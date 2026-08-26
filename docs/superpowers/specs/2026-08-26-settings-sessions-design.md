# Settings 頁面新增 Sessions 設定 — 設計文件

日期：2026-08-26
範圍：`sales-order-backend` + `sales-order-frontend`（App 不在範圍內，見 §8）

## 1. 目標

將 session 行為相關的四個參數從環境變數搬入 settings 表，管理者可在 `/admin/setting` 頁面檢視與修改，**儲存後即時生效**（不需重啟 backend）。

| 原 env | 新 field_id | field_type | 預設值 |
|---|---|---|---|
| `SESSION_DURATION` | `session_duration` | `duration` | `72h` |
| `SESSION_IDLE_TIMEOUT` | `session_idle_timeout` | `duration` | `168h` |
| `SESSION_SLIDING` | `session_sliding` | `bool` | `true` |
| `SESSION_WARN_BEFORE` | `session_warn_before` | `duration` | `5m` |

Cookie 結構參數（`SESSION_NAME`/`PATH`/`DOMAIN`/`HTTP_ONLY`/`SECURE`/`SAME_SITE`）維持 env，不進 settings——改錯會鎖死登入，且需重啟才有意義。

決策記錄：

- 時間長度存 **Go duration 字串**（`72h`、`5m`），`field_type` 為新增的 `duration`；後端以 `time.ParseDuration` 驗證。
- 修改後**即時生效**（使用者明確選擇；重啟生效方案已排除）。

## 2. 現況

- Backend session 組態在 `config/cookie.go` 由 envconfig 讀取，`internal/server/server.go` `newAuthentication()` 啟動時建構 `scs.SessionManager`：`Lifetime = cfg.Session.Duration`；`Sliding=true` 時 `IdleTimeout = cfg.Session.IdleTimeout`（false 則為 0，固定壽命）。
- settings 表已有 31 欄（`internal/domain/settings/fields.go` `FieldRegistry`），API 為 `GET/PUT /api/v1/settings`，`Update` 會以 registry 的 Name/Desc 覆寫（desc 為 registry 控制，前端 PUT 不帶權威 desc）。
- `config.Session.WarnBefore` 目前**無任何 backend 消費者**；前端 `SessionExpiryDialog.tsx` hardcode `WARN_BEFORE_MS = 5 * 60 * 1000`。
- 前端設定頁 `Setting.tsx` 以靜態 `FIELD_GROUPS` 分組渲染，未知 field_id 不渲染；desc 已顯示於欄位名稱下方。
- App（Flutter）的到期提醒用 `FlavorConfig().getSessionWarnBefore()`，不吃 backend 值。

## 3. Backend 設計

### 3.1 FieldRegistry 與驗證

`fields.go` 新增 4 欄（registry 變 35 欄）：

```go
{"session_duration", "Session 絕對壽命", "duration", "到期後需重新登入", "72h"},
{"session_idle_timeout", "Session 閒置逾時", "duration", "滑動續期啟用時生效", "168h"},
{"session_sliding", "滑動續期", "bool", "活動中的 session 不被固定期限登出", "true"},
{"session_warn_before", "到期提醒提前時間", "duration", "Web 到期提醒；App 用自身 flavor 值", "5m"},
```

`validation.go`：

- `switch fieldType` 新增 `case "duration"`：`time.ParseDuration(v)` 須成功且結果 `> 0`，否則 `ErrValidation`。
- `session_duration`、`session_idle_timeout`、`session_warn_before` 加入 `requiredFields`（必填非空）。
- `session_sliding` 走既有 `bool` 規則（`true`/`false`）。

### 3.2 Seed 改為補缺（backfill）

現行 `Seed` 在 `count > 0` 時直接 no-op，既有部署（31 列）永遠拿不到新欄位。改為：

1. 查出現有 `field_id` 集合。
2. 只為 registry 中**缺漏**的欄位建立 insert builder（預設值；env 覆寫僅保留 netsuite/email/frontend_url 既有邏輯）。
3. 全部缺漏欄位於單一 `CreateBulk` + `OnConflictColumns(field_id).Ignore()` 插入（保留防並發語意）。
4. 無缺漏 → no-op。既有列的值**不被覆寫**。

### 3.3 啟動流程

`server.go` 調整順序：settings seed/backfill 完成後、`newAuthentication()` 之前 `LoadAll` settings，解析四個 session 欄位：

- `session_duration` / `session_idle_timeout`：`time.ParseDuration`。
- `session_sliding`：`== "true"`。

解析失敗 → `log.Fatalf`（fail-fast，與 `server.go` 現行風格一致；PUT 已驗證，正常不會發生）。`newAuthentication()` 改用解析後的值設定 `manager.Lifetime` / `manager.IdleTimeout`（sliding=false → IdleTimeout=0）。Cookie 結構參數仍讀 `cfg.Session`。

### 3.4 即時生效

`NewUseCase` 增加可選 hook（如 `NewUseCase(repo, WithSessionApplier(fn))` 或直接第二參數，實作時依程式碼風格定）：

- `Update` 在 `BatchUpsert` 成功後，若本次 payload 含任一 `session_*` 欄位，以**更新後的完整值**（重讀 registry+DB 或直接由 Get 結果解析）呼叫 `ApplySessionConfig`。
- `ApplySessionConfig(manager, duration, idleTimeout, sliding)`：
  - `manager.Lifetime = duration`
  - `sliding` → `manager.IdleTimeout = idleTimeout`；否則 `manager.IdleTimeout = 0`
- `server.go` 以閉包注入 `s.session`。**注入點在 manager 建立之後**（`newAuthentication` 先於 settings usecase 建構，或調整建構順序）。
- `/me` 的 `session_expires_at` 計算（`authentication/repository.go` 讀 `r.session.Lifetime`）讀同一欄位，自動一致。

**已知風險（接受）**：scs 對 `Lifetime`/`IdleTimeout` 的讀取無鎖，PUT 觸發的寫入在 Go 記憶體模型下屬 data race。64-bit 平台對齊 int64（`time.Duration`）讀寫實務上原子，且寫入者僅管理員單次操作；scs 內部 `Commit` 讀自身欄位，外部 atomic holder 繞不開，故採直接寫入並於此記錄。

### 3.5 Env cutover

- `config/cookie.go` `Session` 結構移除 `Duration`、`IdleTimeout`、`Sliding`、`WarnBefore` 四個欄位。
- `hexagon.env` 移除 `SESSION_DURATION`/`SESSION_IDLE_TIMEOUT`/`SESSION_SLIDING`/`SESSION_WARN_BEFORE`。
- README session 章節改寫：四個行為參數改於設定頁管理、即時生效；cookie 參數仍為 env。`SESSION_WARN_BEFORE=300`（App 秒數）的說明標註為 App flavor 設定。

## 4. Frontend 設計

- `src/models/settings.ts`：`SettingFieldType` 加 `"duration"`。
- `src/pages/admin/setting/Setting.tsx` `FIELD_GROUPS` 新增群組：

  ```ts
  { title: "Session 設定", fieldIds: ["session_duration", "session_idle_timeout", "session_sliding", "session_warn_before"] },
  ```

  duration 欄位落入既有非 secret 分支，渲染為 text input；`save()` 的 bool 分支已處理 `session_sliding`（true/false + toast 驗證）；duration 值由後端 422 驗證錯誤 toast 把關（與 URL/email 欄位一致的模式，不加前端專屬驗證）。
- `src/components/session/SessionExpiryDialog.tsx`：刪除 hardcode `WARN_BEFORE_MS`，改以 `useSettingsValue("session_warn_before")` 取得字串，經新增的 `parseGoDuration` 小工具（支援 `s`/`m`/`h` 單位與複合格式如 `1h30m`，回傳毫秒；解析失敗回傳 `null` 時 fallback 5 分鐘）轉換。
- `src/constant/options.ts` `FALLBACK_SETTINGS` 補 4 欄（name/desc/default 與 registry 一致）。

## 5. 測試

Backend：

- `fields_test.go`：更新欄位數斷言（35）；`duration` 為合法 field_type。
- `validation_test.go`：新增 duration 驗證案例——`72h`、`1h30m` 合法；`abc`、`0s`、`-5m`、空字串非法。
- Seed backfill 測試：先建 31 列舊欄位 → 執行 Seed → 4 個 session 欄位被補上、既有列值不變。
- Settings integration：PUT `session_duration` 後，注入的假/真 `scs.SessionManager` 之 `Lifetime` 已更新；PUT `session_sliding=false` 後 `IdleTimeout` 歸 0。

Frontend：

- `parseGoDuration` 單測：`"5m"`→300000、`"72h"`、`"1h30m"`、非法字串→fallback 行為。

## 6. 文件

- `README.md`（backend）session 章節改寫。
- `docs/AGENTS/backend.md`、`docs/AGENTS/frontend.md` 的設定頁/session 段落同步更新。

## 7. 錯誤處理

| 情況 | 行為 |
|---|---|
| PUT 非法 duration / 空值 | 422，`ErrValidation`，前端 toast 顯示後端訊息 |
| 啟動時 DB 中 session 欄位解析失敗 | `log.Fatalf`（欄位名 + 原因） |
| 前端 `session_warn_before` 解析失敗 / query 未就緒 | fallback 5 分鐘 |
| 非 superadmin 修改 | 4 欄皆非 secret，依既有 settings 權限（admin 可改）；不加額外限制 |

## 8. 範圍外

- Flutter App：到期提醒用自身 `FlavorConfig().getSessionWarnBefore()`（秒），不受 `session_warn_before` 影響；本次不改 App。
- Cookie 結構參數（Name/Path/Domain/HttpOnly/Secure/SameSite）：維持 env。
- 即時生效的 data race：接受（§3.4），不自建同步層。
