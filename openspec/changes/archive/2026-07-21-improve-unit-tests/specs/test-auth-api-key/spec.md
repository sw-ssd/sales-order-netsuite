## ADDED REQUIREMENTS

### Requirement: ApiKeyMiddleware 單元測試
測試 `internal/middleware/apiKeyStore.go` 的 ApiKeyMiddleware 邏輯。

#### Scenario: Missing token
- **SHALL** 未帶 `X-Sowinsoft-Token` header 的請求 pass through（不阻擋）
- **SHALL** 非 `/api/` 路徑的請求跳過檢查

#### Scenario: Invalid token
- **SHALL** 帶入不存在的 API key 回傳 401
- **SHALL** 帶入已撤銷（is_active=false）的 API key 回傳 401
- **SHALL** 帶入已過期的 API key 回傳 401

#### Scenario: Valid token
- **SHALL** 帶入有效的 API key 通過驗證
- **SHALL** 驗證通過後 context 帶有 APIKeyInfo（key ID, name, scopes）
- **SHALL** 驗證通過後更新 last_used_at

### Requirement: API Key Domain 單元測試
測試 `internal/domain/apikeys/` 的 repository + usecase。

#### Scenario: Create API Key
- **SHALL** `generateAPIKey()` 回傳 `sk_` 開頭的金鑰
- **SHALL** `generateAPIKey()` 產生的 hash 與 key 匹配（sha256）
- **SHALL** repository `Create` 回傳完整金鑰（一次性）

#### Scenario: Revoke API Key
- **SHALL** repository `Revoke` 設定 `is_active=false`
- **SHALL** 已撤銷的金鑰不可再使用
