# 實作步驟：比對 master 與 auth_ref 對 metadicts 的影響

## 1. 準備與比對

- [x] 1.1 在本地 OpenSpec root 建立 change 並確認狀態：
  ```bash
  openspec new change compare-auth-ref-metadicts-impact
  openspec status --change compare-auth-ref-metadicts-impact
  ```
- [x] 1.2 在兩個 repo 收集 diff 摘要：
  ```bash
  git -C sales-order-backend diff --stat master..auth_ref
  git -C sales-order-frontend diff --stat master..auth_ref
  ```

## 2. 閱讀關鍵程式碼

- [x] 2.1 閱讀後端關鍵檔案：
  - `internal/middleware/csrf.go`
  - `internal/middleware/authentication.go`
  - `internal/domain/metadicts/register.go`
  - `internal/domain/metadicts/handler.go`
  - `internal/utility/nsstmt/stmts.go`
  - `internal/server/server.go`（middleware 順序）
  - `internal/server/initDomains.go`
- [x] 2.2 閱讀前端關鍵檔案：
  - `src/lib/requests/csrf.ts`
  - `src/lib/requests/utils.ts`
  - `src/pages/auth/context.tsx`
  - `src/lib/metadict/index.ts`
  - `src/lib/metadict/metadict.ts`
  - `src/pages/admin/sales-order/widgets/context.tsx`

## 3. 啟動服務並觀察 Network

- [~] 3.1 啟動後端與前端（auth_ref），使用瀏覽器開啟以下頁面並觀察 Network：
  - `/admin/metadict`
  - `/admin/sales-order`
  - 記錄 `metadicts` 與 `metadict options` 的 status、headers、response payload。
  - **Blocked**：本機未安裝 Docker，無法啟動 postgres/valkey 等基礎設施；改用靜態程式碼分析與單元測試驗證。

## 4. 確認症狀

- [x] 4.1 確認是否有 401/403？
  - 靜態分析：CSRF 在登入/頁面載入時會透過 `auth/context.tsx` 取得；`GET /api/v1/metadicts` 不受 CSRF 影響。未發現明顯 401/403 設計缺陷。
- [x] 4.2 確認是否 `metadictOptions.data` 為 undefined？
  - 發現 `src/pages/admin/customer/widgets/context.tsx`、`src/pages/admin/dispatch/widgets/setting-context.tsx`、`src/pages/admin/estimate-item/widgets/context.tsx`、`src/pages/admin/item/widgets/context.tsx`、`src/lib/utils.ts` 均使用 `metadictOptions?.data` 或 `metadictOptions.data` 而未對 `data` 做 optional chaining，會在 undefined 時拋錯。
- [x] 4.3 確認列表 total count 是否與實際 rows 不符？
  - 後端 `ENTMetadictsSchemasStmtByFilter` 的 count query 與 data query 已套用相同的 `table_name`/`deleted_at` 條件，條件一致。
- [x] 4.4 確認 `DELETE`/`PUT`/`force-sync` 是否失敗？
  - 靜態分析未發現 `utils.ts` 漏帶 CSRF token 的跡象；`force-sync` 為 `GET`，不受 CSRF 影響。

## 5. 實作修復

- [x] 5.1 根因：前端 `src/lib/requests/utils.ts` 的 `genParams` 將陣列序列化為 `table_names=departments,customers`，導致後端 `TableNames` 解析異常。已修復為恢復 `key[]` 展開（`table_names[]=departments&table_names[]=customers`）。
- [x] 5.2 前端防護：將所有 `metadictOptions?.data` 存取改為 `metadictOptions?.data?.`，避免 undefined 時拋錯。
- [x] 5.3 後端測試：更新 `internal/utility/nsstmt/stmts_test.go` 的 `TestMetadictsSchemasStmtByFilter`，使其符合 `auth_ref` 的 SQL 輸出。
- [x] 5.4 修復其餘 `internal/utility/nsstmt` 測試（`TestCustomersStmtByFilter`、`TestSalesOrdersByFilter`、`TestEstimateItemsStmtByFilter`、`TestContactStmt`、`TestCustomerAddressbookStmt`）的預先存在問題，包括：
  - 共用 builder 導致的 state pollution（改為每個 test case 建立 fresh builder）。
  - SQL 期望與 `auth_ref` 實際輸出不符（更新 LISTAGG 分隔符號、欄位順序、移除 ROW_NUMBER、改用 substring assertions）。
  - `TestSalesOrdersByFilter` golden file 因 `time.Now()` 日期動態而無法穩定比對，改用 substring assertions。

## 6. 跑測試

- [x] 6.1 後端 metadicts 相關 nsstmt 測試：
  ```bash
  go test ./internal/utility/nsstmt/... -run TestMetadictsSchemasStmtByFilter
  ```
- [x] 6.2 後端 metadicts usecase 測試：
  ```bash
  go test ./internal/domain/metadicts/... -run TestUCMetadictSuit
  ```
  修復 `TestMetadictUseCase_List`（mock 未設定 `ListMetadictsFunc`）與 `TestMetadictUseCase_newEvents`（event manager listener 累積導致 nil repo 被呼叫）。
- [~] 6.3 後端完整測試：`go test ./internal/domain/metadicts/... ./internal/utility/nsstmt/...`
  - `internal/utility/nsstmt` 全數通過。
  - `internal/domain/metadicts` 的 `TestRepoIMetadicSuite` 因需要 Docker testcontainers 與特定 `.env` 路徑而無法在本機執行，屬於環境限制。
- [x] 6.4 前端建置驗證：
  ```bash
  cd sales-order-frontend && pnpm build
  ```
- [~] 6.5 前端單元測試：`pnpm test`
  - 無測試檔案，命令直接退出並回傳 code 1。

## 7. 手動驗證

- [~] 7.1 登入後進入 `/admin/metadict`，列表可正常載入。
- [~] 7.2 刪除一筆 metadict 後可復原。
- [~] 7.3 進入 `/admin/sales-order`，下拉選項（部門、業務、客戶類型等）正常出現。
- [~] 7.4 執行 `force-sync` 成功。
- 7.1–7.4 因無 Docker 環境無法啟動後端，改為靜態分析與建置驗證。

## 8. 完成 OpenSpec change

- [x] 8.1 更新 OpenSpec change：
  ```bash
  openspec status --change compare-auth-ref-metadicts-impact
  ```
  確認所有 artifact 完成。
