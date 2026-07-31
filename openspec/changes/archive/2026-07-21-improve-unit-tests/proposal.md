## Why

上一個 change（`session-auth-improvements`）實作了 CSRF 保護、M2M API Key 系統、Sliding Session、Force Logout 補齊等安全機制，但缺乏對應的測試涵蓋。既有 `_integration_test.go` 主要測試既有認證流程，未涵蓋新功能。

本次變更為新實作的 auth 功能補上單元測試與整合測試，確保行為正確且防止迴歸。

## What Changes

- 為 `CSRFMiddleware` 撰寫 unit test（Go）
- 為 `ApiKeyMiddleware` 撰寫 unit test（Go）
- 為 `internal/domain/apikeys/` 撰寫 repository + usecase unit test
- 為 sliding session 邏輯撰寫 unit test
- 撰寫 auth 領域的整合測試覆蓋 CSRF flow、API key auth、force logout（所有 utype）

## Capabilities

### New Capabilities

- `test-auth-csrf`: CSRFMiddleware 單元測試 — token 驗證、過期、遺失 header
- `test-auth-api-key`: ApiKeyMiddleware + API Key domain 單元測試 — 建立、驗證、撤銷
- `test-auth-session`: Sliding session + force logout 單元測試
- `test-auth-integration`: Auth 整合測試 — 完整 CSRF flow、API key auth

### Modified Capabilities

無

## Impact

- Go test files — 新增 4-5 個 `_test.go` 檔案
- 測試資料 — mock Ent client 或使用 test database
- CI pipeline — `go test ./...` 自動涵蓋
