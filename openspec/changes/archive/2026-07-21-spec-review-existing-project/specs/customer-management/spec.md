## ADDED Requirements

### Requirement: 客戶 (Customer) 管理
系統維護客戶主檔，含聯絡人與地址簿，資料源為 NetSuite 同步。

#### Scenario: 查詢客戶列表
- **WHEN** 使用者 GET `/api/v1/customers` 傳入篩選（is_inactive、company_name 等）
- **THEN** 系統回傳分頁客戶列表，附地址與聯絡人資訊

#### Scenario: 建立客戶
- **WHEN** 使用者 POST `/api/v1/customers` 傳入客戶資料（company_name, phone, tax IDs 等）
- **THEN** 系統建立 Customer 記錄，含 AddressBook 與 Contact

#### Scenario: 客戶軟刪除與復原
- **WHEN** 使用者 DELETE `/api/v1/customers/{id}`
- **THEN** 系統執行軟刪除
- **WHEN** 使用者 PUT `/api/v1/customers/recover/{id}`
- **THEN** 系統復原客戶

#### Scenario: 強制 NetSuite 同步
- **WHEN** 使用者 GET `/api/v1/customers/force-sync`
- **THEN** 系統從 NetSuite 拉取最新客戶資料，upsert 至本地資料庫

### Requirement: 地址簿 (AddressBook) 管理
每個客戶可有多個地址（出貨/帳單地址）。

#### Scenario: 管理地址
- **WHEN** 使用者編輯客戶地址
- **THEN** 系統更新 CustomerAddressBook 及 CustomerAddressBookEntityAddress 記錄
- **THEN** 地址含完整欄位（addr1, addr2, city, state, zip, country, addressee, addrtext, addr_phone）

### Requirement: 聯絡人 (Contact) 管理
每個客戶可有多個聯絡人。

#### Scenario: 管理聯絡人
- **WHEN** 使用者編輯客戶聯絡人
- **THEN** 系統更新 CustomerContact 記錄（title, full_name, email, phone, company）
