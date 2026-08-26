# Settings 頁面 Sessions 設定 Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** 將 `SESSION_DURATION`/`SESSION_IDLE_TIMEOUT`/`SESSION_SLIDING`/`SESSION_WARN_BEFORE` 從 env 搬入 settings 表，`/admin/setting` 可編輯且儲存後即時生效。

**Architecture:** Backend `FieldRegistry` 新增 4 欄（新 `field_type: "duration"`，Go duration 字串）；`Seed` 改為補缺（backfill）；啟動時從 DB 解析 session 參數建構 `scs.SessionManager`；settings `Update` 成功後經注入的 applier 即時寫回 `manager.Lifetime`/`IdleTimeout`。Frontend 設定頁加「Session 設定」群組；`SessionExpiryDialog` 改從 settings 讀 `session_warn_before`。

**Tech Stack:** Go (scs/v2, ent), SolidJS + TanStack Query, Vitest, testify + testcontainers.

**Spec:** `docs/superpowers/specs/2026-08-26-settings-sessions-design.md`

## Global Constraints

- 時間長度值一律為 Go duration 字串（`72h`、`5m`、`1h30m`），後端以 `time.ParseDuration` 驗證且必須 `> 0`。
- Cookie 結構參數（`SESSION_NAME`/`PATH`/`DOMAIN`/`HTTP_ONLY`/`SECURE`/`SAME_SITE`）維持 env，不在本計畫變更。
- 即時生效採直接寫入 `scs.SessionManager.Lifetime/IdleTimeout`；scs 讀取無鎖的 data race 為已知可接受風險（spec §3.4），不加同步層。
- Backend 程式碼在 submodule `sales-order-backend/`；frontend 在 `sales-order-frontend/`；各自獨立 commit。
- UI 標籤用繁體中文；commit message 用 conventional commits。
- Backend 測試指令：`cd sales-order-backend && go test ./internal/domain/settings/ -v`（含 testcontainers 的測試需 Docker；`testing.Short()` 會跳過 integration）。
- Frontend 測試指令：`cd sales-order-frontend && pnpm exec vitest run <file>`。

---

### Task 1: Backend — FieldRegistry 新增 session 欄位 + duration 驗證

**Files:**
- Modify: `sales-order-backend/internal/domain/settings/fields.go`
- Modify: `sales-order-backend/internal/domain/settings/validation.go`
- Test: `sales-order-backend/internal/domain/settings/fields_test.go`
- Test: `sales-order-backend/internal/domain/settings/validation_test.go`
- Modify: `sales-order-backend/internal/domain/settings/repository_test.go`（31→35 斷言）

**Interfaces:**
- Produces: `FieldRegistry` 新增 `session_duration`/`session_idle_timeout`/`session_sliding`/`session_warn_before`（Task 2-4 依賴）；`validateFieldValue` 接受 `field_type == "duration"`。

- [ ] **Step 1: 寫失敗測試 — registry 完整性與 duration 驗證**

`fields_test.go` 修改兩處：

```go
	require.Len(t, FieldRegistry, 35)
```

`switch f.FieldType` 的合法清單加 `"duration"`：

```go
		case "int64", "string", "secret", "bool", "duration":
```

`repository_test.go`：`require.Len(t, rows, 31)` 改 `require.Len(t, rows, 35)`，註解 `31 列` 改 `35 列`。

`validation_test.go` 新增（`strPtr` 已存在於同 package 測試）：

```go
func TestValidateFieldValue_Duration(t *testing.T) {
	for _, ok := range []string{"72h", "5m", "1h30m", "168h"} {
		require.NoError(t, validateFieldValue("session_duration", "duration", strPtr(ok)), ok)
	}
	for _, bad := range []string{"", "abc", "0s", "-5m", "72"} {
		require.ErrorIs(t, validateFieldValue("session_duration", "duration", strPtr(bad)), ErrValidation, bad)
	}
	// duration 欄位不接受 null（非 secret）
	require.ErrorIs(t, validateFieldValue("session_duration", "duration", nil), ErrValidation)
}
```

- [ ] **Step 2: 跑測試確認失敗**

Run: `cd sales-order-backend && go test ./internal/domain/settings/ -run 'TestFieldRegistry_Integrity|TestValidateFieldValue_Duration' -v`
Expected: FAIL（registry 長度 31≠35、未知 field_type "duration"）

- [ ] **Step 3: 實作 — fields.go 新增 4 欄**

`fields.go` `FieldRegistry` 末尾（`email_from` 列之後）新增，並把註解 `31 欄` 改 `35 欄`：

```go
	{"session_duration", "Session 絕對壽命", "duration", "到期後需重新登入", "72h"},
	{"session_idle_timeout", "Session 閒置逾時", "duration", "滑動續期啟用時生效", "168h"},
	{"session_sliding", "滑動續期", "bool", "活動中的 session 不被固定期限登出", "true"},
	{"session_warn_before", "到期提醒提前時間", "duration", "Web 到期提醒；App 用自身 flavor 值", "5m"},
```

- [ ] **Step 4: 實作 — validation.go duration case**

`validation.go` import 加 `"time"`；`switch fieldType` 的 `case "string":` 之前插入：

```go
	case "duration":
		d, err := time.ParseDuration(v)
		if err != nil || d <= 0 {
			return fmt.Errorf("%w: %s 需為有效 duration（如 72h、5m）且大於 0", ErrValidation, fieldID)
		}
```

（空字串會被 `time.ParseDuration` 擋下，不需加進 `requiredFields`。）

- [ ] **Step 5: 跑測試確認通過**

Run: `cd sales-order-backend && go test ./internal/domain/settings/ -run 'TestFieldRegistry_Integrity|TestValidateFieldValue' -v`
Expected: PASS

- [ ] **Step 6: Commit**

```bash
cd sales-order-backend
git add internal/domain/settings/fields.go internal/domain/settings/validation.go internal/domain/settings/fields_test.go internal/domain/settings/validation_test.go internal/domain/settings/repository_test.go
git commit -m "feat(settings): 新增 session_* 欄位與 duration field_type 驗證"
```

---

### Task 2: Backend — Seed 改為補缺（backfill）

**Files:**
- Modify: `sales-order-backend/internal/domain/settings/seed.go`
- Test: `sales-order-backend/internal/domain/settings/seed_test.go`

**Interfaces:**
- Consumes: Task 1 的 `FieldRegistry`（35 欄）。
- Produces: `Seed(ctx, client, nscfg, emailcfg, frontendURL)` 簽名不變；語意改為「只插入 registry 中缺漏的 field_id，不覆寫既有列」。`server.go`（Task 3）與既有呼叫端不需改簽名。

- [ ] **Step 1: 寫失敗測試 — 補缺不覆寫**

`seed_test.go`：既有測試的 `require.Len(t, rows, 31)` 改 `35`，註解 `31 列` 改 `35 列`。新增：

```go
// TestSeed_BackfillsMissingFields 模擬舊部署（已有部分列）：
// Seed 只補上缺漏的 session_* 欄位，既有列的值不被覆寫。
func TestSeed_BackfillsMissingFields(t *testing.T) {
	lc := testcontainers.New()
	defCtx := t.Context()
	db := func() error {
		return lc.ContainerDB(defCtx, &testcontainers.ContainerNames{
			ContainerName: "settings_backfill_test_db",
			NetworkName:   "settings_backfill_test_network",
			VolumeName:    "settings_backfill_test_vol",
		})
	}
	require.NoError(t, lc.CreateContainers(defCtx, db))
	defer lc.Close(defCtx)

	client := lc.DBClient()
	require.NoError(t, lc.CreateMigration(defCtx, client))

	// 舊部署遺留列（值已被管理者改過）
	require.NoError(t, client.Setting.Create().
		SetFieldID("default_department_id").SetName("預設部門 ID").
		SetFieldType("int64").SetDesc("").SetValue("99").Exec(defCtx))

	require.NoError(t, Seed(defCtx, client, config.NetSuite{}, config.Email{}, ""))

	rows, err := client.Setting.Query().All(defCtx)
	require.NoError(t, err)
	require.Len(t, rows, 35) // 1 舊列 + 34 補灌

	// 既有列不覆寫
	require.Equal(t, "99", fieldValue(wrapSchemas(rows), "default_department_id"))
	// 新欄位以 registry 預設補上
	require.Equal(t, "72h", fieldValue(wrapSchemas(rows), "session_duration"))
	require.Equal(t, "true", fieldValue(wrapSchemas(rows), "session_sliding"))
	require.Equal(t, "5m", fieldValue(wrapSchemas(rows), "session_warn_before"))
}
```

- [ ] **Step 2: 跑測試確認失敗**

Run: `cd sales-order-backend && go test ./internal/domain/settings/ -run TestSeed_BackfillsMissingFields -v`
Expected: FAIL（現行 seed-once：count>0 → no-op，只有 1 列）

- [ ] **Step 3: 實作 — Seed 補缺**

`seed.go` 的 `Seed` 函式整體改寫（`envByID` 保留；doc comment 同步更新）：

```go
// Seed 依 FieldRegistry 補上缺漏的 settings 欄位列（backfill）：
// 已存在的 field_id 一律不動（不覆寫管理者已修改的值）。
// env 覆寫：netsuite_*（nscfg）/ email_*（emailcfg）/ frontend_url（frontendURL），
// 僅在「該欄位本次需插入」且對應值非空時覆寫 registry 預設。
func Seed(ctx context.Context, client *gen.Client, nscfg config.NetSuite, emailcfg config.Email, frontendURL string) error {
	rows, err := client.Setting.Query().All(ctx)
	if err != nil {
		return err
	}
	existing := make(map[string]bool, len(rows))
	for _, r := range rows {
		existing[r.FieldID] = true
	}
	envByID := map[string]string{
		"netsuite_account_id":      nscfg.ACCOUNT_ID,
		"netsuite_consumer_key":    nscfg.CONSUMER_KEY,
		"netsuite_consumer_secret": nscfg.CONSUMER_SECRET,
		"netsuite_token_id":        nscfg.TOKEN_ID,
		"netsuite_token_secret":    nscfg.TOKEN_SECRET,
		"email_host":               emailcfg.Host,
		"email_port":               emailcfg.Port,
		"email_identity":           emailcfg.Identity,
		"email_username":           emailcfg.Username,
		"email_password":           emailcfg.Password,
		"email_from":               emailcfg.From,
		"frontend_url":             frontendURL,
	}
	builders := make([]*gen.SettingCreate, 0, len(FieldRegistry))
	for _, f := range FieldRegistry {
		if existing[f.FieldID] {
			continue // 既有列不覆寫
		}
		v := f.Default
		if ev, ok := envByID[f.FieldID]; ok && ev != "" {
			v = ev
		}
		builders = append(builders, client.Setting.Create().
			SetFieldID(f.FieldID).SetName(f.Name).SetFieldType(f.FieldType).
			SetDesc(f.Desc).SetValue(v))
	}
	if len(builders) == 0 {
		return nil
	}
	// OnConflictColumns(field_id).Ignore()：兩個實例同時補灌時，輸方 insert 命中
	// field_id unique index 會被忽略（而非 unique violation → server.go log.Fatalf 崩潰）。
	return client.Setting.CreateBulk(builders...).
		OnConflictColumns(setting.FieldFieldID).
		Ignore().
		Exec(ctx)
}
```

- [ ] **Step 4: 跑測試確認通過**

Run: `cd sales-order-backend && go test ./internal/domain/settings/ -run TestSeed -v`
Expected: PASS（`TestSeed_CreatesRowsFromRegistry` 與 `TestSeed_BackfillsMissingFields`）

- [ ] **Step 5: Commit**

```bash
cd sales-order-backend
git add internal/domain/settings/seed.go internal/domain/settings/seed_test.go
git commit -m "feat(settings): Seed 改為依 field_id 補缺，支援新增欄位 backfill"
```

---

### Task 3: Backend — SessionSettings 解析/套用 + server 啟動接線 + env cutover

**Files:**
- Create: `sales-order-backend/internal/domain/settings/session.go`
- Test: `sales-order-backend/internal/domain/settings/session_test.go`
- Modify: `sales-order-backend/internal/server/server.go`（Server struct、`newDatabase`、`newAuthentication`）
- Modify: `sales-order-backend/config/cookie.go`（移除 4 欄）
- Modify: `sales-order-backend/hexagon.env`（移除 4 個 env）

**Interfaces:**
- Produces:
  - `type SessionSettings struct { Duration, IdleTimeout, WarnBefore time.Duration; Sliding bool }`
  - `func ParseSessionSettings(rows []*SettingSchema) (SessionSettings, error)`
  - `func (ss SessionSettings) Apply(m *scs.SessionManager)`
  - `func hasSessionUpdate(fields []SettingFieldResponse) bool`（Task 4 使用）
  - `Server.sessionSettings settings.SessionSettings`

- [ ] **Step 1: 寫失敗測試**

建立 `session_test.go`：

```go
package settings

import (
	"testing"
	"time"

	"github.com/alexedwards/scs/v2"
	"github.com/hexagon-maker/sales-order-backend/ent/gen"
	"github.com/stretchr/testify/require"
)

func sessionRows(duration, idle, sliding, warn string) []*SettingSchema {
	return []*SettingSchema{
		{Setting: &gen.Setting{FieldID: "session_duration", FieldType: "duration", Value: duration}},
		{Setting: &gen.Setting{FieldID: "session_idle_timeout", FieldType: "duration", Value: idle}},
		{Setting: &gen.Setting{FieldID: "session_sliding", FieldType: "bool", Value: sliding}},
		{Setting: &gen.Setting{FieldID: "session_warn_before", FieldType: "duration", Value: warn}},
	}
}

func TestParseSessionSettings(t *testing.T) {
	ss, err := ParseSessionSettings(sessionRows("72h", "168h", "true", "5m"))
	require.NoError(t, err)
	require.Equal(t, 72*time.Hour, ss.Duration)
	require.Equal(t, 168*time.Hour, ss.IdleTimeout)
	require.True(t, ss.Sliding)
	require.Equal(t, 5*time.Minute, ss.WarnBefore)

	// 非法 duration → 報錯並帶欄位名
	_, err = ParseSessionSettings(sessionRows("abc", "168h", "true", "5m"))
	require.ErrorContains(t, err, "session_duration")

	// 缺欄位（fieldValue 回傳 ""）→ 報錯
	_, err = ParseSessionSettings(nil)
	require.Error(t, err)
}

func TestSessionSettingsApply(t *testing.T) {
	m := scs.New()
	SessionSettings{Duration: 72 * time.Hour, IdleTimeout: 168 * time.Hour, Sliding: true}.Apply(m)
	require.Equal(t, 72*time.Hour, m.Lifetime)
	require.Equal(t, 168*time.Hour, m.IdleTimeout)

	// sliding=false → IdleTimeout 歸 0（固定壽命，與 server.go 語意一致）
	SessionSettings{Duration: 24 * time.Hour, IdleTimeout: 168 * time.Hour, Sliding: false}.Apply(m)
	require.Equal(t, 24*time.Hour, m.Lifetime)
	require.Zero(t, m.IdleTimeout)
}

func TestHasSessionUpdate(t *testing.T) {
	require.True(t, hasSessionUpdate([]SettingFieldResponse{{FieldID: "session_sliding"}}))
	require.False(t, hasSessionUpdate([]SettingFieldResponse{{FieldID: "default_vendor_id"}}))
}
```

- [ ] **Step 2: 跑測試確認失敗**

Run: `cd sales-order-backend && go test ./internal/domain/settings/ -run 'TestParseSessionSettings|TestSessionSettingsApply|TestHasSessionUpdate' -v`
Expected: FAIL（undefined: ParseSessionSettings 等）

- [ ] **Step 3: 實作 session.go**

```go
package settings

import (
	"fmt"
	"strings"
	"time"

	"github.com/alexedwards/scs/v2"
)

// SessionSettings 為 session 行為參數。settings 表為單一來源
// （取代 SESSION_DURATION / SESSION_IDLE_TIMEOUT / SESSION_SLIDING / SESSION_WARN_BEFORE env）。
type SessionSettings struct {
	Duration    time.Duration // session 絕對壽命
	IdleTimeout time.Duration // 閒置逾時（Sliding 啟用時生效）
	Sliding     bool          // 滑動續期
	WarnBefore  time.Duration // 到期提醒提前時間（目前僅 Web 前端消費）
}

// ParseSessionSettings 從 settings 列解析 session 參數；缺欄位或非法 duration 回傳錯誤。
func ParseSessionSettings(rows []*SettingSchema) (SessionSettings, error) {
	var ss SessionSettings
	var err error
	if ss.Duration, err = time.ParseDuration(fieldValue(rows, "session_duration")); err != nil {
		return ss, fmt.Errorf("session_duration: %w", err)
	}
	if ss.IdleTimeout, err = time.ParseDuration(fieldValue(rows, "session_idle_timeout")); err != nil {
		return ss, fmt.Errorf("session_idle_timeout: %w", err)
	}
	ss.Sliding = fieldValue(rows, "session_sliding") == "true"
	if ss.WarnBefore, err = time.ParseDuration(fieldValue(rows, "session_warn_before")); err != nil {
		return ss, fmt.Errorf("session_warn_before: %w", err)
	}
	return ss, nil
}

// Apply 將參數即時寫入 scs.SessionManager。
// 已知風險（spec §3.4）：scs 對 Lifetime/IdleTimeout 的讀取無鎖，此處寫入屬技術性
// data race；64-bit 平台對齊 int64 讀寫實務原子、寫入者僅管理員單次操作，接受。
func (ss SessionSettings) Apply(m *scs.SessionManager) {
	m.Lifetime = ss.Duration
	if ss.Sliding {
		m.IdleTimeout = ss.IdleTimeout
	} else {
		m.IdleTimeout = 0
	}
}

// hasSessionUpdate 判斷 settings PUT payload 是否含 session_* 欄位。
func hasSessionUpdate(fields []SettingFieldResponse) bool {
	for _, f := range fields {
		if strings.HasPrefix(f.FieldID, "session_") {
			return true
		}
	}
	return false
}
```

- [ ] **Step 4: 跑測試確認通過**

Run: `cd sales-order-backend && go test ./internal/domain/settings/ -run 'TestParseSessionSettings|TestSessionSettingsApply|TestHasSessionUpdate' -v`
Expected: PASS

- [ ] **Step 5: server.go 接線**

`Server` struct（`sessionCloser` 之後）加欄位：

```go
	sessionSettings settings.SessionSettings
```

`newDatabase()` 末尾（`s.cfg.Email = settings.ToEmailConfig(srows)` 之後）加：

```go
	// session 行為參數以 settings 表為單一來源（非 env）
	ss, err := settings.ParseSessionSettings(srows)
	if err != nil {
		log.Fatalf("failed to parse session settings: %v", err)
	}
	s.sessionSettings = ss
```

`newAuthentication()` 中：

```go
	manager.Lifetime = s.cfg.Session.Duration
	// 滑動續期：啟用時設定 IdleTimeout，活動中的 session 不會在固定期限被登出
	if s.cfg.Session.Sliding {
		manager.IdleTimeout = s.cfg.Session.IdleTimeout
	}
```

改為：

```go
	manager.Lifetime = s.sessionSettings.Duration
	// 滑動續期：啟用時設定 IdleTimeout，活動中的 session 不會在固定期限被登出
	if s.sessionSettings.Sliding {
		manager.IdleTimeout = s.sessionSettings.IdleTimeout
	}
```

- [ ] **Step 6: config/cookie.go 移除 4 欄**

`Session` struct 刪除 `Duration`、`IdleTimeout`、`Sliding`、`WarnBefore` 四行，保留：

```go
type Session struct {
	Name     string          `split_words:"true" default:"__session"`
	Path     string          `default:"/"`
	Domain   string          `default:""`
	Secret   string          `default:"true"`
	HttpOnly bool            `split_words:"true" default:"true"`
	Secure   bool            `split_words:"true" default:"true"`
	SameSite SameSiteDecoder `split_words:"true" default:"lax"`
}
```

- [ ] **Step 7: hexagon.env 移除 4 個 env**

刪除以下行（含各自註解行）：

```
# SESSION_DURATION：session 絕對壽命，到期後需重新登入
SESSION_DURATION=72h
# SESSION_IDLE_TIMEOUT：滑動續期啟用時的閒置逾時
SESSION_IDLE_TIMEOUT=168h
# SESSION_SLIDING：啟用滑動續期
SESSION_SLIDING=true
# SESSION_WARN_BEFORE：到期前提醒的提前時間
SESSION_WARN_BEFORE=5m
```

- [ ] **Step 8: 編譯 + 全 settings 測試**

Run: `cd sales-order-backend && go build ./... && go test ./internal/domain/settings/ -v`
Expected: build 成功；測試 PASS

- [ ] **Step 9: Commit**

```bash
cd sales-order-backend
git add internal/domain/settings/session.go internal/domain/settings/session_test.go internal/server/server.go config/cookie.go hexagon.env
git commit -m "feat(settings): session 行為參數改由 settings 表提供，移除對應 env"
```

---

### Task 4: Backend — PUT 後即時套用（applier hook）

**Files:**
- Modify: `sales-order-backend/internal/domain/settings/usecase.go`
- Test: `sales-order-backend/internal/domain/settings/usecase_test.go`
- Modify: `sales-order-backend/internal/server/initDomains.go`（`initSettings`）

**Interfaces:**
- Consumes: Task 3 的 `SessionSettings`、`ParseSessionSettings`、`hasSessionUpdate`、`Apply`。
- Produces: `type SessionApplier func(SessionSettings)`；`NewUseCase(repo IRepository, appliers ...SessionApplier) IUseCase`（ variadic，既有呼叫端 `NewUseCase(repo)` 不需改）。

- [ ] **Step 1: 寫失敗測試**

`usecase_test.go` 新增（import 加 `"time"`）：

```go
func TestUpdate_AppliesSessionSettings(t *testing.T) {
	var got SessionSettings
	uc := NewUseCase(repoFuncs{list: listOf(
		&SettingSchema{Setting: &gen.Setting{FieldID: "session_duration", FieldType: "duration", Value: "90s"}},
		&SettingSchema{Setting: &gen.Setting{FieldID: "session_idle_timeout", FieldType: "duration", Value: "10m"}},
		&SettingSchema{Setting: &gen.Setting{FieldID: "session_sliding", FieldType: "bool", Value: "true"}},
		&SettingSchema{Setting: &gen.Setting{FieldID: "session_warn_before", FieldType: "duration", Value: "5m"}},
	)}, func(ss SessionSettings) { got = ss })

	_, err := uc.Update(context.Background(), []SettingFieldResponse{
		{FieldID: "session_duration", Value: strPtr("90s")},
	}, true)
	require.NoError(t, err)
	require.Equal(t, 90*time.Second, got.Duration)
	require.Equal(t, 10*time.Minute, got.IdleTimeout)
	require.True(t, got.Sliding)
	require.Equal(t, 5*time.Minute, got.WarnBefore)
}

func TestUpdate_NoSessionFields_ApplierNotCalled(t *testing.T) {
	called := false
	uc := NewUseCase(repoFuncs{list: listOf(
		&SettingSchema{Setting: &gen.Setting{FieldID: "default_vendor_id", FieldType: "int64", Value: "807"}},
	)}, func(ss SessionSettings) { called = true })

	_, err := uc.Update(context.Background(), []SettingFieldResponse{
		{FieldID: "default_vendor_id", Value: strPtr("888")},
	}, true)
	require.NoError(t, err)
	require.False(t, called)
}
```

- [ ] **Step 2: 跑測試確認失敗**

Run: `cd sales-order-backend && go test ./internal/domain/settings/ -run 'TestUpdate_AppliesSessionSettings|TestUpdate_NoSessionFields' -v`
Expected: FAIL（applier 未被呼叫，`got` 為零值 / `called` false 斷言失敗；或 NewUseCase 參數不符編譯錯）

- [ ] **Step 3: 實作 usecase.go**

```go
// SessionApplier 於 session_* 欄位更新成功後套用新值（如即時寫入 scs.SessionManager）。
type SessionApplier func(SessionSettings)

type usecase struct {
	repo     IRepository
	appliers []SessionApplier
}

func NewUseCase(repo IRepository, appliers ...SessionApplier) IUseCase {
	return &usecase{repo: repo, appliers: appliers}
}
```

`Update` 的結尾：

```go
	if err := u.repo.BatchUpsert(ctx, rows); err != nil {
		return nil, err
	}
	return u.Get(ctx)
```

改為：

```go
	if err := u.repo.BatchUpsert(ctx, rows); err != nil {
		return nil, err
	}
	updated, err := u.Get(ctx)
	if err != nil {
		return nil, err
	}
	// session_* 欄位更新 → 即時套用（值已過 validateFieldValue，Parse 理論不會失敗）
	if len(u.appliers) > 0 && hasSessionUpdate(fields) {
		ss, err := ParseSessionSettings(updated)
		if err != nil {
			return nil, err
		}
		for _, apply := range u.appliers {
			apply(ss)
		}
	}
	return updated, nil
```

- [ ] **Step 4: 跑測試確認通過**

Run: `cd sales-order-backend && go test ./internal/domain/settings/ -run TestUpdate -v`
Expected: PASS（含既有 Update 測試——`NewUseCase(repo)` 舊呼叫因 variadic 不需改）

- [ ] **Step 5: initDomains.go 注入 applier**

`initSettings()` 改為：

```go
func (s *Server) initSettings() {
	repo := settings.NewRepository(s.ent)
	uc := settings.NewUseCase(repo, func(ss settings.SessionSettings) {
		ss.Apply(s.session) // PUT settings 後即時生效，不需重啟
	})
	settings.RegisterHTTPEndPoints(s.router, s.session, uc)
}
```

（`Init()` 順序為 `newAuthentication()` → `InitDomains()`，`s.session` 此時已建立。）

- [ ] **Step 6: 編譯 + 全測試**

Run: `cd sales-order-backend && go build ./... && go test ./internal/domain/settings/ -v`
Expected: PASS

- [ ] **Step 7: Commit**

```bash
cd sales-order-backend
git add internal/domain/settings/usecase.go internal/domain/settings/usecase_test.go internal/server/initDomains.go
git commit -m "feat(settings): PUT session_* 後即時套用至 SessionManager"
```

---

### Task 5: Backend — 文件更新（README + AGENTS）

**Files:**
- Modify: `sales-order-backend/README.md`
- Modify: `docs/AGENTS/backend.md`（superproject；若有 SESSION_* env 說明）

**Interfaces:**
- Consumes: Task 1-4 的最終行為。

- [ ] **Step 1: README session 章節改寫**

`README.md` 中 session env 說明段（含 `SESSION_DURATION=72h`、`SESSION_SLIDING=true`、`SESSION_IDLE_TIMEOUT=168h`、`SESSION_WARN_BEFORE=5m`、`若停用滑動續期…` 等 bullets）替換為：

```markdown
- Session 行為參數（絕對壽命、閒置逾時、滑動續期、到期提醒提前時間）由 settings 表管理，
  於 Web 設定頁（/admin/setting「Session 設定」）修改，儲存後即時生效，不需重啟。
  對應 field_id：`session_duration`、`session_idle_timeout`、`session_sliding`、`session_warn_before`
  （值為 Go duration 字串 / `true`|`false`）。
- 行為：只要持續使用就不會登出；停止使用超過 `session_idle_timeout` 才需重新登入。
- `/me` 回應含 `session_expires_at`（RFC3339，伺服器時間），供前端/App 計算到期提醒。
- Cookie 結構參數（`SESSION_NAME`/`PATH`/`DOMAIN`/`HTTP_ONLY`/`SECURE`/`SAME_SITE`）仍為 env。

若停用滑動續期（`session_sliding=false`），session 將在 `session_duration` 到期後立即失效，回復固定壽命行為。
```

「測「到期提醒 / 401」的項目需縮短 `SESSION_DURATION`…」一段改為：

```markdown
- 測「到期提醒 / 401」的項目可在設定頁把 `session_duration` 改短（建議 `90s`–`6m`），測完改回 `12h`；儲存即時生效，不需重建。
```

- [ ] **Step 2: docs/AGENTS/backend.md 同步**

在 superproject 的 `docs/AGENTS/backend.md` 中找到 session/env 相關段落，將四個 env 的敘述改為指向 settings 表（field_id 同上），並註明 cookie 參數仍為 env。若該文件無 SESSION_* 敘述，於 settings 相關段落補一句：session 行為參數（`session_duration`/`session_idle_timeout`/`session_sliding`/`session_warn_before`）由 settings 表管理、PUT 即時生效。

- [ ] **Step 3: Commit**

```bash
cd sales-order-backend && git add README.md && git commit -m "docs: session 行為參數改由 settings 表管理"
cd /Volumes/UTM2/Developer/sales-order-netsuite && git add docs/AGENTS/backend.md && git commit -m "docs(backend): session 參數來源更新為 settings 表"
```

---

### Task 6: Frontend — 型別、fallback、設定頁群組

**Files:**
- Modify: `sales-order-frontend/src/models/settings.ts`
- Modify: `sales-order-frontend/src/constant/options.ts`（FALLBACK_SETTINGS）
- Modify: `sales-order-frontend/src/pages/admin/setting/Setting.tsx`（FIELD_GROUPS）
- Test: `sales-order-frontend/src/pages/admin/setting/Setting.test.tsx`

**Interfaces:**
- Produces: `SettingFieldType` 含 `"duration"`；「Session 設定」群組渲染 4 個 text input（`session_sliding` 走既有 bool 文字輸入 + true/false toast 驗證）。

- [ ] **Step 1: 修改測試（先紅）— 群組數 5→6**

`Setting.test.tsx`：

```ts
const GROUP_TITLES = ["系統常數", "App 設定", "Session 設定", "前端 URL", "NetSuite 憑證", "EMAIL 設定"];
```

測試名稱 `"superadmin: renders all 5 group titles and masked placeholders for unset secrets"` 改為 `"superadmin: renders all 6 group titles and masked placeholders for unset secrets"`。

save 測試內加斷言（fallback 值原樣送出）：

```ts
    expect(valueOf("session_duration")).toBe("72h");
    expect(valueOf("session_sliding")).toBe("true");
```

- [ ] **Step 2: 跑測試確認失敗**

Run: `cd sales-order-frontend && pnpm exec vitest run src/pages/admin/setting/Setting.test.tsx`
Expected: FAIL（找不到「Session 設定」標題）

- [ ] **Step 3: 實作 — 型別 + fallback + 群組**

`src/models/settings.ts`：

```ts
export type SettingFieldType = "int64" | "string" | "secret" | "bool" | "duration";
```

`src/constant/options.ts` `FALLBACK_SETTINGS.fields` 末尾（`email_from` 列之後）加：

```ts
    { field_id: "session_duration", name: "Session 絕對壽命", field_type: "duration", desc: "到期後需重新登入", value: "72h" },
    { field_id: "session_idle_timeout", name: "Session 閒置逾時", field_type: "duration", desc: "滑動續期啟用時生效", value: "168h" },
    { field_id: "session_sliding", name: "滑動續期", field_type: "bool", desc: "活動中的 session 不被固定期限登出", value: "true" },
    { field_id: "session_warn_before", name: "到期提醒提前時間", field_type: "duration", desc: "Web 到期提醒；App 用自身 flavor 值", value: "5m" },
```

`src/pages/admin/setting/Setting.tsx` `FIELD_GROUPS` 在「App 設定」群組之後插入：

```ts
  { title: "Session 設定", fieldIds: ["session_duration", "session_idle_timeout", "session_sliding", "session_warn_before"] },
```

（duration 欄位落入既有非 secret 分支渲染 text input；`save()` 的 bool 分支已處理 `session_sliding`；duration 值由後端 422 toast 把關，不加前端專屬驗證。）

- [ ] **Step 4: 跑測試確認通過**

Run: `cd sales-order-frontend && pnpm exec vitest run src/pages/admin/setting/Setting.test.tsx`
Expected: PASS（6 個群組標題、secret 數量 6 不變）

- [ ] **Step 5: Commit**

```bash
cd sales-order-frontend
git add src/models/settings.ts src/constant/options.ts src/pages/admin/setting/Setting.tsx src/pages/admin/setting/Setting.test.tsx
git commit -m "feat(setting): 設定頁新增 Session 設定群組"
```

---

### Task 7: Frontend — SessionExpiryDialog 改用 session_warn_before

**Files:**
- Create: `sales-order-frontend/src/lib/setting/duration.ts`
- Test: `sales-order-frontend/src/lib/setting/duration.test.ts`
- Modify: `sales-order-frontend/src/lib/setting/index.ts`（export）
- Modify: `sales-order-frontend/src/components/session/SessionExpiryDialog.tsx`
- Modify: `docs/AGENTS/frontend.md`（superproject；設定頁段落）

**Interfaces:**
- Consumes: `useSettingsValue()`（既有，簽名 `(enabled?) => (field_id) => string | null`）。
- Produces: `parseGoDuration(s: string): number | null`（毫秒；非法/空字串回傳 `null`），自 `~/lib/setting` 匯出。

- [ ] **Step 1: 寫失敗測試**

`src/lib/setting/duration.test.ts`：

```ts
import { describe, expect, it } from "vitest";
import { parseGoDuration } from "./duration";

describe("parseGoDuration", () => {
  it("parses simple units to milliseconds", () => {
    expect(parseGoDuration("5m")).toBe(300_000);
    expect(parseGoDuration("72h")).toBe(259_200_000);
    expect(parseGoDuration("90s")).toBe(90_000);
    expect(parseGoDuration("500ms")).toBe(500);
  });

  it("parses compound durations", () => {
    expect(parseGoDuration("1h30m")).toBe(5_400_000);
  });

  it("returns null for invalid or empty input", () => {
    expect(parseGoDuration("")).toBeNull();
    expect(parseGoDuration("abc")).toBeNull();
    expect(parseGoDuration("72")).toBeNull();
    expect(parseGoDuration("5x")).toBeNull();
  });
});
```

- [ ] **Step 2: 跑測試確認失敗**

Run: `cd sales-order-frontend && pnpm exec vitest run src/lib/setting/duration.test.ts`
Expected: FAIL（找不到模組 ./duration）

- [ ] **Step 3: 實作 duration.ts**

`src/lib/setting/duration.ts`：

```ts
// Go time.ParseDuration 格式（如 "72h"、"5m"、"1h30m"）轉毫秒。
// 非法或空字串回傳 null（呼叫端自行 fallback）。
const UNIT_MS: Record<string, number> = {
  ns: 1e-6,
  us: 1e-3,
  "µs": 1e-3,
  ms: 1,
  s: 1_000,
  m: 60_000,
  h: 3_600_000,
};

export const parseGoDuration = (s: string): number | null => {
  const t = s.trim();
  if (t === "") return null;
  const re = /(\d+(?:\.\d+)?)(ns|us|µs|ms|s|m|h)/g;
  let total = 0;
  let matched = 0;
  let m: RegExpExecArray | null;
  while ((m = re.exec(t)) !== null) {
    total += parseFloat(m[1]) * UNIT_MS[m[2]];
    matched += m[0].length;
  }
  return matched > 0 && matched === t.length ? total : null;
};
```

`src/lib/setting/index.ts` 末尾加：

```ts
export { parseGoDuration } from "./duration";
```

- [ ] **Step 4: 跑測試確認通過**

Run: `cd sales-order-frontend && pnpm exec vitest run src/lib/setting/duration.test.ts`
Expected: PASS

- [ ] **Step 5: SessionExpiryDialog 接線**

`src/components/session/SessionExpiryDialog.tsx`：

刪除：

```ts
const WARN_BEFORE_MS = 5 * 60 * 1000; // SESSION_WARN_BEFORE：到期前 5 分鐘提醒
```

import 區加：

```ts
import { parseGoDuration, useSettingsValue } from "~/lib/setting";
```

元件內（`const [authState, { setAuthInfo }] = useAuth();` 之後）加：

```ts
const settingsValue = useSettingsValue();
// 到期提醒提前時間由 settings 表 session_warn_before 提供；解析失敗/未就緒 fallback 5 分鐘
const warnBeforeMs = () =>
  parseGoDuration(settingsValue("session_warn_before") ?? "") ?? 5 * 60 * 1000;
```

`showDialog` 中 `remaining()! <= WARN_BEFORE_MS` 改為 `remaining()! <= warnBeforeMs()`。

- [ ] **Step 6: 全前端測試 + 型別檢查**

Run: `cd sales-order-frontend && pnpm exec vitest run && pnpm run build`
Expected: 測試 PASS；build 成功

- [ ] **Step 7: docs/AGENTS/frontend.md 設定頁段落更新**

在 superproject `docs/AGENTS/frontend.md` 的「設定頁（/admin/setting）」段落補充：`FIELD_GROUPS` 含「Session 設定」群組（4 個 `session_*` 欄位，duration 為 Go duration 字串）；`SessionExpiryDialog` 的提醒提前時間改由 `useSettingsValue("session_warn_before")` + `parseGoDuration` 取得，不再 hardcode。

- [ ] **Step 8: Commit**

```bash
cd sales-order-frontend
git add src/lib/setting/duration.ts src/lib/setting/duration.test.ts src/lib/setting/index.ts src/components/session/SessionExpiryDialog.tsx
git commit -m "feat(session): 到期提醒提前時間改由 settings session_warn_before 提供"
cd /Volumes/UTM2/Developer/sales-order-netsuite
git add docs/AGENTS/frontend.md
git commit -m "docs(frontend): 設定頁 Session 群組與 warn_before 來源說明"
```

---

### Task 8: 驗收 — E2E 煙霧測試（手動/本機）

**Files:** 無（執行驗證）

- [ ] **Step 1: 啟動 backend + frontend**

後端需可連 Postgres。啟動後於 DB 確認 settings 表含 4 個 `session_*` 列（舊 DB 由 backfill 補上）。

- [ ] **Step 2: 設定頁操作**

superadmin（ssd@sowinsoft.com）登入 → `/admin/setting` → 確認「Session 設定」群組顯示 4 欄及目前值；把 `session_duration` 改為 `90s` 儲存 → toast「設定已儲存」。

- [ ] **Step 3: 即時生效驗證**

不重启 backend，重新整理頁面呼叫 `/me`，確認 `session_expires_at` 距現在約 90 秒（表示 `Lifetime` 已即時更新）；輸入非法值（如 `abc`）儲存 → toast 顯示後端 422 錯誤訊息。

- [ ] **Step 4: 還原**

`session_duration` 改回 `72h` 儲存。
