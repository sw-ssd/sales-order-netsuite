## MODIFIED Requirements

### Requirement: 多角色認證（Mobile）
Mobile App 的登入流程應使用 session-bound token。

#### Scenario: 登入成功取得 token
- **WHEN** 使用者登入成功
- **THEN** 後端 response body 新增 `accessToken` 欄位（HS256 JWT，含 user identity, 24h expiry）
- **THEN** Mobile `AuthInterceptor` 從 session 讀取動態 token
- **THEN** 後續請求的 `X-Sowinsoft-Token` header 使用此動態 token
- **THEN** `FlavorConfig.getApiAccessToken()` 不再使用
- **MODIFIED** 從靜態 per-env token 改為 session-bound 動態 token

#### Scenario: 登出清除 token
- **WHEN** 使用者登出
- **THEN** 清除記憶體中的 token
- **WHEN** 401 回應
- **THEN** `AuthInterceptor` 清除 session（含 token），跳轉登入頁

## ADDED Requirements

### Requirement: Mobile 啟動時 session 驗證
App 啟動時應確認 server 端 session 仍有效。

#### Scenario: 啟動驗證
- **WHEN** App 啟動時偵測到 `AutherSessionInfo.isAuth=true`
- **THEN** 在進入主頁面之前呼叫 `GET /restricted/me`
- **THEN** 若 me API 回傳 200，維持現有狀態
- **THEN** 若 me API 回傳 401，清除 session state 並跳轉登入頁
- **THEN** 若 API 網路錯誤（無網路等），保留本機 session 但不阻擋進入（離線模式）
