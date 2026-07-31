## 1. 產生 migration

- [ ] 1.1 啟動本地基礎設施：`task infra:start`
- [ ] 1.2 使用 Atlas diff 產生 Goose migration：`task db:migrate -- create_api_keys_table`
- [ ] 1.3 檢查產生的 `database/goose/YYYYMMDDHMMSS_create_api_keys_table.sql`，確認 `api_keys` 表欄位、型別、index 與 `ent/schema/api_key.go` 一致

## 2. 驗證 migration

- [ ] 2.1 執行 migration：`task goose:up`
- [ ] 2.2 用 psql 或 `task goose:status` 確認 `api_keys` 表已建立
- [ ] 2.3 啟動後端：`task dev`，確認不再出現 `relation "api_keys" does not exist`

## 3. 執行測試

- [ ] 3.1 執行 auth 相關測試：`task tests:auth`
- [ ] 3.2 確認 `TestApiKey*` 與 `TestGenerateAPIKey` 全部通過
- [ ] 3.3 若測試失敗，回頭檢查 migration 與 `internal/domain/apikeys/` 的預期是否一致

## 4. 提交變更

- [ ] 4.1 將新增的 migration 檔案加入 git index
- [ ] 4.2 Commit：`chore: add api_keys table migration`
- [ ] 4.3 執行 `openspec validate --change fix-api-keys-table-missing` 確認 artifact 與實作一致

## 5. 文件更新（如需要）

- [ ] 5.1 若 `AGENTS.md` 中有提到 API key 相關 migration 資訊，確認是否需同步更新
- [ ] 5.2 記錄此 migration 已包含在本次 change 中
