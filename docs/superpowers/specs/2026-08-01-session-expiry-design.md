# Session 體驗優化設計（登入能維持多久）

- 日期：2026-08-01
- 狀態：草案（待審核）
- 範圍：sales-order-backend（Go）、sales-order-frontend（SolidJS）、sales-order-app（Flutter）

## 背景與現況

登入狀態以純 cookie session 實現：後端 Go（chi + `alexedwards/scs/v2` v2.8.0）發放 HttpOnly/Secure cookie（`__session`），session 資料存 Postgres（`PostgresStoreEnt`）。壽命由 `SESSION_DURATION` 控制（開發 `.env` 為 `12h`，正式 `hexagon.env` 為 `168h`，程式預設 `24h`），`Cookie.Persist=true`（關閉瀏覽器/App 重開仍有效直到絕對到期）。

已確認的機制缺口：

1. **固定到期、無滑動**：`LoadAndSave`（`middleware/authentication.go`）已有半套滑動（剩餘 < 24h 時 `SetDeadline(now + Lifetime)`），但門檻寫死 24h、無設定開關、無閒置逾時語意。scs `IdleTimeout` 未設定。
2. **CSRF 期限不同步**：`repository.go` 的 `CommitCtx(..., time.Now().Add(session.Lifetime))` 在滑動續期後不會同步更新，可能比 session 早失效。
3. **無到期前提醒**：Web 收到 401 → 清 auth state → `/signin?redirect=…`；App 收到 401 → 清 cookie + session info → 登入頁。使用者操作到一半被強制登出，且無預警。
4. **App 401 後不還原頁面**：Web 已有 `redirect` 參數還原；App 沒有。

## 目標 / 非目標

**目標**
- 活動中的使用者不被固定期限登出（滑動續期）。
- 閒置超過設定時間才失效。
- 到期前主動提醒（Web + App），可一鍵續期。
- 401 重新登入後還原原本頁面（Web 維持、App 補上）。
- 設定值文件化，開發與正式環境一致。

**非目標**
- 不做「記住我」勾選（使用者明確表示不要）。
- 不修 OAuth2 登入缺陷（`handler_oauth2.go` TODO，另開 change）。
- 不導入 refresh token / Bearer token 架構（維持純 cookie session）。
- 不做多裝置 session 管理 UI。

## 核心行為

| 情境 | 行為 |
|---|---|
| 持續使用（請求間隔 < 30 分鐘） | 永不登出——每次請求滑動續期，絕對 deadline 持續向後延 |
| 閒置 ≥ 30 分鐘 | session 失效，下次請求 401 |
| 到期前 5 分鐘（前端在線） | 彈窗「即將登出，剩餘 X 分鐘」+「繼續操作」 |
| 點「繼續操作」 | 呼叫輕量受保護請求（`/csrf`）觸發續期，關閉彈窗 |
| 已到期（401） | Web：清 auth state → `/signin?redirect=…`；App：清 cookie+session → 登入頁 → 登入後還原原頁 |

**「登入能維持多久」的答案**：只要持續使用就不會登出；停止使用超過 30 分鐘才需要重新登入。絕對上限仍為 12h（每次續期把 deadline 推回 `now + 12h`，故實際是「12h 滑動視窗」）。

## 架構與元件

### 後端（sales-order-backend）

**設定（`config/cookie.go`）**
- 既有：`SESSION_DURATION`（預設 24h，開發 12h、正式 168h → 統一為 12h）
- 新增：
  - `SESSION_IDLE_TIMEOUT`（預設 30m）：閒置逾時
  - `SESSION_SLIDING`（bool，預設 false → 開發/正式設 true）：啟用滑動
  - `SESSION_WARN_BEFORE`（預設 5m）：前端提醒提前量（後端僅提供到期時間，提醒由前端算）

**Session 初始化（`internal/server/server.go`）**
- `Sliding=true` 時：`manager.IdleTimeout = cfg.Session.IdleTimeout`
- `Lifetime = cfg.Session.Duration`（不變）

**滑動續期（`internal/middleware/authentication.go`）**
- 現有寫死 `24*time.Hour` 門檻改為設定驅動：當 `s.IdleTimeout > 0` 且剩餘壽命 < `s.IdleTimeout` 時 `s.SetDeadline(infoCtx, time.Now().Add(s.Lifetime))`
- scs 原生：`IdleTimeout > 0` 時每次 `Load` 標記 Modified → 每次請求重新 commit，store expiry = `min(deadline, now + idle)`

**CSRF 期限同步（`internal/domain/authentication/repository.go`）**
- `CommitCtx` 期限改為 `session.Deadline(ctx)`（若為零值則 fallback `now + Lifetime`），與滑動後 session 到期同步

**`/me` 回應（`internal/domain/authentication/handler_restricted.go` + `third_party/postgres_entstore/models.go`）**
- `SessionInfoResource` 新增 `session_expires_at`（RFC3339，`omitempty`）；只在 `Me` handler 中填寫（`h.session.Deadline(ctx)`），session blob 中留空，不污染儲存內容。回應 shape 維持 flat（`user`/`salesrep`/`customer` 在頂層），前端 `SessionInfoMe` 只需加一個可選欄位。

### 前端 Web（sales-order-frontend）

- `src/models/base.ts`：`SessionInfoMe` 加 `session_expires_at?: string`
- `src/lib/auth/global.ts` / `src/lib/requests/utils.ts`：401 handler 記錄到期時間；到期前 `SESSION_WARN_BEFORE` 觸發提醒（以伺服器時間為準，不用裝置本地絕對時刻）
- 到期提醒元件：倒數 modal，「繼續操作」呼叫 `/csrf` 觸發續期、更新到期時間、關閉彈窗
- 401 還原：維持現狀（`/signin?redirect=…` 已實作）

### App（sales-order-app / Flutter）

- `layer_data/models/auther/auther_session_info.dart`：`AutherSessionInfo` 加 `session_expires_at`
- `layer_business/network/auth_interceptor.dart`：記錄到期時間、401 清除 session 時記住來源頁面
- 到期提醒 dialog + 續期動作（呼叫輕量受保護請求）
- 登入後還原來源頁面
- 環境變數：`prod.env.hexagon` / `dev` env 加 `SESSION_WARN_BEFORE`

## 資料流

```
登入成功 → RenewToken → Put(KeyAccountID) → 回應含 session_expires_at
使用者操作 → 每次請求 LoadAndSave 檢查剩餘壽命
  ├─ 剩餘 < IdleTimeout → SetDeadline(now + Lifetime) → commit 新 expiry
  └─ 閒置 > IdleTimeout → store 過期 → 下次請求 401
前端/App 倒數 → 剩 SESSION_WARN_BEFORE → 彈窗
  ├─ 「繼續操作」→ GET /csrf → 續期 → 新 session_expires_at → 關窗
  └─ 不做動作 → 到期 → 401 → 清 session → 登入頁(帶 redirect) → 登入後還原
```

## 錯誤處理

- **401 處理**：Web 維持 `errorDefaultChecker` 清 `auth_state` + 導向 `/signin?redirect=`；App `AuthInterceptor.onError` 清 session 並導向登入頁（記住來源）。
- **續期失敗**：`/csrf` 呼叫失敗 → 彈窗維持並顯示「續期失敗，即將登出」；到期後自然落入 401 流程。
- **伺服器時間差異**：提醒一律以 `/me` 回應的 `session_expires_at`（伺服器時間）為準。

## 測試

- **後端單元/整合測試**（`internal/domain/authentication/`、`internal/middleware/`）：
  - 滑動續期：活動中 session 到期時間延長；`SESSION_SLIDING=false` 時維持固定壽命
  - CSRF 期限與 session 到期同步
  - `/me` 回應含 `session_expires_at`
- **前端/App**：提醒時機、續期動作、401 還原以手動驗證為主（既無測試框架涵蓋此路徑）

## 風險 / 取捨

- [滑動續期延長 session 被竊取後的可用時間] → 依賴既有 Secure/HttpOnly/SameSite；閒置逾時 30 分鐘縮小暴露窗。
- [12h 滑動視窗 vs 固定 12h 的既有期待] → 文件化；`SESSION_SLIDING` 可關閉回復舊行為。
- [前端/App 各自實作提醒，時鐘不同步] → 以伺服器 `session_expires_at` 為唯一依據。
- [正式環境壽命從 168h 縮為 12h 滑動] → 對「每天使用」的使用者影響為零（持續使用永不登出）；對「一週開一次」的使用者需重登一次。屬預期行為，文件化。

## 遷移計畫

1. 後端：設定欄位 → `server.go` → `LoadAndSave` → CSRF 期限 → `/me` 回應
2. 後端測試
3. Web：型別 → 提醒元件 → 401 整合
4. App：型別 → 提醒 → 401 還原
5. 部署：`.env` / `hexagon.env` / `prod.env.hexagon` 更新；`SESSION_SLIDING=true`
6. **Rollback**：`SESSION_SLIDING=false` 即回復固定壽命；其餘為獨立 commit 可個別 revert

## 待決問題

- 正式環境 `SESSION_DURATION` 由 168h 改為 12h 是否可接受（滑動續期下對日常使用者無感）？
- `SESSION_WARN_BEFORE=5m` 是否足夠？
- App 401 還原的「來源頁面」範圍（最近一次路由 vs 登出前所在 tab）？
