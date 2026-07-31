## ADDED Requirements

### Requirement: NetSuite 讀取同步（SuiteQL）
系統定期從 NetSuite 透過 SuiteQL 查詢讀取最新資料至本地 PostgreSQL。

#### Scenario: 強制同步
- **WHEN** 使用者 GET `/{domain}/force-sync`
- **THEN** 系統發送 SuiteQL 查詢至 NetSuite REST API
- **THEN** 解析回傳 JSON，upsert 至對應 ent 表
- **THEN** 支援同步的領域：Customer, Item, SalesOrder, Salesrep, Department, EstimateItem, Metadicts

#### Scenario: 定時同步（gocron）
- **WHEN** 系統啟動 cron scheduler
- **THEN** 根據 crontab 設定自動執行同步任務
- **THEN** 記錄執行結果至 Cron 表

#### Scenario: 事件觸發同步
- **WHEN** 本地建立/修改關鍵實體（如 SalesOrder）
- **THEN** 系統透過 gookit/event 發布事件
- **THEN** 非同步處理事件，寫入 NetSuite（Record API）

### Requirement: NetSuite 寫入同步（Record API）
系統將本地建立的訂單等資料回寫至 NetSuite。

#### Scenario: 建立訂單觸發回寫
- **WHEN** 使用者在 App 建立 SalesOrder
- **THEN** 系統寫入本地資料庫
- **THEN** 非同步透過 NetSuite Record API 建立對應交易
- **THEN** 記錄 NetSuite internal_id 回本地 sync_id

#### Scenario: Cloud Scheduler 觸發
- **WHEN** Google Cloud Scheduler 呼叫 `/api/v1/cron/{type}`
- **THEN** 系統執行對應同步任務
