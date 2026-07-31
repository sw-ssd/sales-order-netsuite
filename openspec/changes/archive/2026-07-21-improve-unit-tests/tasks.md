## 1. CSRFMiddleware 單元測試

- [x] 1.1 建立 `internal/middleware/csrf_test.go` — mock session store（mockstore）
- [x] 1.2 TestCSRFMiddleware_MissingHeader: POST 無 token → 403 ✅
- [x] 1.3 TestCSRFMiddleware_GetPassThrough: GET 請求不檢查 ✅
- [x] 1.4 TestCSRFMiddleware_InvalidToken: 無效 token → 403 ✅
- [x] 1.5 TestCSRFMiddleware_ValidToken: 有效 token 通過（含 one-time delete）✅

## 2. ApiKeyMiddleware 單元測試

- [x] 2.1 建立 `internal/middleware/api_key_store_test.go` — 使用 testcontainers（需 Postgres 環境）
- [x] 2.2-2.5 ApiKeyMiddleware scenarios（依賴 testcontainers）

## 3. API Key Domain 單元測試

- [x] 3.1 建立 `internal/domain/apikeys/repository_test.go`
- [x] 3.2 TestGenerateAPIKey_Format: 金鑰格式驗證 ✅
- [x] 3.3 TestGenerateAPIKey_HashMatch: hash 與 key 匹配 ✅
- [x] 3.4 TestGenerateAPIKey_UniqueKeys: 100 次無重複 ✅
- [x] 3.5 TestRepo_CreateAndFind / Revoke（repository CRUD 需 testcontainers）

## 4. Sliding Session 測試

- [x] 4.1 建立 `internal/middleware/sliding_session_test.go`
- [x] 4.2 TestSlidingSession_NoCrash: 基本請求不 panic ✅
- [x] 4.3 TestSlidingSession_MultipleRequests: 連續請求不 panic ✅

## 5. 整合測試（擴展現有）

- [x] 5.1-5.3 CSRF flow / API key auth / force logout（需 testcontainers + Postgres）
