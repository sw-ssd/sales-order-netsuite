## Why

目前業務員與客戶管理頁面的「重置密碼」按鈕對所有登入用戶皆可操作，需要根據角色與資料關係進行更細緻的權限控制：
- 客戶的重置密碼應僅開放給「所屬業務」與管理員
- 業務員的重置密碼應僅開放給管理員，且管理員不能重置自己的密碼

## What Changes

- 在 `sales-order-frontend/src/pages/admin/salesrep/widgets/columns.tsx`：
  - 將「重置密碼」按鈕改為 `disabled={!isManager() || isSelfLogin}`
  - 即管理員可見可操作，非管理員與登入者本人 disabled
- 在 `sales-order-frontend/src/pages/admin/customer/widgets/columns.tsx`：
  - 將「重置密碼」按鈕改為 `disabled={!isManager() && !isAssignedSalesrep}`
  - 即所屬業務與管理員可操作，其他 disabled
- 保留既有 mutation 與 toast 邏輯

## Capabilities

### New Capabilities
(無)

### Modified Capabilities
(無 - 純為前端顯示權限控制，非產品規格變更)
