## ADDED Requirements

### Requirement: `api_keys` 資料表必須透過 Goose migration 建立

此 capability 只新增一條 migration，讓現有資料庫補上 API key 功能所需的資料表。

#### Scenario: 在乾淨資料庫上啟動後端
- **WHEN** 後端啟動並執行 `goose.Up()`
- **THEN** 資料庫中必須存在 `api_keys` 表，且欄位與 `ent/schema/api_key.go` 一致

#### Scenario: 在已啟動的開發資料庫上執行 migration
- **WHEN** 開發人員執行 `task goose:up`
- **THEN** `api_keys` 表被建立，不影響其他既有資料表

### Requirement: migration 檔案必須與 Ent schema 一致

migration 必須完全對應 `ent/schema/api_key.go` 與 `ent/gen/migrate/schema.go` 所定義的 `api_keys` 表結構。

#### Scenario: 比對 schema 與 migration
- **WHEN** 檢視 `database/goose/YYYYMMDDHMMSS_create_api_keys_table.sql`
- **THEN** 表名、欄位名稱、型別、長度、預設值、nullable、index 必須與 `APIKeysTable` 定義一致

### Requirement: API key 相關測試必須通過

補上 migration 後，既有 API key 測試應能在有資料表的環境下正常執行。

#### Scenario: 執行 auth 相關測試
- **WHEN** 執行 `task tests:auth`
- **THEN** `TestApiKeyMiddleware_*`、`TestGenerateAPIKey` 等測試全部通過

## MODIFIED Requirements

無。

## REMOVED Requirements

無。
