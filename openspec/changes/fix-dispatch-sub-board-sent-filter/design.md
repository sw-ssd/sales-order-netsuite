## Context

派車子板 `SubBoard` 負責依據部門、出貨日期與「未/已發送」開關顯示卡片。資料來源為 WebSocket 推播的 dispatches，經 `setting-context.tsx` 的 `_data()` 轉為 `TBoard`（columns 依車號分類）。`SubBoard` 再對 `board.columns` 進行二次篩選後渲染。

目前使用者在切換「未 / 已發送」開關時，發現卡片定位異常：切換後卡片未出現在預期欄位，或「派車確認」選取的卡片集合不正確。

## Goals / Non-Goals

**Goals:**
- 修正 `sub-board.tsx` 中「未 / 已發送」狀態與卡片篩選、確認邏輯的對應關係。
- 確保切換開關時，卡片能正確出現在對應車號欄位。
- 確保「派車確認」按鈕只針對當前檢視狀態下應處理的卡片。

**Non-Goals:**
- 不改變後端派車 API 或資料庫欄位。
- 不調整 WebSocket 連線機制與訂閱參數。
- 不更動車號分類欄位邏輯（`makeTColumns`）。

## Decisions

1. **修正 `confirmByDepartmentDispatch` 的 `onlyDispatches` 篩選條件**
   - 目前邏輯：
     ```ts
     filterStore.has_sended
       ? !card.custbody_hf_is_car_dispatched
       : card.custbody_hf_is_car_dispatched
     ```
   - 問題：在「未發送」檢視 (`has_sended=false`) 時選取已派車 (`is_car_dispatched=true`) 的卡片，與業務語義相反。
   - 修正為：
     ```ts
     filterStore.has_sended
       ? card.custbody_hf_is_car_dispatched
       : !card.custbody_hf_is_car_dispatched
     ```
   - 理由：「未發送」應處理尚未派車的卡片；「已發送」應處理已派車的卡片。

2. **保留 `sub-board.tsx` 的 client-side `has_sended` 篩選**
   - 雖然 WebSocket 已依 `has_sended` 回傳資料，但 client-side 篩選可作為防護，避免資料短暫不一致時顯示錯誤卡片。
   - 不會移除 `_checkHasSended`，但會確認其判斷式正確。

3. **使用型別強化 `has_sended` 與 `custbody_hf_is_car_dispatched` 的區別**
   - 若 `Dispatch` model 的 `has_sended` 為 optional boolean，將在條件判斷補上 `?? false`，避免 undefined 造成比對失敗。
   - 已在 `_checkHasSended` 使用 `(card.has_sended ?? false) === filterStore.has_sended`，保持不變。

4. **確保 `columns` memo 正確追蹤 `props.board` 變更**
   - 將原本抽離的 `_getTColumn()` 改為直接在 `createMemo` 內讀取 `props.board.columns`，避免 SolidJS prop destructuring 可能導致的 reactivity loss。
   - 同時移除未使用的 `_checkCardExists` 輔助函式，讓過濾邏輯更直接。
   - 如此一來，當 WebSocket 更新 board store 或使用者切換 `has_sended` 時，`columns` 都會重新計算。

5. **切換 `has_sended` 時清空 board 舊資料**
   - 在 `setting-context.tsx` 的 `onChangeHasSended` 加入 `setStore("columns", [])`。
   - 避免切換開關後，前一次 WebSocket payload 中殘留的卡片（其 `has_sended` 與目前檢視不符）仍被顯示。
   - 新的 WebSocket 連線會在 filter 更新後自動重新取得符合當前 `has_sended` 的資料。

6. **修正 `only_dispatches` 型別以配合 `auth_ref` 後端變更**
   - 背景：`auth_ref` 後端將 `DispatchBody.OnlyDispatches` 由 `[]int64` 改為 `[]string`，但前端 `DispatchFilter` 仍宣告為 `number[]`，且 `confirmMutationByFilter` 直接傳入 `number[]`。
   - 結果：後端 JSON unmarshal 失敗，導致「派車確認」回傳「更新失敗」。
   - 修正：
     - `models/netsuite/dispatch.ts`：`only_dispatches` 改為 `string[]`。
     - `setting-context.tsx`：`confirmMutationByFilter` 將 `cards` 透過 `cards?.map((id) => id.toString())` 轉為字串陣列後送出。

## Risks / Trade-offs

- [Risk] `has_sended` 與 `custbody_hf_is_car_dispatched` 的業務語義若與預期不同，修正後可能反而選錯卡片。 → Mitigation: 與需求方確認「未 / 已發送」對應的欄位；本設計已依目前最合理語義推斷。
- [Risk] 切換開關後若 WebSocket 回傳延遲，畫面可能短暫空白。 → Mitigation: 這屬於現有載入行為，不屬本次修復範圍；可後續加入 optimistic UI 或 skeleton。
- [Risk] 若 WebSocket 回傳 stale `has_sended` 資料（例如資料庫已為 true 但 payload 仍為 false），前端會依據 payload 顯示，導致卡片出現在錯誤檢視。 → Mitigation: 本次已強化 `columns` memo 的 reactivity 與 client-side 過濾；若仍出現，需檢查後端 WebSocket 資料新鮮度。
