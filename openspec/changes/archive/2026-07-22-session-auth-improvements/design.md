## Context

目前認證系統的狀態與待解決問題：

```
現況問題矩陣
┌──────────────────────────┬──────────┬─────────────────────────┐
│ 問題                     │ 嚴重程度  │ 解法方向                │
├──────────────────────────┼──────────┼─────────────────────────┤
│ 無 CSRF 保護             │ 🔴 High  │ 啟用 CSRF endpoint      │
│ API token 永不過期+共用  │ 🔴 High  │ 建立 M2M API Key 系統   │
│ 三端 401 行為不一致       │ 🟡 Mid   │ 統一格式 + 全域攔截器    │
│ Session TTL 24h 偏短     │ 🟡 Mid   │ 7 天 + sliding          │
│ Force logout 不完整      │ 🟡 Mid   │ 補齊 salesrep/customer  │
│ Dead code（OAuth/OTP）   │ 🟢 Low   │ 清理                    │
└──────────────────────────┴──────────┴─────────────────────────┘
```

## Goals / Non-Goals

**Goals:**
- CSRF 保護（寫操作驗證 token）
- M2M API Key 系統（DB 儲存、scope-based、可撤銷）
- 三端 401 統一處理
- Session TTL 7 天 + sliding extension
- Force logout 完整支援所有 user type
- 清理 dead code

**Non-Goals:**
- 不變更 SameSite 設定（留 `none`）
- 不引入 refresh token pattern
- 不重構完整 auth 架構
- 不改變 SCS 函式庫（維持 scs/v2 + PostgresStoreEnt）
- 不處理密碼強度政策

## Decisions

### 1. CSRF 保護：啟用既有 endpoint + middleware

**選擇**：使用後端既有的 `GET /api/v1/restricted/csrf` endpoint，前端在寫操作帶入 `X-CSRF-Token` header，後端新增 middleware 驗證。

**流程**：
```
Page Load → GET /restricted/csrf → store token in session → 存入 memory
  ↓
POST/PUT/PATCH/DELETE → 帶 X-CSRF-Token header → middleware 驗證
  ↓
Token valid  → next()
Token invalid → 403
```

**理由**：
- 既有 endpoint 已實作（`handler_restricted.go` 的 `Csrf` handler），只需前端串接 + middleware 驗證
- 不需要引入新依賴
- token 綁定 session store，登出後自動失效

**不採用**：Double Submit Cookie（不支援 `SameSite=lax` 下的安全需求）、自訂 header 檢查（不夠安全，攻擊者可偽造）

### 2. M2M API Key 系統

**選擇**：新的 `api_keys` ent schema，Bearer token 認證，scope-based 權限。

```
Ent Schema: api_keys
┌──────────────────────────┐
│ id          UUID         │
│ name        string       │ ← 如「排程同步用」
│ key_hash    string       │ ← sha256(api_key)
│ key_prefix  string       │ ← key 前 8 個 hex 字元（不含 `sk_`）
│ scopes      []string     │ ← ["sales_orders:read", "items:write"]
│ expires_at  timestamptz  │ ← nullable
│ is_active   bool         │
│ created_by  UUID→user    │
│ created_at  timestamptz  │
│ last_used_at timestamptz │ ← nullable
└──────────────────────────┘
```

```
Middleware: ApiKeyMiddleware
位置：global middleware chain，在 LoadAndSave 之後
路徑：只對 /api/v1/* 生效

流程：
1. 讀 Authorization: Bearer <key>（或沿用 `X-Sowinsoft-Token` header）
2. 金鑰格式為 `sk_<8 hex>_<56 hex>`，取前 8 個 hex 字元作為 `key_prefix` 查找候選
3. sha256(key) === stored.hash
4. is_active && (expires_at === null || expires_at > now())
5. 檢查 handler-defined scope 是否在 key.scopes 中
6. 更新 last_used_at
7. context 寫入 key info（name, scopes）
```

**不採用**：
- JWT 作為 API key（不可撤銷 — 需要 stateful storage）
- 環境變數共用 token（無 scope、無稽核）

### 3. 三端 401 統一處理

**後端**：統一使用既有 `respond.Error` 回傳 401，將錯誤訊息調整為清楚的中文說明（例如「未授權，請重新登入」）。不引入新的 response body schema，維持現有 `application/problem+json` 格式與錯誤碼結構。

**Frontend（SolidJS）**：在 `fetchRequest` 層加入全域 401 攔截：
```
fetchRequest → resp.ok?
  ├─ Yes → return data
  └─ No → resp.status === 401?
       ├─ Yes → clear auth_state + CSRF token → router.navigate('/signin')
       ├─ No  → throw HttpError
```
401 時不 retry。需注意 router 在全域 scope 的可用性。

**Mobile（Flutter）**：現有 AuthInterceptor 保留，補強啟動時驗證：
```
App Start → AutherSessionInfo.isAuth?
  ├─ Yes → GET /restricted/me
  │        ├─ 200 → OK
  │        └─ 401 → clearSession() → navigate SelectSignin
  └─ No  → navigate SelectSignin
```

### 4. Session TTL：7 天 + Sliding

**選擇**：修改 `config/cookie.go` 的 `Duration` 從 `24h` 改為 `168h`（7 天）。實際值由 `SESSION_DURATION` env 變數覆蓋。Sliding 實作在 `LoadAndSave` middleware。

```
LoadAndSave Middleware（擴充）
1. SCS 原本的 Load session + save cookie
2. 額外檢查：session.Expiry - time.Now() < 24h
3. 若低於 threshold：session.SetExpiry(7d)
4. SCS 自動處理 DB upsert（CommitCtx）
5. 獨立 cleanup goroutine 每 30 分鐘刪除 expiry < now() 的 session rows
```

**24h threshold 理由**：7 天內最多延長 7 次，每次延長寫一次 DB。若每請求都檢查，低於 threshold 才寫入，效能開銷極小。

### 5. Force Logout 補齊

**選擇**：repository 層 `SignoutUser` 改為依 `info_id` + `user_type` 刪除。`user_type` 在登入時寫入 session row，避免不同角色 ID 碰撞導致誤登出。

**影響範圍**：
- `repository.go` — `SignoutUser` 改為 `WHERE info_id = ? AND user_type = ?`
- `handler_restricted.go` — `ForceLogout` 從 URL 讀取 `utype` 並傳入 use case
- `ent/schema/session.go` — 新增 `user_type` 欄位
- `SessionInfoWithContext` / `PostgresStoreEnt.CommitCtx` — 從 session 資料解析並儲存 `user_type`

## Risks / Trade-offs

| 風險 | 影響 | 緩解 |
|------|------|------|
| CSRF token 新增 DB write | 每個寫操作多一次 session store 查詢 | token 存於 session data 中，與 session 查詢合併 |
| API key hash 碰撞 | 理論上 sha256 碰撞率極低 | 可忽略 |
| 7 天 session 被盜風險 | cookie 暴露視窗從 12h 變 7 天 | CSRF 保護補償（攻擊者無法跨站寫入）+ 管理者可 force logout |
| Sliding 寫入頻率 | 7 天內最多 ~7 次額外 upsert | threshold 24h 確保低頻 |
| 既有 X-Sowinsoft-Token 使用者 | 部署後現有 token 失效 | 公告+遷移期：新舊 middleware 並行直到所有 client 更新 |
