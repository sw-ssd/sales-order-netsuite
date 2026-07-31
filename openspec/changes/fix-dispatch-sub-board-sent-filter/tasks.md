# 實作步驟：修正派車子板「未 / 已發送」卡片定位

## 1. 確認現行邏輯

- [x] 1.1 閱讀 `src/pages/admin/dispatch/widgets/sub-board.tsx`，確認 `_checkHasSended`、`confirmByDepartmentDispatch` 與 `columns` 的關聯。
- [x] 1.2 閱讀 `src/pages/admin/dispatch/widgets/setting-context.tsx`，確認 `filterStore.has_sended` 初始化與 `onChangeHasSended` 邏輯。
- [x] 1.3 閱讀 `src/pages/admin/dispatch/widgets/card.tsx`，確認卡片顯示欄位與 `has_sended`、`custbody_hf_is_car_dispatched` 的對應。

## 2. 修正篩選與確認邏輯

- [x] 2.1 修正 `sub-board.tsx` 中 `confirmByDepartmentDispatch` 的 `onlyDispatches` 篩選：
  - 當 `filterStore.has_sended` 為 false 時，選取 `!card.custbody_hf_is_car_dispatched` 的卡片。
  - 當 `filterStore.has_sended` 為 true 時，選取 `card.custbody_hf_is_car_dispatched` 的卡片。
- [x] 2.2 確認 `_checkHasSended` 的判斷式維持 `(card.has_sended ?? false) === filterStore.has_sended`。已確認無誤。
- [x] 2.3 確認 `cacheDispatchesByDepartment` 取出的卡片在「未 / 已發送」兩種模式下都正確。`cacheDispatchesByDepartment` 僅按部門與 `TEMP_CAR_NUMBER` 過濾，後續由 `newCards` 的 `_checkHasSended` 確保只保留當前 `has_sended` 狀態的卡片，邏輯正確。

- [x] 2.4 修正 `sub-board.tsx` 中 `columns` memo 的 reactivity：
  - 移除 `_getTColumn` 與 `_checkCardExists`，改為在 `createMemo` 內直接讀取 `props.board.columns`，避免 SolidJS 因 prop destructuring 無法正確追蹤 store 變更。
  - 確保切換 `has_sended` 開關時，`columns` 會重新計算並正確過濾卡片。
- [x] 2.5 修正 `setting-context.tsx` 中 `onChangeHasSended`：切換 `has_sended` 時同步清空 `store.columns`，避免舊檢視的 stale cards 在新檢視中殘留。
- [x] 2.6 修正 `auth_ref` 後端 `only_dispatches` 型別變更導致的「派車確認」更新失敗：
  - 後端 `DispatchBody.OnlyDispatches` 在 `auth_ref` 由 `[]int64` 改為 `[]string`，但前端仍傳 `number[]`，造成 JSON unmarshal 失敗。
  - 將 `models/netsuite/dispatch.ts` 的 `DispatchFilter.only_dispatches` 改為 `string[]`。
  - 在 `setting-context.tsx` 的 `confirmMutationByFilter` 將 `cards` 透過 `cards?.map((id) => id.toString())` 轉為字串後送出。

## 3. 建置與測試

- [x] 3.1 執行前端建置：
  ```bash
  cd sales-order-frontend && pnpm build
  ```
  結果：✓ built in 4.46s（含 2.4/2.5/2.6 所有修正）
- [x] 3.2 執行 TypeScript 型別檢查（若專案設定可行）：
  ```bash
  cd sales-order-frontend && pnpm exec tsc --noEmit
  ```
  結果：無法執行，`tsconfig.json` 的 `--ignoreDeprecations` 設定導致 TS5103 錯誤，屬於專案既有設定問題。
- [~] 3.3 手動驗證：
  - 登入後進入派車子板。
  - 切換「未發送」與「已發送」開關，確認卡片出現在正確車號欄位。
  - 在「未發送」下點擊「派車確認」，確認請求 body 中的 `only_dispatches` 只包含未派車卡片，且型別為 `string[]`。
  - 確認 card 36667、36668 的「派車確認」不再回傳「更新失敗」。
  - 因本機未安裝 Docker，無法啟動後端進行手動驗證。

## 4. 完成 OpenSpec change

- [x] 4.1 執行：
  ```bash
  openspec status --change fix-dispatch-sub-board-sent-filter
  ```
  確認所有 artifact 完成。

## 5. 後續觀察

- 若 card 在資料庫為 `has_sended=true` 但仍出現在「未發送」檢視，請確認 WebSocket 回傳的 payload 是否即時更新；前端已依據回傳值與 `filterStore.has_sended` 進行雙重過濾，並在切換開關時清空舊資料。
- 若「派車確認」仍失敗，請確認後端是否又調整 `DispatchBody` 欄位型別，或是否有其他 `only_dispatches` 呼叫點未同步。
