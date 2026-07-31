## ADDED Requirements

### Requirement: 訂單 (SalesOrder) 管理
使用者能建立、檢視、修改與刪除銷售訂單。訂單為核心業務實體，綁定客戶、業務員、條款與幣別。

#### Scenario: 建立訂單
- **WHEN** 使用者 POST `/api/v1/sales_orders` 提供訂單資訊（客戶、明細、交期等）
- **THEN** 系統建立 SalesOrder 記錄，回傳訂單資料與 ID
- **THEN** 明細項目寫入 SalesOrderItem 表
- **THEN** 系統發送事件觸發 NetSuite 非同步同步

#### Scenario: 查詢訂單列表
- **WHEN** 使用者 GET `/api/v1/sales_orders` 傳入篩選條件（日期、客戶、狀態等）
- **THEN** 系統回傳分頁訂單列表，含明細摘要與 meta 訊息

#### Scenario: 訂單軟刪除與復原
- **WHEN** 使用者 DELETE `/api/v1/sales_orders/{id}`
- **THEN** 系統標記 deleted_at 執行軟刪除
- **WHEN** 使用者 PUT `/api/v1/sales_orders/recover/{id}`
- **THEN** 系統清除 deleted_at 復原訂單

### Requirement: 出貨管理 (Dispatch)
業務能管理訂單的出貨排程，含車輛派遣、確認出貨。

#### Scenario: 編輯出貨
- **WHEN** 使用者 PATCH `/api/v1/sales_orders/dispatches/upsert` 傳入出貨資料
- **THEN** 系統更新或新增 Dispatch 記錄

#### Scenario: 確認出貨
- **WHEN** 使用者 PATCH `/api/v1/sales_orders/dispatches/confirm/{id}`
- **THEN** 系統標記 has_sended = true，寫入出貨記錄

#### Scenario: 即時推送出貨狀態
- **WHEN** 前端連線到 `/api/v1/sales_orders/dispatches/sse` 或 `/ws`
- **THEN** 系統推送即時出貨變更事件給訂閱者

### Requirement: 明細項目 (SalesOrderItem) 管理
訂單中的逐筆產品明細，含數量、單價、加工要求。

#### Scenario: 查詢明細
- **WHEN** 使用者載入訂單明細
- **THEN** 系統回傳 SalesOrderItem 列表，關聯產品（Item）、加工要求（ListRequest / ListSpecification）
