## ADDED Requirements

### Requirement: 字典表管理
系統維護 13 組字典表，資料源為 NetSuite 同步。每個字典表僅含基本欄位（is_inactive, display_name, table_name 等）與對應關聯。

#### Scenario: 查詢字典表
- **WHEN** 前端需要下拉選單資料（如幣別、付款條件、車號等）
- **THEN** 系統從對應 List* 表回傳資料

#### Scenario: 字典表同步
- **WHEN** 使用者 GET `/api/v1/metadicts/force-sync`
- **THEN** 系統從 NetSuite 同步所有 List* 表資料

#### Scenario: 字典表修復
- **WHEN** 使用者 DELETE `/api/v1/metadicts/{id}/{table_name}`
- **THEN** 系統軟刪除該字典項目
- **WHEN** 使用者 PUT `/api/v1/metadicts/recover/{id}/{table_name}`
- **THEN** 系統復原該字典項目

### 字典表列表
| 實體 | 用途 |
|------|------|
| ListCurrency | 幣別 |
| ListTerm | 付款條件 |
| ListApprovalStatus | 核准狀態 |
| ListCarNumber | 車輛號碼 |
| ListCheckoutMethod | 結帳方式 |
| ListCreatedFrom | 訂單來源 |
| ListCustomerType | 客戶類型 |
| ListInvoiceType | 發票類型 |
| ListPaymentMethod | 付款方式 |
| ListRequest | 加工要求（代切、實重肉片等） |
| ListSpecification | 加工規格（對切、十字切等） |
| ListUnitstype | 單位類型 |
| ListUnitstypeUom | 單位換算 |
