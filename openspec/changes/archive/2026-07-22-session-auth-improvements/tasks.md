## 1. CSRF 保護

- [x] 1.1 新增 `CSRFMiddleware` — 攔截 POST/PUT/PATCH/DELETE，驗證 `X-CSRF-Token` header 是否匹配 session store 中的 token
- [x] 1.2 將 CSRFMiddleware 加入 global middleware chain（在 Authenticate 之後）
- [x] 1.3 Frontend 串接 CSRF token：`apiReq` 在寫操作時從 memory 讀取 CSRF token 帶入 header
- [x] 1.4 Frontend 頁面載入時呼叫 `GET /restricted/csrf` 取得 token 並暫存（memory，非 localStorage）
- [x] 1.5 登出/401 時清除 CSRF token

## 2. M2M API Key 系統（後端）

- [x] 2.1 建立 `api_keys` ent schema（id, name, key_hash, key_prefix, scopes, expires_at, is_active, created_by, created_at, last_used_at）
- [x] 2.2 執行 `go generate ./ent` 產生 ent 程式碼
- [x] 2.3 建立 `internal/domain/apikeys/` 目錄，含 handler + usecase + repository + register + model
- [x] 2.4 實作 API key 產生：`sk_<prefix>_<random64>`，回傳完整金鑰（一次性），hash 存 DB
- [x] 2.5 實作 CRUD endpoint：建立、列表（不回傳 key）、撤銷（is_active=false）
- [x] 2.6 實作 `ApiKeyMiddleware` — 讀取 `Authorization: Bearer`，sha256 比對 hash，檢查 active+expiry，更新 last_used
- [x] 2.7 註冊 API key endpoint 路由

## 3. M2M API Key 遷移（前端/mobile）

- [x] 3.1 從 frontend `apiReq` 移除 `X-Sowinsoft-Token` header 寫入（由 CSRF token 取代寫操作保護）
- [x] 3.2 從 mobile `getApiAccessToken()` 移除靜態 token，改用登入後取得的動態 token（或 API key）

## 4. 401 統一處理

- [x] 4.1 調整 `internal/middleware/authentication.go` 的 401 錯誤訊息為清楚中文說明
- [x] 4.2 調整新版 ApiKeyMiddleware 的 401 錯誤訊息為一致中文說明
- [x] 4.3 Frontend `fetchRequest` 加入全域 401 攔截：clear auth + 導向 `/signin`
- [x] 4.4 Mobile app startup 加入 session 驗證流程（startup me check）

## 5. Session TTL 7 天 + Sliding

- [x] 5.1 確認 `config/cookie.go` 的 `Duration` 預設為 `24h`（由環境變數覆蓋）
- [x] 5.2 修改 `hexagon.env` 的 `SESSION_DURATION` 為 `168h`
- [x] 5.3 在 `LoadAndSave` middleware 加入 sliding extension 邏輯（剩餘 < 24h → 延長至 7 天）

## 6. Force Logout 補齊

- [x] 6.1 在 `ent/schema/session.go` 新增 `user_type` 欄位，並讓 `SessionInfoWithContext` / `CommitCtx` 解析與儲存 `user_type`
- [x] 6.2 調整 `repository.go` 的 `SignoutUser` 為 `WHERE info_id = ? AND user_type = ?`
- [x] 6.3 調整 `handler_restricted.go` 的 `ForceLogout` — 從 URL 讀取 `utype` 並傳入 use case
- [x] 6.4 確認 frontend / app 的 force logout 呼叫已帶正確 `utype`
- [x] 6.5 新增 Goose migration：在 `sessions` 表加入 `user_type` 欄位

## 7. 清理 Dead Code

- [x] 7.1 移除 `handler_oauth2.go` 中已註解的 `OAuth2Login` handler 與其他 stub
- [x] 7.2 移除 usecase.go 中 magic link / forgot password stub
- [x] 7.3 從 authentication register.go 移除 OTP routes 與相關註解

## 8. 驗證與部署準備

- [x] 8.1 新版 ApiKeyMiddleware 已在 server.go 取代舊版 ApiTokenMiddleware
- [x] 8.2 `cmd/token/main.go` 標註 deprecated，指向新的 API key 管理端點
- [x] 8.3 整合測試：CSRF flow、API key auth、force logout（各 utype）、sliding session
