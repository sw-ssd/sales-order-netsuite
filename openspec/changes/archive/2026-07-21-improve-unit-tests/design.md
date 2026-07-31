## Context

既有測試架構：
- `_integration_test.go`（1354 行）— 使用 `testcontainers` + Postgres + `httptest`
- 測試輔助：`stretchr/testify` (assert)、`testcontainers` (LocalTestContainer)
- 涵蓋：既有 auth login/logout 流程
- 未涵蓋：CSRFMiddleware、ApiKeyMiddleware、API Key domain、sliding session、force logout 全部 utype

## Goals / Non-Goals

**Goals:**
- CSRFMiddleware unit test（純邏輯，可 mock session store）
- ApiKeyMiddleware unit test（需 ent client — 用 testcontainers 或 mock）
- API Key domain unit test（generateAPIKey、repository CRUD）
- Sliding session unit test（檢查 last_touch timestamp 行為）
- Integration test：CSRF flow、API key auth、force logout 全 utype

**Non-Goals:**
- 不新增前端測試（SolidJS）
- 不新增 mobile 測試（Flutter）
- 不修改既有測試邏輯
- 不新增 CI/CD 設定

## Decisions

### 1. CSRFMiddleware 測試策略：純 unit test + mock session store

**選擇**：不啟動 testcontainers，直接 mock PostgresStoreEnt。

**理由**：
- CSRFMiddleware 只依賴 `scs.CtxStore` interface（FindCtx, DeleteCtx）
- 可用 `mockstore`（SCS 提供的 in-memory mock）或自訂 stub
- 測試放在 `internal/middleware/csrf_test.go`

### 2. ApiKeyMiddleware 測試策略：testcontainers

**選擇**：比照既有 `_integration_test.go` 使用 testcontainers。

**理由**：
- ApiKeyMiddleware 需要真實的 ent client 來查詢 `api_keys` 表
- 既有測試已建立完整的 testcontainers 基礎設施
- 測試放在 `internal/middleware/api_key_store_test.go`

### 3. API Key Domain 測試策略：testcontainers

**選擇**：使用 testcontainers 測試 repository CRUD + generateAPIKey 邏輯。

**理由**：
- `generateAPIKey()` 是純函式，可在 unit test 中測試
- repository 需要真實 DB，使用 testcontainers

### 4. Sliding Session 測試策略：unit test（mock SCS session）

**選擇**：建立 mock session manager 驗證 last_touch 行為。

**理由**：
- sliding 邏輯只操作 SCS 的 Get/Put，可 mock
- 不涉及 DB

### 5. Integration Test 策略：擴展現有 `_integration_test.go`

**選擇**：在既有 `_integration_test.go` 中加入 CSRF flow、API key auth、force logout 的測試案例。

**理由**：
- 避免重複的 testcontainers setup
- 與既有 auth test 共享 setup/teardown

## Risks / Trade-offs

| 風險 | 影響 | 緩解 |
|------|------|------|
| testcontainers 啟動時間 | 每次跑測試需 ~10s 啟動容器 | 既有測試已有此開銷；CI 可 cache container |
| CSRF token one-time 測試依賴 store | 若 mock 不完整，測試覆蓋不足 | 使用 `mockstore` + 驗證 DeleteCtx 被呼叫 |
| generateAPIKey 隨機性 | 無法 assert 預期值 | 只 assert 格式（`sk_` prefix, 長度, hash 匹配性） |
