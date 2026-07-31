## ADDED Requirements

### Requirement: 釐清 auth_ref 對 metadicts 資料取得的影響

本次 change MUST 比對 `sales-order-backend` 與 `sales-order-frontend` 的 `master` 與 `auth_ref` 分支，明確指出導致 metadicts 資料取得異常的根因。

#### Scenario: 後端 diff 分析
- **WHEN** 檢視 `sales-order-backend` 的 `auth_ref` 分支與 `master` 的差異
- **THEN** 必須列出影響 `GET /api/v1/metadicts`、`DELETE /api/v1/metadicts/{id}/{table_name}`、`PUT /api/v1/metadicts/recover/{id}/{table_name}`、`GET /api/v1/metadicts/force-sync` 的關鍵改動。

#### Scenario: 前端 diff 分析
- **WHEN** 檢視 `sales-order-frontend` 的 `auth_ref` 分支與 `master` 的差異
- **THEN** 必須列出影響 `metadictsQuery`、`metadictOptionsQuery` 與相關寫入請求的關鍵改動。

---

### Requirement: 修復 metadicts 資料取得異常

本次 change MUST 根據根因假說實作修復，使 `auth_ref` 環境下 metadicts 的讀取與寫入操作恢復正常。

#### Scenario: 列表與選項正常載入
- **WHEN** 使用者登入後進入 `/admin/metadict` 與 `/admin/sales-order`
- **THEN** `GET /api/v1/metadicts` 與 `GET /api/v1/metadicts?table_names=...` 必須回傳 200 且資料完整，無 401/403。

#### Scenario: 寫入操作恢復
- **WHEN** 使用者執行 metadict 的刪除、復原或 `force-sync`
- **THEN** 這些請求必須成功，且 `DELETE`/`PUT` 請求正確帶入 `X-CSRF-Token`。

#### Scenario: 測試與手動驗證通過
- **WHEN** 執行後端測試 `go test ./internal/domain/metadicts/... ./internal/utility/nsstmt/...` 與手動驗證
- **THEN** 測試全數通過，且瀏覽器頁面下拉選項與列表顯示正常。
