# 設計：比對 master 與 auth_ref 對 metadicts 的影響

## 比對範圍

- 後端分支：`sales-order-backend/master` vs `sales-order-backend/auth_ref`
- 前端分支：`sales-order-frontend/master` vs `sales-order-frontend/auth_ref`

## 後端關鍵變更

1. `internal/middleware/csrf.go`（新增）
   - 只對 `POST/PUT/PATCH/DELETE` 檢查 `X-CSRF-Token`。
   - `GET /api/v1/metadicts` 與 `GET /api/v1/metadicts/force-sync` 不受 CSRF 影響。
   - `DELETE /api/v1/metadicts/{id}/{table_name}` 與 `PUT /api/v1/metadicts/recover/{id}/{table_name}` 必須帶有效的 `X-CSRF-Token`。

2. `internal/middleware/authentication.go`
   - 改為中文錯誤訊息，邏輯不變。
   - 加入 sliding session：剩餘壽命 < 24h 時自動延長。
   - 銷毀 session 時 cookie 過期時間從 `time.Time{}` 改為 `time.Unix(0,0)`。

3. `internal/domain/metadicts/register.go`
   - 所有 `/api/v1/metadicts` 路由都套用 `middleware.Authenticate(session)`。
   - 未使用 `Authorize`（未來可視權限需求調整）。

4. `internal/utility/nsstmt/stmts.go`
   - `ENTMetadictsSchemasStmtByFilter` 的 count query 改為套用與 data query 相同的 `table_name` 與 `deleted_at` 條件。
   - 可能修正 master 上 count 與實際回傳筆數不一致的問題，但也可能改變分頁顯示。

5. `internal/server/initDomains.go` / `internal/server/server.go`
   - 新增 `apikeys` domain。
   - 全域 middleware 順序：`CORS` → `JSON` → `PublicCache` → `LoadAndSave` → `Audit` → `ApiKeyMiddleware` → `CSRFMiddleware`。
   - API key 請求會跳過 CSRF，但一般 session 請求仍須符合 CSRF。

## 前端關鍵變更

1. `src/lib/requests/csrf.ts`（新增）
   - 透過 `GET /api/v1/restricted/csrf` 取得 token，存在記憶體。
   - 登出或 401 時清除。

2. `src/lib/requests/utils.ts`
   - 寫入請求（POST/PUT/PATCH/DELETE）自動帶入 `X-CSRF-Token`。
   - 收到 401 時清除 CSRF token 與 `localStorage.auth_state`，並導回 `/signin`。
   - URL 產生方式從 `new URL(endpoint, VITE_API_URL)` 改為 `new URL(VITE_API_URL + endpoint)`，對絕對路徑結果相同，但需確認 `VITE_API_URL` 是否含尾斜線。

3. `src/pages/auth/context.tsx`
   - 登入成功後與頁面載入時（若已登入）會呼叫 `fetchCsrfToken()`。
   - 若 `fetchCsrfToken()` 失敗，後續寫入操作會因缺少 token 被後端拒絕。

4. `src/lib/metadict/index.ts`
   - `metadictsQuery` 與 `metadictOptionsQuery` 都是 GET，不帶 CSRF header。
   - 若後端回傳 401，會觸發 `utils.ts` 的全域導回登入。

5. `src/pages/admin/sales-order/widgets/context.tsx`
   - 將 `props.metadictOptions.data.forEach` 改為 `props.metadictOptions?.data?.forEach`，顯示在 `auth_ref` 上 `metadictOptions` 可能為 undefined。

## 根因假說

- 假說 A：前端登入後未成功取得 CSRF token，導致 metadicts 的 `DELETE`/`PUT`/`force-sync` 請求收到 403，寫入操作無法完成。
- 假說 B：後端 `nsstmt` count query 條件改變，導致列表分頁 total 與實際資料不符，看起來像是「取得資料不完整」。
- 假說 C：session/cookie 調整導致 `GET /api/v1/metadicts` 偶發 401，被前端導回登入，頁面無法載入 metadict options。
- 假說 D：前端傳入 filter 時 `table_names` 或 `is_options` 參數格式因 `utils.ts` 改變而異常，導致後端解析不到條件。

## 確認後的根因

經由 diff 比對，發現 `sales-order-frontend/src/lib/requests/utils.ts` 的 `genParams` 在 `auth_ref` 中改變了陣列參數的序列化方式：

- `master`：陣列會展開為 `table_names[]=departments&table_names[]=customers`。
- `auth_ref`：陣列被轉為 `String(table_names)`，變成 `table_names=departments,customers`。

後端 `Filter` 的 `TableNames []string` 使用 `in:"query=table_names[],table_names"` 解析，雖然同時註冊了 `table_names` 名稱，但單一值 `departments,customers` 通常不會被自動拆分為 slice，導致 `table_names` 解析為單一元素 `[]string{"departments,customers"}` 或解析失敗，進而使 `metadictOptionsQuery` 無法取得指定表格的 options。

此外，auth_ref 環境下 `metadictOptions` 可能為 undefined，多個 widget 使用 `props.metadictOptions?.data.forEach` 存取 `data` 時會觸發 runtime exception，導致頁面掛掉。

## 修復方向

1. 前端 `src/lib/requests/utils.ts`：恢復陣列參數的 `key[]` 展開序列化，保留 scalar 與 null/undefined 的處理。
2. 前端各 widget：將 `metadictOptions?.data` 存取改為 `metadictOptions?.data?.`，避免 undefined 時拋錯。
3. 後端 `internal/utility/nsstmt/stmts_test.go`：更新 `TestMetadictsSchemasStmtByFilter` 的期望，使其符合 `auth_ref` 的 SQL 輸出。
4. 其他 nsstmt 測試（`TestCustomersStmtByFilter`、`TestSalesOrdersByFilter` 等）在 `master` 與 `auth_ref` 上均有預先存在的失敗，不屬於本次 metadicts 根因，需另行整理。

## 驗證方式

1. 同時啟動後端與前端在 `auth_ref`。
2. 開啟瀏覽器 DevTools Network，進入 `/admin/metadict` 與 `/admin/sales-order`。
3. 觀察 `GET /api/v1/metadicts`、`GET /api/v1/metadicts?table_names[]=...`、`DELETE /api/v1/metadicts/...`、`PUT /api/v1/metadicts/recover/...` 的 status 與 response。
4. 比較 `master` 與 `auth_ref` 上的請求與回應差異。
5. 執行後端測試：`go test ./internal/domain/metadicts/...` 與 `go test ./internal/utility/nsstmt/... -run TestMetadictsSchemasStmtByFilter`。
6. 手動驗證：登入後 `/admin/sales-order` 的下拉選項正常出現；`/admin/metadict` 列表與刪除/復原/force-sync 操作正常。
