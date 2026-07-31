## Why

後端執行時出現 `ERROR: relation "api_keys" does not exist`，導致 `ApiKeyMiddleware` 無法驗證 `X-Sowinsoft-Token`，也讓 `/api/v1/api-keys` 相關功能無法使用。`api_keys` 的 Ent schema 與 gen 程式碼已經存在，但缺少對應的 database migration，因此現有資料庫中沒有這張表。

## What Changes

- 在 `sales-order-backend/database/goose/` 新增一條 Goose migration，建立 `api_keys` 資料表，欄位與 index 對齊 `ent/schema/api_key.go`。
- 新增 migration 後，現有開發/測試/生產環境在下次啟動 server（或執行 `task goose:up`）時會自動補上 `api_keys` 表。
- 補上 migration 後執行既有 API key 相關測試，確認 `ApiKeyMiddleware` 與 `internal/domain/apikeys/` 運作正常。

## Capabilities

### New Capabilities

- `api-keys-table-migration`: 建立並維護 `api_keys` 資料表的 Goose migration。

### Modified Capabilities

- 無。此變更只補上原本就預期存在的資料表，不改變任何現有 API 行為或介面。

## Success Criteria

1. 在乾淨的資料庫上啟動後端，不會再出現 `relation "api_keys" does not exist` 錯誤。
2. `task tests:auth` 中與 `TestApiKey*` 相關的測試全部通過。
3. `api_keys` 表的欄位、型別、index 與 `ent/schema/api_key.go` 完全一致。

## Out of Scope

- 不改變 API key 產生、驗證、授權的業務邏輯。
- 不新增或修改 API endpoint。
- 不處理舊版 JWT-based API token 的相容性（已標示為 legacy）。
