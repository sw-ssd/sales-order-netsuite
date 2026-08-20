# Settings Field-Based Schema — Backend Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** 將 `settings` 表由單列 31 欄改為 field 導向（每欄位一列：`id, field_id, name, field_type, desc, value`），API 改為陣列批次契約，superadmin 判定改為 email。

**Architecture:** 固定 `FieldRegistry`（Go 常數，31 欄）為欄位定義單一來源；`settings` 表每列一個設定（value 字串儲存）；GET 回傳全欄位（secret 遮罩）、PUT 批次 upsert；權限以 session email == `ssd@sowinsoft.com` 判定 superadmin；啟動 seed（表空才建）+ cfg 覆寫不變。

**Tech Stack:** Go 1.25, entgo.io/ent, goose, chi, dockertest。

## Global Constraints

- 所有程式碼置於 `sales-order-backend/`（submodule，分支 `setting`；BASE 為現行 settings 實作 6070969）。
- 新表欄位固定為 `id, field_id, name, field_type, desc, value`；`field_type ∈ {int64, string, secret, bool}`；value 一律字串儲存。
- 31 欄 registry 內容以 spec §3 為準（field_id/name/type/desc/預設值逐字對應）。
- 遮罩：`field_type == "secret"` 且值非空 → `••••末4碼`；空 → `""`。
- PUT：全欄位物件 round-trip；secret `value == null` = 不變；`••••` 前綴 → 400；未知 field_id 忽略。
- 權限：PUT 需 admin+（`ui.User.BindNames.RoleName ∈ constants.EntAdminRole`）；含 secret 更新需 email == `constants.SuperAdminEmail`（`ssd@sowinsoft.com`）。
- 錯誤碼：401/403/400/422/500（spec §8）。
- 產生檔 `ent/gen/*` 入版控；`frontend_types/` 維持 gitignored（不入版控）。
- 每個 task 結束 commit（backend submodule 內）。

---

### Task 1: Ent schema 改版 + migration

**Files:**
- Modify: `sales-order-backend/ent/schema/setting.go`（31 欄 → 6 欄）
- Generate: `ent/gen/*`（`task ent:gen`）
- Create: `sales-order-backend/database/goose/<ts>_rebuild_settings_field_table.sql`

**Interfaces:**
- Produces: ent `Setting`（欄位 `field_id`（unique）、`name`、`field_type`、`desc`、`value` + BaseMixin id/created_at/updated_at）；migration **drop 舊表 + 建新表**（不插列）。

- [ ] **Step 1: 改 schema**

`ent/schema/setting.go` 的 `Fields()` 改為：

```go
func (Setting) Fields() []ent.Field {
	return []ent.Field{
		field.String("field_id").Unique().NotEmpty(),
		field.String("name"),
		field.String("field_type"),
		field.String("desc").Default(""),
		field.String("value").Default(""),
	}
}
```

（`BaseMixin{}` 提供 id/created_at/updated_at；`Policy()` 維持 nil。）

- [ ] **Step 2: 產生 ent 碼**

Run: `cd sales-order-backend && task ent:gen && go build ./...`
Expected: `ent/gen/setting*` 更新（`SetFieldID` 等）、build 通過。

- [ ] **Step 3: migration（drop 舊表 + 建新表）**

Run: `cd sales-order-backend && task db:migrate -- rebuild_settings_field_table`（若 Taskfile 傳參失敗，依既有 migration 檔格式手寫）。
確認 DDL：`DROP TABLE IF EXISTS settings;` + `CREATE TABLE settings (id bigint PK, field_id varchar UNIQUE NOT NULL, name varchar NOT NULL, field_type varchar NOT NULL, desc varchar NOT NULL DEFAULT '', value varchar NOT NULL DEFAULT '', created_at timestamptz NOT NULL, updated_at timestamptz NOT NULL);` — **不含 INSERT**。

Run: `cd sales-order-backend && task goose:up && task goose:status`
Expected: migration applied、表結構正確。

- [ ] **Step 4: Commit**

```bash
cd sales-order-backend && git add ent/schema/setting.go ent/gen database/goose && git commit -m "feat(ent): rebuild settings as field rows"
```

---

### Task 2: FieldRegistry（TDD）

**Files:**
- Create: `sales-order-backend/internal/domain/settings/fields.go`
- Test: `sales-order-backend/internal/domain/settings/fields_test.go`

**Interfaces:**
- Produces: `type FieldDef struct { FieldID, Name, FieldType, Desc, Default string }`；`var FieldRegistry []FieldDef`（31 列，spec §3 逐字）；`func IsSecretField(fieldID string) bool`（type==secret）；`func FieldByID(fieldID string) (FieldDef, bool)`。

- [ ] **Step 1: 寫失敗測試（registry 完整性）**

```go
package settings

import (
	"testing"

	"github.com/stretchr/testify/require"
)

func TestFieldRegistry_Integrity(t *testing.T) {
	require.Len(t, FieldRegistry, 31)
	seen := map[string]bool{}
	secretCount := 0
	for _, f := range FieldRegistry {
		require.NotEmpty(t, f.FieldID, "field_id 不可空")
		require.False(t, seen[f.FieldID], "重複 field_id: %s", f.FieldID)
		seen[f.FieldID] = true
		require.NotEmpty(t, f.Name)
		switch f.FieldType {
		case "int64", "string", "secret", "bool":
		default:
			t.Fatalf("未知 field_type %q for %s", f.FieldType, f.FieldID)
		}
		if f.FieldType == "secret" {
			secretCount++
		}
	}
	require.Equal(t, 6, secretCount)
	// 關鍵 field_id 存在（後續任務依賴）
	for _, id := range []string{"default_department_id", "system_salesrep_id", "frontend_url", "netsuite_consumer_key", "email_password", "default_vendor_id"} {
		_, ok := FieldByID(id)
		require.True(t, ok, "registry 缺 %s", id)
	}
	require.True(t, IsSecretField("email_password"))
	require.False(t, IsSecretField("email_host"))
}
```

- [ ] **Step 2: 執行確認失敗**

Run: `cd sales-order-backend && go test ./internal/domain/settings/ -run TestFieldRegistry`
Expected: FAIL（undefined: FieldRegistry）。

- [ ] **Step 3: 實作 fields.go**

（完整 31 列，spec §3 逐字；型別統計 14 int64 / 11 string / 6 secret / 0 bool）

```go
package settings

// FieldDef 為 settings 欄位定義（固定 registry）。
type FieldDef struct {
	FieldID   string
	Name      string
	FieldType string // int64 | string | secret | bool
	Desc      string
	Default   string
}

// FieldRegistry 為 settings 欄位單一來源（31 欄；值為預設種子）。
var FieldRegistry = []FieldDef{
	{"default_department_id", "預設部門 ID", "int64", "NetSuite 預設部門", "6"},
	{"approval", "核准狀態", "int64", "訂單核准狀態值", "4"},
	{"system_department_id", "系統管理員部門 ID", "int64", "系統部門（排除於篩選）", "-16888"},
	{"system_salesrep_id", "系統管理員業務 ID", "int64", "系統業務（排除於下拉）", "-16888"},
	{"system_test_salesrep_id", "系統測試業務 ID", "int64", "測試業務（排除於下拉）", "-17888"},
	{"system_customer_id", "系統測試客戶 ID", "int64", "種子客戶", "-17888"},
	{"system_customer_address_id", "系統客戶地址 ID", "int64", "種子地址", "-17888"},
	{"system_customer_address_entity_address_id", "系統客戶實體地址 ID", "int64", "種子實體地址", "-17888"},
	{"system_customer_contact_id", "系統客戶聯絡人 ID", "int64", "種子聯絡人", "-17888"},
	{"system_customer_entity_id", "系統客戶實體 ID", "string", "NetSuite Entity ID", "SW17888"},
	{"system_customer_name", "系統測試客戶名稱", "string", "", "系統測試客戶"},
	{"default_vendor_id", "預設供應商 ID", "int64", "供應商客戶", "807"},
	{"default_vendor_name", "預設供應商名稱", "string", "", "樹森開發股份有限公司"},
	{"default_salesrep_id", "預設業務 ID", "int64", "系統預設業務員", "119"},
	{"temp_car_number", "暫存車牌號碼", "int64", "未分配車牌之派車", "1"},
	{"company_admin_salesrep_id", "公司管理員業務 ID", "int64", "app 表單代換", "-5"},
	{"about_url", "關於我們 URL", "string", "app profile 用", "https://www.hexagonty.com"},
	{"support_email", "客服 Email", "string", "", "hexagon@hexagonty.com"},
	{"default_timeout", "預設逾時（秒）", "int64", "", "30"},
	{"frontend_url", "前端站台 URL", "string", "deeplink/manuals 用", ""},
	{"netsuite_account_id", "Account ID", "secret", "NetSuite TBA 憑證", ""},
	{"netsuite_consumer_key", "Consumer Key", "secret", "NetSuite TBA 憑證", ""},
	{"netsuite_consumer_secret", "Consumer Secret", "secret", "NetSuite TBA 憑證", ""},
	{"netsuite_token_id", "Token ID", "secret", "NetSuite TBA 憑證", ""},
	{"netsuite_token_secret", "Token Secret", "secret", "NetSuite TBA 憑證", ""},
	{"email_host", "SMTP Host", "string", "", "smtp.gmail.com"},
	{"email_port", "SMTP Port", "string", "字串儲存（config.Email.Port）", "587"},
	{"email_identity", "SMTP Identity", "string", "", ""},
	{"email_username", "SMTP Username", "string", "", ""},
	{"email_password", "SMTP Password", "secret", "", ""},
	{"email_from", "寄件人 Email", "string", "", ""},
}

func IsSecretField(fieldID string) bool {
	f, ok := FieldByID(fieldID)
	return ok && f.FieldType == "secret"
}

func FieldByID(fieldID string) (FieldDef, bool) {
	for _, f := range FieldRegistry {
		if f.FieldID == fieldID {
			return f, true
		}
	}
	return FieldDef{}, false
}
```

- [ ] **Step 4: 執行確認通過**

Run: `cd sales-order-backend && go test ./internal/domain/settings/ -run TestFieldRegistry`
Expected: PASS。

- [ ] **Step 5: Commit**

```bash
cd sales-order-backend && git add internal/domain/settings/fields.go internal/domain/settings/fields_test.go && git commit -m "feat(settings): add field registry"
```

---

### Task 3: DTO + transformation 改版（TDD）

**Files:**
- Modify: `sales-order-backend/internal/domain/settings/model.go`（`SettingDTO` → `SettingFieldDTO` + 批次容器）
- Modify: `sales-order-backend/internal/domain/settings/transformation.go`（`ToDTO` → by field_id 遮罩；`ToNetSuiteConfig`/`ToEmailConfig` by field_id）
- Test: `internal/domain/settings/transformation_test.go`（改版）

**Interfaces:**
- Produces: `SettingFieldDTO{FieldID, Name, FieldType, Desc string; Value *string}`；`SettingsResponse{Fields []SettingFieldDTO}`；`ToFieldDTOs(rows []*gen.Setting) []SettingFieldDTO`（secret 遮罩）；`ToNetSuiteConfig(rows []*gen.Setting) config.NetSuite`；`ToEmailConfig(rows []*gen.Setting) config.Email`；`fieldValue(rows, fieldID) string`。

- [ ] **Step 1: 寫失敗測試（遮罩 + config 組裝）**

```go
func TestToFieldDTOs_MasksSecrets(t *testing.T) {
	rows := []*gen.Setting{
		{FieldID: "default_department_id", Name: "預設部門 ID", FieldType: "int64", Value: "6"},
		{FieldID: "netsuite_consumer_key", Name: "Consumer Key", FieldType: "secret", Value: "7094cbcfc0e588445262fe5699513c7745418bee1339a7075eeaddd8529425f7"},
		{FieldID: "email_password", Name: "SMTP Password", FieldType: "secret", Value: ""},
	}
	dtos := ToFieldDTOs(rows)
	require.Len(t, dtos, 3)
	require.Equal(t, "6", *dtos[0].Value)
	require.Equal(t, "••••25f7", *dtos[1].Value)
	require.Equal(t, "", *dtos[2].Value) // 空 secret → ""
}

func TestToNetSuiteConfig_ByFieldID(t *testing.T) {
	rows := []*gen.Setting{
		{FieldID: "netsuite_account_id", Value: "11126151_SB2"},
		{FieldID: "netsuite_consumer_key", Value: "ck"},
		{FieldID: "netsuite_consumer_secret", Value: "cs"},
		{FieldID: "netsuite_token_id", Value: "ti"},
		{FieldID: "netsuite_token_secret", Value: "ts"},
	}
	cfg := ToNetSuiteConfig(rows)
	require.Equal(t, "11126151_SB2", cfg.ACCOUNT_ID)
	require.Equal(t, "ck", cfg.CONSUMER_KEY)
	require.Equal(t, "cs", cfg.CONSUMER_SECRET)
	require.Equal(t, "ti", cfg.TOKEN_ID)
	require.Equal(t, "ts", cfg.TOKEN_SECRET)
}
```

- [ ] **Step 2: 執行確認失敗**

Run: `cd sales-order-backend && go test ./internal/domain/settings/ -run 'TestToFieldDTOs|TestToNetSuiteConfig'`
Expected: FAIL（undefined 或舊簽名）。

- [ ] **Step 3: 實作 model.go + transformation.go**

`model.go`（取代舊 `SettingDTO`）：

```go
type SettingFieldDTO struct {
	FieldID   string  `json:"field_id"`
	Name      string  `json:"name"`
	FieldType string  `json:"field_type"`
	Desc      string  `json:"desc"`
	Value     *string `json:"value"` // GET 遮罩；PUT null=不變（secret）
}

type SettingsResponse struct {
	Fields []SettingFieldDTO `json:"fields"`
}
```

`transformation.go`（沿用 `maskPrefix`/`maskSecret`）：

```go
func ToFieldDTOs(rows []*gen.Setting) []SettingFieldDTO {
	out := make([]SettingFieldDTO, 0, len(rows))
	for _, r := range rows {
		v := r.Value
		if IsSecretField(r.FieldID) {
			m := maskSecret(v)
			v = m
		}
		out = append(out, SettingFieldDTO{
			FieldID: r.FieldID, Name: r.Name, FieldType: r.FieldType, Desc: r.Desc,
			Value: &v,
		})
	}
	return out
}

func fieldValue(rows []*gen.Setting, fieldID string) string {
	for _, r := range rows {
		if r.FieldID == fieldID {
			return r.Value
		}
	}
	return ""
}

func ToNetSuiteConfig(rows []*gen.Setting) config.NetSuite {
	return config.NetSuite{
		CONSUMER_KEY:    fieldValue(rows, "netsuite_consumer_key"),
		CONSUMER_SECRET: fieldValue(rows, "netsuite_consumer_secret"),
		TOKEN_ID:        fieldValue(rows, "netsuite_token_id"),
		TOKEN_SECRET:    fieldValue(rows, "netsuite_token_secret"),
		ACCOUNT_ID:      fieldValue(rows, "netsuite_account_id"),
	}
}

func ToEmailConfig(rows []*gen.Setting) config.Email {
	return config.Email{
		Identity: fieldValue(rows, "email_identity"),
		Host:     fieldValue(rows, "email_host"),
		Port:     fieldValue(rows, "email_port"),
		Username: fieldValue(rows, "email_username"),
		Password: fieldValue(rows, "email_password"),
		From:     fieldValue(rows, "email_from"),
	}
}
```

- [ ] **Step 4: 執行確認通過**

Run: `cd sales-order-backend && go test ./internal/domain/settings/ -run 'TestToFieldDTOs|TestToNetSuiteConfig'`
Expected: PASS。

- [ ] **Step 5: Commit**

```bash
cd sales-order-backend && git add internal/domain/settings/model.go internal/domain/settings/transformation.go internal/domain/settings/transformation_test.go && git commit -m "refactor(settings): field-row DTO and config assembly"
```

---

### Task 4: Repository（List/BatchUpsert）+ Seed 改版（TDD）

**Files:**
- Modify: `sales-order-backend/internal/domain/settings/repository.go`
- Modify: `sales-order-backend/internal/domain/settings/seed.go`
- Test: `internal/domain/settings/seed_test.go`（改版）

**Interfaces:**
- Produces: `IRepository`：`List(ctx) ([]*gen.Setting, error)`、`BatchUpsert(ctx, rows []*gen.Setting) error`（upsert by `field_id`）；`Seed(ctx, client *gen.Client, nscfg config.NetSuite, emailcfg config.Email, frontendURL string) error`（表空 → 依 registry 插 31 列；secret/frontend_url 以 env/參數覆寫；表有列 → 不重灌）；`LoadAll(ctx, client) ([]*gen.Setting, error)`。

- [ ] **Step 1: 寫失敗測試（seed 規則）**

```go
func TestSeed_CreatesRowsFromRegistry(t *testing.T) {
	lc := testcontainers.NewLocalTestContainer(t) // 依實際 API（Task-4 舊測試）
	client := lc.NewEntClient(t)
	ctx := context.Background()

	ns := config.NetSuite{CONSUMER_KEY: "ck", CONSUMER_SECRET: "cs", TOKEN_ID: "ti", TOKEN_SECRET: "ts", ACCOUNT_ID: "acc1"}
	email := config.Email{Host: "smtp.example.com", Port: "587", Username: "u", Password: "p", From: "f@example.com"}

	require.NoError(t, Seed(ctx, client, ns, email, "https://frontend.example.com"))
	rows, err := client.Setting.Query().Order(ent.Asc(setting.FieldID)).All(ctx)
	require.NoError(t, err)
	require.Len(t, rows, 31)
	// 值檢查
	require.Equal(t, "ck", fieldValue(rows, "netsuite_consumer_key"))
	require.Equal(t, "https://frontend.example.com", fieldValue(rows, "frontend_url"))
	require.Equal(t, "6", fieldValue(rows, "default_department_id"))
	require.Equal(t, "樹森開發股份有限公司", fieldValue(rows, "default_vendor_name"))

	// 有列 → 不重灌（改值後再 seed 不覆寫）
	require.NoError(t, client.Setting.Update().Where(setting.FieldIDEQ("netsuite_consumer_key")).SetValue("changed").Exec(ctx))
	require.NoError(t, Seed(ctx, client, ns, email, "https://frontend.example.com"))
	rows2, _ := client.Setting.Query().Where(setting.FieldIDEQ("netsuite_consumer_key")).All(ctx)
	require.Equal(t, "changed", rows2[0].Value)
}
```

- [ ] **Step 2: 執行確認失敗**

Run: `cd sales-order-backend && go test ./internal/domain/settings/ -run TestSeed`
Expected: FAIL（舊 Seed 簽名）。

- [ ] **Step 3: 實作 repository.go + seed.go**

`repository.go`：

```go
func (r *repository) List(ctx context.Context) ([]*gen.Setting, error) {
	return r.client.Setting.Query().All(ctx)
}

// BatchUpsert 依 field_id upsert（ON CONFLICT UPDATE）。
func (r *repository) BatchUpsert(ctx context.Context, rows []*gen.Setting) error {
	for _, row := range rows {
		_, err := r.client.Setting.Create().
			SetFieldID(row.FieldID).SetName(row.Name).SetFieldType(row.FieldType).
			SetDesc(row.Desc).SetValue(row.Value).
			OnConflictColumns(setting.FieldFieldID).
			UpdateNewValues().
			Save(ctx)
		if err != nil {
			return err
		}
	}
	return nil
}
```

`seed.go`（取代舊單列 seed）：

```go
func Seed(ctx context.Context, client *gen.Client, nscfg config.NetSuite, emailcfg config.Email, frontendURL string) error {
	count, err := client.Setting.Query().Count(ctx)
	if err != nil {
		return err
	}
	if count > 0 {
		return nil // seed-once：表已有列不重灌
	}
	envByID := map[string]string{
		"netsuite_account_id":   nscfg.ACCOUNT_ID,
		"netsuite_consumer_key": nscfg.CONSUMER_KEY,
		"netsuite_consumer_secret": nscfg.CONSUMER_SECRET,
		"netsuite_token_id":     nscfg.TOKEN_ID,
		"netsuite_token_secret": nscfg.TOKEN_SECRET,
		"email_host":            emailcfg.Host,
		"email_port":            emailcfg.Port,
		"email_identity":        emailcfg.Identity,
		"email_username":        emailcfg.Username,
		"email_password":        emailcfg.Password,
		"email_from":            emailcfg.From,
		"frontend_url":          frontendURL,
	}
	for _, f := range FieldRegistry {
		v := f.Default
		if ev, ok := envByID[f.FieldID]; ok && ev != "" {
			v = ev
		}
		_, err := client.Setting.Create().
			SetFieldID(f.FieldID).SetName(f.Name).SetFieldType(f.FieldType).
			SetDesc(f.Desc).SetValue(v).
			Save(ctx)
		if err != nil {
			return err
		}
	}
	return nil
}

func LoadAll(ctx context.Context, client *gen.Client) ([]*gen.Setting, error) {
	return client.Setting.Query().All(ctx)
}
```

- [ ] **Step 4: 執行確認通過**

Run: `cd sales-order-backend && go test ./internal/domain/settings/ -run TestSeed`
Expected: PASS（需容器環境，同前次配方：DOCKER_HOST=unix:///tmp/podman-docker.sock + API_SECRET/DB_DRIVER from hexagon.env）。

- [ ] **Step 5: Commit**

```bash
cd sales-order-backend && git add internal/domain/settings/repository.go internal/domain/settings/seed.go internal/domain/settings/seed_test.go && git commit -m "refactor(settings): field-row repository and registry seed"
```

---

### Task 5: Validation 改版（TDD）

**Files:**
- Modify: `sales-order-backend/internal/domain/settings/usecase.go`（驗證部分 → 抽 `validation.go`）
- Create: `sales-order-backend/internal/domain/settings/validation.go`
- Test: `internal/domain/settings/validation_test.go`

**Interfaces:**
- Produces: `ErrValidation`（沿用）、`ErrMaskedSecretValue`（沿用）、`ErrNeedSuperadmin`（沿用）；`validateFieldValue(fieldID, fieldType string, value *string) error`（type-level + 特殊規則 + required set）；`hasSecretUpdate(fields []SettingFieldDTO) bool`。

- [ ] **Step 1: 寫失敗測試**

```go
func TestValidateFieldValue(t *testing.T) {
	require.NoError(t, validateFieldValue("default_department_id", "int64", strPtr("6")))
	require.Error(t, validateFieldValue("default_department_id", "int64", strPtr("abc")))
	require.Error(t, validateFieldValue("default_department_id", "int64", nil)) // 非 secret 不接受 null
	require.NoError(t, validateFieldValue("email_password", "secret", nil))    // secret null = 不變
	require.NoError(t, validateFieldValue("email_password", "secret", strPtr("new")))
	require.Error(t, validateFieldValue("email_password", "secret", strPtr("••••25f7"))) // 遮罩值
	require.NoError(t, validateFieldValue("about_url", "string", strPtr("https://www.hexagonty.com")))
	require.Error(t, validateFieldValue("about_url", "string", strPtr("not-a-url")))
	require.Error(t, validateFieldValue("email_port", "string", strPtr("99999")))
	require.Error(t, validateFieldValue("default_timeout", "int64", strPtr("0")))
	require.NoError(t, validateFieldValue("email_username", "string", strPtr(""))) // 非必填可空
	require.Error(t, validateFieldValue("system_customer_name", "string", strPtr(""))) // 必填
}
```

- [ ] **Step 2: 執行確認失敗**

Run: `cd sales-order-backend && go test ./internal/domain/settings/ -run TestValidateFieldValue`
Expected: FAIL（undefined）。

- [ ] **Step 3: 實作 validation.go**

```go
package settings

import (
	"errors"
	"fmt"
	"net/mail"
	"net/url"
	"strconv"
	"strings"
)

var (
	ErrNeedSuperadmin    = errors.New("更新憑證需要 superadmin 權限（ssd@sowinsoft.com）")
	ErrMaskedSecretValue = errors.New("不允許回寫遮罩值，請重新輸入完整值")
	ErrValidation        = errors.New("validation failed")
)

const maskPrefix = "••••" // 沿用（若已於 transformation.go 宣告則不重複）

// requiredFields 必填非空（spec §4.3）。
var requiredFields = map[string]bool{
	"system_customer_entity_id": true,
	"system_customer_name":      true,
	"default_vendor_name":       true,
	"about_url":                 true,
	"support_email":             true,
	"email_from":                true,
}

// validateFieldValue 單一欄位值驗證：型別層級 + 特殊規則 + 必填。
// value == nil 僅 secret 允許（= 不變）。
func validateFieldValue(fieldID, fieldType string, value *string) error {
	if IsSecretField(fieldID) {
		if value == nil {
			return nil
		}
		if strings.HasPrefix(*value, maskPrefix) {
			return fmt.Errorf("%w: %s", ErrMaskedSecretValue, fieldID)
		}
		return nil // secret 可空（清除）
	}
	if value == nil {
		return fmt.Errorf("%w: %s 不接受 null", ErrValidation, fieldID)
	}
	v := *value
	switch fieldType {
	case "int64":
		n, err := strconv.ParseInt(v, 10, 64)
		if err != nil {
			return fmt.Errorf("%w: %s 需為整數", ErrValidation, fieldID)
		}
		if fieldID == "default_timeout" && n <= 0 {
			return fmt.Errorf("%w: default_timeout 需大於 0", ErrValidation)
		}
	case "bool":
		if v != "true" && v != "false" {
			return fmt.Errorf("%w: %s 需為 true/false", ErrValidation, fieldID)
		}
	case "string":
		// 特殊規則
		switch fieldID {
		case "about_url", "frontend_url":
			if v != "" {
				if _, err := url.ParseRequestURI(v); err != nil {
					return fmt.Errorf("%w: %s 無效 URL", ErrValidation, fieldID)
				}
			}
		case "support_email", "email_from", "email_username":
			if v != "" {
				if _, err := mail.ParseAddress(v); err != nil {
					return fmt.Errorf("%w: %s 無效 email", ErrValidation, fieldID)
				}
			}
		case "email_port":
			if v != "" {
				p, err := strconv.Atoi(v)
				if err != nil || p < 1 || p > 65535 {
					return fmt.Errorf("%w: email_port 需為 1-65535", ErrValidation)
				}
			}
		}
		if requiredFields[fieldID] && strings.TrimSpace(v) == "" {
			return fmt.Errorf("%w: %s 為必填", ErrValidation, fieldID)
		}
	default:
		return fmt.Errorf("%w: 未知 field_type %q", ErrValidation, fieldType)
	}
	return nil
}

// hasSecretUpdate 檢查批次中是否有 secret 欄位帶非 null 值。
func hasSecretUpdate(fields []SettingFieldDTO) bool {
	for _, f := range fields {
		if IsSecretField(f.FieldID) && f.Value != nil {
			return true
		}
	}
	return false
}
```

- [ ] **Step 4: 執行確認通過**

Run: `cd sales-order-backend && go test ./internal/domain/settings/ -run TestValidateFieldValue`
Expected: PASS。（若 `maskPrefix` 已在 transformation.go 宣告，validation.go 移除重複宣告。）

- [ ] **Step 5: Commit**

```bash
cd sales-order-backend && git add internal/domain/settings/validation.go internal/domain/settings/validation_test.go && git commit -m "feat(settings): field-level validation"
```

---

### Task 6: UseCase + Handler + Register 改版（TDD）

**Files:**
- Modify: `sales-order-backend/internal/domain/settings/usecase.go`
- Modify: `sales-order-backend/internal/domain/settings/handler.go`
- Modify: `sales-order-backend/internal/domain/settings/register.go`（路由不變，僅確認）
- Test: `internal/domain/settings/usecase_test.go`、`handler_test.go`（改版）

**Interfaces:**
- Produces: `IUseCase`：`Get(ctx) (*SettingsResponse, error)`、`Update(ctx, fields []SettingFieldDTO, isSuperadmin bool) (*SettingsResponse, error)`；handler 以 `ui.User.Email == constants.SuperAdminEmail` 判定 superadmin、`ui.User.BindNames.RoleName ∈ constants.EntAdminRole` 判定 admin。

- [ ] **Step 1: 寫失敗測試（權限 + 語意）**

```go
func TestUpdate_RejectsNonSuperadminSecret(t *testing.T) {
	uc := NewUseCase(repoFuncs{upsert: func(ctx context.Context, rows []*gen.Setting) error {
		t.Fatal("upsert 不應被呼叫")
		return nil
	}})
	fields := []SettingFieldDTO{{FieldID: "email_password", Value: strPtr("new")}}
	_, err := uc.Update(context.Background(), fields, false)
	require.ErrorIs(t, err, ErrNeedSuperadmin)
}

func TestUpdate_MaskedValueRejected(t *testing.T) {
	uc := NewUseCase(repoFuncs{})
	fields := []SettingFieldDTO{{FieldID: "email_password", Value: strPtr("••••gcnj")}}
	_, err := uc.Update(context.Background(), fields, true)
	require.ErrorIs(t, err, ErrMaskedSecretValue)
}

func TestUpdate_SuperadminOK(t *testing.T) {
	uc := NewUseCase(repoFuncs{upsert: func(ctx context.Context, rows []*gen.Setting) error {
		require.Len(t, rows, 1)
		require.Equal(t, "new", rows[0].Value)
		return nil
	}})
	fields := []SettingFieldDTO{{FieldID: "email_password", FieldType: "secret", Value: strPtr("new")}}
	_, err := uc.Update(context.Background(), fields, true)
	require.NoError(t, err)
}
```

（`repoFuncs` 改為實作新 `IRepository`（List/BatchUpsert）；測試補 `TestGet_ReturnsMaskedFields`、`TestUpdate_UnknownFieldIgnored`、`TestUpdate_Validation422`。）

- [ ] **Step 2: 執行確認失敗**

Run: `cd sales-order-backend && go test ./internal/domain/settings/ -run 'TestUpdate|TestGet'`
Expected: FAIL（舊介面）。

- [ ] **Step 3: 實作 usecase.go + handler.go**

`usecase.go`：

```go
type IUseCase interface {
	Get(ctx context.Context) (*SettingsResponse, error)
	Update(ctx context.Context, fields []SettingFieldDTO, isSuperadmin bool) (*SettingsResponse, error)
}

func (u *usecase) Get(ctx context.Context) (*SettingsResponse, error) {
	rows, err := u.repo.List(ctx)
	if err != nil {
		return nil, err
	}
	return &SettingsResponse{Fields: ToFieldDTOs(rows)}, nil
}

func (u *usecase) Update(ctx context.Context, fields []SettingFieldDTO, isSuperadmin bool) (*SettingsResponse, error) {
	if !isSuperadmin && hasSecretUpdate(fields) {
		return nil, ErrNeedSuperadmin
	}
	var rows []*gen.Setting
	for _, f := range fields {
		def, ok := FieldByID(f.FieldID)
		if !ok {
			continue // 未知 field_id 忽略
		}
		if f.Value == nil && IsSecretField(f.FieldID) {
			continue // secret null = 不變（不寫入，避免清空）
		}
		if err := validateFieldValue(f.FieldID, def.FieldType, f.Value); err != nil {
			return nil, err
		}
		v := ""
		if f.Value != nil {
			v = *f.Value
		}
		rows = append(rows, &gen.Setting{
			FieldID: f.FieldID, Name: def.Name, FieldType: def.FieldType,
			Desc: def.Desc, Value: v,
		})
	}
	if len(rows) == 0 {
		return nil, fmt.Errorf("%w: 沒有可更新的欄位", ErrValidation)
	}
	if err := u.repo.BatchUpsert(ctx, rows); err != nil {
		return nil, err
	}
	return u.Get(ctx)
}
```

（secret null = 不變：null 的 secret 欄位**不加入 rows**（`continue`），避免以空字串覆寫原值。）

`handler.go`：

```go
// currentSessionInfo 解析 session（email + role）。
func (h *Handler) currentSessionInfo(r *http.Request) (*pe.SessionInfoResource, error) {
	return pe.GetContextToSessionInfo(r.Context(), h.session)
}

func (h *Handler) GetSetting(w http.ResponseWriter, r *http.Request) {
	resp, err := h.uc.Get(r.Context())
	if err != nil {
		respond.Error(w, http.StatusInternalServerError, err)
		return
	}
	respond.Json(w, http.StatusOK, resp)
}

func (h *Handler) UpdateSetting(w http.ResponseWriter, r *http.Request) {
	var req SettingsResponse
	if err := json.NewDecoder(r.Body).Decode(&req); err != nil {
		respond.Error(w, http.StatusBadRequest, err)
		return
	}
	ui, err := h.currentSessionInfo(r)
	if err != nil || ui == nil {
		respond.Error(w, http.StatusUnauthorized, err)
		return
	}
	rname := ""
	if ui.User != nil && ui.User.BindNames != nil {
		rname = ui.User.BindNames.RoleName
	}
	if !slices.Contains(constants.EntAdminRole, rname) {
		respond.Error(w, http.StatusForbidden, errors.New("需要管理員權限"))
		return
	}
	isSuper := ui.User != nil && ui.User.Email == constants.SuperAdminEmail
	updated, err := h.uc.Update(r.Context(), req.Fields, isSuper)
	if err != nil {
		switch {
		case errors.Is(err, ErrNeedSuperadmin):
			respond.Error(w, http.StatusForbidden, err)
		case errors.Is(err, ErrMaskedSecretValue):
			respond.Error(w, http.StatusBadRequest, err)
		default:
			respond.Error(w, http.StatusUnprocessableEntity, err)
		}
		return
	}
	respond.Json(w, http.StatusOK, updated)
}
```

（`register.go` 不變：GET/PUT `/api/v1/settings` + `middleware.Authenticate(session)`。`pe` = `third_party/postgres_entstore`。）

- [ ] **Step 4: 執行確認通過**

Run: `cd sales-order-backend && go test ./internal/domain/settings/`
Expected: 全部 PASS（含舊 masking/handler 測試改版後）。

- [ ] **Step 5: Commit**

```bash
cd sales-order-backend && git add internal/domain/settings/usecase.go internal/domain/settings/handler.go internal/domain/settings/usecase_test.go internal/domain/settings/handler_test.go && git commit -m "refactor(settings): batch update with email superadmin gate"
```

---

### Task 7: 啟動整合 + tygo

**Files:**
- Modify: `sales-order-backend/internal/server/server.go`（`newDatabase()` seed 呼叫 + cfg 覆寫）
- Modify: `sales-order-backend/tygo.yaml`

**Interfaces:**
- Consumes: `Seed(ctx, cli, ns, email, frontendURL)`、`LoadAll`、`ToNetSuiteConfig(rows)`、`ToEmailConfig(rows)`。
- Produces: 啟動後 cfg 由 DB rows 覆寫（行為同前）；tygo 產出 `SettingFieldDTO`/`SettingsResponse`。

- [ ] **Step 1: server.go 更新**

`newDatabase()` 中取代舊 settings 區塊：

```go
	// settings：表空時依 registry 種子（env 覆寫 secret/frontend_url）；其後以 DB rows 建構 NS/EMAIL client
	frontendURL := os.Getenv("FRONTEND_URL")
	if err := settings.Seed(ctx, cli, s.cfg.NetSuite, s.cfg.Email, frontendURL); err != nil {
		log.Fatalf("failed to seed settings: %v", err)
	}
	srows, err := settings.LoadAll(ctx, cli)
	if err != nil {
		log.Fatalf("failed to load settings: %v", err)
	}
	s.cfg.NetSuite = settings.ToNetSuiteConfig(srows)
	s.cfg.Email = settings.ToEmailConfig(srows)
```

（import 補 `os`；`initSettings` 路由註冊不變。）

- [ ] **Step 2: tygo.yaml**

`frontend_types/settings.ts` 產出更新（`SettingFieldDTO` + `SettingsResponse`）。Run: `cd sales-order-backend && task typego`；產生檔不入版控（gitignored），僅確認生成。

- [ ] **Step 3: 驗證**

Run: `cd sales-order-backend && go build ./... && go vet ./internal/domain/settings/`
Expected: 通過。

- [ ] **Step 4: Commit**

```bash
cd sales-order-backend && git add internal/server/server.go tygo.yaml && git commit -m "feat(server): seed field rows and build clients from DB"
```

---

### Task 8: 整合測試 + 文件 + 收尾

**Files:**
- Modify: `sales-order-backend/internal/domain/settings/handler_integration_test.go`（改版：GET 陣列遮罩 / PUT 批次 / 403 email / 400 遮罩值）
- Modify: `docs/AGENTS/backend.md`（superproject root 提交）

**Interfaces:**
- Produces: 整合測試涵蓋新契約；backend.md 更新（field 表、email superadmin）。

- [ ] **Step 1: 整合測試改版**

依現有 integration 模式：GET 200 回傳 `fields` 陣列且 secret 遮罩（`••••` 前綴）；PUT 批次更新非 secret（200 + DB 值變）；PUT 含 secret 非 superadmin email session → 403；PUT 遮罩值 → 400。seed 於 SetupTest 以 `Seed(ctx, client, ns, email, "https://frontend.example.com")` 建立 31 列。

Run: `cd sales-order-backend && go test ./internal/domain/settings/ -run Integration`
Expected: PASS（需容器環境）。

- [ ] **Step 2: 更新 backend.md**

於 backend 指引：settings 改為 field 表（每欄位一列）、GET 遮罩、PUT 批次、superadmin 以 email `ssd@sowinsoft.com` 判定、seed 依 registry。

- [ ] **Step 3: 全量驗證**

Run: `cd sales-order-backend && go test ./internal/domain/settings/ && go build ./...`
Expected: PASS。

- [ ] **Step 4: Commit**

```bash
cd sales-order-backend && git add internal/domain/settings/handler_integration_test.go && git commit -m "test(settings): integration for field-row API"
cd .. && git add docs/AGENTS/backend.md && git commit -m "docs(backend): settings field-based schema guide"
```
