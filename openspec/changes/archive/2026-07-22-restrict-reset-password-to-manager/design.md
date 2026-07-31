## Context

`salesrep` 與 `customer` 管理後台的 `rud_acts` column cell 中，各有一個「重置密碼」`TableSheetButton`。經修正後的權限需求如下：

- **Customer「重置密碼」**：只有「所屬業務」（該客戶的 `salesrep`）與 `isManager` 可見並可操作；其他用戶看到按鈕但為 disabled。
- **Salesrep「重置密碼」**：只有 `isManager` 可見並可操作；且管理員不能重置「自己」的密碼；其他用戶看到按鈕但為 disabled。

## Goals / Non-Goals

**Goals:**
- Customer 重置密碼按鈕：disabled 條件為 `!isManager() && currentSalesrepId !== rowSalesrepId`
- Salesrep 重置密碼按鈕：disabled 條件為 `!isManager() || currentSalesrepId === rowSalesrepId`
- 按鈕始終渲染，非權限者僅 disabled，避免介面跳動
- 保持現有的 mutation 行為與 toast 提示不變

**Non-Goals:**
- 不改變後端 reset-null-password API 權限（前端先控制按鈕 disabled）
- 不調整 reset password 的 mutation 邏輯

## Decisions

### Decision: 使用 `disabled` 控制可見性，而非 `Show` 條件渲染

將按鈕始終渲染，但根據權限動態設定 `disabled`。這樣非權限者仍能看到按鈕佔位，避免不同角色看到不同列寬或介面跳動。

**Customer 權限計算：**
```tsx
const [{ info }, { isManager }] = useAuth();
const currentSalesrepId = info?.salesrep?.id;
const rowSalesrepId = row.getValue("salesrep") as number;
const canResetCustomerPassword = () =>
  isManager() || currentSalesrepId === rowSalesrepId;
```

**Salesrep 權限計算：**
```tsx
const [{ info }, { isManager }] = useAuth();
const currentSalesrepId = info?.salesrep?.id;
const rowSalesrepId = row.getValue("id") as number;
const canResetSalesrepPassword = () =>
  isManager() && currentSalesrepId !== rowSalesrepId;
```

按鈕上直接使用 `disabled={!canResetXxxPassword()}`。

**Alternatives considered:**
- 使用 `Show when={...}` 隱藏按鈕 — 會導致不同角色看到不同列寬，捨棄。

## Risks / Trade-offs

- `isManager` 與 `info` 來自 `useAuth` hook；兩個 cell 中已調整解構方式。
- Customer 的所屬業務透過 `row.getValue("salesrep")` 取得，該欄位已在 model 中定義。
- 僅為前端控制，後端仍應補上對應權限檢查以確保安全。
