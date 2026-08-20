# Settings Field-Based Schema 設計（改版）

日期：2026-08-20
狀態：待審核
前置：`2026-08-20-settings-constants-integration-design.md`（現行單列 31 欄實作，本 spec 為其 schema 改版）

## 1. 背景與目標

### 現況問題

現行 `settings` 表為單列 31 個型別欄位（id=1），欄位中繼資料（label/type/secret）硬編碼在前端 `Setting.tsx` 的 GROUPS/FieldDef；backend 無欄位名稱/說明/型別來源。新增或調整欄位需同時改 schema + migration + backend DTO + frontend GROUPS + app model，五處同步。

### 目標

將 `settings` schema 改為 **field 導向**：每列一個設定欄位，含中繼資料與值。

```
id | field_id | name | field_type | desc | value
```

- 欄位定義（field_id/name/field_type/desc/預設值）由 backend 單一來源（固定 registry）
- frontend 頁面 data-driven：名稱/型別/說明來自 API；分組為前端靜態對應
- API 回傳陣列；secret 遮罩；批次 PUT
- superadmin 判定改為 **email == `ssd@sowinsoft.com`**（`constants.SuperAdminEmail`）

### 已確認決策（brainstorming 過程）

| 題目 | 決定 |
|---|---|
| 值存放 | 同表 `value` 欄（每列 = 一個設定） |
| PUT 語意 | 批次 upsert（GET 全部、PUT 全部、secret null=不變） |
| 分組 | 前端靜態 `群組 → field_id[]`（5 群組） |
| field_type | `int64 / string / secret / bool` 四種 |
| 實作方式 | A1 固定 Registry（編譯期常數） |
| superadmin | email == `ssd@sowinsoft.com`（取代 role name 判定） |

## 2. 資料模型

表名維持 `settings`（內容改版）。舊單列 31 欄表 **drop**；新表：

| 欄位 | 型別 | 說明 |
|---|---|---|
| `id` | bigint PK（BaseMixin） | |
| `field_id` | varchar **unique** | 欄位識別字，如 `default_department_id` |
| `name` | varchar | 顯示名稱（繁中） |
| `field_type` | varchar | `int64 / string / secret / bool` |
| `desc` | varchar | 說明文字（繁中） |
| `value` | varchar | 值一律以字串儲存；int64/bool 於 API 邊界轉換 |

`created_at` / `updated_at`（BaseMixin）。不掛 tenant。migration 僅建表、**不插列**（seed 職責在啟動邏輯，維持 seed-once 語意）。

## 3. Field Registry（31 欄）

`internal/domain/settings/fields.go` 定義 `FieldRegistry`（Go slice，固定）：

| field_id | name | type | desc | 預設 |
|---|---|---|---|---|
| default_department_id | 預設部門 ID | int64 | NetSuite 預設部門 | 6 |
| approval | 核准狀態 | int64 | 訂單核准狀態值 | 4 |
| system_department_id | 系統管理員部門 ID | int64 | 系統部門（排除於篩選） | -16888 |
| system_salesrep_id | 系統管理員業務 ID | int64 | 系統業務（排除於下拉） | -16888 |
| system_test_salesrep_id | 系統測試業務 ID | int64 | 測試業務（排除於下拉） | -17888 |
| system_customer_id | 系統測試客戶 ID | int64 | 種子客戶 | -17888 |
| system_customer_address_id | 系統客戶地址 ID | int64 | 種子地址 | -17888 |
| system_customer_address_entity_address_id | 系統客戶實體地址 ID | int64 | 種子實體地址 | -17888 |
| system_customer_contact_id | 系統客戶聯絡人 ID | int64 | 種子聯絡人 | -17888 |
| system_customer_entity_id | 系統客戶實體 ID | string | NetSuite Entity ID | SW17888 |
| system_customer_name | 系統測試客戶名稱 | string | | 系統測試客戶 |
| default_vendor_id | 預設供應商 ID | int64 | 供應商客戶 | 807 |
| default_vendor_name | 預設供應商名稱 | string | | 樹森開發股份有限公司 |
| default_salesrep_id | 預設業務 ID | int64 | 系統預設業務員 | 119 |
| temp_car_number | 暫存車牌號碼 | int64 | 未分配車牌之派車 | 1 |
| company_admin_salesrep_id | 公司管理員業務 ID | int64 | app 表單代換 | -5 |
| about_url | 關於我們 URL | string | app profile 用 | https://www.hexagonty.com |
| support_email | 客服 Email | string | | hexagon@hexagonty.com |
| default_timeout | 預設逾時（秒） | int64 | | 30 |
| frontend_url | 前端站台 URL | string | deeplink/manuals 用 | env `FRONTEND_URL` |
| netsuite_account_id | Account ID | secret | | env `NETSUITE_ACCOUNT_ID` |
| netsuite_consumer_key | Consumer Key | secret | | env `NETSUITE_CONSUMER_KEY` |
| netsuite_consumer_secret | Consumer Secret | secret | | env `NETSUITE_CONSUMER_SECRET` |
| netsuite_token_id | Token ID | secret | | env `NETSUITE_TOKEN_ID` |
| netsuite_token_secret | Token Secret | secret | | env `NETSUITE_TOKEN_SECRET` |
| email_host | SMTP Host | string | | smtp.gmail.com |
| email_port | SMTP Port | string | 字串儲存（config.Email.Port） | 587 |
| email_identity | SMTP Identity | string | | "" |
| email_username | SMTP Username | string | | env `EMAIL_USERNAME` |
| email_password | SMTP Password | secret | | env `EMAIL_PASSWORD` |
| email_from | 寄件人 Email | string | | env `EMAIL_FROM` |

統計：14 int64、11 string、6 secret、0 bool（bool 型別備用）。`secret` 為 string 子型別，觸發遮罩 + superadmin 權限。name/desc 繁中沿用現行前端 label。

## 4. Backend

### 4.1 結構

- `ent/schema/setting.go`：改為上述 6 欄（`field_id` unique index）。
- `internal/domain/settings/fields.go`：`FieldRegistry`（§3 表格；含 `Default` 與 `Secret bool` 輔助）。
- `seed.go`：表空 → 依 registry 插入 31 列；secret / frontend_url 值以 env 覆寫；表已有列 → 不重灌（seed-once）。
- `repository.go`：`List() ([]*gen.Setting, error)`、`BatchUpsert(rows []*SettingDTO) error`（upsert by `field_id`，`OnConflictColumns(setting.FieldID).UpdateNewValues()`）。
- `transformation.go`：ent ↔ DTO；`ToNetSuiteConfig` / `ToEmailConfig` 改為 by field_id 組裝（行為不變）。
- `handler.go` / `register.go`：路由不變（`/api/v1/settings` GET/PUT，Authenticate）。

### 4.2 API 契約

**GET /api/v1/settings**（登入即可）→ 200

```json
{ "fields": [
  { "field_id": "default_department_id", "name": "預設部門 ID", "field_type": "int64", "desc": "...", "value": "6" },
  { "field_id": "netsuite_consumer_key", "name": "Consumer Key", "field_type": "secret", "desc": "...", "value": "••••25f7" }
] }
```

- 遮罩：`field_type == "secret"` 且 value 非空 → `••••末4碼`；空值 → `""`。
- 排序：依 registry 順序。

**PUT /api/v1/settings**（批次 upsert）→ 200

- body：`{ "fields": [ { "field_id", "name", "field_type", "desc", "value" } ] }` — **全欄位物件**（與 GET 回傳同形狀，前端直接 round-trip）。
- 語意：
  - 非 secret 欄位：`value` 字串直接更新（int64/bool 依型別驗證）。
  - secret 欄位：`value == null` = 不變；非 null = 更新（完整值）；含 `••••` 前綴 → 400。
  - 未知 `field_id`：忽略（或 400，依實作；spec 採忽略 + 不影響其餘欄位）。
- 權限（**email 判定**）：
  - PUT 全部：需要 admin+（session `ui.User.BindNames.RoleName ∈ constants.EntAdminRole`）→ 否則 403。
  - PUT 含任一 secret 欄位更新（非 null value）：目前使用者 email 須為 `constants.SuperAdminEmail`（`ssd@sowinsoft.com`）→ 否則 403。
  - 解析：`pe.GetContextToSessionInfo(ctx, session)` → `ui.User.Email` / `ui.User.BindNames.RoleName`。

### 4.3 驗證

型別層級（`field_type` 驅動）：
- `int64`：`strconv.ParseInt` 通過。
- `bool`：`"true"/"false"`。
- `string`：格式不限（除下方特殊規則）。
- `secret`：可空（空 = 清除）；非空即更新。

必填集合（field_id → 非空，屬特殊規則 map）：`system_customer_entity_id`、`system_customer_name`、`default_vendor_name`、`about_url`、`support_email`、`email_from`。

已知欄位特殊規則（field_id → rule map，`validation.go`）：
- `about_url`、`frontend_url`：URL 格式（可為空）。
- `support_email`、`email_from`、`email_username`：email 格式（可為空）。
- `email_port`：數字 1–65535。
- `default_timeout`：> 0。

錯誤碼：401（未登入）、403（非 admin / secret 非 superadmin）、400（遮罩值）、422（型別/格式驗證）、500（DB）。

### 4.4 啟動整合

- `server.go newDatabase()`：`settings.Seed(ctx, cli, ns, email, frontendURL)`（表空才建）→ 讀 rows → `s.cfg.NetSuite` / `s.cfg.Email` 由 `ToNetSuiteConfig` / `ToEmailConfig` 覆寫（行為與現行一致）。
- tygo：`SettingFieldDTO` → `frontend_types/settings.ts`（gitignored 慣例同前，不入版控）。

## 5. Frontend

### 5.1 資料層

- `src/models/settings.ts`：`SettingsField { field_id, name, field_type, desc, value: string | null }`；`Settings = { fields: SettingsField[] }`；`SECRET_TYPES` 常數（`field_type === "secret"` 判定改用型別）。
- `FALLBACK_SETTINGS`（options.ts）：改為 31 列 `SettingsField[]`（值 = backend 種子）。
- `src/lib/setting/`：query/mutation 契約同前（GET/PUT `/api/v1/settings`）。
- hook：`useSettings()` 回傳 `() => SettingsField[]`；提供 `settingsValue(field_id): string | null` helper。

### 5.2 設定頁

- GROUPS 改為靜態 `{ title, fieldIds: string[] }`（5 群組，field_id 對應 §3 registry）。
- 表單 data-driven：label/type/desc 由 API rows 提供；未知 field_id 不渲染（或顯示於「其他」）。
- secret 欄位：遮罩 placeholder、留空 = null（不變）、非 superadmin disabled。
- superadmin 判斷：`authState.info?.user?.email === "ssd@sowinsoft.com"`（取代 `isSystemAdmin()` — 修正 F4 的 salesrep-id 妥協）。
- 保存：批次 PUT 全欄位；int64/bool 值以字串送出；數值欄位沿用「raw string + save 時轉換 + 整數驗證 + 空白 trim = 不變」慣例。
- loader 預取；toast 同前。

### 5.3 使用點遷移（11 處）

`settings().default_department_id` 等 → `settingsValue("default_department_id")`（hook 回傳 rows 後的取值 helper）；各使用點語意不變（fallback = FALLBACK_SETTINGS 對應列）。

## 6. App

- `SettingsModel` → `SettingsFieldModel[]`（freezed；field_id/name/field_type/desc/value）。
- `SettingsService` 內建 **typed adapter**：從 rows 組出 `int? systemSalesrepId`、`int? systemDepartmentId`、`int? systemTestSalesrepId`、`int? companyAdminSalesrepId`、`int? defaultDepartmentId`、`String? aboutUrl`、`String? supportEmail`、`int? defaultTimeout`、`String? frontendUrl` 等 getter（by field_id + 型別轉換），**9 個使用點語法不變**（仍 `s.systemSalesrepId ?? SystemConstants.systemSalesrepId`）。
- Sembast cache 存 rows；fallback = SystemConstants 組 adapter。
- 登入/啟動 refresh + loadCache 邏輯不變。

## 7. 資料流

```
啟動（backend）：migration 建表 → seed（表空：registry 31 列 + env 值）→ load rows → cfg 覆寫 → NS/EMAIL client
執行期：GET（遮罩陣列）─┬→ frontend 頁面 data-driven 表單
                       └→ app 登入後 fetch → Sembast cache → typed adapter
        PUT（批次 upsert，secret null=不變）→ 前端重取
```

## 8. 錯誤處理與驗證

| 情境 | 處理 |
|---|---|
| 未登入 | 401 → 前端跳登入 |
| 非 admin PUT | 403 |
| secret 更新非 superadmin（email ≠ ssd@sowinsoft.com） | 403 → 前端 secret 區塊唯讀提示 |
| 遮罩值回寫 | 400 |
| 型別/格式驗證失敗 | 422 → 前端欄位錯誤顯示 |
| app fetch 失敗 | 回退 Sembast 快取 → SystemConstants adapter |
| registry 缺列 | GET 仍回傳既有列；前端未知 field_id 不渲染（記錄 warn） |

## 9. 測試

| 層 | 範圍 |
|---|---|
| backend 單元 | registry 完整性（31 列、field_id 無重複、6 secret）、seed（表空建/有列不重灌）、masking、型別驗證（int64/bool/string）、特殊規則（URL/email/port/timeout）、批次 upsert 語意（secret null=不變）、權限（email 判定 403） |
| backend 整合 | dockertest：GET 遮罩陣列、PUT 批次、403（非 superadmin email）、400（遮罩值） |
| frontend | vitest：data-driven 渲染（label/type/desc 來自 mock rows）、遮罩 placeholder、非 superadmin 唯讀、保存 payload（secret null）、數值 raw-string + trim 慣例 |
| app | flutter test：adapter fallback（無 rows → SystemConstants）、rows → typed getter 轉換 |

## 10. 遷移與風險

| 項目 | 決定 |
|---|---|
| 舊單列表 | migration drop；無資料遷移（feature 未上線/未合併，dev DB 重灌即可） |
| 值修正 | 沿用現行種子值（vendor 807、dept 6、frontend_url 由 FRONTEND_URL 種子） |
| superadmin 判定 | email == `ssd@sowinsoft.com`（backend 403 為權威；frontend 以 session email 做 UX 門控） |
| 動態欄位 | 範圍外（A2 不做）；新增欄位需改 registry + seed |
| bool 型別 | 備用（目前無 bool 欄位） |
| 稽核紀錄 | 範圍外 |

## 11. 實施順序

1. backend：ent schema 改版 → migration（drop 舊表 + 建新表）→ `fields.go` registry → seed/repository/transformation/validation/handler 改版 → 啟動整合 → tygo → 測試。
2. frontend：型別 + FALLBACK_SETTINGS 改列 → hook helper → 頁面 data-driven → superadmin email 判定 → 11 使用點遷移 → vitest。
3. app：model 改 rows → service typed adapter → 使用點驗證（語法不變）→ flutter test。
4. 文件：更新三份 AGENTS docs + changelog。
