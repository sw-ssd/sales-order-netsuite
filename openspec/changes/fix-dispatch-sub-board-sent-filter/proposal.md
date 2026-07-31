## Why

派車子板（Dispatch SubBoard）在切換「未 / 已發送」開關時，卡片篩選與定位邏輯異常，導致卡片未出現在預期欄位或開關狀態與畫面不一致。需要修正 `src/pages/admin/dispatch/widgets/sub-board.tsx` 及相關狀態管理，確保卡片依據 `has_sended` 正確分類與顯示。

## What Changes

- 修正 `sub-board.tsx` 中卡片過濾條件與 `has_sended` 開關狀態的對應關係。
- 確認 `confirmByDepartmentDispatch` 在「未 / 已發送」兩種模式下選取的卡片集合正確。
- 修正（如需要）`setting-context.tsx` 中 `filterStore` 與 `has_sended` 的初始值與切換邏輯。
- 增加或調整相關型別與輔助函式，避免卡片在不同開關狀態下錯位或遺失。
- 不影響後端 API；此變更為前端篩選與狀態同步修正。

## Capabilities

### New Capabilities
- `dispatch-sub-board-filter`: 定義派車子板依 `has_sended` 篩選與定位卡片的正確行為。

### Modified Capabilities
- 無既有 spec 需要變更；本變更為單一前端元件修復，未改變系統層級規格。
