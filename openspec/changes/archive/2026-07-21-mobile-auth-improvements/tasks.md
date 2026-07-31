## 1. 後端：登入 Response 加入 accessToken

- [x] 1.1 在 `model.go` 加入 `LoginResponse` struct（`user_id`, `access_token`）
- [x] 1.2 更新 user/salesrep/customer 登入 handler — 成功時產生 JWT 並回傳
- [x] 1.3 新增 `generateAccessToken()` helper（HS256, 24h expiry, user identity in claims）

## 2. Mobile：AuthInterceptor 改用動態 Token

- [x] 2.1 `AuthSessionManager` 新增 `accessToken` field（in-memory）
- [x] 2.2 `signinHandler` / `customerSigninHandler` 從 response 讀取 `access_token` 並存入
- [x] 2.3 `AuthInterceptor.onRequest` 改用 `sessionManager.accessToken`
- [x] 2.4 `clearSession()` 清除 token；`onError` 401 自動清除

## 3. Mobile：啟動時 Session 驗證

- [x] 3.1 在 `initializeMainApp()` 中 `setupLocator()` 後加入 session 驗證
- [x] 3.2 `isLoggedIn` 為 true 時 `GET /restricted/me`
- [x] 3.3 401 → `clearSession()`；網路錯誤 → 保留 session（離線模式）

## 4. 清理

- [x] 4.1 `AuthInterceptor` 不再引用 `FlavorConfig.getApiAccessToken()`
- [x] 4.2 `FlavorConfig.getApiAccessToken()` 保留未刪（向後相容）
