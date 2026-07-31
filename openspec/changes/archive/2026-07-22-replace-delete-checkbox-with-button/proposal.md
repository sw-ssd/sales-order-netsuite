## Why

目前所有管理後台資料表的「刪除列表」功能使用 Checkbox 切換，UX 不明確且容易被誤觸。需更換為 Button，點擊時切換顯示已軟刪除記錄的狀態，同時 Button 視覺化反映當前模式（顯示刪除列表 vs 顯示正常列表）。

## What Changes

- 將 8 個管理頁面資料表 header 中的 Checkbox + Label 取代為 Toggle Button
- Button 需顯示當前狀態：點擊後切換 `skip_softdelete`，Button 文字/樣式反映當前模式
- 影響頁面：article, customer, department, estimate-item, item, metadict, sales-order, salesrep

## Capabilities

### New Capabilities
(無)

### Modified Capabilities
(無 — UI 元件行為變更，非產品規格變更)
