## 1. 產生 migration

- [x] 1.1 啟動本地基礎設施：`task infra:start`
- [x] 1.2 使用 Atlas diff 產生 Goose migration（因既有 migration replay 會失敗，改為手動撰寫）
- [x] 1.3 檢查 `database/goose/20260731170056_create_api_keys_table.sql`，確認 `api_keys` 表欄位、型別、index 與 `ent/schema/api_key.go` 一致

## 2. 驗證 migration

- [x] 2.1 執行 migration：`task goose:up`
- [x] 2.2 用 `atlas schema inspect` 確認 `api_keys` 表已建立
- [x] 2.3 啟動後端：`task run`，確認不再出現 `relation "api_keys" does not exist`

## 3. 執行測試

- [ ] 3.1 執行 auth 相關測試：`task test:auth`（受限於本機無 Docker socket，未能直接執行）
- [ ] 3.2 已改由啟動後端與 `atlas schema inspect` 驗證 `api_keys` 表
- [x] 3.3 migration 與 `internal/domain/apikeys/` 的預期一致

## 4. 提交變更

- [x] 4.1 將新增的 migration 檔案加入 git index
- [ ] 4.2 Commit：`chore: add api_keys table migration`
- [ ] 4.3 執行 `openspec validate --change fix-api-keys-table-missing` 確認 artifact 與實作一致

## 5. 文件更新（如需要）

- [x] 5.1 `AGENTS.md` 中無 API key migration 資訊需更新
- [x] 5.2 已記錄此 migration 包含在本次 change 中
