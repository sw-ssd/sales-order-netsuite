# 系統設定整合（Settings）設計

日期：2026-08-20
狀態：待審核

## 1. 背景與目標

### 現況問題

同一批「整合常數」目前散落在三個 codebase，且數值互相衝突：

| 常數 | backend `third_party/constants/constants.go` | frontend `src/constant/options.ts` | app `system_constants.dart` |
|---|---|---|---|
| `default_department_id` | `6` | `6` | `1` ← 衝突 |
| `default_vendor_id` | `807`（seeder 實際建立 S000001） | `16` ← 衝突 | — |
| `system_department_id` / `system_salesrep_id` | `-16888` | `-16888` | `-16888` |
| `system_test_salesrep_id` | `-17888` | `-17888` | `-17888` |
| `default_salesrep_id` | `119` | `119` | — |
| `temp_car_number` | `1` | `1` | — |

衝突根源：無單一來源，各端各自硬編碼。另 frontend 已有 `/admin/setting` 空路由 + 空 `Setting.tsx`（stub）+ sidemenu 註解掉的「設定管理」項目。

### 目標

1. backend 新增 `setting` ent schema，成為整合常數（含 NetSuite / EMAIL 憑證）的**唯一來源**。
2. frontend 新增設定頁（`/admin/setting`），可檢視/編輯設定。
3. frontend 與 app 所需的整合資料皆由 backend API 取得，不再各自硬編碼（保留編譯期值作為 fallback）。
4. 憑證採 **write-only 遮罩**：GET 不回傳明文，僅 ssd superadmin 可更新。

### 已確認決策（brainstorming 過程）

- **範圍**：全部常數納入（Go block 15 個 + app 的 `companyAdminSalesrepId` / `aboutUrl` / `supportEmail` / `defaultTimeout`）+ NetSuite / EMAIL 憑證。
- **憑證處理**：存 DB，GET 遮罩（寫入後不顯示），僅 ssd superadmin 編輯，每次寫入需重新輸入完整值。
- **backend 內部**：折衷 — 憑證啟動時從 DB 讀取（env 僅首次種子），系統 ID 常數內部暫留 Go constants（種子值一致）；DB 設定 + API 提供給 frontend/app。

## 2. 資料模型

表名：`settings`。**固定單列，`id = 1`**（upsert 語意）。加入 `mixin_base`（created_at / updated_at）。不掛 tenant（全域設定）。

### 欄位清單（31 欄）

| 群組 | 欄位 | 型別 | 種子值 | 備註 |
|---|---|---|---|---|
| 系統常數 | `default_department_id` | int64 | 6 | 值修正來源（app 現為 1） |
| | `approval` | int64 | 4 | 目前僅註解程式碼使用，保留 |
| | `system_department_id` | int64 | -16888 | |
| | `system_salesrep_id` | int64 | -16888 | |
| | `system_test_salesrep_id` | int64 | -17888 | |
| | `system_customer_id` | int64 | -17888 | |
| | `system_customer_address_id` | int64 | -17888 | |
| | `system_customer_address_entity_address_id` | int64 | -17888 | |
| | `system_customer_contact_id` | int64 | -17888 | |
| | `system_customer_entity_id` | string | "SW17888" | 必填非空 |
| | `system_customer_name` | string | "系統測試客戶" | 必填非空 |
| | `default_vendor_id` | int64 | 807 | 值修正來源（frontend 現為 16） |
| | `default_vendor_name` | string | "樹森開發股份有限公司" | 必填非空 |
| | `default_salesrep_id` | int64 | 119 | |
| | `temp_car_number` | int64 | 1 | |
| App 常數 | `company_admin_salesrep_id` | int64 | -5 | app 專用 |
| | `about_url` | string | "https://www.hexagonty.com" | URL 格式驗證 |
| | `support_email` | string | "hexagon@hexagonty.com" | email 格式驗證 |
| | `default_timeout` | int | 30 | > 0 |
| 前端 URL | `frontend_url` | string | env `FRONTEND_URL`（無則空） | deeplink / manuals 用；URL 格式驗證 |
| NetSuite 憑證 🔒 | `netsuite_account_id` | string | env `NETSUITE_ACCOUNT_ID` | secret |
| | `netsuite_consumer_key` | string | env `NETSUITE_CONSUMER_KEY` | secret |
| | `netsuite_consumer_secret` | string | env `NETSUITE_CONSUMER_SECRET` | secret |
| | `netsuite_token_id` | string | env `NETSUITE_TOKEN_ID` | secret |
| | `netsuite_token_secret` | string | env `NETSUITE_TOKEN_SECRET` | secret |
| EMAIL | `email_host` | string | env `EMAIL_HOST` 預設 "smtp.gmail.com" | 非 secret |
| | `email_port` | string | env `EMAIL_PORT` 預設 "587" | 非 secret；PUT 時驗證為數字 |
| | `email_identity` | string | env `EMAIL_IDENTITY` 預設 "" | 非 secret |
| | `email_username` | string | env `EMAIL_USERNAME` | 非 secret |
| | `email_password` 🔒 | string | env `EMAIL_PASSWORD` | secret |
| | `email_from` | string | env `EMAIL_FROM` | 非 secret；email 格式驗證 |

**secret 欄位集合**：`netsuite_account_id`、`netsuite_consumer_key`、`netsuite_consumer_secret`、`netsuite_token_id`、`netsuite_token_secret`、`email_password`（6 欄）。

**種子規則**（首次啟動，行不存在時）：非 secret 欄位用上表種子值（schema default 即為 Go constants 現值）；secret 欄位由 env 覆寫；若 `settings` 列已存在，**不覆寫**任何值。v1 不實作 secret at-rest 加密（見 §9 風險）。

## 3. Backend

### 3.1 Ent schema 與遷移

- 新檔 `ent/schema/setting.go`（見 §2 欄位；`id` 固定 1，含 `BaseMixin`）。
- 執行 `task db:migrate -- add_settings_table` 產生 goose migration；migration 內 `INSERT` id=1 列（非 secret 種子值，secret 留空由啟動種子補）。
- `task ent:gen` 產生 `ent/gen/setting*`。

### 3.2 新 domain `internal/domain/settings/`

依既有 domain 慣例（departments 為範本）：

- `model.go`：GET 回應 DTO（含遮罩後的 secret）、PUT request DTO（`*string` / `*int64` 指針語意）、`filter.go` 不需要（單列）。
- `repository.go`：`Get()` 讀 id=1；`Upsert()` 更新 id=1（不存在則插入）。
- `usecase.go`：遮罩邏輯、secret 更新權限檢查、驗證。
- `transformation.go`：ent model ↔ DTO。
- `handler.go`：`GET /`、`PUT /`。
- `register.go`：`router.Route("/api/v1/settings", ...)`，`r.Use(middleware.Authenticate(session))`。

### 3.3 API 契約

**`GET /api/v1/settings`**（登入即可）

（欄位省略示範遮罩格式，完整欄位見 §2）

```json
{
  "default_department_id": 6,
  "approval": 4,
  "...": "...",
  "frontend_url": "https://frontend.hexagonty.com",
  "netsuite_account_id": "••••SB2",
  "netsuite_consumer_key": "••••25f7",
  "netsuite_consumer_secret": "••••47dc",
  "netsuite_token_id": "••••7d1a",
  "netsuite_token_secret": "••••4ca4",
  "email_host": "smtp.gmail.com",
  "email_port": "587",
  "email_identity": "",
  "email_username": "hexagon@hexagonty.com",
  "email_password": "••••gcnj",
  "email_from": "hexagon@hexagonty.com"
}
```

- **遮罩規則**：secret 欄位回傳末 4 字元，前綴 `••••`；值為空則回傳空字串。

**`PUT /api/v1/settings`**（upsert 單列）

- 非 secret 欄位：**全量更新** — body 須含所有非 secret 欄位，直接以 body 值覆寫（必填欄位驗證：`system_customer_entity_id`、`system_customer_name`、`default_vendor_name`、`about_url`、`support_email`、`email_from` 非空；`about_url`/`frontend_url` 為 URL；`support_email`/`email_from`/`email_username` 為 email；`email_port` 為 1–65535 數字字串；`default_timeout` > 0）。
- secret 欄位：body 傳 `null` 或省略 = 保持不變；傳完整字串 = 更新（重新輸入語意，GET 遮罩值不可回寫）。

### 3.4 權限

| 動作 | 條件 |
|---|---|
| GET | 已登入（`middleware.Authenticate`） |
| PUT（僅非 secret 欄位） | admin / superadmin 角色（casbin） |
| PUT（含任一 secret 欄位） | 僅 superadmin（ssd，`is_manager` 或 superadmin 角色，依 handler 內既有角色檢查慣例，如 apikeys `requireAPIKeyManager`） |

- casbin policy 新增 `settings` resource 的 read/write 規則（superadmin、admin 角色）。
- secret 權限在 handler/usecase 層檢查：PUT body 含 secret 欄位且目前使用者非 superadmin → 403。

### 3.5 啟動整合

`internal/server/server.go` `newDatabase()` 中，於既有 `c.Seeder(...)` 之後：

1. 檢查 `settings` 列是否存在；不存在則以 §2 種子規則建立（env 優先 → 常數預設）。
2. 讀取設定存入 `s.settings`（`settings.Setting` 值型別）。

`internal/server/initDomains.go`：

- `newNSClient(cfg config.NetSuite)` → 改為接收 `s.settings` 的 netsuite 五欄，建構 `netsuite.NewNetsuiteClient` / `NewSuiteQLClient`。
- mail 建構（現 `s.cfg.Email.*`）→ 改用 `s.settings` 的 email 六欄。
- `config.New()` 仍載入 env（作為種子來源），`config/email.go`、`config/netsuite.go` 保留不刪。

### 3.6 tygo

`tygo.yaml` 新增：

```yaml
- path: "github.com/hexagon-maker/sales-order-backend/internal/domain/settings"
  type_mappings:
    time.Time: "string /* RFC3339 */"
  output_path: "frontend_types/settings.ts"
  include_files:
    - "model.go"
```

## 4. Frontend（SolidJS）

### 4.1 資料層

- `src/models/settings.ts`：手寫介面，欄位對照 backend tygo 產出 `frontend_types/settings.ts`（依既有慣例，如 `Department` 對照 `frontend_types/departments.ts`），並由 `src/models/index.ts` 匯出。
- `src/lib/setting/setting.ts`：request builder（`getSettings`、`putSettings`）。
- `src/lib/setting/index.ts`：`settingsQuery()`（TanStack Query `queryOptions`）。
- `src/constant/api.ts`：新增 `getSettings`、`putSettings` 路徑（`apiVersion + "/settings"`）。

### 4.2 設定頁

`src/pages/admin/setting/Setting.tsx` 由空 stub 改為分組表單：

- 分組：系統常數 / App 設定 / 前端 URL / NetSuite 憑證 🔒 / EMAIL 設定（含憑證 🔒）。
- secret 欄位：遮罩值顯示（如 `••••25f7`）、輸入框留空 = 不變、填入完整值 = 更新；非 superadmin 使用者 secret 區塊整個唯讀（disabled + 提示）。
- 儲存：PUT；成功 toast + 重取 query；失敗顯示錯誤（含 403 提示）。
- superadmin 判斷沿用 auth context（`authState.info` 角色 / 現有 `isAppAdmin` 慣例）。
- 路由 `src/routes/admin/setting.tsx` 已存在，loader 加 `ensureQueryData(settingsQuery())`。

### 4.3 sidemenu

`src/constant/sidemenu.ts` `systemMenus` 取消註解「設定管理」項目（icon `RiSystemSettingsFill`，需 import 恢復）。

### 4.4 `options.ts` 遷移（11 個使用點）

`src/constant/options.ts` 同步常數改為執行期資料。新增 `useSettings()`（包裝 `settingsQuery`，loading 期間回傳編譯期 fallback 值）。使用點：

| 檔案 | 現用常數 | 改為 |
|---|---|---|
| `components/datatable/DepartmentFilter.tsx` | `SYSTEM_ADMIN_DEPARTMENT` | `useSettings().system_department_id` |
| `pages/admin/customer/widgets/qrcode-form.tsx` | `DEEPLINK_COMPANY_URL` | `${useSettings().frontend_url}/customer_account_qrcode` |
| `pages/admin/dispatch/widgets/boards.tsx` | `DEFAULT_DEPARTMENT` | `useSettings().default_department_id` |
| `pages/admin/dispatch/widgets/card.tsx` | `TEMP_CAR_NUMBER` | `useSettings().temp_car_number` |
| `pages/admin/dispatch/widgets/column.tsx` | `TEMP_CAR_NUMBER` | 同上 |
| `pages/admin/dispatch/widgets/setting-context.tsx` | `SYSTEM_ADMIN_DEPARTMENT`, `TEMP_CAR_NUMBER` | 同上對應 |
| `pages/admin/dispatch/widgets/sub-board.tsx` | `DEFAULT_DEPARTMENT` | `useSettings().default_department_id` |
| `pages/admin/sales-order/widgets/create-vendorreturn-form.tsx` | `DEFAULT_VENDOR_ID` | `useSettings().default_vendor_id` |
| `pages/admin/sales-order/widgets/form/main-from.tsx` | `SYSTEM_ADMIN_DEPARTMENT`, `SYSTEM_ADMIN_SALESREP_ID`, `SYSTEM_TEST_SALESREP_ID` | 對應欄位 |
| `pages/admin/sales-order/widgets/form/vendor-main-from.tsx` | `DEFAULT_VENDOR_ID`, `DEFAULT_VENDOR_NAME`, `SYSTEM_ADMIN_*` | 對應欄位 |
| `pages/auth/context.tsx` | `SYSTEM_ADMIN_SALESREP_ID` | `useSettings().system_salesrep_id` |

`options.ts` 保留僅作 fallback 預設（改由 `src/constant/` 或遷移後內聯），`DEEPLINK_COMPANY_URL` 由 `settings.frontend_url` 取代。

## 5. App（Flutter）

### 5.1 資料層

- 新 model `lib/layer_data/models/setting/setting.dart`（freezed + json_serializable，欄位對應 §2，secret 欄位 app 不使用仍解析為可空欄位）。
- 新 API `lib/layer_business/network/api/settings_api.dart`（僅 `get()`，`CacheOptionsMixin` + `lgCache`）+ 抽象介面 `network/abstract/settings_api_type.dart` + `Endpoints.getSettings = "$_apiVersion/settings"`。
- provider 註冊：`lib/layer_business/services/settings/provider.dart`（disco `Provider`）。

### 5.2 SettingsService

`lib/layer_business/services/settings/`：

- 登入成功後 fetch；結果存 Sembast `settings_store`（新 store，沿用 `sembast_kv_storage`）。
- 對外以 `Signal<SettingsModel?>` 提供；fetch 失敗時回退：Sembast 快取 → 編譯期 fallback（`SystemConstants` 保留為 fallback 類別，數值更新為種子值）。
- 每次 app 啟動/登入時嘗試 refresh；離線可用快取。

### 5.3 使用點遷移（9 處）

| 檔案 | 現用常數 | 改為 |
|---|---|---|
| `services/customer/customer_service.dart` | `systemDepartmentId`, `deeplinkCompanyLink` | service 讀取；deeplink 用 `frontend_url` |
| `services/profile/profile_service.dart` | `aboutUrl` | `about_url` |
| `services/salesorder/salesorder_service.dart` | `systemSalesrepId`, `defaultTestSalesrepId`, `companyAdminSalesrepId` | 對應欄位 |
| `stories/admin/customer/customer_create_screen.dart` | `systemSalesrepId` | 對應欄位 |
| `stories/admin/customer/form/main_edit_form.dart` / `main_form.dart` | `systemSalesrepId` | 對應欄位 |
| `stories/admin/salesorder_item/salesorder_item_screen.dart` | `systemSalesrepId`, `companyAdminSalesrepId`, `systemDepartmentId`, `defaultDepartment` | 對應欄位 |
| `stories/admin/tabs/profile/widgets/profile_list.dart` | `systemSalesrepId` | 對應欄位 |
| `stories/admin/tabs/salesorder/form/salesorder_cards.dart` | `systemSalesrepId`, `defaultTestSalesrepId` | 對應欄位 |

- `SystemConstants.defaultDepartment` 改 6；`deeplinkCompanyLink` 改由 `frontend_url` 組裝。
- app 不需要 secret 欄位（GET 已遮罩），僅取非 secret 值。

## 6. 資料流

```
啟動（backend）
  config.New(env) → DB migrate → seeder → settings seed(env+constants, 列不存在才建)
  → load settings → s.settings → newNSClient / mail 以 DB 值建構

執行期
  GET /api/v1/settings（遮罩）─┬→ frontend settingsQuery → 設定頁表單
                               └→ app 登入後 fetch → Sembast cache → Signal
  PUT /api/v1/settings（upsert，secret null=不變）→ 前端重取
```

## 7. 錯誤處理與驗證

| 情境 | 處理 |
|---|---|
| GET/PUT 未登入 | 401 → 前端跳登入 |
| PUT 含 secret 非 superadmin | 403 → 前端 secret 區塊唯讀提示 |
| PUT 驗證失敗（格式/必填） | 422 → 前端欄位錯誤顯示 |
| app fetch 失敗 | 回退 Sembast 快取 → 編譯期 fallback；下次啟動重試 |
| secret 欄位遮罩回寫 | PUT 收到 GET 遮罩格式值（含 `••••`）→ 400 拒絕（避免誤存遮罩字串） |
| env 種子缺值（secret 空） | 列仍建立、欄位空；NetSuite/EMAIL 功能以空值處理（現行行為：空憑證即無法呼叫） |

## 8. 測試

| 層 | 範圍 |
|---|---|
| backend 單元 | 遮罩規則（末 4 碼 / 空值）、PUT 語意（null 不變、遮罩值拒絕）、種子規則（存在不覆寫、缺列建立）、驗證規則、secret 權限 403 |
| backend 整合 | dockertest：GET 遮罩、PUT upsert、啟動種子 |
| frontend | vitest：遮罩顯示、secret 欄位 null 語意表單提交、非 superadmin 唯讀 |
| app | flutter test：SettingsService fallback（fetch 失敗 → cache → 編譯期值）、解析 |

遵循既有測試慣例（backend `*_test.go` + testify；frontend `*.test.tsx`；app 目前無 test/ 目錄，新增以 `fvm flutter test` 執行）。

## 9. 風險與決定

| 項目 | 決定 | 理由 |
|---|---|---|
| 憑證存 DB（明文） | 接受；GET 遮罩 + superadmin 限定 | 使用者已確認「仍比 env 差，但可接受」；v1 不做 at-rest 加密，列為後續 hardening（AES-GCM + `SETTINGS_SECRET_KEY` env） |
| 值衝突 | 種子以 backend 現值為準（vendor 807、department 6），frontend/app 修正 | backend 值為 seeder/transformation 實際使用 |
| backend 內部常數 | 暫留 Go constants | 折衷方案；DB 僅憑證與 API 層。後續可逐域遷移 |
| `Approval`（=4） | 納入設定但無行為影響 | 僅註解程式碼使用；保留完整性 |
| `frontend_url` 種子 | env `FRONTEND_URL`（新增 backend env）；未設則由設定頁補 | frontend/app 的 deeplink/manuals 改由此值 |
| 稽核紀錄 | v1 不做 | 範圍外 |

## 10. 實施順序

1. backend：`ent/schema/setting.go` → `task ent:gen` → `task db:migrate -- add_settings_table` → `internal/domain/settings/` → register 路由 → 啟動整合（server.go / initDomains.go）→ tygo → 測試。
2. frontend：models + `src/lib/setting/` → api.ts → 設定頁 + sidemenu → `options.ts` 使用點遷移 → vitest。
3. app：model + settingsApi + SettingsService → `SystemConstants` 使用點遷移 → flutter test。
4. 文件：更新 `docs/AGENTS/backend.md` / `frontend.md` / `app.md`（常數改為 DB 驅動）、changelog。
