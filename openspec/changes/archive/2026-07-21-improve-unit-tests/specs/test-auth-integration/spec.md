## ADDED Requirements

### Requirement: Auth 整合測試
測試 CSRF + API key + force logout 的完整流程。

#### Scenario: CSRF flow
- **SHALL** 未登入的使用者 GET `/api/v1/restricted/csrf` 回傳 400
- **SHALL** 已登入的使用者 GET `/api/v1/restricted/csrf` 回傳 CSRF token
- **SHALL** 帶有效 CSRF token 的 POST 請求通過
- **SHALL** 帶無效 CSRF token 的 POST 請求回傳 403
- **SHALL** CSRF token 使用後即失效（one-time）

#### Scenario: API key auth
- **SHALL** 透過 POST `/api/v1/api-keys` 建立金鑰回傳 201 + 完整金鑰
- **SHALL** 使用新建立的金鑰認證通過
- **SHALL** 撤銷後使用該金鑰回傳 401
- **SHALL** 列表 API keys 不回傳金鑰本身

#### Scenario: Force logout
- **SHALL** 管理員可 force logout User 類型的 session
- **SHALL** 管理員可 force logout Salesrep 類型的 session
