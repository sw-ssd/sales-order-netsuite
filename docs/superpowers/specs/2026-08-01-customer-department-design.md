# Customer ↔ Department 關聯設計（部門跟隨業務）

## 背景與現況

需求：後端新增 customer 與 department 的關聯。

`sales-order-backend` submodule 已有未提交的 in-flight 改動：

- `ent/schema/customer.go` 的 `Mixin()` 已加入 `DepartmentMixin{}`，帶來 `department_id` 欄位與 `belong_department` edge（M2O、可空），`ent/gen` 生成碼已同步（`SetDepartmentID`、`WithBelongDepartment`、`customer.FieldDepartmentID` 等皆已存在）。
- 另新增一個 post-mutate hook（`ent.OpUpdate|ent.OpUpdateOne`），意圖在更新時將 salesrep 的部門複製到 customer。

經查證，該 hook 是壞的，且主流程不會觸發：

1. **post-mutate 寫不進 DB**：`next.Mutate(ctx, m)` 執行後 SQL 已送出，之後 `m.SetDepartmentID(...)` 只影響記憶體中的 entity，不會更新資料列。
2. **`crpm.Edges.OwnerSalesrepOrErr()` 會回傳 `NotLoadedError`**：`UpdateOneID` 未 eager-load `owner_salesrep` edge 時，`CustomerEdges.loadedTypes[3]` 為 false → 回傳 error → hook 直接 `return nil, err` → 任何直接本地更新（如 `RecoverCustomer`）都會失敗。
3. **掛錯 operation**：客戶的所有寫入（create / NetSuite 同步 / update）都走 `repo.CreateUpsertCustomersWithAddressBooksAndContacts` → `OnConflictColumns(customer.FieldID).UpdateNewValues()`，這是 **OpCreate** 的 upsert，`OpUpdate|OpUpdateOne` hook 根本不會執行。

其他現況：

- `customers` 表（`database/goose/20251114152452_base_tables.sql`）**沒有** `department_id` 欄位；`salesreps`、`items`、`dispatches`、`sales_orders`、`estimate_items` 都有（含 FK，`ON DELETE SET NULL`，見 `20251210091912_add_dispatch_shippingdate.sql`）。
- NetSuite 同步來源 `models.SQLCustomer` **沒有** department 欄位（NetSuite customer record 在此整合中不帶部門），故部門是**本地端**資料，不會從 NetSuite 回流。
- 讀取端：`Resource` / `ResourceOption` 目前從 **salesrep 的** `belong_department` edge 取 `Department` / `DepartmentName`，從未讀 customer 自己的 edge。
- 篩選端：`predicateCustomer` 的 `f.DepartmentID`（query param `department_id`）目前用 `customer.HasOwnerSalesrepWith(salesrep.DepartmentID(...))` 篩選。
- 排序端：`customerOrder` 不支援 `department`，但前端表格已有 `department` 欄位（`accessorKey: "department"`）。

## 目標 / 非目標

目標：

- 讓 customer 的 `department_id` 欄位真正被寫入、被讀取，並在每次寫入時**自動跟隨 salesrep 的部門**（Salesrep-follow only，使用者已確認）。
- 舊資料（`department_id` 為 NULL）在下次同步時自動補齊（self-healing）。
- API 回應格式維持相容：未補齊前 fallback 到 salesrep 部門，輸出與現況一致。

非目標：

- 不新增 API 寫入欄位：customer create/update request **不接受** `department`，部門永遠鏡像 salesrep。
- 不新增 Department 的 inverse edge（不支援從部門反查客戶列表）。
- 不改 NetSuite 同步模型（`SQLCustomer` 不加 department）。

## 核心行為

- **寫入（鏡像）**：customer 每次寫入（create / sync / update，皆為 OpCreate upsert）時，若 mutation 未明確設定 `department_id`，則以 `m.Salesrep()` 查 `Salesrep.DepartmentID` 並 `SetDepartmentID`。明確設定優先（目前無任何程式會明確設定）。
- **讀取**：`Resource` / `ResourceOption` 優先讀 customer 自己的 `belong_department`；未設定時 fallback 到 salesrep 的部門。
- **篩選**：維持 salesrep-based 篩選（在鏡像語意下，對已補齊與未補齊資料皆正確，零回歸）。
- **排序**：支援 `sort=department` → `customer.ByDepartmentID`。

## 架構與元件

全部在 `sales-order-backend` 內：

### 1. Migration（資料模型）

- 新增 goose migration：`ALTER TABLE customers ADD COLUMN department_id bigint NULL` + FK `customers_departments_belong_department` → `departments(internal_id)` `ON DELETE SET NULL`，與 `20251210091912` 對其他表的做法一致。
- **用 Atlas 產生**：`go run -mod=mod cmd/migrate_ent/main.go add_customer_department_id`（需可連的 dev/test DB，Atlas 對 `ent/gen/migrate/schema.go` diff，`atlas.sum` 會自動更新）。ent schema 不需再改。

### 2. 鏡像 Hook（`ent/schema/customer.go`）

- **刪除**現有壞掉的 post-mutate `OpUpdate|OpUpdateOne` hook。
- **新增** pre-mutate hook，掛 `ent.OpCreate|ent.OpUpdate|ent.OpUpdateOne`，在 `next.Mutate` **之前**執行：

```go
hook.On(
    func(next ent.Mutator) ent.Mutator {
        return hook.CustomerFunc(func(ctx context.Context, m *gen.CustomerMutation) (ent.Value, error) {
            // 1. 明確設定的 department_id 優先
            if _, ok := m.DepartmentID(); ok {
                return next.Mutate(ctx, m)
            }
            // 2. 沒有 salesrep 則不動
            sid, ok := m.Salesrep()
            if !ok || sid == 0 {
                return next.Mutate(ctx, m)
            }
            // 3. 查 salesrep 的部門並帶入
            sp, err := m.Client().Salesrep.Query().
                Where(salesrep.IDEQ(sid)).
                Only(ctx)
            if err != nil {
                if gen.IsNotFound(err) {
                    return next.Mutate(ctx, m)
                }
                return nil, err
            }
            if sp.DepartmentID != 0 {
                m.SetDepartmentID(sp.DepartmentID)
            }
            return next.Mutate(ctx, m)
        })
    },
    ent.OpCreate|ent.OpUpdate|ent.OpUpdateOne,
)
```

- 因 create / sync / update 皆走 OpCreate upsert，此 hook 在 insert 時寫入部門，`UpdateNewValues()` 的 conflict update 也會一併更新 → 舊資料自動補齊。
- `RecoverCustomer`（`OpUpdateOne`）不再觸發 `NotLoadedError` 而失敗。
- 需新增 import：`ent/gen/salesrep`。

### 3. 讀取路徑（`internal/domain/customers`）

- `repository.go`：`GetCustomer` 與 `Search` 的 query 加 `.WithBelongDepartment()`。
- `transformation.go`：
  - `Resource`：先讀 `a.Edges.BelongDepartmentOrErr()`；成功且有值 → `r.Department = d.ID; r.DepartmentName = d.Name`；否則維持現有邏輯（salesrep 的部門）。
  - `ResourceOption`：同上。
- `model.go`：不需改（`CustomerResponse.Department/DepartmentName`、`CustomerOptionResponse` 已存在）。

### 4. 篩選與排序（`internal/domain/customers/transformation.go`）

- 篩選：`predicateCustomer` 的 `f.DepartmentID` 分支**維持** `customer.HasOwnerSalesrepWith(salesrep.DepartmentID(f.DepartmentID))`，僅更新註解。
- 排序：`customerOrder` 加 `case customer.FieldDepartmentID:` → `customer.ByDepartmentID(sql.OrderAsc()/Desc(), sql.OrderNullsLast())`。

## 資料流

```
NetSuite 同步 / create / update
        │
        ▼
repo.CreateUpsertCustomersWithAddressBooksAndContacts (OpCreate upsert)
        │  mutation: department_id 未明確設定
        ▼
pre-mutate hook：m.Salesrep() → 查 Salesrep.DepartmentID → m.SetDepartmentID(...)
        │
        ▼
INSERT ... ON CONFLICT (internal_id) DO UPDATE SET ... department_id = salesrep 部門
        │
        ▼
GetCustomer / Search (WithBelongDepartment) → Resource/ResourceOption
        │  own edge 有值 → 用 own；無值 → fallback salesrep 部門
        ▼
GET /api/.../customers 回應：department / department_name
```

## 錯誤處理

- hook 查詢 salesrep 不存在（`IsNotFound`）→ 跳過，不設定部門，不回錯誤（部門留空，下次寫入再補）。
- hook 查詢發生其他錯誤 → 回傳錯誤，寫入失敗（與現有 hook 錯誤處理一致，可被呼叫端觀察）。
- migration 由 Atlas diff 產生，若 dev/test DB 不可連，改用現有 testcontainers 流程產生。

## 測試

沿用 `internal/domain/customers/repository_test.go` / `usecase_test.go` 的 pattern（含 `seeder`）：

- **Hook**：
  - create/upsert 時未設 department_id → 結果列 `department_id` = salesrep 的部門。
  - mutation 明確設定 department_id → 不被覆寫。
  - 無 salesrep / salesrep 無部門 → `department_id` 不動（NULL）。
- **Repository**：
  - `GetCustomer` / `Search` 載入 own `belong_department` edge（`WithBelongDepartment` 生效）。
  - `RecoverCustomer` 不再回錯誤。
- **Transformation**：
  - 客戶自己有部門 → 回傳 own 的 `department` / `department_name`。
  - 客戶無部門、salesrep 有部門 → fallback salesrep 的值（與現況輸出一致）。
- Migration 由既有 testcontainers 測試套件覆蓋（需先產生 migration 檔）。

## 風險 / 取捨

- **鏡像語意下的自我修正**：salesrep 換部門後，customer 在下一次該客戶被同步/更新時自動跟上；在此之前回應可能短暫顯示舊部門（fallback 讀 own edge 已設定 → 顯示舊值，直到重寫）。可接受，與「跟隨 salesrep」語意一致。
- **既有 hook 是地雷**：若本設計未執行（僅保留現狀），`RecoverCustomer` 會壞。本設計已將刪除/取代列入。
- **不回寫 NetSuite**：部門僅本地端，NetSuite 重新同步不會覆蓋本地值（upsert 未設 department_id 時 conflict update 不觸碰該欄位；hook 會重設為 salesrep 部門）。

## 遷移計畫

1. 產生 goose migration（Atlas diff）。
2. 套用 migration 到 dev/test DB（既有 `Taskfile.yml` / testcontainers 流程）。
3. 同步後舊資料自動補齊，無需一次性 backfill 指令。

## 待決問題

- 無（使用者已確認「Salesrep-follow only」，不開放 API 直接設定部門）。
