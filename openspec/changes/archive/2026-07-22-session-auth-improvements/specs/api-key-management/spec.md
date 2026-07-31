## ADDED Requirements

### Requirement: API 金鑰管理
系統提供正式的機器對機器 API 金鑰，取代現有靜態 X-Sowinsoft-Token。每支金鑰獨立管理、含 scope 權限、可撤銷。

#### Scenario: 建立金鑰
- **WHEN** 管理者建立 API 金鑰
- **THEN** 系統產生唯一金鑰字串：`sk_<8 hex chars>_<56 hex chars>`
- **THEN** `key_prefix` 儲存金鑰中 `sk_` 之後、第一個 `_` 之前的 8 個 hex 字元
- **THEN** 儲存金鑰 hash（sha256）至 `api_keys` 表
- **THEN** 儲存名稱、scope、到期日
- **THEN** 回傳完整金鑰給管理者（一次性，不再顯示）

#### Scenario: API 金鑰認證
- **WHEN** client 發送請求帶 `Authorization: Bearer sk_...`（或 `X-Sowinsoft-Token: sk_...`）
- **THEN** middleware 解析 `key_prefix`（8 hex 字元）找到對應記錄
- **THEN** sha256(key) 比對儲存的 hash
- **THEN** 檢查 `is_active = true`、`expires_at > now()`
- **THEN** 更新 `last_used_at` 時間戳
- **THEN** 驗證通過後將 scope、key name 寫入 request context

#### Scenario: Scope 權限檢查
- **WHEN** API key 驗證通過後進入 handler
- **THEN** middleware 檢查 handler 所需 scope 是否在 key 的 scopes 中
- **THEN** scope 不足回傳 403
- **THEN** scope 格式：`<domain>:<action>`（如 `sales_orders:read`, `items:write`）

#### Scenario: 撤銷金鑰
- **WHEN** 管理者撤銷金鑰
- **THEN** 系統將 `is_active` 設為 false
- **THEN** 已發出的金鑰立即無效

#### Scenario: 金鑰列表與狀態
- **WHEN** 管理者查詢金鑰列表
- **THEN** 系統回傳：名稱、前綴、scope、建立時間、到期時間、最後使用時間、是否啟用
- **THEN** 不回傳金鑰本身（hash 無法逆推）

### api_keys 資料表設計

| 欄位 | 類型 | 說明 |
|------|------|------|
| id | UUID | 主鍵 |
| name | string | 金鑰名稱，如「排程同步用」 |
| key_hash | string | sha256(api_key) |
| key_prefix | string | 前 8 個 hex 字元（不含 `sk_`） |
| scopes | text[] | 權限範圍陣列 |
| expires_at | timestamptz | 到期日，null = 永不到期 |
| is_active | bool | 是否啟用 |
| created_by | UUID → user | 建立者 |
| created_at | timestamptz | 建立時間 |
| last_used_at | timestamptz | 最後使用時間，null = 從未使用 |
