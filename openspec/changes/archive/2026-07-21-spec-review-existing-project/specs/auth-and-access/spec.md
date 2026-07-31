## ADDED Requirements

### Requirement: 多角色認證
系統支援三種登入角色類型：系統使用者（User）、業務員（Salesrep）、客戶（Customer）。

#### Scenario: 登入流程
- **WHEN** 使用者 POST `/api/v1/authentication/login/{utype}`（utype = user/salesrep/customer）傳入帳號密碼
- **THEN** 系統驗證 Credential（argon2id hash），建立 session
- **THEN** 設定 session cookie + X-Sowinsoft-Token header（HS256 JWT）
- **THEN** 回傳對應角色資訊（含權限）

#### Scenario: 帳號驗證
- **WHEN** 使用者 POST `/api/v1/authentication/valid-account/{utype}`
- **THEN** 系統檢查帳號是否存在與可用

#### Scenario: 密碼設定與重設
- **WHEN** 使用者 POST `/api/v1/authentication/newset-password/{utype}`
- **THEN** 系統更新密碼（argon2id hash）
- **WHEN** 管理員 POST `/api/v1/authentication/reset-null-password/{id}/{utype}`
- **THEN** 系統重設密碼為空（強制重設流程）

#### Scenario: OAuth2 登入（Google）
- **WHEN** 使用者透過 Google 登入
- **THEN** 系統接收 OAuth callback，建立/關聯 Provider 記錄
- **THEN** 建立 session，回傳登入成功

#### Scenario: 登出
- **WHEN** 使用者 POST `/api/v1/authentication/logout/{utype}`
- **THEN** 系統清除 session，清除 auth cookies

#### Scenario: 取得當前使用者
- **WHEN** 使用者 GET `/api/v1/restricted/me`
- **THEN** 系統回傳當前登入者資訊（SessionInfoMe）

### Requirement: 角色權限管理（RBAC）
系統使用 Casbin 實現基於角色的存取控制（RBAC）。

#### Scenario: 權限定義
- **WHEN** 管理員管理策略
- **THEN** POST/PATCH/DELETE `/api/v1/policies` 與 `/api/v1/roles/policies`
- **THEN** Casbin 引擎即時生效權限變更

#### Scenario: API 權限檢查
- **WHEN** 使用者呼叫受保護 API
- **THEN** middleware.Authorize 檢查 Casbin 策略
- **THEN** 無權限回傳 403

### Requirement: 多租戶隔離
系統支援多 Tenant 隔離。

#### Scenario: Tenant 管理
- **WHEN** 管理員 POST/PATCH/DELETE `/api/v1/tenants`
- **THEN** 系統建立/修改/刪除 Tenant 記錄
- **THEN** User、Role 等實體綁定租戶範圍，查詢自動過濾

### Requirement: OTP 驗證
系統支援一次性密碼用於特定驗證流程。

#### Scenario: OTP 驗證
- **WHEN** 使用者觸發 OTP 流程
- **THEN** 系統建立 OTP 記錄（含 code、expires_at），綁定 User
- **THEN** 驗證通過後標記 used = true
