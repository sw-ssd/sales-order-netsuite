## ADDED Requirements

### Requirement: 派車子板依「未 / 已發送」狀態正確篩選卡片

`SubBoard` SHALL 依據 `filterStore.has_sended` 正確篩選並定位卡片，切換開關時不得出現錯位或遺失。

#### Scenario: 切換至「未發送」顯示未發送卡片
- **WHEN** 使用者將開關切換為「未發送」(`has_sended = false`)
- **THEN** 畫面只顯示 `has_sended` 為 false 的卡片，且卡片依車號欄位正確分組。

#### Scenario: 切換至「已發送」顯示已發送卡片
- **WHEN** 使用者將開關切換為「已發送」(`has_sended = true`)
- **THEN** 畫面只顯示 `has_sended` 為 true 的卡片，且卡片依車號欄位正確分組。

---

### Requirement: 派車確認按鈕選取正確卡片集合

`confirmByDepartmentDispatch` SHALL 針對當前檢視狀態，選取符合業務語義的卡片 ID 送給後端確認。

#### Scenario: 未發送檢視下確認未派車卡片
- **WHEN** 使用者在「未發送」檢視下點擊「派車確認」
- **THEN** 只送出 `custbody_hf_is_car_dispatched` 為 false 的卡片 ID。

#### Scenario: 已發送檢視下確認已派車卡片
- **WHEN** 使用者在「已發送」檢視下點擊「派車確認」
- **THEN** 只送出 `custbody_hf_is_car_dispatched` 為 true 的卡片 ID。
