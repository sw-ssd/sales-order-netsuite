## ADDED Requirements

### Requirement: CSRFMiddleware 單元測試
測試 `internal/middleware/csrf.go` 的 CSRFMiddleware 邏輯。

#### Scenario: Missing header
- **SHALL** 未帶 `X-CSRF-Token` header 的 POST/PUT/PATCH/DELETE 請求回傳 403
- **SHALL** GET/HEAD/OPTIONS 請求不檢查 CSRF token（pass through）

#### Scenario: Invalid token
- **SHALL** 帶入不存在的 CSRF token 回傳 403
- **SHALL** 帶入已過期的 CSRF token 回傳 403

#### Scenario: Valid token
- **SHALL** 帶入有效的 CSRF token 的 POST 請求通過驗證
- **SHALL** token 使用後被刪除（one-time use）
