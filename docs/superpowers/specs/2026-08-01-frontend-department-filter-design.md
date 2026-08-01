# Customer / Sales-Order 頁部門下拉過濾設計

## 背景與現況

需求（frontend）：customer 與 sales-order 頁新增「部門」下拉過濾：

- 一般用戶（非管理者）：只能看到自己所屬部門的資料，下拉 **disabled**（鎖定自己部門）。
- 管理者：可以看到所有部門，下拉 **enabled**（可選任一部門或全部）。

現況盤點：

- **先例**：dispatch 頁已有部門 tab + `disabledDepartment` 邏輯（`isManager()` 管理者全開、一般用戶鎖自己部門）——純前端 UX，後端無 viewer 部門強制。本次沿用同一模式（使用者已確認**純前端**，不做後端強制）。
- **auth state**（`src/pages/auth/context.tsx`）：`isManager()` 讀 `authState.info?.salesrep?.session_info_base?.is_manager`；`infoDepartment()` 讀 `authState.info?.salesrep?.department`（ProfileCard 另有 `department_name`）。
- **頁面結構**：customer 與 sales-order 頁同構——route loader prefetch 基礎 query（無參數），頁面 `useQuery(xxxQuery)` 顯示，server 搜尋走 `smu`（`headerMutater`）：`smu.mutate({...params})` → `fetchQuery(xxxQuery({...p}))` → `listInvalidate`。
- **後端 filter 參數已存在**：
  - customers：`department_id`（`filter.Nsfilter.DepartmentID`，`predicateCustomer` 用 salesrep 部門篩選）。
  - sales_orders：`department`（`SalesOrderFilter.Department` → `salesorder.DepartmentIDEQ`），另已有 department 排序。
- **metadict 部門資料源**：`metadictOptionsQuery(["departments"])`；customer 的 `CUSTOMER_METADICT_KEYS` 未含 `departments`，sales-order 的 `SALES_ORDER_METADICT_KEYS` 中 `"departments"` 被註解——兩頁都需補上。
- **UI 元件**：`~/components/ui/select.tsx`（ark-ui）可用。
- **測試基建**：vitest（`npm test`）+ playwright e2e（`npm run test:e2e`）。

## 目標 / 非目標

目標：

- customer / sales-order 兩頁新增部門下拉過濾；非管理者鎖定自己部門，管理者可選全部或單一部門。
- 非管理者進入頁面時，列表直接只顯示自己部門（初始載入即限縮，不得閃現全部資料）。

非目標：

- **不做後端權限強制**（使用者已確認純前端，與 dispatch 一致；懂 API 者可繞過，內部系統可接受）。
- 不改 dispatch 頁（其 tab UX 維持現狀，共用元件可作為後續 follow-up 遷移）。
- 不改後端（filter 參數與 predicate 均已存在）。
- 不做部門資料的 CRUD（沿用 metadict 既有流程）。

## 核心行為

- **非管理者**：下拉 disabled，值 = 自己部門（`infoDepartment`），永不觸發 `onChange`；列表查詢恆帶 `department_id`（customer）/ `department`（sales-order）= 自己部門。
- **管理者**：下拉 enabled，選項 =「全部部門」(0) + 所有啟用部門（排除 `is_inactive`）；預設自己部門；選「全部部門」→ 省略部門參數（等同不篩選）。
- **變更時**：觸發該頁既有 server-search（`smu`），帶部門參數；失敗依既有 toast 錯誤處理，下拉值回滾。
- **初始載入**：頁面基礎 query 依 auth 帶入初始部門參數（非管理者 = 自己部門；管理者 = 自己部門）；參數化 query 尚未載入時表格為空（Loading 後才顯示），避免非管理者閃現全部資料。

## 架構與元件

全部在 `sales-order-frontend` 內：

### 1. 共用元件 `src/components/datatable/DepartmentFilter.tsx`

- Props：
  - `options: MetadictOption[]`（`table_name === "departments"` 的 metas）
  - `value: number`（0 = 全部）
  - `onChange: (id: number) => void`
- 內部 `useAuth()`：
  - `isManager()` 為 false → `<Select disabled>`，值 = `infoDepartment`；顯示名稱優先從 `options` 匹配，找不到時用 auth 的 `salesrep.department_name`。
  - 管理者 → `<Select enabled>`：選項 `[{label: "全部部門", value: 0}, ...啟用部門]`，預設值由外部 `value` 控制。
- `infoDepartment` 缺失（salesrep 無部門）：非管理者顯示「無部門」disabled、外部不帶部門參數；管理者預設「全部」。
- options 尚未載入：disabled，載入後啟用。

### 2. Customer 頁

- `src/routes/admin/customer.tsx`：`CUSTOMER_METADICT_KEYS` 加 `"departments"`。
- `src/pages/admin/customer/widgets/context.tsx`：新增部門 signal；toolbar 加入 `<DepartmentFilter>`；`onChange` → `smu.mutate({ department_id: id })`（id = 0 時省略參數）。
- `src/pages/admin/customer/Customer.tsx`：基礎 query 改為 `customersQuery(initialDepartmentParams())`，由 `useAuth()` 算出初始參數。

### 3. Sales-order 頁

- `src/routes/admin/sales-order.tsx`：`SALES_ORDER_METADICT_KEYS` 取消 `"departments"` 註解。
- `src/pages/admin/sales-order/widgets/context.tsx`：同 customer 頁，`onChange` → `smu.mutate({ department: id })`。
- `src/pages/admin/sales-order/SalesOrder.tsx`：基礎 query 帶初始 `department` 參數。

## 資料流

```
auth (isManager / infoDepartment) ──> DepartmentFilter（選項來自 metadictOptions["departments"]）
        │ 非管理者：disabled 鎖定自己部門；管理者：可選 全部/單一部門
        ▼
onChange(id) ──> smu.mutate({ department_id | department: id })  ──> fetchQuery(xxxQuery) ──> listInvalidate
        ▼
列表 query（含部門參數）──> 後端既有 predicate（customers: DepartmentID via salesrep；sales_orders: DepartmentIDEQ）
```

## 錯誤處理

- 部門查詢失敗：沿用該頁既有 `smu` mutation 的 error toast（與其他 column filter 行為一致，不做下拉值回滾）。
- metadict 部門資料未載入：下拉 disabled（不阻擋頁面其他功能）。
- `infoDepartment` 無值：非管理者不帶部門參數（無法限縮，記錄於 spec）；管理者預設「全部」。

## 測試

- **vitest unit（元件）**：
  - 非管理者：disabled、值 = 自己部門、`onChange` 不觸發。
  - 管理者：含「全部部門」+ 啟用部門、預設自己部門、選擇後 `onChange` 觸發。
  - `infoDepartment` 缺失與 options 未載入的 fallback。
- **playwright e2e**：
  - 非管理者登入：兩頁僅顯示自己部門資料、下拉 disabled。
  - 管理者登入：下拉 enabled，切「全部 / 指定部門」列表對應變化，查詢參數正確（customer `department_id`、sales-order `department`）。
- 後端無改動，不需後端測試。

## 風險 / 取捨

- **純前端限制可被繞過**（API 直接呼叫）——已與使用者確認可接受（內部系統、與 dispatch 一致）。
- **初始載入多一次請求**：參數化 query 與 loader 的無參數 prefetch key 不同，非管理者進入頁面時 loader 的 prefetch 不被重用（無害的 cache 殘留）；以「表格先空再顯示」避免全部資料閃現。
- **部門參數軸線獨立**：與既有 column filter 的合併策略維持現狀（各自觸發 server-search），不在此次改變既有行為。

## 待決問題

- 無（使用者已確認：純前端 UX、管理者預設自己部門且有「全部部門」選項）。
