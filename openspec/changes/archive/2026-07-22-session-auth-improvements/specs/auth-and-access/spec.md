## MODIFIED Requirements

### Requirement: 多角色認證
系統支援三種登入角色類型：系統使用者（User）、業務員（Salesrep）、客戶（Customer）。

#### Scenario: 登入流程
- **WHEN** 使用者 POST `/api/v1/authentication/login/{utype}`（utype = user/salesrep/customer）傳入帳號密碼
- **THEN** 系統驗證 Credential（argon2id hash），建立 session
- **THEN** 設定 session cookie（HttpOnly=true, Secure=true, SameSite=none）
- **THEN** 設定 `X-Sowinsoft-Token` header（HS256 JWT，含 24h `exp` claim），供 Mobile 作為動態 access token 使用
- **THEN** 回傳對應角色資訊（含權限）
- **MODIFIED** API JWT 加入 `exp` claim（24 小時到期）

#### Scenario: CSRF 保護（寫操作）
- **WHEN** 前端頁面載入，GET `/api/v1/restricted/csrf`
- **THEN** 後端產生 CSRF token，存入 session store，回傳 token
- **WHEN** 前端發起 POST/PUT/PATCH/DELETE 請求
- **THEN** 前端在 header 帶入 `X-CSRF-Token: <token>`
- **THEN** 後端 middleware 驗證 token 是否匹配 session store
- **THEN** token 不匹配或不合法回傳 403

#### Scenario: 登出
- **WHEN** 使用者 POST `/api/v1/authentication/logout/{utype}`
- **THEN** 系統清除 session（DB 刪除 session row）
- **THEN** 清除 auth cookies（Set-Cookie 含過期時間）
- **THEN** 前端同時清除 localStorage auth_state 與 CSRF token

#### Scenario: 強制登出（Force Logout）
- **WHEN** 管理員 DELETE `/api/v1/restricted/logout/{id}/{utype}`
- **THEN** 系統刪除該 user 的所有 active session rows（by `info_id` + `user_type`）
- **THEN** 支援 utype = user / salesrep / customer
- **MODIFIED** 補齊 salesrep 與 customer 的 force logout 邏輯

### Requirement: 401 回應統一格式
所有 401 回應使用一致結構，供前端全域攔截器解析。

#### Scenario: 401 回應格式
- **WHEN** 未授權請求抵達
- **THEN** 後端回傳 HTTP 401
- **THEN** response body 使用既有 `respond.Error` 產生的 `application/problem+json` 格式
- **THEN** 錯誤訊息內容為清楚的中文說明，例如「未授權，請重新登入」
- **THEN** 前端 / Mobile 的全域攔截器依 HTTP 401 status 觸發重新導向，不解析特定 body 欄位

## ADDED Requirements

### Requirement: 全域 401 處理（Frontend）
前端要有統一的 401 未授權處理機制，而非僅依賴個別 route loader。

#### Scenario: 401 自動跳轉登入頁
- **WHEN** 任何 API 回傳 401
- **THEN** 全域 response interceptor 清除 auth state + CSRF token
- **THEN** 自動導向 `/signin?redirect=<current_path>`
- **THEN** 不重試失敗請求（避免 auth loop）

### Requirement: 401 處理（Mobile）
行動端現有 AuthInterceptor 機制應補強啟動時驗證。

#### Scenario: 啟動時 session 驗證
- **WHEN** App 啟動時偵測到 `AutherSessionInfo.isAuth=true`
- **THEN** 呼叫 `/restricted/me` 確認 server 端 session 有效
- **THEN** 若 me API 回傳 401，清除 session state 並跳轉登入頁
- **THEN** 若 me API 成功，維持現有狀態

### Requirement: Session TTL 調整為 7 天
Session 有效時間從 12h 延長至 7 天，支援 sliding extension。

#### Scenario: Session 建立
- **WHEN** 使用者登入成功
- **THEN** session TTL 設定為 7 天（604800 秒）
- **THEN** Set-Cookie: `Max-Age=604800`

#### Scenario: Sliding extension
- **WHEN** 已登入使用者發起 API 請求
- **THEN** 後端檢查 session 剩餘時間
- **THEN** 若剩餘時間低於 threshold（如 24h），自動延長至完整 7 天
- **THEN** 更新 DB session row 的 expiry 與 cookie

#### Scenario: 長時間閒置
- **WHEN** 使用者超過 7 天未有任何請求
- **THEN** DB session row 因 expiry 過期被 cleanup goroutine 刪除
- **THEN** 下一次請求回傳 401

## REMOVED Requirements

### Requirement: OAuth2 登入啟動（Google）
**Reason**: OAuth2 login initiation handler 已註解，callback handler 仍殘留。此 flow 未完成且無使用計畫。callback handler 保留以維持向後相容，initiation 端移除殘留程式碼。

**Migration**: 無 — 此功能從未正式啟用。

### Requirement: Magic Link / Forgot Password 未完成 stub
**Reason**: 這些 flow 只有 handler 簽章但未實作，造成維護混淆。

**Migration**: 完全移除相關 stub 程式碼。
