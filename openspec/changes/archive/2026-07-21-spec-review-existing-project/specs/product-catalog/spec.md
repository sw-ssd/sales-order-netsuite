## ADDED Requirements

### Requirement: 產品 (Item) 目錄管理
系統維護產品主檔，資料源為 NetSuite 同步。

#### Scenario: 查詢產品列表
- **WHEN** 使用者 GET `/api/v1/items` 傳入篩選條件
- **THEN** 系統回傳分頁產品列表（item_id, display_name, description, units_type 等）

#### Scenario: 產品軟刪除與復原
- **WHEN** 使用者 DELETE `/api/v1/items/{id}`
- **THEN** 系統執行軟刪除
- **WHEN** 使用者 PUT `/api/v1/items/recover/{id}`
- **THEN** 系統復原產品

#### Scenario: 強制 NetSuite 同步
- **WHEN** 使用者 GET `/api/v1/items/force-sync`
- **THEN** 系統從 NetSuite 拉取最新產品目錄

### Requirement: 估價單明細 (EstimateItem) 管理
業務員為客戶建立估價記錄。

#### Scenario: 查詢估價清單
- **WHEN** 使用者 GET `/api/v1/estimate_items` 傳入篩選條件
- **THEN** 系統回傳估價明細，含客戶（Customer）、業務員（Salesrep）、產品（Item）關聯

#### Scenario: 估價軟刪除與復原
- **WHEN** 使用者 DELETE `/api/v1/estimate_items/{id}`
- **THEN** 系統執行軟刪除
- **WHEN** 使用者 PUT `/api/v1/estimate_items/recover/{id}`
- **THEN** 系統復原估價

#### Scenario: 強制同步
- **WHEN** 使用者 GET `/api/v1/estimate_items/force-sync`
- **THEN** 系統從 NetSuite 拉取最新估價資料
