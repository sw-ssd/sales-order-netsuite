## Why

此專案已發展為三端架構（Go 後端、SolidJS 前端、Flutter App），涵蓋約 35 個實體與 20 個領域 API，但尚未建立正式的規格文件。隨著規模增長，缺乏規格文件導致：
- 新功能開發需反覆探索既有行為
- 跨端（Web/Mobile）行為不一致難以察覺
- NetSuite 同步邏輯分散，無單一真相來源
- 新人 onboarding 成本高

本次變更的目的：為 **整個現有專案** 建立一份完整的規格總整理，涵蓋所有領域、實體、API、同步邏輯，作為後續開發的共同基礎。

## What Changes

- **`openspec/specs/`** 下建立完整規格目錄，每個領域一個子目錄
- 規格涵蓋三端（Backend / Frontend / Mobile）的行為定義
- 定義每個領域的：實體模型、API 合約、NetSuite 同步行為、跨端一致規範
- 建立 **design.md** 描述整體架構與跨域關係
- 建立 **tasks.md** 列出對應實作任務

### 不包含的範圍
- 不修改任何程式碼
- 不引入新功能
- 不改動既有行為
- 不建立測試

## Capabilities

### New Capabilities

- `core-sales-order`: 訂單領域 — SalesOrder, SalesOrderItem, Dispatch。含 CRUD、出貨管理（Dispatch）、即時推送（SSE/WebSocket）
- `customer-management`: 客戶管理 — Customer, AddressBook, Contact。含 NetSuite 同步
- `product-catalog`: 產品目錄 — Item, EstimateItem。含估價單邏輯
- `salesrep-management`: 業務員管理 — Salesrep。含 Credential 自動建立
- `auth-and-access`: 認證與授權 — User, Role, Tenant, Credential, OTP, Provider, Session, CasbinRule。含多角色登入（User/Salesrep/Customer）、OAuth2、RBAC
- `dictionary-lists`: 字典表群 — 13 個 List* 實體（Currency, Term, ApprovalStatus, CarNumber, CheckoutMethod, CreatedFrom, CustomerType, InvoiceType, PaymentMethod, Request, Specification, Unitstype, UnitstypeUom）
- `department-management`: 部門管理 — Department。含 NetSuite 同步
- `netsuite-sync`: NetSuite 雙向同步機制 — SuiteQL 讀取、Record API 寫入、定時排程（gocron）、Cloud Scheduler 觸發
- `infrastructure-cron`: 排程任務管理 — Cron 任務記錄與監控
- `cms-article`: 內容管理 — Article（CMS 公告/文章）
- `notification`: 通知系統 — Email 通知、系統事件（gookit/event）

### Modified Capabilities

無 — 本次為全新規格建立，不修改既有規格。
