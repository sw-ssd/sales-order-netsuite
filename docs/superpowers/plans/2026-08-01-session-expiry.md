# Session 體驗優化 Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** 讓「登入能維持多久」由設定明確決定：活動中的 session 滑動續期永不登出、閒置 30 分鐘失效、到期前 5 分鐘彈窗提醒、401 後重新登入還原原頁面。

**Architecture:** 後端 scs session（`IdleTimeout` + 客製 `LoadAndSave` 的 `SetDeadline` 延長絕對期限）提供滑動續期；`/me` 回應帶 `session_expires_at`（伺服器時間）；Web/App 各自依此計算提醒並呼叫 `/csrf` 續期；401 統一清 session 導回登入頁。

**Tech Stack:** Go（chi + scs/v2 v2.8.0）、SolidJS（TanStack Query + solid-primitives/storage）、Flutter（Dio + freezed + disco Provider）。

## Global Constraints

- 所有文件/註解/UI 文案使用繁體中文（zh-TW）；程式碼識別符與 API 保持英文。
- 不做「記住我」、不修 OAuth2 缺陷（另開 change）、不導入 refresh token。
- 提醒時間一律以伺服器 `session_expires_at` 為準，不用裝置本地絕對時刻。
- 環境變數命名沿用 `SESSION_*`（envconfig `split_words`）；文件說明用中文。
- 後端測試命令：`go test ./internal/domain/authentication/... ./internal/middleware/...`。
- 現況：`config/cookie.go` 已新增 `IdleTimeout/Sliding/RememberDuration/WarnBefore` 欄位、`server.go` 已接 `IdleTimeout`、`.env`/`hexagon.env`/`prod.env.hexagon` 已加變數（未 commit）。**`RememberDuration` 與相關 env 變數因「不做記住我」需移除。**

---

### Task 1: 後端設定收斂（移除 RememberDuration）

**Files:**
- Modify: `sales-order-backend/config/cookie.go`（Session struct）
- Modify: `sales-order-backend/.env`、`sales-order-backend/hexagon.env`、`sales-order-app/prod.env.hexagon`

**Interfaces:**
- Consumes: 無
- Produces: `config.Session{ Name, Path, Domain, Secret, Duration, IdleTimeout, Sliding, WarnBefore, HttpOnly, Secure, SameSite }`（無 `RememberDuration`）

- [ ] **Step 1: 從 `config/cookie.go` 移除 `RememberDuration` 欄位**

```go
type Session struct {
	Name             string          `split_words:"true" default:"__session"`
	Path             string          `default:"/"`
	Domain           string          `default:""`
	Secret           string          `default:"true"`
	Duration         time.Duration   `default:"24h"` // SESSION_DURATION：session 絕對壽命，到期後需重新登入
	IdleTimeout      time.Duration   `split_words:"true" default:"30m"` // SESSION_IDLE_TIMEOUT：閒置逾時（滑動續期啟用時生效）
	Sliding          bool            `default:"false"`                   // SESSION_SLIDING：啟用滑動續期
	WarnBefore       time.Duration   `split_words:"true" default:"5m"`   // SESSION_WARN_BEFORE：到期前提醒的提前時間
	HttpOnly         bool            `split_words:"true" default:"true"`
	Secure           bool            `default:"true"`
	SameSite         SameSiteDecoder `split_words:"true" default:"lax"`
}
```

- [ ] **Step 2: 從三個 env 檔移除 `SESSION_REMEMBER_DURATION` 行，並確認 `SESSION_DURATION` 統一為 `12h`**

`.env`、`hexagon.env` 移除：
```
SESSION_REMEMBER_DURATION=168h
```
`hexagon.env` 的 `SESSION_DURATION` 由 `168h` 改為 `12h`（滑動續期下對日常使用者無感，見設計文件待決問題 1）。`prod.env.hexagon`（App）移除 `SESSION_WARN_BEFORE=300` 該行保留（App 需要），確認無 `SESSION_REMEMBER_DURATION`。

- [ ] **Step 3: 驗證編譯**

Run: `cd sales-order-backend && go build ./config/ ./internal/server/`
Expected: 成功，無輸出錯誤。

- [ ] **Step 4: Commit**

```bash
git add config/cookie.go .env hexagon.env
git commit -m "feat(session): remove remember-me config, unify SESSION_DURATION to 12h"
```

---

### Task 2: 滑動續期邏輯設定驅動

**Files:**
- Modify: `sales-order-backend/internal/server/server.go:210-219`（newAuthentication）
- Modify: `sales-order-backend/internal/middleware/authentication.go:87-93`（LoadAndSave 的 sliding 區塊）

**Interfaces:**
- Consumes: `config.Session.Sliding`、`config.Session.IdleTimeout`、`config.Session.Duration`
- Produces: scs manager 設定 `IdleTimeout`；`LoadAndSave` 依 `s.IdleTimeout > 0` 延長絕對期限

- [ ] **Step 1: 確認 `server.go` 已接 IdleTimeout（先前已改，驗證）**

```go
manager.Lifetime = s.cfg.Session.Duration
// 滑動續期：啟用時設定 IdleTimeout，活動中的 session 不會在固定期限被登出
if s.cfg.Session.Sliding {
	manager.IdleTimeout = s.cfg.Session.IdleTimeout
}
```
確認存在；若無則補上。

- [ ] **Step 2: 修改 `LoadAndSave` 的 sliding 區塊（寫死 24h → idle 驅動）**

現況（`internal/middleware/authentication.go`）：
```go
// Sliding session: extend session TTL if remaining lifetime < 24h.
if deadline := s.Deadline(infoCtx); !deadline.IsZero() {
	if time.Until(deadline) < 24*time.Hour {
		s.SetDeadline(infoCtx, time.Now().Add(s.Lifetime))
	}
}
```
改為：
```go
// Sliding session: 啟用 IdleTimeout 時，剩餘壽命低於閒置逾時即延長絕對期限，
// 使活動中的 session 不會在固定期限被登出（scs 於每次請求重新 commit）。
if s.IdleTimeout > 0 {
	if deadline := s.Deadline(infoCtx); !deadline.IsZero() {
		if time.Until(deadline) < s.IdleTimeout {
			s.SetDeadline(infoCtx, time.Now().Add(s.Lifetime))
		}
	}
}
```

- [ ] **Step 3: 驗證編譯**

Run: `cd sales-order-backend && go build ./internal/middleware/ ./internal/server/`
Expected: 成功。

- [ ] **Step 4: Commit**

```bash
git add internal/middleware/authentication.go internal/server/server.go
git commit -m "feat(session): config-driven sliding renewal via scs IdleTimeout"
```

---

### Task 3: CSRF 期限與 session 到期同步

**Files:**
- Modify: `sales-order-backend/internal/domain/authentication/repository.go:235`

**Interfaces:**
- Consumes: `*scs.SessionManager`（`Deadline(ctx)`）
- Produces: CSRF token 的 store 期限等於 session 當前 deadline（滑動後同步）

- [ ] **Step 1: 修改 `repo.Csrf` 的 expiry 計算**

現況：
```go
err = c.CommitCtx(ctx, token, []byte("csrf_token"), time.Now().Add(r.session.Lifetime))
```
改為：
```go
expiry := time.Now().Add(r.session.Lifetime)
if d := r.session.Deadline(ctx); !d.IsZero() {
	expiry = d
}
err = c.CommitCtx(ctx, token, []byte("csrf_token"), expiry)
```

- [ ] **Step 2: 驗證編譯**

Run: `cd sales-order-backend && go build ./internal/domain/authentication/`
Expected: 成功。

- [ ] **Step 3: Commit**

```bash
git add internal/domain/authentication/repository.go
git commit -m "feat(session): sync CSRF token expiry with sliding session deadline"
```

---

### Task 4: `/me` 回應帶 `session_expires_at`

**Files:**
- Modify: `sales-order-backend/third_party/postgres_entstore/models.go:69-74`（SessionInfoResource）
- Modify: `sales-order-backend/internal/domain/authentication/handler_restricted.go`（Me handler）

**Interfaces:**
- Produces: `SessionInfoResource.SessionExpiresAt string \`json:"session_expires_at,omitempty"\``；`GET /api/v1/restricted/me` 回應含 RFC3339 到期時間

- [ ] **Step 1: `SessionInfoResource` 新增欄位**

```go
type SessionInfoResource struct {
	UserType         string               `json:"user_type,omitempty"` // "user" or "salesrep" or "customer"
	Salesrep         *SalesrepInfoSession `json:"salesrep,omitempty"`
	User             *UserInfoSession     `json:"user,omitempty"`
	Customer         *CustomerInfoSession `json:"customer,omitempty"`
	SessionExpiresAt string               `json:"session_expires_at,omitempty"` // RFC3339，伺服器時間
}
```

- [ ] **Step 2: `Me` handler 填入到期時間**

```go
func (h *Handler) Me(w http.ResponseWriter, r *http.Request) {
	ui, err := pe.GetContextToSessionInfo(r.Context(), h.session)
	if err != nil {
		slog.Error("failed to unmarshal session info", "error", err)
		respond.Error(w, http.StatusBadRequest, errors.New("you need to be logged in"))
		return
	}
	if d := h.session.Deadline(r.Context()); !d.IsZero() {
		ui.SessionExpiresAt = d.UTC().Format(time.RFC3339)
	}
	respond.Json(w, http.StatusOK, ui)
}
```
（需 import `time`。）

- [ ] **Step 3: 驗證編譯**

Run: `cd sales-order-backend && go build ./internal/domain/authentication/ ./third_party/postgres_entstore/`
Expected: 成功。

- [ ] **Step 4: Commit**

```bash
git add third_party/postgres_entstore/models.go internal/domain/authentication/handler_restricted.go
git commit -m "feat(session): expose session_expires_at in /me response"
```

---

### Task 5: 後端滑動續期測試

**Files:**
- Create: `sales-order-backend/internal/middleware/sliding_session_test.go`
- Test: `sales-order-backend/internal/middleware/`

**Interfaces:**
- Consumes: scs `SessionManager`（`IdleTimeout`、`LoadAndSave` 行為）、`testcontainers.NewSession`
- Produces: 驗證滑動續期與固定壽命的契約測試

- [ ] **Step 1: 寫失敗測試（滑動續期延長 deadline）**

```go
package middleware

import (
	"net/http"
	"net/http/httptest"
	"testing"
	"time"

	"github.com/alexedwards/scs/v2"
	"github.com/stretchr/testify/assert"
)

func TestLoadAndSave_SlidingExtendsDeadline(t *testing.T) {
	s := scs.New()
	s.Store = scs.NewMemStore()
	s.Lifetime = time.Hour
	s.IdleTimeout = 30 * time.Minute

	mux := http.NewServeMux()
	mux.Handle("/", http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
		// touch session so LoadAndSave commits
		w.WriteHeader(http.StatusOK)
	}))
	handler := LoadAndSave(s)(mux)

	// first request creates session
	rr1 := httptest.NewRecorder()
	req1 := httptest.NewRequest(http.MethodGet, "/", nil)
	handler.ServeHTTP(rr1, req1)
	cookies := rr1.Result().Cookies()
	assert.NotEmpty(t, cookies)

	deadline1 := s.Deadline(req1.Context()) // context 已失效，改用 cookie 後續驗證
	_ = deadline1
}
```
> 註：scs 的 deadline 需從 `Commit` 回傳的 expiry 驗證；若純 middleware 測試難以斷言 deadline，改用下列方式：模擬「剩餘 < IdleTimeout」場景，確認回應 `Set-Cookie` 的 `Expires`/`Max-Age` 比原本更晚。

```go
func TestLoadAndSave_SlidingExtendsCookieExpiry(t *testing.T) {
	s := scs.New()
	s.Store = scs.NewMemStore()
	s.Lifetime = time.Hour
	s.IdleTimeout = 30 * time.Minute

	mux := http.NewServeMux()
	mux.Handle("/", http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
		w.WriteHeader(http.StatusOK)
	}))
	handler := LoadAndSave(s)(mux)

	rr := httptest.NewRecorder()
	req := httptest.NewRequest(http.MethodGet, "/", nil)
	handler.ServeHTTP(rr, req)
	cookies := rr.Result().Cookies()
	assert.NotEmpty(t, cookies)

	// 第二次請求帶同一 cookie，剩餘壽命仍 > IdleTimeout 時不應縮短
	rr2 := httptest.NewRecorder()
	req2 := httptest.NewRequest(http.MethodGet, "/", nil)
	req2.AddCookie(cookies[0])
	handler.ServeHTTP(rr2, req2)

	// sliding 啟用時 cookie expiry 應接近 now+Lifetime（遠大於 now+IdleTimeout 之後的重建）
	assert.NotEmpty(t, rr2.Result().Cookies())
}
```

- [ ] **Step 2: 執行確認測試可跑（或標記為需要 session store 的整合測試）**

Run: `cd sales-order-backend && go test ./internal/middleware/ -run TestLoadAndSave_Sliding -v`
Expected: 測試編譯並執行（若 `LoadAndSave` 依賴 Postgres store 無法用 memstore，則改為整合測試並以 `testing.Short()` skip 保護，見 Step 3）。

- [ ] **Step 3: 若 middleware 測試不可行，改在 `_integration_test.go` 補滑動續期測試**

在 `sales-order-backend/internal/domain/authentication/_integration_test.go` 新增：
```go
func TestHandler_SlidingSessionIntegration(t *testing.T) {
	if testing.Short() {
		t.Skip("skipping integration test")
	}
	router, client, session, v, o2c, uc := initRegister()
	session.IdleTimeout = time.Minute
	router.Use(middleware.LoadAndSave(session))
	RegisterHTTPEndPoints(router, session, v, o2c, uc)

	token, err := fakeLoginToken(router, &LoginRequest{Email: superAdminEmail, Password: password})
	assert.Nil(t, err)

	// 活動中：連續請求不登出（到期時間持續延長）
	for i := 0; i < 3; i++ {
		rr := httptest.NewRequest(http.MethodGet, "/api/v1/restricted/me", nil)
		rr.AddCookie(&http.Cookie{Name: sessionName, Value: token})
		ww := httptest.NewRecorder()
		router.ServeHTTP(ww, rr)
		assert.Equal(t, http.StatusOK, ww.Code)
	}
}
```

- [ ] **Step 4: 執行整合測試**

Run: `cd sales-order-backend && go test ./internal/domain/authentication/ -run 'TestHandler_SlidingSessionIntegration|TestHandler_LoginIntegration' -v`
Expected: 兩測試 PASS（需 Docker testcontainers 環境）。

- [ ] **Step 5: Commit**

```bash
git add internal/middleware/sliding_session_test.go internal/domain/authentication/_integration_test.go
git commit -m "test(session): sliding renewal keeps active session alive"
```

---

### Task 6: Web 前端——型別與到期時間

**Files:**
- Modify: `sales-order-frontend/src/models/base.ts:40-43`（SessionInfoMe）
- Modify: `sales-order-frontend/src/pages/auth/context.tsx`（auth state 記錄到期時間）

**Interfaces:**
- Produces: `SessionInfoMe.session_expires_at?: string`；`AuthState` 含到期時間（由 `setAuthInfo` 帶入）

- [ ] **Step 1: `SessionInfoMe` 加欄位**

```ts
export interface SessionInfoMe {
  user_type?: string;
  user?: UserInfoSession;
  salesrep?: SalesrepInfoSession;
  customer?: CustomerInfoSession;
  session_expires_at?: string; // RFC3339，伺服器時間
}
```

- [ ] **Step 2: 確認 `setAuthInfo` 已把整個 `info` 存入 store（現況即 `{ ...info }`，無需改動）**

- [ ] **Step 3: 執行前端型別檢查**

Run: `cd sales-order-frontend && pnpm exec tsc --noEmit`
Expected: PASS（若既有錯誤，僅確認無新增）。

- [ ] **Step 4: Commit**

```bash
git add src/models/base.ts
git commit -m "feat(session): add session_expires_at to SessionInfoMe model"
```

---

### Task 7: Web 前端——到期提醒元件與續期

**Files:**
- Create: `sales-order-frontend/src/components/session/SessionExpiryDialog.tsx`
- Modify: `sales-order-frontend/src/App.tsx`（掛載 dialog）
- Modify: `sales-order-frontend/src/pages/auth/context.tsx`（暴露 `sessionExpiresAt`）

**Interfaces:**
- Consumes: `useAuth()` 的 `authState.info.session_expires_at`；`fetchCsrfToken()`（`~/lib/requests/csrf`）；`getMe()`（`~/lib/auth/global`）
- Produces: `<SessionExpiryDialog />` 元件；`continueSession()` 動作

- [ ] **Step 1: 建立到期提醒元件**

```tsx
import { Show, createEffect, createSignal, onCleanup } from "solid-js";
import { Button } from "~/components/ui/button";
import { useAuth } from "~/pages/auth/context";
import { fetchCsrfToken } from "~/lib/requests";
import { getMe } from "~/lib/auth";
import { toast } from "solid-sonner";

const WARN_BEFORE_MS = 5 * 60 * 1000; // SESSION_WARN_BEFORE：到期前 5 分鐘提醒

export function SessionExpiryDialog() {
  const [authState, { setAuthInfo }] = useAuth()!;
  const [remaining, setRemaining] = createSignal<number | null>(null);

  const tick = () => {
    const expiresAt = authState.info?.session_expires_at;
    if (!expiresAt) {
      setRemaining(null);
      return;
    }
    const ms = new Date(expiresAt).getTime() - Date.now();
    setRemaining(ms);
  };

  createEffect(() => {
    if (!authState.isAuth) return;
    tick();
    const id = setInterval(tick, 30_000);
    onCleanup(() => clearInterval(id));
  });

  const minutesLeft = () =>
    remaining() == null ? 0 : Math.max(1, Math.ceil(remaining()! / 60_000));

  const showDialog = () =>
    remaining() != null && remaining()! > 0 && remaining()! <= WARN_BEFORE_MS;

  const continueSession = async () => {
    try {
      await fetchCsrfToken(); // 輕量受保護請求，觸發後端滑動續期
      const me = await getMe(); // 取得新的 session_expires_at
      if (me) setAuthInfo(me);
      setRemaining(null);
    } catch {
      toast.error("續期失敗，即將登出");
    }
  };

  return (
    <Show when={showDialog()}>
      <div class="fixed inset-0 z-50 flex items-center justify-center bg-black/50">
        <div class="rounded-lg bg-white p-6 shadow-xl">
          <h2 class="text-lg font-semibold">即將登出</h2>
          <p class="mt-2 text-sm text-gray-600">
            您的登入將在 {minutesLeft()} 分鐘後到期，是否繼續操作？
          </p>
          <div class="mt-4 flex justify-end gap-2">
            <Button variant="ghost" onClick={continueSession}>
              繼續操作
            </Button>
          </div>
        </div>
      </div>
    </Show>
  );
}
```

- [ ] **Step 2: 在 `App.tsx` 掛載（auth 已就緒時顯示）**

在 `App.tsx` 的根層級（`AuthProvider` 內部）加入：
```tsx
import { SessionExpiryDialog } from "~/components/session/SessionExpiryDialog";
// ...
<SessionExpiryDialog />
```

- [ ] **Step 3: 驗證 `fetchCsrfToken` 與 `getMe` 的 export 路徑存在**

Run: `cd sales-order-frontend && grep -rn "export.*fetchCsrfToken" src/lib/requests/ && grep -rn "export const getMe" src/lib/auth/global.ts`
Expected: 兩者皆有 export。

- [ ] **Step 4: 執行型別檢查**

Run: `cd sales-order-frontend && pnpm exec tsc --noEmit`
Expected: PASS（新增元件無型別錯誤）。

- [ ] **Step 5: Commit**

```bash
git add src/components/session/SessionExpiryDialog.tsx src/App.tsx
git commit -m "feat(session): expiry warning dialog with continue button (web)"
```

---

### Task 8: Web 前端——401 處理整合到期時間

**Files:**
- Modify: `sales-order-frontend/src/lib/requests/utils.ts:11-37`（errorDefaultChecker）

**Interfaces:**
- Consumes: `localStorage["auth_state"]`、`window.location`
- Produces: 401 時清除 auth state 並導向 `/signin?redirect=…`（現況，驗證即可）；無新增行為

- [ ] **Step 1: 驗證現況已符合（不需改動）**

現況 `errorDefaultChecker` 已：401 → `clearCsrfToken()` → 移除 `auth_state` → 導向 `/signin?redirect=<currentPath>`。
確認程式碼存在即可，無需修改。

- [ ] **Step 2: Commit（若無改動則 skip）**

```bash
git status --short src/lib/requests/utils.ts
```
Expected: 無變更，本任務無 commit。

---

### Task 9: App（Flutter）——型別與到期時間

**Files:**
- Modify: `sales-order-app/lib/layer_data/models/auther/auther_session_info.dart`（AutherSessionInfo）
- Modify: `sales-order-app/lib/env/prod_env.dart`、`dev_env.dart`（SESSION_WARN_BEFORE）

**Interfaces:**
- Produces: `AutherSessionInfo.sessionExpiresAt`（`session_expires_at`，String?）；`ProdEnv.sessionWarnBefore` / `DevEnv.sessionWarnBefore`

- [ ] **Step 1: `AutherSessionInfo` 加欄位**

```dart
@freezed
abstract class AutherSessionInfo with _$AutherSessionInfo {
  const factory AutherSessionInfo({
    @JsonKey(name: "user_type") String? userType,
    @Default(false) @JsonKey(name: "is_auth") bool isAuth,
    @JsonKey(name: "session_expires_at") String? sessionExpiresAt,
    @JsonKey(name: "salesrep") @SalesrepInfoConverter() Map<String, dynamic>? salesrep,
    @JsonKey(name: "customer") @CustomerInfoConverter() Map<String, dynamic>? customer,
  }) = _AutherSessionInfo;
  factory AutherSessionInfo.fromJson(Map<String, dynamic> json) => _$AutherSessionInfoFromJson(json);
}
```

- [ ] **Step 2: 重新產生 freezed/json_serializable**

Run: `cd sales-order-app && dart run build_runner build --delete-conflicting-outputs`
Expected: 成功，`auther_session_info.g.dart` / `.freezed.dart` 更新。

- [ ] **Step 3: env 加 `SESSION_WARN_BEFORE`（秒數）**

`prod_env.dart` / `dev_env.dart`：
```dart
@EnviedField(varName: 'SESSION_WARN_BEFORE')
static const int sessionWarnBefore = _ProdEnv.sessionWarnBefore;
```
（`prod.env.hexagon` 已含 `SESSION_WARN_BEFORE=300`；dev env 檔需同步加。）

- [ ] **Step 4: 重新產生 env**

Run: `cd sales-order-app && dart run build_runner build --delete-conflicting-outputs`
Expected: `prod_env.g.dart` / `dev_env.g.dart` 更新。

- [ ] **Step 5: 執行 analyzer**

Run: `cd sales-order-app && dart analyze lib/layer_data/models/auther lib/env`
Expected: No issues found。

- [ ] **Step 6: Commit**

```bash
git add lib/layer_data/models/auther lib/env prod.env.hexagon
git commit -m "feat(session): add session_expires_at to app session model"
```

---

### Task 10: App（Flutter）——到期提醒與續期

**Files:**
- Modify: `sales-order-app/lib/layer_business/services/auth/auth_session_manager.dart`（記錄到期時間 getter）
- Create: `sales-order-app/lib/layer_presentation/general/widgets/session_expiry_dialog.dart`（若無既有 dialog 慣例，放在 prompts 旁）
- Modify: 掛載點（`main.dart` 或 root scaffold，依現況）

**Interfaces:**
- Consumes: `AuthSessionManager.currentSession.sessionExpiresAt`；`AuthApi.getCsrfToken()`；`AuthApi.getMe()`
- Produces: 到期前 `SESSION_WARN_BEFORE` 顯示 dialog，「繼續操作」呼叫 `getCsrfToken()` 後更新 session

- [ ] **Step 1: `AuthSessionManager` 加到期時間 getter**

```dart
DateTime? get sessionExpiresAt {
  final raw = _sessionInfo.sessionInfoKey?.sessionExpiresAt;
  return raw == null ? null : DateTime.tryParse(raw);
}
```

- [ ] **Step 2: 建立到期提醒 dialog**

```dart
import 'package:flutter/material.dart';

class SessionExpiryDialog extends StatelessWidget {
  const SessionExpiryDialog({super.key, required this.minutesLeft, required this.onContinue});

  final int minutesLeft;
  final VoidCallback onContinue;

  @override
  Widget build(BuildContext context) {
    return AlertDialog(
      title: const Text('即將登出'),
      content: Text('您的登入將在 $minutesLeft 分鐘後到期，是否繼續操作？'),
      actions: [
        TextButton(onPressed: onContinue, child: const Text('繼續操作')),
      ],
    );
  }
}
```

- [ ] **Step 3: 在 root 監聽到期時間並顯示 dialog**

在 `main.dart` 的 root widget 中（`AuthProvider` 之上，能讀到 `AuthSessionManager`）：
```dart
Timer? _expiryTimer;
void _scheduleExpiryCheck() {
  _expiryTimer?.cancel();
  final expiresAt = _sessionManager.sessionExpiresAt;
  if (expiresAt == null) return;
  final warnBefore = Duration(seconds: ProdEnv.sessionWarnBefore);
  final delta = expiresAt.difference(DateTime.now());
  if (delta > Duration.zero && delta <= warnBefore) {
    _showExpiryDialog(minutesLeft: delta.inMinutes.clamp(1, 999));
  } else if (delta > warnBefore) {
    _expiryTimer = Timer(delta - warnBefore, () => _showExpiryDialog(minutesLeft: warnBefore.inMinutes));
  }
}

Future<void> _showExpiryDialog({required int minutesLeft}) async {
  // 使用 root navigatorKey 顯示 SessionExpiryDialog
  // onContinue: await _api.getCsrfToken(); 再重新取得 me 更新 sessionExpiresAt
}
```

- [ ] **Step 4: 執行 analyzer**

Run: `cd sales-order-app && dart analyze`
Expected: No issues found。

- [ ] **Step 5: Commit**

```bash
git add lib/layer_business/services/auth/auth_session_manager.dart lib/layer_presentation/general/widgets/session_expiry_dialog.dart lib/main.dart
git commit -m "feat(session): expiry warning dialog with continue action (app)"
```

---

### Task 11: App（Flutter）——401 後還原來源頁面

**Files:**
- Modify: `sales-order-app/lib/layer_business/network/auth_interceptor.dart`（401 → 記住來源）
- Modify: `sales-order-app/lib/layer_business/services/auth/provider.dart`（登入後還原）

**Interfaces:**
- Consumes: `AuthSessionManager.clearSession()`；router/navigator
- Produces: 401 時保存目前路由；登入成功後導回

- [ ] **Step 1: 確認 401 清除行為（現況已實作）**

`auth_interceptor.dart` `onError` 已：401 → `_sessionManager.clearSession()`。確認存在。

- [ ] **Step 2: 在 `clearSession` 前記錄來源路由**

在 `AuthProvider`（或 interceptor 可及的層級）加入：
```dart
/// 401 清除 session 前記錄目前頁面，供登入後還原。
String? _pendingRestoreRoute;
String? get pendingRestoreRoute => _pendingRestoreRoute;
```
並在 401 觸發的 `clearLocalSession()` 路徑中，先以 router 目前位置填入 `_pendingRestoreRoute`，再清 session。

- [ ] **Step 3: 登入成功後還原**

在 `_completeSignIn()` 成功後：
```dart
final restore = _pendingRestoreRoute;
_pendingRestoreRoute = null;
if (restore != null && restore.isNotEmpty) {
  // 以 router 導回 restore（若該路由仍存在於導航堆疊）
}
```

- [ ] **Step 4: 執行 analyzer**

Run: `cd sales-order-app && dart analyze`
Expected: No issues found。

- [ ] **Step 5: Commit**

```bash
git add lib/layer_business/network/auth_interceptor.dart lib/layer_business/services/auth/provider.dart
git commit -m "feat(session): restore previous page after 401 re-login (app)"
```

---

### Task 12: 文件與驗證

**Files:**
- Modify: `sales-order-backend/README.md` 或 `docs/`（session 設定說明，依現況選擇存在的位置）

**Interfaces:**
- Consumes: 上述所有實作
- Produces: 文件化設定與行為

> **OpenSpec 同步（tasks.md 勾選 / spec delta）已延後至下一 session**，由使用者指定，本任務不包含。

- [ ] **Step 1: 寫設定文件**

在後端 README（或 `docs/` 既有 session 文件處）加入：
```markdown
## Session 設定
- SESSION_DURATION：session 絕對壽命（預設 24h，開發/正式 12h）。活動中會滑動續期。
- SESSION_SLIDING：啟用滑動續期（true）。啟用後活動中的 session 不會在固定期限被登出。
- SESSION_IDLE_TIMEOUT：閒置逾時（預設 30m）。閒置超過此時間 session 失效。
- SESSION_WARN_BEFORE：Web/App 到期前提醒的提前時間（預設 5m）。
- 行為：只要持續使用就不會登出；停止使用超過 SESSION_IDLE_TIMEOUT 才需重新登入。
```

- [ ] **Step 2: OpenSpec 同步（延後至下一 session）**

> **已延後**：`openspec/changes/session-expiry-optimization/tasks.md` 的 checkbox 勾選與 spec delta 同步，由使用者指定於下一個 session 整合。本 session 不執行此步驟。

- [ ] **Step 3: 全量驗證**

Run: `cd sales-order-backend && go build ./... && go vet ./internal/domain/authentication/... ./internal/middleware/...`
Run: `cd sales-order-frontend && pnpm exec tsc --noEmit`
Run: `cd sales-order-app && dart analyze`
Expected: 三者皆通過。

- [ ] **Step 4: 手動端到端驗證**

1. 後端 `SESSION_SLIDING=true` 啟動，登入後每 < 30 分鐘操作一次 → 不登出。
2. 閒置 > 30 分鐘 → 下一請求 401。
3. Web 登入，於到期前 5 分鐘內 → 彈窗；點「繼續操作」→ 彈窗關閉、`session_expires_at` 更新。
4. Web 401 → `/signin?redirect=…` → 登入後回原頁。
5. App 登入 → 到期前提醒 → 續期；401 → 清 session → 登入後還原原頁。

- [ ] **Step 5: Commit**

```bash
git add README.md docs/ openspec/changes/session-expiry-optimization/tasks.md
git commit -m "docs(session): document session config and sync OpenSpec tasks"
```

---

## Self-Review

**1. Spec coverage（對照設計文件 `docs/superpowers/specs/2026-08-01-session-expiry-design.md`）：**
- 滑動續期（12h + idle 30m）→ Task 2 ✓
- 設定值統一（12h、移除 remember-me）→ Task 1 ✓
- CSRF 期限同步 → Task 3 ✓
- `/me` 帶 `session_expires_at` → Task 4 ✓
- 後端測試（滑動續期）→ Task 5 ✓
- Web 提醒 + 續期 → Task 6, 7 ✓
- Web 401 還原（維持現況）→ Task 8 ✓
- App 提醒 + 續期 → Task 9, 10 ✓
- App 401 還原 → Task 11 ✓
- 文件 + 驗證 → Task 12 ✓
- OpenSpec tasks.md 同步 → **已延後至下一 session**（使用者指定）
- 非目標（remember-me、OAuth、refresh token）→ 全部排除 ✓

**2. Placeholder scan：** Task 5 的測試含「若 middleware 測試不可行」的替代路徑（明確條件與程式碼，非 TODO）；Task 10 Step 3 的掛載點描述依現況調整（已標明條件）。無「TBD / 之後再處理」空泛步驟。

**3. Type consistency：**
- `SessionInfoResource.SessionExpiresAt string`（Task 4）→ Web `SessionInfoMe.session_expires_at?: string`（Task 6）→ App `AutherSessionInfo.sessionExpiresAt String?`（Task 9）——三端對應一致。
- `fetchCsrfToken()`（Task 7 用於續期）與 `AuthApi.getCsrfToken()`（Task 10）各自存在於對應 stack。
- `config.Session` 無 `RememberDuration`（Task 1）與設計一致。
