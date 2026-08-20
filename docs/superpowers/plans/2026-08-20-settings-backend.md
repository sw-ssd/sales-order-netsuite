# Settings Backend Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** 建立單列 `settings` ent schema + `/api/v1/settings` REST API（secret 遮罩、admin/superadmin 權限），啟動時以 env 種子並讓 NetSuite/EMAIL 憑證改由 DB 設定建構 client。

**Architecture:** 單列 `settings` 表（id=1）為唯一來源；新 domain `internal/domain/settings/` 提供 GET（遮罩）/PUT（upsert）與種子邏輯；`server.go` 啟動時 seed → load → 覆寫 `s.cfg.NetSuite`/`s.cfg.Email`，`initDomains.go` 完全不用改。內部 transformation 的 Go constants 暫留（折衷方案）。

**Tech Stack:** Go 1.25, entgo.io/ent, goose migrations, chi, casbin（角色檢查用 session info + constants），dockertest。

## Global Constraints

- 所有新程式碼置於 `sales-order-backend/`（submodule，路徑前綴一律以 `sales-order-backend/` 起）。
- 欄位名與 JSON key 一律 snake_case，與 spec §2 完全一致；secrets（6 欄）GET 一律遮罩 `••••<末4碼>`，空值回傳 `""`。
- 權限：GET = 任何已登入 session；PUT 非 secret = admin+；PUT 含 secret = 僅 superadmin（`constants.SuperAdminRoleName`）。
- PUT 語意：secret 欄位 `null`/省略 = 不變；非 `null` = 更新（完整值）；值含 `••••` 前綴 → 400。
- `settings` 列僅在不存在時建立（種子不覆寫既有值）。
- 不修改任何 NetSuite 同步 transformation 的常數讀取。
- 產生檔（`ent/gen/*`、`frontend_types/settings.ts`）需入版控。
- 每個 task 結束需 `git commit`（在 `sales-order-backend/` submodule 內提交）。

---

### Task 1: Ent schema + 產生碼

**Files:**
- Create: `sales-order-backend/ent/schema/setting.go`
- Modify: `sales-order-backend/internal/utility/nsstmt/stmts.go`（加 table 常數）
- Generate: `sales-order-backend/ent/gen/*`（`task ent:gen`）

**Interfaces:**
- Produces: ent schema `Setting`（table `settings`），生成 `ent/gen/setting*`（`gen.Setting`、`gen.SettingCreate`、`gen.SettingUpdateOne`、`setting.IDEQ` 等 predicate）；`nsstmt.Setting` 常數。

- [ ] **Step 1: nsstmt 加 table 常數**

於 `internal/utility/nsstmt/stmts.go` 的常數區塊（`// Metadict ent table_name constants` 之後）新增：

```go
// Settings ent table
const (
	Setting = "settings"
)
```

- [ ] **Step 2: 建立 ent schema**

建立 `ent/schema/setting.go`：

```go
package schema

import (
	"entgo.io/ent"
	"entgo.io/ent/dialect/entsql"
	"entgo.io/ent/schema"
	"entgo.io/ent/schema/field"
	"github.com/hexagon-maker/sales-order-backend/internal/utility/nsstmt"
)

// Setting holds the schema definition for the single-row settings entity.
type Setting struct {
	ent.Schema
}

func (Setting) Annotations() []schema.Annotation {
	return []schema.Annotation{
		entsql.Annotation{Table: nsstmt.Setting},
	}
}

func (Setting) Mixin() []ent.Mixin {
	return []ent.Mixin{
		BaseMixin{}, // id int64 + created_at / updated_at
	}
}

func (Setting) Fields() []ent.Field {
	return []ent.Field{
		// 系統常數
		field.Int64("default_department_id").Default(6),
		field.Int64("approval").Default(4),
		field.Int64("system_department_id").Default(-16888),
		field.Int64("system_salesrep_id").Default(-16888),
		field.Int64("system_test_salesrep_id").Default(-17888),
		field.Int64("system_customer_id").Default(-17888),
		field.Int64("system_customer_address_id").Default(-17888),
		field.Int64("system_customer_address_entity_address_id").Default(-17888),
		field.Int64("system_customer_contact_id").Default(-17888),
		field.String("system_customer_entity_id").Default("SW17888"),
		field.String("system_customer_name").Default("系統測試客戶"),
		field.Int64("default_vendor_id").Default(807),
		field.String("default_vendor_name").Default("樹森開發股份有限公司"),
		field.Int64("default_salesrep_id").Default(119),
		field.Int64("temp_car_number").Default(1),
		// App 常數
		field.Int64("company_admin_salesrep_id").Default(-5),
		field.String("about_url").Default("https://www.hexagonty.com"),
		field.String("support_email").Default("hexagon@hexagonty.com"),
		field.Int("default_timeout").Default(30),
		// 前端 URL
		field.String("frontend_url").Default(""),
		// NetSuite 憑證 (secret)
		field.String("netsuite_account_id").Default(""),
		field.String("netsuite_consumer_key").Default(""),
		field.String("netsuite_consumer_secret").Default(""),
		field.String("netsuite_token_id").Default(""),
		field.String("netsuite_token_secret").Default(""),
		// EMAIL
		field.String("email_host").Default("smtp.gmail.com"),
		field.String("email_port").Default("587"),
		field.String("email_identity").Default(""),
		field.String("email_username").Default(""),
		field.String("email_password").Default(""),
		field.String("email_from").Default(""),
	}
}

// Policy defines the privacy policy. 全域單列，僅需登入即可讀取，不需 tenant 過濾。
func (Setting) Policy() ent.Policy {
	return nil
}
```

- [ ] **Step 3: 產生 ent 碼**

Run: `cd sales-order-backend && task ent:gen`
Expected: `ent/gen/setting.go`、`ent/gen/setting_create.go`、`ent/gen/setting_update.go`、`ent/gen/setting_query.go`、`ent/gen/setting_where.go`、`ent/gen/setting/` 產生完成，`go build ./...` 通過。

- [ ] **Step 4: Commit**

```bash
cd sales-order-backend && git add ent/schema/setting.go internal/utility/nsstmt/stmts.go ent/gen && git commit -m "feat(ent): add Setting schema"
```

---

### Task 2: Goose migration（建表）

**Files:**
- Create: `sales-order-backend/database/goose/<timestamp>_add_settings_table.sql`

**Interfaces:**
- Produces: `settings` 表（欄位見 Task 1 schema；不插入列 — 列由啟動種子建立）。

- [ ] **Step 1: 產生 migration**

Run: `cd sales-order-backend && task db:migrate -- add_settings_table`
（若需 DB：先 `task infra:start`。）Atlas diff 依 ent schema 產生 `database/goose/*_add_settings_table.sql`。

- [ ] **Step 2: 確認 DDL 內容**

確認 migration 內為 `CREATE TABLE settings (...)` 且**不含 INSERT**。若含任何 seed 資料列，移除該 INSERT（種子職責在啟動邏輯，Task 7）。

- [ ] **Step 3: 驗證 migration 可套用**

Run: `cd sales-order-backend && task goose:up`
Expected: 最後一項為 `OK add_settings_table`；`task goose:status` 顯示 applied。

- [ ] **Step 4: Commit**

```bash
cd sales-order-backend && git add database/goose && git commit -m "feat(db): add settings table migration"
```

---

### Task 3: DTO + 遮罩 transformation（TDD）

**Files:**
- Create: `sales-order-backend/internal/domain/settings/model.go`
- Create: `sales-order-backend/internal/domain/settings/transformation.go`
- Test: `sales-order-backend/internal/domain/settings/transformation_test.go`

**Interfaces:**
- Produces: `SettingDTO` struct（欄位名 = spec §2 JSON key，secret 欄位 `*string`）；`SecretFieldNames []string`；`ToDTO(s *gen.Setting) *SettingDTO`（secret 遮罩）；`ApplyToGen(dto *SettingDTO) (nonSecretSet map[string]any, secretSet map[string]*string)`（供 repository 使用）。

- [ ] **Step 1: 寫失敗測試（遮罩規則）**

```go
package settings

import (
	"testing"

	"github.com/hexagon-maker/sales-order-backend/ent/gen"
	"github.com/stretchr/testify/require"
)

func TestToDTO_MasksSecrets(t *testing.T) {
	s := &gen.Setting{
		ID:                   1,
		NetsuiteAccountID:    "11126151_SB2",
		NetsuiteConsumerKey:  "abcd",
		NetsuiteTokenSecret:  "",
		EmailPassword:        "dqtn sblt oskn gcnj",
		DefaultDepartmentID:  6,
		DefaultVendorName:    "樹森開發股份有限公司",
	}
	dto := ToDTO(s)
	require.Equal(t, "••••SB2", *dto.NetsuiteAccountID)
	require.Equal(t, "••••", *dto.NetsuiteConsumerKey) // 長度 <= 4
	require.Equal(t, "", *dto.NetsuiteTokenSecret)     // 空值回傳 ""
	require.Equal(t, "••••gcnj", *dto.EmailPassword)
	require.Equal(t, int64(6), dto.DefaultDepartmentID)
	require.Equal(t, "樹森開發股份有限公司", dto.DefaultVendorName)
}

func TestSecretFieldNames(t *testing.T) {
	require.ElementsMatch(t, []string{
		"netsuite_account_id",
		"netsuite_consumer_key",
		"netsuite_consumer_secret",
		"netsuite_token_id",
		"netsuite_token_secret",
		"email_password",
	}, SecretFieldNames)
}
```

- [ ] **Step 2: 執行確認失敗**

Run: `cd sales-order-backend && go test ./internal/domain/settings/ -run 'TestToDTO|TestSecretFieldNames'`
Expected: FAIL（package 不存在 / undefined: ToDTO）。

- [ ] **Step 3: 實作 model.go**

```go
package settings

// SettingDTO 為 GET/PUT 共用 DTO；secret 欄位為 *string：
// GET 回傳遮罩後字串；PUT 時 null/省略 = 不變，非 null = 更新（完整值）。
type SettingDTO struct {
	DefaultDepartmentID                  int64   `json:"default_department_id"`
	Approval                             int64   `json:"approval"`
	SystemDepartmentID                   int64   `json:"system_department_id"`
	SystemSalesrepID                     int64   `json:"system_salesrep_id"`
	SystemTestSalesrepID                 int64   `json:"system_test_salesrep_id"`
	SystemCustomerID                     int64   `json:"system_customer_id"`
	SystemCustomerAddressID              int64   `json:"system_customer_address_id"`
	SystemCustomerAddressEntityAddressID int64   `json:"system_customer_address_entity_address_id"`
	SystemCustomerContactID              int64   `json:"system_customer_contact_id"`
	SystemCustomerEntityID               string  `json:"system_customer_entity_id"`
	SystemCustomerName                   string  `json:"system_customer_name"`
	DefaultVendorID                      int64   `json:"default_vendor_id"`
	DefaultVendorName                    string  `json:"default_vendor_name"`
	DefaultSalesrepID                    int64   `json:"default_salesrep_id"`
	TempCarNumber                        int64   `json:"temp_car_number"`
	CompanyAdminSalesrepID               int64   `json:"company_admin_salesrep_id"`
	AboutURL                             string  `json:"about_url"`
	SupportEmail                         string  `json:"support_email"`
	DefaultTimeout                       int     `json:"default_timeout"`
	FrontendURL                          string  `json:"frontend_url"`
	// secrets（遮罩 / 權限處理）
	NetsuiteAccountID     *string `json:"netsuite_account_id"`
	NetsuiteConsumerKey   *string `json:"netsuite_consumer_key"`
	NetsuiteConsumerSecret *string `json:"netsuite_consumer_secret"`
	NetsuiteTokenID       *string `json:"netsuite_token_id"`
	NetsuiteTokenSecret   *string `json:"netsuite_token_secret"`
	EmailPassword         *string `json:"email_password"`
	// EMAIL（非 secret）
	EmailHost     string `json:"email_host"`
	EmailPort     string `json:"email_port"`
	EmailIdentity string `json:"email_identity"`
	EmailUsername string `json:"email_username"`
	EmailFrom     string `json:"email_from"`
}

// SecretFieldNames 遮罩與權限檢查用（與 spec §2 的 6 欄 secret 一致）。
var SecretFieldNames = []string{
	"netsuite_account_id",
	"netsuite_consumer_key",
	"netsuite_consumer_secret",
	"netsuite_token_id",
	"netsuite_token_secret",
	"email_password",
}
```

- [ ] **Step 4: 實作 transformation.go**

```go
package settings

import (
	"github.com/hexagon-maker/sales-order-backend/ent/gen"
	"github.com/hexagon-maker/sales-order-backend/third_party/tools"
)

const maskPrefix = "••••"

// maskSecret 回傳遮罩字串：空值 → ""；長度 <= 4 → "••••"；否則 "••••"+末4字元。
func maskSecret(v string) string {
	if v == "" {
		return ""
	}
	if len(v) <= 4 {
		return maskPrefix
	}
	return maskPrefix + v[len(v)-4:]
}

// ToDTO 將 ent 列轉為 GET/PUT 共用 DTO，secret 一律遮罩。
func ToDTO(s *gen.Setting) *SettingDTO {
	mask := func(v string) *string { return tools.ToPtr(maskSecret(v)) }
	return &SettingDTO{
		DefaultDepartmentID:                  s.DefaultDepartmentID,
		Approval:                             s.Approval,
		SystemDepartmentID:                   s.SystemDepartmentID,
		SystemSalesrepID:                     s.SystemSalesrepID,
		SystemTestSalesrepID:                 s.SystemTestSalesrepID,
		SystemCustomerID:                     s.SystemCustomerID,
		SystemCustomerAddressID:              s.SystemCustomerAddressID,
		SystemCustomerAddressEntityAddressID: s.SystemCustomerAddressEntityAddressID,
		SystemCustomerContactID:              s.SystemCustomerContactID,
		SystemCustomerEntityID:               s.SystemCustomerEntityID,
		SystemCustomerName:                   s.SystemCustomerName,
		DefaultVendorID:                      s.DefaultVendorID,
		DefaultVendorName:                    s.DefaultVendorName,
		DefaultSalesrepID:                    s.DefaultSalesrepID,
		TempCarNumber:                        s.TempCarNumber,
		CompanyAdminSalesrepID:               s.CompanyAdminSalesrepID,
		AboutURL:                             s.AboutURL,
		SupportEmail:                         s.SupportEmail,
		DefaultTimeout:                       s.DefaultTimeout,
		FrontendURL:                          s.FrontendURL,
		NetsuiteAccountID:                    mask(s.NetsuiteAccountID),
		NetsuiteConsumerKey:                  mask(s.NetsuiteConsumerKey),
		NetsuiteConsumerSecret:               mask(s.NetsuiteConsumerSecret),
		NetsuiteTokenID:                      mask(s.NetsuiteTokenID),
		NetsuiteTokenSecret:                  mask(s.NetsuiteTokenSecret),
		EmailPassword:                        mask(s.EmailPassword),
		EmailHost:                            s.EmailHost,
		EmailPort:                            s.EmailPort,
		EmailIdentity:                        s.EmailIdentity,
		EmailUsername:                        s.EmailUsername,
		EmailFrom:                            s.EmailFrom,
	}
}
```

（`third_party/tools` 提供 `tools.ToPtr` — 若有 `tools.ToPtr` 不存在請用 `&v` 區域變數替代。）

- [ ] **Step 5: 執行確認通過**

Run: `cd sales-order-backend && go test ./internal/domain/settings/ -run 'TestToDTO|TestSecretFieldNames'`
Expected: PASS。

- [ ] **Step 6: Commit**

```bash
cd sales-order-backend && git add internal/domain/settings && git commit -m "feat(settings): add DTO and secret masking"
```

---

### Task 4: Repository + Seed（TDD）

**Files:**
- Create: `sales-order-backend/internal/domain/settings/repository.go`
- Create: `sales-order-backend/internal/domain/settings/seed.go`
- Create: `sales-order-backend/internal/domain/settings/seed_test.go`

**Interfaces:**
- Produces: `IRepository`（`Get(ctx) (*gen.Setting, error)`、`Upsert(ctx, dto *SettingDTO) (*gen.Setting, error)`）；`Seed(ctx, client *gen.Client, nscfg config.NetSuite, emailcfg config.Email) error`（列不存在才建立，secret 由 env 填入，非 secret 用 schema 預設）；`Load(ctx, client *gen.Client) (*gen.Setting, error)`；`ToNetSuiteConfig(s *gen.Setting) config.NetSuite`；`ToEmailConfig(s *gen.Setting) config.Email`。

- [ ] **Step 1: 寫失敗測試（seed 規則 + config 轉換）**

```go
package settings

import (
	"context"
	"testing"

	"github.com/hexagon-maker/sales-order-backend/config"
	"github.com/hexagon-maker/sales-order-backend/ent/gen"
	"github.com/hexagon-maker/sales-order-backend/ent/gen/setting"
	"github.com/hexagon-maker/sales-order-backend/testcontainers"
	"github.com/stretchr/testify/require"
)

func TestSeed_CreatesRowWithEnvSecrets(t *testing.T) {
	lc := testcontainers.NewLocalTestContainer(t)
	client := lc.NewEntClient(t)

	ns := config.NetSuite{
		CONSUMER_KEY:    "ck-secret",
		CONSUMER_SECRET: "cs-secret",
		TOKEN_ID:        "ti-secret",
		TOKEN_SECRET:    "ts-secret",
		ACCOUNT_ID:      "acc-1",
	}
	email := config.Email{
		Host: "smtp.example.com", Port: "587", Username: "u", Password: "p", From: "f@example.com",
	}

	err := Seed(context.Background(), client, ns, email)
	require.NoError(t, err)

	row, err := client.Setting.Get(context.Background(), 1)
	require.NoError(t, err)
	require.Equal(t, "ck-secret", row.NetsuiteConsumerKey)
	require.Equal(t, "acc-1", row.NetsuiteAccountID)
	require.Equal(t, "p", row.EmailPassword)
	require.Equal(t, int64(6), row.DefaultDepartmentID) // schema 預設
	require.Equal(t, "樹森開發股份有限公司", row.DefaultVendorName)

	// 再次 seed 不覆寫
	row.NetsuiteConsumerKey = "changed"
	_, err = client.Setting.UpdateOneID(1).SetNetsuiteConsumerKey("changed").Save(context.Background())
	require.NoError(t, err)
	err = Seed(context.Background(), client, ns, email)
	require.NoError(t, err)
	row2, _ := client.Setting.Get(context.Background(), 1)
	require.Equal(t, "changed", row2.NetsuiteConsumerKey)

	// 不存在的列以 Get 回傳 not found
	_, err = client.Setting.Query().Where(setting.IDEQ(999)).Count(context.Background())
	require.NoError(t, err)
}
```

- [ ] **Step 2: 執行確認失敗**

Run: `cd sales-order-backend && go test ./internal/domain/settings/ -run TestSeed -short`
Expected: FAIL（undefined: Seed / NewLocalTestContainer 用法不符則依 `testcontainers/local_containers.go` 現有 API 調整 — 參照 `customers/repository_test.go` 的 container 建立方式）。

- [ ] **Step 3: 實作 repository.go**

```go
package settings

import (
	"context"

	"github.com/hexagon-maker/sales-order-backend/ent/gen"
	"github.com/hexagon-maker/sales-order-backend/ent/gen/setting"
)

type IRepository interface {
	Get(ctx context.Context) (*gen.Setting, error)
	Upsert(ctx context.Context, dto *SettingDTO) (*gen.Setting, error)
}

type repository struct {
	client *gen.Client
}

func NewRepository(client *gen.Client) IRepository {
	return &repository{client: client}
}

func (r *repository) Get(ctx context.Context) (*gen.Setting, error) {
	return r.client.Setting.Get(ctx, 1)
}

// Upsert 更新 id=1：非 secret 欄位全量覆寫；secret 欄位僅在 dto 非 nil 時更新。
func (r *repository) Upsert(ctx context.Context, dto *SettingDTO) (*gen.Setting, error) {
	exists, err := r.client.Setting.Query().Where(setting.IDEQ(1)).Exist(ctx)
	if err != nil {
		return nil, err
	}
	if !exists {
		return r.client.Setting.Create().
			SetID(1).
			SetDefaultDepartmentID(dto.DefaultDepartmentID).
			SetApproval(dto.Approval).
			SetSystemDepartmentID(dto.SystemDepartmentID).
			SetSystemSalesrepID(dto.SystemSalesrepID).
			SetSystemTestSalesrepID(dto.SystemTestSalesrepID).
			SetSystemCustomerID(dto.SystemCustomerID).
			SetSystemCustomerAddressID(dto.SystemCustomerAddressID).
			SetSystemCustomerAddressEntityAddressID(dto.SystemCustomerAddressEntityAddressID).
			SetSystemCustomerContactID(dto.SystemCustomerContactID).
			SetSystemCustomerEntityID(dto.SystemCustomerEntityID).
			SetSystemCustomerName(dto.SystemCustomerName).
			SetDefaultVendorID(dto.DefaultVendorID).
			SetDefaultVendorName(dto.DefaultVendorName).
			SetDefaultSalesrepID(dto.DefaultSalesrepID).
			SetTempCarNumber(dto.TempCarNumber).
			SetCompanyAdminSalesrepID(dto.CompanyAdminSalesrepID).
			SetAboutURL(dto.AboutURL).
			SetSupportEmail(dto.SupportEmail).
			SetDefaultTimeout(dto.DefaultTimeout).
			SetFrontendURL(dto.FrontendURL).
			SetEmailHost(dto.EmailHost).
			SetEmailPort(dto.EmailPort).
			SetEmailIdentity(dto.EmailIdentity).
			SetEmailUsername(dto.EmailUsername).
			SetEmailFrom(dto.EmailFrom).
			applySecretSetters(dto).
			Save(ctx)
	}

	upd := r.client.Setting.UpdateOneID(1).
		SetDefaultDepartmentID(dto.DefaultDepartmentID).
		SetApproval(dto.Approval).
		SetSystemDepartmentID(dto.SystemDepartmentID).
		SetSystemSalesrepID(dto.SystemSalesrepID).
		SetSystemTestSalesrepID(dto.SystemTestSalesrepID).
		SetSystemCustomerID(dto.SystemCustomerID).
		SetSystemCustomerAddressID(dto.SystemCustomerAddressID).
		SetSystemCustomerAddressEntityAddressID(dto.SystemCustomerAddressEntityAddressID).
		SetSystemCustomerContactID(dto.SystemCustomerContactID).
		SetSystemCustomerEntityID(dto.SystemCustomerEntityID).
		SetSystemCustomerName(dto.SystemCustomerName).
		SetDefaultVendorID(dto.DefaultVendorID).
		SetDefaultVendorName(dto.DefaultVendorName).
		SetDefaultSalesrepID(dto.DefaultSalesrepID).
		SetTempCarNumber(dto.TempCarNumber).
		SetCompanyAdminSalesrepID(dto.CompanyAdminSalesrepID).
		SetAboutURL(dto.AboutURL).
		SetSupportEmail(dto.SupportEmail).
		SetDefaultTimeout(dto.DefaultTimeout).
		SetFrontendURL(dto.FrontendURL).
		SetEmailHost(dto.EmailHost).
		SetEmailPort(dto.EmailPort).
		SetEmailIdentity(dto.EmailIdentity).
		SetEmailUsername(dto.EmailUsername).
		SetEmailFrom(dto.EmailFrom).
		applySecretSetters(dto)
	return upd.Save(ctx)
}

// applySecretSetters 對 Create builder 套用非 nil 的 secret 值。
func (b *gen.SettingCreate) applySecretSetters(dto *SettingDTO) *gen.SettingCreate {
	if dto.NetsuiteAccountID != nil {
		b = b.SetNetsuiteAccountID(*dto.NetsuiteAccountID)
	}
	if dto.NetsuiteConsumerKey != nil {
		b = b.SetNetsuiteConsumerKey(*dto.NetsuiteConsumerKey)
	}
	if dto.NetsuiteConsumerSecret != nil {
		b = b.SetNetsuiteConsumerSecret(*dto.NetsuiteConsumerSecret)
	}
	if dto.NetsuiteTokenID != nil {
		b = b.SetNetsuiteTokenID(*dto.NetsuiteTokenID)
	}
	if dto.NetsuiteTokenSecret != nil {
		b = b.SetNetsuiteTokenSecret(*dto.NetsuiteTokenSecret)
	}
	if dto.EmailPassword != nil {
		b = b.SetEmailPassword(*dto.EmailPassword)
	}
	return b
}

// applySecretSetters 對 UpdateOne builder 套用非 nil 的 secret 值。
func (b *gen.SettingUpdateOne) applySecretSetters(dto *SettingDTO) *gen.SettingUpdateOne {
	if dto.NetsuiteAccountID != nil {
		b = b.SetNetsuiteAccountID(*dto.NetsuiteAccountID)
	}
	if dto.NetsuiteConsumerKey != nil {
		b = b.SetNetsuiteConsumerKey(*dto.NetsuiteConsumerKey)
	}
	if dto.NetsuiteConsumerSecret != nil {
		b = b.SetNetsuiteConsumerSecret(*dto.NetsuiteConsumerSecret)
	}
	if dto.NetsuiteTokenID != nil {
		b = b.SetNetsuiteTokenID(*dto.NetsuiteTokenID)
	}
	if dto.NetsuiteTokenSecret != nil {
		b = b.SetNetsuiteTokenSecret(*dto.NetsuiteTokenSecret)
	}
	if dto.EmailPassword != nil {
		b = b.SetEmailPassword(*dto.EmailPassword)
	}
	return b
}
```

（ent 產生的 `Set*` 方法回傳 `*gen.SettingCreate` / `*gen.SettingUpdateOne`，故以 method receiver 形式定義 `applySecretSetters` 以維持串接；若生成 API 不同以 `go build` 為準調整。）

- [ ] **Step 4: 實作 seed.go**

```go
package settings

import (
	"context"

	"github.com/hexagon-maker/sales-order-backend/config"
	"github.com/hexagon-maker/sales-order-backend/ent/gen"
	"github.com/hexagon-maker/sales-order-backend/ent/gen/setting"
)

// Seed 建立 settings 列（僅當 id=1 不存在時）。非 secret 欄位用 schema 預設；
// NetSuite / EMAIL 憑證以 env 載入的 config 填入。
func Seed(ctx context.Context, client *gen.Client, nscfg config.NetSuite, emailcfg config.Email) error {
	exists, err := client.Setting.Query().Where(setting.IDEQ(1)).Exist(ctx)
	if err != nil {
		return err
	}
	if exists {
		return nil
	}
	_, err = client.Setting.Create().
		SetID(1).
		SetNetsuiteAccountID(nscfg.ACCOUNT_ID).
		SetNetsuiteConsumerKey(nscfg.CONSUMER_KEY).
		SetNetsuiteConsumerSecret(nscfg.CONSUMER_SECRET).
		SetNetsuiteTokenID(nscfg.TOKEN_ID).
		SetNetsuiteTokenSecret(nscfg.TOKEN_SECRET).
		SetEmailHost(emailcfg.Host).
		SetEmailPort(emailcfg.Port).
		SetEmailIdentity(emailcfg.Identity).
		SetEmailUsername(emailcfg.Username).
		SetEmailPassword(emailcfg.Password).
		SetEmailFrom(emailcfg.From).
		Save(ctx)
	return err
}

// Load 讀取 id=1 的設定列。
func Load(ctx context.Context, client *gen.Client) (*gen.Setting, error) {
	return client.Setting.Get(ctx, 1)
}

// ToNetSuiteConfig 將 DB 設定轉回 config.NetSuite（供 client 建構）。
func ToNetSuiteConfig(s *gen.Setting) config.NetSuite {
	return config.NetSuite{
		CONSUMER_KEY:    s.NetsuiteConsumerKey,
		CONSUMER_SECRET: s.NetsuiteConsumerSecret,
		TOKEN_ID:        s.NetsuiteTokenID,
		TOKEN_SECRET:    s.NetsuiteTokenSecret,
		ACCOUNT_ID:      s.NetsuiteAccountID,
	}
}

// ToEmailConfig 將 DB 設定轉回 config.Email（供 mail 建構）。
func ToEmailConfig(s *gen.Setting) config.Email {
	return config.Email{
		Identity: s.EmailIdentity,
		Host:     s.EmailHost,
		Port:     s.EmailPort,
		Username: s.EmailUsername,
		Password: s.EmailPassword,
		From:     s.EmailFrom,
	}
}
```

- [ ] **Step 5: 執行確認通過**

Run: `cd sales-order-backend && go test ./internal/domain/settings/ -run TestSeed`
Expected: PASS（整合測試會啟動 dockertest container；需 Docker 在線）。

- [ ] **Step 6: Commit**

```bash
cd sales-order-backend && git add internal/domain/settings && git commit -m "feat(settings): add repository, seed and config conversion"
```

---

### Task 5: UseCase（驗證 + secret 語意 + 權限，TDD）

**Files:**
- Create: `sales-order-backend/internal/domain/settings/usecase.go`
- Create: `sales-order-backend/internal/domain/settings/usecase_test.go`

**Interfaces:**
- Consumes: `IRepository`（Task 4）、`SettingDTO`、`SecretFieldNames`、`maskSecret`（Task 3）。
- Produces: `IUseCase`（`Get(ctx) (*SettingDTO, error)`、`Update(ctx, dto *SettingDTO, isSuperadmin bool) (*SettingDTO, error)`）；sentinel errors `ErrNeedSuperadmin`、`ErrMaskedSecretValue`；`validateNonSecret(dto) error`（供 handler 端測試/直接使用）。

- [ ] **Step 1: 寫失敗測試**

```go
package settings

import (
	"context"
	"errors"
	"testing"

	"github.com/stretchr/testify/require"
)

type repoFuncs struct {
	get    func(ctx context.Context) (*gen.Setting, error)
	upsert func(ctx context.Context, dto *SettingDTO) (*gen.Setting, error)
}

func (f repoFuncs) Get(ctx context.Context) (*gen.Setting, error)          { return f.get(ctx) }
func (f repoFuncs) Upsert(ctx context.Context, d *SettingDTO) (*gen.Setting, error) { return f.upsert(ctx, d) }

func ptr[T any](v T) *T { return &v }

func TestUpdate_RejectsNonSuperadminSecretChange(t *testing.T) {
	uc := NewUseCase(repoFuncs{
		upsert: func(ctx context.Context, d *SettingDTO) (*gen.Setting, error) {
			t.Fatal("upsert 不應被呼叫")
			return nil, nil
		},
	})
	dto := &SettingDTO{
		DefaultDepartmentID: 6,
		NetSuiteConsumerKey: ptr("new-key"),
	}
	_, err := uc.Update(context.Background(), dto, false)
	require.ErrorIs(t, err, ErrNeedSuperadmin)
}

func TestUpdate_RejectsMaskedSecretValue(t *testing.T) {
	uc := NewUseCase(repoFuncs{})
	dto := &SettingDTO{
		DefaultDepartmentID: 6,
		NetSuiteConsumerKey: ptr("••••25f7"), // GET 遮罩值不可回寫
	}
	_, err := uc.Update(context.Background(), dto, true)
	require.ErrorIs(t, err, ErrMaskedSecretValue)
}

func TestUpdate_AllowsSuperadminSecretChange(t *testing.T) {
	uc := NewUseCase(repoFuncs{
		upsert: func(ctx context.Context, d *SettingDTO) (*gen.Setting, error) {
			require.Equal(t, "new-key", *d.NetSuiteConsumerKey)
			return &gen.Setting{ID: 1}, nil
		},
	})
	dto := &SettingDTO{DefaultDepartmentID: 6, NetSuiteConsumerKey: ptr("new-key")}
	_, err := uc.Update(context.Background(), dto, true)
	require.NoError(t, err)
}

func TestValidateNonSecret(t *testing.T) {
	require.Error(t, validateNonSecret(&SettingDTO{DefaultDepartmentID: 6})) // 必填缺漏
	require.NoError(t, validateNonSecret(&SettingDTO{
		DefaultDepartmentID: 6,
		SystemCustomerEntityID: "SW17888",
		SystemCustomerName:     "系統測試客戶",
		DefaultVendorName:      "樹森開發股份有限公司",
		AboutURL:               "https://www.hexagonty.com",
		SupportEmail:           "hexagon@hexagonty.com",
		EmailFrom:              "hexagon@hexagonty.com",
		EmailPort:              "587",
		DefaultTimeout:         30,
	}))
	require.Error(t, validateNonSecret(&SettingDTO{
		DefaultDepartmentID: 6,
		SystemCustomerEntityID: "SW17888",
		SystemCustomerName:     "系統測試客戶",
		DefaultVendorName:      "樹森開發股份有限公司",
		AboutURL:               "not-a-url",
		SupportEmail:           "hexagon@hexagonty.com",
		EmailFrom:              "hexagon@hexagonty.com",
		EmailPort:              "587",
		DefaultTimeout:         30,
	}))
	require.Error(t, validateNonSecret(&SettingDTO{
		DefaultDepartmentID: 6,
		SystemCustomerEntityID: "SW17888",
		SystemCustomerName:     "系統測試客戶",
		DefaultVendorName:      "樹森開發股份有限公司",
		AboutURL:               "https://www.hexagonty.com",
		SupportEmail:           "hexagon@hexagonty.com",
		EmailFrom:              "hexagon@hexagonty.com",
		EmailPort:              "99999", // port 超出範圍
		DefaultTimeout:         30,
	}))
}
```

（測試需 import `"github.com/hexagon-maker/sales-order-backend/ent/gen"`；`repoFuncs` 實作 `IRepository`。）

- [ ] **Step 2: 執行確認失敗**

Run: `cd sales-order-backend && go test ./internal/domain/settings/ -run 'TestUpdate|TestValidate'`
Expected: FAIL（undefined: NewUseCase / validateNonSecret / ErrNeedSuperadmin / ErrMaskedSecretValue）。

- [ ] **Step 3: 實作 usecase.go**

```go
package settings

import (
	"context"
	"errors"
	"fmt"
	"net/mail"
	"net/url"
	"strconv"
	"strings"
)

var (
	ErrNeedSuperadmin   = errors.New("更新憑證需要 superadmin 權限")
	ErrMaskedSecretValue = errors.New("不允許回寫遮罩值，請重新輸入完整值")
)

type IUseCase interface {
	Get(ctx context.Context) (*SettingDTO, error)
	Update(ctx context.Context, dto *SettingDTO, isSuperadmin bool) (*SettingDTO, error)
}

type usecase struct {
	repo IRepository
}

func NewUseCase(repo IRepository) IUseCase {
	return &usecase{repo: repo}
}

func (u *usecase) Get(ctx context.Context) (*SettingDTO, error) {
	row, err := u.repo.Get(ctx)
	if err != nil {
		return nil, err
	}
	return ToDTO(row), nil
}

func (u *usecase) Update(ctx context.Context, dto *SettingDTO, isSuperadmin bool) (*SettingDTO, error) {
	if err := validateNonSecret(dto); err != nil {
		return nil, err
	}
	if !isSuperadmin && hasSecretChange(dto) {
		return nil, ErrNeedSuperadmin
	}
	if err := rejectMaskedSecrets(dto); err != nil {
		return nil, err
	}
	row, err := u.repo.Upsert(ctx, dto)
	if err != nil {
		return nil, err
	}
	return ToDTO(row), nil
}

// hasSecretChange 檢查 body 是否含任一非 nil 的 secret 欄位。
func hasSecretChange(dto *SettingDTO) bool {
	return dto.NetsuiteAccountID != nil ||
		dto.NetsuiteConsumerKey != nil ||
		dto.NetsuiteConsumerSecret != nil ||
		dto.NetsuiteTokenID != nil ||
		dto.NetsuiteTokenSecret != nil ||
		dto.EmailPassword != nil
}

// rejectMaskedSecrets 拒絕以 GET 遮罩值（含 "••••" 前綴）回寫。
func rejectMaskedSecrets(dto *SettingDTO) error {
	secrets := []*string{
		dto.NetsuiteAccountID, dto.NetsuiteConsumerKey, dto.NetsuiteConsumerSecret,
		dto.NetsuiteTokenID, dto.NetsuiteTokenSecret, dto.EmailPassword,
	}
	for _, s := range secrets {
		if s != nil && strings.HasPrefix(*s, maskPrefix) {
			return ErrMaskedSecretValue
		}
	}
	return nil
}

// validateNonSecret 驗證所有非 secret 欄位（必填 + 格式）。
func validateNonSecret(dto *SettingDTO) error {
	required := []struct {
		name  string
		value string
	}{
		{"system_customer_entity_id", dto.SystemCustomerEntityID},
		{"system_customer_name", dto.SystemCustomerName},
		{"default_vendor_name", dto.DefaultVendorName},
		{"about_url", dto.AboutURL},
		{"support_email", dto.SupportEmail},
		{"email_from", dto.EmailFrom},
	}
	for _, r := range required {
		if strings.TrimSpace(r.value) == "" {
			return fmt.Errorf("欄位 %s 為必填", r.name)
		}
	}
	for _, u := range []string{dto.AboutURL, dto.FrontendURL} {
		if u == "" {
			continue
		}
		if _, err := url.ParseRequestURI(u); err != nil {
			return fmt.Errorf("無效的 URL: %s", u)
		}
	}
	for _, e := range []string{dto.SupportEmail, dto.EmailFrom, dto.EmailUsername} {
		if e == "" {
			continue
		}
		if _, err := mail.ParseAddress(e); err != nil {
			return fmt.Errorf("無效的 email: %s", e)
		}
	}
	if dto.EmailPort != "" {
		p, err := strconv.Atoi(dto.EmailPort)
		if err != nil || p < 1 || p > 65535 {
			return fmt.Errorf("email_port 需為 1-65535 的數字，收到: %q", dto.EmailPort)
		}
	}
	if dto.DefaultTimeout <= 0 {
		return fmt.Errorf("default_timeout 需大於 0，收到: %d", dto.DefaultTimeout)
	}
	return nil
}
```

- [ ] **Step 4: 執行確認通過**

Run: `cd sales-order-backend && go test ./internal/domain/settings/ -run 'TestUpdate|TestValidate'`
Expected: PASS。

- [ ] **Step 5: Commit**

```bash
cd sales-order-backend && git add internal/domain/settings && git commit -m "feat(settings): add usecase with validation and secret semantics"
```

---

### Task 6: Handler + Routes（TDD）

**Files:**
- Create: `sales-order-backend/internal/domain/settings/handler.go`
- Create: `sales-order-backend/internal/domain/settings/register.go`
- Create: `sales-order-backend/internal/domain/settings/handler_test.go`

**Interfaces:**
- Consumes: `IUseCase`、`SettingDTO`（Task 5）；session role 解析：`pe.UnmarshalContextToSessionInfo`（`third_party/postgres_entstore`）、`constants.KeySession`、`constants.EntAdminRole`、`constants.SuperAdminRoleName`。
- Produces: `Handler`（`GetSetting` / `UpdateSetting`）；`RegisterHTTPEndPoints(router chi.Router, session *scs.SessionManager, uc IUseCase) *Handler`，路由 `/api/v1/settings`（GET、PUT，皆 `middleware.Authenticate(session)`）。

- [ ] **Step 1: 寫失敗測試（handler 遮罩回應 + 403 情境）**

```go
package settings

import (
	"context"
	"encoding/json"
	"net/http"
	"net/http/httptest"
	"testing"

	"github.com/hexagon-maker/sales-order-backend/ent/gen"
	"github.com/stretchr/testify/require"
)

func TestGetSetting_ReturnsMasked(t *testing.T) {
	h := NewHandler(nil, &fakeUC{
		get: func(ctx context.Context) (*SettingDTO, error) {
			row := &gen.Setting{ID: 1, NetsuiteAccountID: "11126151_SB2", NetsuiteConsumerKey: "ck"}
			return ToDTO(row), nil
		},
	})
	req := httptest.NewRequest(http.MethodGet, "/api/v1/settings", nil)
	rec := httptest.NewRecorder()
	h.GetSetting(rec, req)
	require.Equal(t, http.StatusOK, rec.Code)
	var body SettingDTO
	require.NoError(t, json.Unmarshal(rec.Body.Bytes(), &body))
	require.Equal(t, "••••SB2", *body.NetsuiteAccountID)
	require.Equal(t, "••••", *body.NetsuiteConsumerKey)
}
```

（`fakeUC` 為 IUseCase 的測試替身，依 Task 5 介面撰寫；`handler_test.go` 內補 `fakeUC` struct 與其餘情境 — non-admin PUT → 403、admin 非 superadmin + secret → ErrNeedSuperadmin。）

- [ ] **Step 2: 執行確認失敗**

Run: `cd sales-order-backend && go test ./internal/domain/settings/ -run TestGetSetting`
Expected: FAIL（undefined: NewHandler）。

- [ ] **Step 3: 實作 handler.go**

```go
package settings

import (
	"encoding/json"
	"errors"
	"net/http"
	"slices"

	"github.com/alexedwards/scs/v2"
	"github.com/go-chi/chi/v5"
	"github.com/hexagon-maker/sales-order-backend/internal/utility/respond"
	pe "github.com/hexagon-maker/sales-order-backend/third_party/postgres_entstore"
	"github.com/hexagon-maker/sales-order-backend/third_party/constants"
)

type Handler struct {
	session *scs.SessionManager
	uc      IUseCase
}

func NewHandler(session *scs.SessionManager, uc IUseCase) *Handler {
	return &Handler{session: session, uc: uc}
}

// currentRoleName 自 session info 解析目前使用者角色名（與舊 Authorize middleware 同機制）。
func (h *Handler) currentRoleName(r *http.Request) (string, error) {
	ui, err := pe.UnmarshalContextToSessionInfo(r.Context(), h.session, constants.KeySession)
	if err != nil {
		return "", err
	}
	_, _, rname := ui.SessionInfoAuthorizeData()
	return rname, nil
}

// GetSetting 回傳全部設定，secret 已遮罩。
func (h *Handler) GetSetting(w http.ResponseWriter, r *http.Request) {
	dto, err := h.uc.Get(r.Context())
	if err != nil {
		respond.Error(w, http.StatusInternalServerError, err)
		return
	}
	respond.Json(w, http.StatusOK, dto)
}

// UpdateSetting 更新設定；含 secret 欄位時僅 superadmin 可更新。
func (h *Handler) UpdateSetting(w http.ResponseWriter, r *http.Request) {
	var dto SettingDTO
	if err := json.NewDecoder(r.Body).Decode(&dto); err != nil {
		respond.Error(w, http.StatusBadRequest, err)
		return
	}

	rname, err := h.currentRoleName(r)
	if err != nil {
		respond.Error(w, http.StatusUnauthorized, err)
		return
	}
	if !slices.Contains(constants.EntAdminRole, rname) {
		respond.Error(w, http.StatusForbidden, errors.New("需要管理員權限"))
		return
	}

	updated, err := h.uc.Update(r.Context(), &dto, rname == constants.SuperAdminRoleName)
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

- [ ] **Step 4: 實作 register.go**

```go
package settings

import (
	"github.com/alexedwards/scs/v2"
	"github.com/go-chi/chi/v5"
	"github.com/hexagon-maker/sales-order-backend/internal/middleware"
)

// RegisterHTTPEndPoints 註冊 /api/v1/settings 路由。
func RegisterHTTPEndPoints(router chi.Router, session *scs.SessionManager, uc IUseCase) *Handler {
	handler := NewHandler(session, uc)

	router.Route("/api/v1/settings", func(r chi.Router) {
		r.Use(middleware.Authenticate(session))
		r.Get("/", handler.GetSetting)
		r.Put("/", handler.UpdateSetting)
	})

	return handler
}
```

- [ ] **Step 5: 執行確認通過**

Run: `cd sales-order-backend && go test ./internal/domain/settings/ -run 'TestGetSetting|TestUpdateSetting'`
Expected: PASS。（若 `pe.UnmarshalContextToSessionInfo` 簽名與此處不同，以實際簽名調整 — 參照 `internal/middleware/authorization.go` 註解中的呼叫方式。）

- [ ] **Step 6: Commit**

```bash
cd sales-order-backend && git add internal/domain/settings && git commit -m "feat(settings): add handler and routes"
```

---

### Task 7: 啟動整合（seed + cfg 覆寫）

**Files:**
- Modify: `sales-order-backend/internal/server/server.go`（`newDatabase()`，既有 `c.Seeder(...)` 之後）
- Modify: `sales-order-backend/internal/server/initDomains.go` — 無需變更（cfg 已被覆寫），僅驗證。

**Interfaces:**
- Consumes: `settings.Seed`、`settings.Load`、`settings.ToNetSuiteConfig`、`settings.ToEmailConfig`（Task 4）。
- Produces: 啟動後 `s.cfg.NetSuite` / `s.cfg.Email` 為 DB 值；`newNSClient(s.cfg.NetSuite)` 與 mail 建構沿用不變。

- [ ] **Step 1: server.go 加入 seed + load + 覆寫**

於 `newDatabase()` 中 `c.Seeder(ctx, c.DefaultDepartment, ...)` 之後插入：

```go
	// settings：首次啟動以 env + constants 種子建立單列；其後 NetSuite/EMAIL client 以 DB 值建構
	if err := settings.Seed(ctx, cli, s.cfg.NetSuite, s.cfg.Email); err != nil {
		log.Fatalf("failed to seed settings: %v", err)
	}
	st, err := settings.Load(ctx, cli)
	if err != nil {
		log.Fatalf("failed to load settings: %v", err)
	}
	s.cfg.NetSuite = settings.ToNetSuiteConfig(st)
	s.cfg.Email = settings.ToEmailConfig(st)
```

並在 import 加上 `"github.com/hexagon-maker/sales-order-backend/internal/domain/settings"`。

- [ ] **Step 2: 驗證 build**

Run: `cd sales-order-backend && go build ./...`
Expected: 通過。`initDomains.go` 不需修改（`s.cfg.NetSuite`/`s.cfg.Email` 已為 DB 值）。

- [ ] **Step 3: Commit**

```bash
cd sales-order-backend && git add internal/server/server.go && git commit -m "feat(server): seed settings and build NS/email clients from DB"
```

---

### Task 8: tygo 產生 frontend types

**Files:**
- Modify: `sales-order-backend/tygo.yaml`
- Generate: `sales-order-backend/frontend_types/settings.ts`

**Interfaces:**
- Produces: `frontend_types/settings.ts`（`SettingDTO` → TS interface，供 frontend plan 使用）。

- [ ] **Step 1: tygo.yaml 加 settings**

於 `packages:` 最後一個 entry 之後新增：

```yaml
  - path: "github.com/hexagon-maker/sales-order-backend/internal/domain/settings"
    type_mappings:
      time.Time: "string /* RFC3339 */"
    output_path: "frontend_types/settings.ts"
    include_files:
      - "model.go"
```

- [ ] **Step 2: 產生並驗證**

Run: `cd sales-order-backend && task typego`
Expected: `frontend_types/settings.ts` 產生，含 `SettingDTO` interface（secrets 為 `string | null`）。

- [ ] **Step 3: Commit**

```bash
cd sales-order-backend && git add tygo.yaml frontend_types/settings.ts && git commit -m "feat(tygo): generate settings frontend types"
```

---

### Task 9: 整合測試（dockertest：GET 遮罩 / PUT upsert / 403）

**Files:**
- Create: `sales-order-backend/internal/domain/settings/handler_integration_test.go`

**Interfaces:**
- Consumes: `RegisterHTTPEndPoints`、`Seed`、`testcontainers.NewLocalTestContainer`（參照 `internal/domain/users/handler_integration_test.go` 的 session/casbin 建立方式）。

- [ ] **Step 1: 寫整合測試**

依 `internal/domain/users/handler_integration_test.go` 模式建立 container、`client.Schema.Create`、session、seed，並以 `httptest` 打 handler：

- `GET /api/v1/settings` → 200，secret 欄位為遮罩字串（以 `••••` 前綴斷言）。
- `PUT /api/v1/settings`（非 secret 全量 + secret 為 null）→ 200，`default_vendor_id` 已更新。
- `PUT /api/v1/settings` 含 secret 欄位（以非 superadmin session 角色）→ 403。
- `PUT` 帶 `••••` 前綴 secret → 400。

（session 角色以 testcontainers/session 建立方式注入 — 參照現有 integration tests；若角色注入過於繁瑣，403 情境以 usecase 單元測試涵蓋、整合層僅驗證 200 路徑，並在測試註解說明。）

- [ ] **Step 2: 執行**

Run: `cd sales-order-backend && task test:integration -run Settings`（或 `go test ./internal/domain/settings/ -run Integration`）
Expected: PASS。

- [ ] **Step 3: Commit**

```bash
cd sales-order-backend && git add internal/domain/settings/handler_integration_test.go && git commit -m "test(settings): add integration test for API"
```

---

### Task 10: 文件 + 收尾驗證

**Files:**
- Modify: `docs/AGENTS/backend.md`（此檔位於 superproject root，於 root repo commit）

**Interfaces:**
- Produces: 文件說明 settings domain、API、啟動種子行為。

- [ ] **Step 1: 更新 backend 指引**

於 `docs/AGENTS/backend.md` 的網域清單補 `settings`（`/api/v1/settings`：GET 遮罩、PUT admin+/superadmin secret），並於 §4 環境設定補註「NetSuite/EMAIL 憑證首次啟動 seed 進 settings 表後以 DB 為準」。

- [ ] **Step 2: 全量驗證**

Run: `cd sales-order-backend && task test:unit && go build ./...`
Expected: 全部 PASS、build 通過。

- [ ] **Step 3: 啟動冒煙**

Run: `cd sales-order-backend && task infra:start && task dev`（背景啟動）→ 另開終端 `curl -s -X GET localhost:3080/api/v1/settings`（未登入應 401）；登入後 GET 應回傳遮罩 JSON。
（若無測試帳號可跳過登入驗證，以 `task test` 通過為準。）

- [ ] **Step 4: Commit**

```bash
git add docs/AGENTS/backend.md && git commit -m "docs(backend): add settings domain guide"
```
（於 superproject root 執行；backend submodule 已各自 commit。）
