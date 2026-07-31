## Context

8 個管理後台資料表在 `rud_acts` column header 中使用 `Checkbox` + `Label`/`TableColumnHeader` 來切換「刪除列表」模式（即顯示/隱藏已軟刪除的記錄）。

當前實作：

```tsx
<Checkbox
  id="skip1"
  indeterminate={false}
  onChange={(b) => { smu.mutate({ skip_softdelete: b }); }}
  aria-label="Skip soft delete"
  disabled={!isManager()}
/>
<Label for="skip1-input" class="text-xs text-ellipsis">刪除列表</Label>
```

`skip_softdelete` 由 smu mutation 處理，結果寫入 `tableStore.isRecover`，控制 cell 中顯示刪除/復原按鈕。

各頁面 header 實作略有差異：
- customer, department, estimate-item, item, metadict, sales-order: 用 `TableColumnHeader`
- salesrep: 用 `Label`

## Goals / Non-Goals

**Goals:**
- Checkbox + Label 全數取代為 `Button`，點擊切換 `skip_softdelete`
- Button 視覺化顯示當前狀態（例如：未點擊 = 「刪除列表」，已點擊 = 「顯示正常列表」）
- 保留 `isManager` 權限控制
- 所有 7 個實際有該功能的頁面一致實作
- 將重複 Button 邏輯抽象為共享元件 `TableDeleteListButton`，確保格式統一

**Non-Goals:**
- 不改變 cell 中 delete/recover 按鈕行為
- 不改變 `smu` mutation 邏輯
- 不改變 datatable context/store 結構
- article 頁面目前無此 Checkbox，不強行加入

## Decisions

### Decision 1: 使用 `Button` + 條件文字/樣式

用 `Button` 取代 Checkbox，variant 隨 `isRecover` 狀態切換：

```tsx
<Button
  variant={store.isRecover ? "default" : "destructive"}
  onClick={() => smu.mutate({ skip_softdelete: !store.isRecover })}
  disabled={!isManager()}
>
  {store.isRecover ? "顯示正常列表" : "刪除列表"}
</Button>
```

**Alternatives considered:**
- `Toggle` 元件 — 需額外引入，語意上無 checkbox 直觀，捨棄。
- 保留 Checkbox 僅改樣式 — 仍保留 Checkbox 固有 UX 問題，捨棄。

### Decision 2: 抽出共享元件 `TableDeleteListButton`

將重複的 Button 樣式、文字切換與權限控制集中到 `~/components/datatable/TableDeleteListButton.tsx`，各頁面 header 只負責傳入 `smu` 與 `store`：

```tsx
header: ({ header, table }) => {
  const { headerMutater } = header.headerActions!;
  const { tableStore: [store] } = table.tableActions!;
  const smu = headerMutater!();

  return (
    <div class="flex justify-end items-end space-x-2">
      <TableDeleteListButton smu={smu} store={store} />
    </div>
  );
},
```

共享元件內部統一處理 `variant`、`onClick`、`disabled` 與 `isManager` 判斷。

## Risks / Trade-offs

- 共享元件透過 `any` 型別接收 `smu`/`store`，避免與各頁 mutation 型別緊耦合。若未來 mutation 介面改變，只需調整一處。
- article 頁面原本無此功能，未進行修改，避免範圍蔓延。
