# dio cache 優化 Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** 讓 Flutter App 的 dio cache 真正運作 — 後端加 ETag 支援 HTTP revalidation，App 端修正 cache 配置，Web frontend 零影響。

**Architecture:** 後端新增 ETag middleware（GET response body → SHA-256 → `If-None-Match` → 304）並調整 `/api` 的 Cache-Control 從 `no-cache, no-store` 改為 `private, no-cache`；App 端修正 `InCacheStorage` 配置（`hitCacheOnErrorCodes` 移除 400、啟用 `hitCacheOnNetworkFailure`）並清理 `mdCache` 死碼。兩個 repo 獨立變更，frontend 完全不動。

**Tech Stack:** Go 1.25 (chi router, httptest) / Flutter 3.35.2 (dio_cache_interceptor 4.0.7, http_cache_core 1.1.4)

## Global Constraints

- Backend：Go 1.25，chi router，測試用標準 `httptest` + `go test ./...`，module `github.com/hexagon-maker/sales-order-backend`
- App：Flutter 3.35.2（`.fvmrc`），`fvm flutter analyze` 0 errors、`fvm flutter test` 全過
- **frontend（sales-order-frontend）不得有任何變更**
- **Client 區分（設計修正）**：後端依 `X-App-Client: mobile` header 區分 — App 拿 `private, max-age=300`（零請求命中），瀏覽器（無 header）拿 `private, no-cache`（revalidate，零影響）
- `Cache-Control` 一律 `private`、**絕不 `public`**（避免已登入用戶資料被共享快取串到其他用戶）
- ETag 只套用 GET + HTTP 200 + 非空 body；非 GET / 非 200 / 空 body 不設 ETag
- App 端 dio 送 `X-App-Client: mobile` header（AuthInterceptor）
- 每個 task 完成後可獨立測試；commit message 用 conventional format

---

## 階段 1：後端 ETag 支援

### Task 1: ETag middleware + 單元測試

**Files:**
- Create: `internal/middleware/etag.go`
- Test: `internal/middleware/etag_test.go`

**Interfaces:**
- Consumes: 無（獨立 middleware，標準 `http.Handler` 簽名）
- Produces: `func ETagMiddleware(next http.Handler) http.Handler` — 供 Task 2 在 `server.go` 掛載

- [ ] **Step 1: 寫 failing test**

```go
// internal/middleware/etag_test.go
package middleware

import (
	"net/http"
	"net/http/httptest"
	"strings"
	"testing"
)

func testETagHandler() http.Handler {
	return ETagMiddleware(http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
		w.Header().Set("Content-Type", "application/json")
		w.WriteHeader(http.StatusOK)
		_, _ = w.Write([]byte(`{"data":"hello"}`))
	}))
}

func TestETagMiddleware_SetsETagOnOK(t *testing.T) {
	req := httptest.NewRequest(http.MethodGet, "/api/v1/customers", nil)
	rec := httptest.NewRecorder()

	testETagHandler().ServeHTTP(rec, req)

	if rec.Code != http.StatusOK {
		t.Fatalf("status = %d, want 200", rec.Code)
	}
	etag := rec.Header().Get("ETag")
	if etag == "" {
		t.Fatal("ETag header not set")
	}
	if !strings.HasPrefix(etag, `"`) || !strings.HasSuffix(etag, `"`) {
		t.Fatalf("ETag %q not quoted", etag)
	}
	if rec.Body.String() != `{"data":"hello"}` {
		t.Fatalf("body = %q, want unchanged response body", rec.Body.String())
	}
}

func TestETagMiddleware_SameBodySameETag(t *testing.T) {
	rec1 := httptest.NewRecorder()
	testETagHandler().ServeHTTP(rec1, httptest.NewRequest(http.MethodGet, "/api/v1/customers", nil))
	rec2 := httptest.NewRecorder()
	testETagHandler().ServeHTTP(rec2, httptest.NewRequest(http.MethodGet, "/api/v1/customers", nil))

	if rec1.Header().Get("ETag") != rec2.Header().Get("ETag") {
		t.Fatalf("ETags differ: %q vs %q", rec1.Header().Get("ETag"), rec2.Header().Get("ETag"))
	}
}

func TestETagMiddleware_MatchReturns304(t *testing.T) {
	rec1 := httptest.NewRecorder()
	testETagHandler().ServeHTTP(rec1, httptest.NewRequest(http.MethodGet, "/api/v1/customers", nil))
	etag := rec1.Header().Get("ETag")

	req := httptest.NewRequest(http.MethodGet, "/api/v1/customers", nil)
	req.Header.Set("If-None-Match", etag)
	rec := httptest.NewRecorder()
	testETagHandler().ServeHTTP(rec, req)

	if rec.Code != http.StatusNotModified {
		t.Fatalf("status = %d, want 304", rec.Code)
	}
	if rec.Body.Len() != 0 {
		t.Fatalf("304 body = %q, want empty", rec.Body.String())
	}
}

func TestETagMiddleware_MismatchReturns200(t *testing.T) {
	req := httptest.NewRequest(http.MethodGet, "/api/v1/customers", nil)
	req.Header.Set("If-None-Match", `"different"`)
	rec := httptest.NewRecorder()
	testETagHandler().ServeHTTP(rec, req)

	if rec.Code != http.StatusOK {
		t.Fatalf("status = %d, want 200", rec.Code)
	}
	if rec.Body.String() != `{"data":"hello"}` {
		t.Fatalf("body = %q, want full body", rec.Body.String())
	}
}

func TestETagMiddleware_NonGETNoETag(t *testing.T) {
	req := httptest.NewRequest(http.MethodPost, "/api/v1/customers", strings.NewReader(`{}`))
	rec := httptest.NewRecorder()

	testETagHandler().ServeHTTP(rec, req)

	if rec.Header().Get("ETag") != "" {
		t.Fatal("ETag set on POST")
	}
	if rec.Body.String() != `{"data":"hello"}` {
		t.Fatalf("body = %q, want unchanged", rec.Body.String())
	}
}

func TestETagMiddleware_Non200PassesThrough(t *testing.T) {
	handler := ETagMiddleware(http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
		w.WriteHeader(http.StatusNotFound)
		_, _ = w.Write([]byte(`{"message":"not found"}`))
	}))

	req := httptest.NewRequest(http.MethodGet, "/api/v1/nope", nil)
	rec := httptest.NewRecorder()
	handler.ServeHTTP(rec, req)

	if rec.Code != http.StatusNotFound {
		t.Fatalf("status = %d, want 404 (flush regression)", rec.Code)
	}
	if rec.Header().Get("ETag") != "" {
		t.Fatal("ETag set on 404")
	}
	if rec.Body.String() != `{"message":"not found"}` {
		t.Fatalf("body = %q, want 404 body preserved", rec.Body.String())
	}
}

func TestETagMiddleware_EmptyBodyNoETag(t *testing.T) {
	handler := ETagMiddleware(http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
		w.WriteHeader(http.StatusOK)
	}))

	req := httptest.NewRequest(http.MethodGet, "/api/v1/empty", nil)
	rec := httptest.NewRecorder()
	handler.ServeHTTP(rec, req)

	if rec.Header().Get("ETag") != "" {
		t.Fatal("ETag set on empty body")
	}
	if rec.Code != http.StatusOK {
		t.Fatalf("status = %d, want 200", rec.Code)
	}
}

func TestETagMiddleware_HeaderPreserved(t *testing.T) {
	req := httptest.NewRequest(http.MethodGet, "/api/v1/customers", nil)
	rec := httptest.NewRecorder()

	testETagHandler().ServeHTTP(rec, req)

	if rec.Header().Get("Content-Type") != "application/json" {
		t.Fatalf("Content-Type = %q, want application/json preserved", rec.Header().Get("Content-Type"))
	}
}

func TestETagMiddleware_WebSocketUpgradePassthrough(t *testing.T) {
	// WS 需要 http.Hijacker；緩衝會破壞升級 — middleware 必須透傳。
	hijacked := false
	handler := ETagMiddleware(http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
		_, ok := w.(http.Hijacker)
		if !ok {
			t.Fatal("handler received non-Hijacker writer — upgrade would fail")
		}
		hijacked = true
		w.WriteHeader(http.StatusSwitchingProtocols)
	}))

	req := httptest.NewRequest(http.MethodGet, "/api/v1/dispatches/ws", nil)
	req.Header.Set("Upgrade", "websocket")
	req.Header.Set("Connection", "Upgrade")
	rec := httptest.NewRecorder()
	handler.ServeHTTP(rec, req)

	if !hijacked {
		t.Fatal("handler not reached with Hijacker writer")
	}
	if rec.Header().Get("ETag") != "" {
		t.Fatal("ETag set on WS upgrade")
	}
}

func TestETagMiddleware_SSEPassthrough(t *testing.T) {
	// SSE 需要 http.Flusher；緩衝會讓串流 byte 卡到 handler 結束才送出。
	flushed := false
	handler := ETagMiddleware(http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
		_, ok := w.(http.Flusher)
		if !ok {
			t.Fatal("handler received non-Flusher writer — SSE streaming would fail")
		}
		flushed = true
		_, _ = w.Write([]byte("data: ping\n\n"))
		w.(http.Flusher).Flush()
	}))

	req := httptest.NewRequest(http.MethodGet, "/api/v1/sales_orders/dispatches/sse", nil)
	req.Header.Set("Accept", "text/event-stream")
	rec := httptest.NewRecorder()
	handler.ServeHTTP(rec, req)

	if !flushed {
		t.Fatal("handler not reached with Flusher writer")
	}
	if rec.Header().Get("ETag") != "" {
		t.Fatal("ETag set on SSE")
	}
}
```

- [ ] **Step 2: 執行測試確認 fail**

```bash
cd sales-order-backend
go test ./internal/middleware/ -run TestETag -v
```

預期：compile error（`ETagMiddleware` undefined）。

- [ ] **Step 3: 實作 ETag middleware**

```go
// internal/middleware/etag.go
package middleware

import (
	"bytes"
	"crypto/sha256"
	"encoding/hex"
	"net/http"
	"strings"
)

// ETagMiddleware 為 GET 回應計算 ETag，並處理 If-None-Match → 304。
// 只套用於 API GET 請求；非 GET 直接放行。
// WebSocket（Upgrade）與 SSE（text/event-stream）請求直接透傳 —
// bodyBufferWriter 不實作 http.Flusher/http.Hijacker，緩衝會破壞串流。
func ETagMiddleware(next http.Handler) http.Handler {
	return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
		if r.Method != http.MethodGet {
			next.ServeHTTP(w, r)
			return
		}
		// 串流/升級請求不緩衝：WS 需要 Hijacker、SSE 需要 Flusher
		if r.Header.Get("Upgrade") != "" ||
			strings.Contains(r.Header.Get("Accept"), "text/event-stream") {
			next.ServeHTTP(w, r)
			return
		}

		// 包一層 ResponseWriter 緩衝 header + body，handler 結束後統一輸出，
		// 這樣才能先算 hash 再決定回 304 或完整 body。
		bw := &bodyBufferWriter{ResponseWriter: w}
		next.ServeHTTP(bw, r)

		// 非 200 或無 body：原樣輸出（不設 ETag，不判 304）
		if bw.statusCode != http.StatusOK || bw.body.Len() == 0 {
			bw.flush()
			return
		}

		hash := sha256.Sum256(bw.body.Bytes())
		etag := `"` + hex.EncodeToString(hash[:16]) + `"` // 128-bit 足夠

		// If-None-Match 相符 → 304 無 body
		if match := r.Header.Get("If-None-Match"); match != "" {
			if strings.Contains(match, etag) || match == "*" {
				bw.Header().Set("ETag", etag)
				bw.WriteHeader(http.StatusNotModified)
				bw.flush()
				return
			}
		}

		bw.Header().Set("ETag", etag)
		bw.flush()
	})
}

// bodyBufferWriter 緩衝 statusCode、headers、body，由 flush() 統一輸出到
// 底層 ResponseWriter。避免「先寫出再後悔」——ETag 需完整 body 才能計算。
type bodyBufferWriter struct {
	http.ResponseWriter
	header     http.Header
	body       bytes.Buffer
	statusCode int
	flushed    bool
}

func (b *bodyBufferWriter) Header() http.Header {
	if b.header == nil {
		b.header = http.Header{}
	}
	return b.header
}

func (b *bodyBufferWriter) WriteHeader(code int) {
	b.statusCode = code
}

func (b *bodyBufferWriter) Write(p []byte) (int, error) {
	return b.body.Write(p)
}

// flush 將緩衝內容實際寫入底層 ResponseWriter。
// 先複製 header（含 handler 設定的 Content-Type 等），再寫 status + body。
func (b *bodyBufferWriter) flush() {
	if b.flushed {
		return
	}
	b.flushed = true

	// 複製所有 header（ETag 等由 middleware 在 flush 前設定）
	for k, vv := range b.header {
		for _, v := range vv {
			b.ResponseWriter.Header().Add(k, v)
		}
	}

	code := b.statusCode
	if code == 0 {
		code = http.StatusOK
	}
	b.ResponseWriter.WriteHeader(code)
	if code != http.StatusNoContent && code != http.StatusNotModified {
		_, _ = b.ResponseWriter.Write(b.body.Bytes())
	}
}
```

- [ ] **Step 4: 執行測試確認 pass**

```bash
cd sales-order-backend
go test ./internal/middleware/ -run TestETag -v
```

預期：全部 PASS（8 個測試）。

- [ ] **Step 5: 全套測試確認無回歸**

```bash
cd sales-order-backend
go test ./...
```

預期：全部 PASS（既有 middleware 測試不受影響）。

- [ ] **Step 6: Commit**

```bash
git add internal/middleware/etag.go internal/middleware/etag_test.go
git commit -m "feat: add ETag middleware with 304 revalidation for GET responses"
```

---

### Task 2: Cache-Control 調整 + middleware 掛載

**Files:**
- Modify: `internal/middleware/publicCache.go`
- Modify: `internal/server/server.go`（`setGlobalMiddleware()`）
- Test: `internal/middleware/public_cache_test.go`

**Interfaces:**
- Consumes: Task 1 的 `ETagMiddleware`
- Produces: `/api` 回應 `Cache-Control: private, no-cache`；GET 回應帶 ETag

- [ ] **Step 1: 寫 failing test**

```go
// internal/middleware/public_cache_test.go
package middleware

import (
	"net/http"
	"net/http/httptest"
	"testing"
)

func TestPublicCacheMiddleware_APIPathBrowserNoCache(t *testing.T) {
	handler := PublicCacheMiddleware(http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
		w.WriteHeader(http.StatusOK)
	}))

	req := httptest.NewRequest(http.MethodGet, "/api/v1/customers", nil)
	rec := httptest.NewRecorder()
	handler.ServeHTTP(rec, req)

	got := rec.Header().Get("Cache-Control")
	if got != "private, no-cache" {
		t.Fatalf("Cache-Control = %q, want \"private, no-cache\" (browser)", got)
	}
}

func TestPublicCacheMiddleware_APIPathMobileMaxAge(t *testing.T) {
	handler := PublicCacheMiddleware(http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
		w.WriteHeader(http.StatusOK)
	}))

	req := httptest.NewRequest(http.MethodGet, "/api/v1/customers", nil)
	req.Header.Set("X-App-Client", "mobile")
	rec := httptest.NewRecorder()
	handler.ServeHTTP(rec, req)

	got := rec.Header().Get("Cache-Control")
	if got != "private, max-age=300" {
		t.Fatalf("Cache-Control = %q, want \"private, max-age=300\" (mobile)", got)
	}
}

func TestPublicCacheMiddleware_NonAPIPathPublicMaxAge(t *testing.T) {
	handler := PublicCacheMiddleware(http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
		w.WriteHeader(http.StatusOK)
	}))

	req := httptest.NewRequest(http.MethodGet, "/docs/index.html", nil)
	rec := httptest.NewRecorder()
	handler.ServeHTTP(rec, req)

	got := rec.Header().Get("Cache-Control")
	if got != "public, max-age=300, s-maxage=600" {
		t.Fatalf("Cache-Control = %q, want \"public, max-age=300, s-maxage=600\"", got)
	}
}
```

- [ ] **Step 2: 執行測試確認 fail**

```bash
cd sales-order-backend
go test ./internal/middleware/ -run TestPublicCache -v
```

預期：FAIL（目前 `/api` 是 `no-cache, no-store`，且無 client 區分）。

- [ ] **Step 3: 修改 publicCache.go**

```go
// internal/middleware/publicCache.go
package middleware

import (
	"net/http"
	"strings"
)

func PublicCacheMiddleware(next http.Handler) http.Handler {
	return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
		// API 路由：依 client 區分 Cache-Control。
		// - Flutter App（送 X-App-Client: mobile）→ private, max-age=300，
		//   dio 可直接命中 cache（零請求），過期後靠 ETag 304 更新。
		// - 瀏覽器（frontend，無該 header）→ private, no-cache，每次 revalidate，
		//   與現況等價、零影響。
		// 一律 private、絕不 public（避免已登入用戶資料被共享快取串到其他用戶）。
		if strings.HasPrefix(r.URL.Path, "/api") {
			if r.Header.Get("X-App-Client") == "mobile" {
				w.Header().Set("Cache-Control", "private, max-age=300")
			} else {
				w.Header().Set("Cache-Control", "private, no-cache")
			}
			next.ServeHTTP(w, r)
			return
		}
		w.Header().Set("Cache-Control", "public, max-age=300, s-maxage=600")
		next.ServeHTTP(w, r)
	})
}
```

- [ ] **Step 4: 執行測試確認 pass**

```bash
cd sales-order-backend
go test ./internal/middleware/ -run TestPublicCache -v
```

預期：全部 PASS。

- [ ] **Step 5: 掛載 ETagMiddleware**

在 `internal/server/server.go` 的 `setGlobalMiddleware()` 中，`PublicCacheMiddleware` 之後加入：

```go
	s.router.Use(middleware.PublicCacheMiddleware)
	s.router.Use(middleware.ETagMiddleware)
```

- [ ] **Step 6: 編譯 + 全套測試**

```bash
cd sales-order-backend
go build ./...
go test ./...
```

預期：build 成功、全部 PASS。

- [ ] **Step 7: Commit**

```bash
git add internal/middleware/publicCache.go internal/middleware/public_cache_test.go internal/server/server.go
git commit -m "feat: client-aware Cache-Control (mobile max-age, browser no-cache) and mount ETag middleware"
```

---

## 階段 2：App 端 cache 配置

### Task 3: InCacheStorage 配置修正 + mdCache 死碼清理

**Files:**
- Modify: `lib/layer_data/repositories/cache_storage.dart`
- Modify: `lib/layer_data/repositories/abstract/abstract_cache_storage.dart`
- Modify: `lib/layer_business/network/api/cache_options_mixin.dart`
- Modify: `lib/layer_business/network/auth_interceptor.dart`（送 `X-App-Client: mobile` header）
- Modify: `test/unit/services/auth_service_test.dart`（stub 移除 mdCache）
- Modify: `test/unit/services/auth_interceptor_401_test.dart`（stub 移除 mdCache）
- Modify: `test/auth/auth_interceptor_test.dart`（stub 移除 mdCache）
- Create: `test/unit/repositories/cache_storage_test.dart`（配置測試）
- Create: `test/integration/dio_cache_etag_test.dart`（dio ETag 整合測試）
- Create: `test/unit/network/auth_interceptor_client_header_test.dart`（X-App-Client header 測試）

**Interfaces:**
- Consumes: 無（純配置調整）
- Produces: `CacheOptions` 的 `hitCacheOnErrorCodes = [404, 500]`、`hitCacheOnNetworkFailure = true`；移除 `mdCache`/`mediumCacheOptions`

- [ ] **Step 0: 加入 X-App-Client header**

`lib/layer_business/network/auth_interceptor.dart` 的 `onRequest` 中，與現有 `Accept` header 設定同位置加入：

```dart
    options.headers['Accept'] = ['application/json'];
    options.headers['Access-Control-Allow-Origin'] = 'true';
    // 後端依此區分 Cache-Control：mobile → max-age（零請求命中），
    // 瀏覽器（無此 header）→ no-cache（每次 revalidate，不受影響）。
    options.headers['X-App-Client'] = 'mobile';
```

- [ ] **Step 0b: X-App-Client header 測試**

`test/unit/network/auth_interceptor_client_header_test.dart`：

```dart
// test/unit/network/auth_interceptor_client_header_test.dart
import 'dart:io';
import 'package:flutter_test/flutter_test.dart';
import 'package:hexagon_food_app/layer_business/network/auth_interceptor.dart';
import 'package:hexagon_food_app/layer_business/services/auth/auth_session_manager.dart';
import 'package:hexagon_food_app/layer_data/repositories/session_info_storage.dart';
import 'package:cookie_jar/cookie_jar.dart';
import 'package:cached_memory_image/cached_image_base64_manager.dart';
import 'package:dio/dio.dart';
import 'package:dio_cache_interceptor/dio_cache_interceptor.dart';
import 'package:sqflite_common_ffi/sqflite_ffi.dart';
import '../../auth/fake_path_provider.dart';

class StubCacheStorage extends CacheStorage {
  @override Future<void> clearCache() async {}
  @override CacheOptions get defaultOptions => CacheOptions(store: MemCacheStore());
  @override CacheOptions get smallCacheOptions => CacheOptions(store: MemCacheStore());
  @override CacheOptions get largeCacheOptions => CacheOptions(store: MemCacheStore());
  @override CacheOptions get noCacheOptions => CacheOptions(store: MemCacheStore());
  @override CacheOptions get refreshOptions => CacheOptions(store: MemCacheStore());
  @override CacheOptions get forceCacheOptions => CacheOptions(store: MemCacheStore());
  @override CacheOptions get refreshForceCacheOptions => CacheOptions(store: MemCacheStore());
  @override Options get smCache => Options();
  @override Options get lgCache => Options();
  @override Options get noCache => Options();
  @override Options get refresh => Options();
  @override Options get forceRefetch => Options();
  @override Options get refreshForceCache => Options();
  @override DioCacheInterceptor get interceptor => DioCacheInterceptor(options: defaultOptions);
  @override Future<void> clean({CachePriority priorityOrBelow = CachePriority.high, bool staleOnly = false}) async {}
}

void main() {
  TestWidgetsFlutterBinding.ensureInitialized();
  registerFakePathProvider();
  sqfliteFfiInit();
  databaseFactory = databaseFactoryFfi;

  late AuthSessionManager sessionManager;
  late AuthInterceptor interceptor;
  late Directory tmpDir;

  setUp(() async {
    tmpDir = Directory.systemTemp.createTempSync('client_header_test_');
    final sessionStorage = await InSessionInfoStorage.create(tmpDir.path);
    sessionManager = AuthSessionManager(
      sessionInfo: sessionStorage,
      cookieJar: PersistCookieJar(ignoreExpires: true),
      cacheStorage: StubCacheStorage(),
      imageCache: CachedImageBase64Manager.instance(),
    );
    interceptor = AuthInterceptor(sessionManager: sessionManager);
  });

  tearDown(() {
    tmpDir.deleteSync(recursive: true);
  });

  test('onRequest sets X-App-Client: mobile header', () async {
    final options = RequestOptions(path: '/api/v1/test');
    await interceptor.onRequest(options, RequestInterceptorHandler());
    expect(options.headers['X-App-Client'], 'mobile');
  });
}
```

- [ ] **Step 1: 確認 mdCache 無業務使用**

```bash
cd sales-order-app
grep -rn "mdCache\|mediumCacheOptions" lib/ --include="*.dart"
```

預期：只在 `abstract_cache_storage.dart`、`cache_storage.dart`、`cache_options_mixin.dart` 出現（純定義，無業務呼叫）。若發現業務呼叫，先記錄再決定（此 task 以 spec 為準：死碼清理）。

- [ ] **Step 2: 修改 cache_storage.dart 配置 + 移除 mdCache**

`lib/layer_data/repositories/cache_storage.dart`：

```dart
  CacheOptions _buildCacheOptions({
    CachePolicy policy = CachePolicy.request,
    CachePriority priority = CachePriority.normal,
    Duration maxStale = const Duration(minutes: 5),
  }) {
    return CacheOptions(
      store: _store,
      policy: policy,
      priority: priority,
      maxStale: maxStale,
      // 400 是 client 錯誤，不該命中 cache；保留 404/500 做錯誤 fallback
      hitCacheOnErrorCodes: [404, 500],
      // 斷網時嘗試從 cache 讀取（需 cache 已有 entry）
      hitCacheOnNetworkFailure: true,
      keyBuilder: CacheOptions.defaultCacheKeyBuilder,
    );
  }
```

移除 `mediumCacheOptions` getter（原本 15min）：

```dart
  @override
  CacheOptions get mediumCacheOptions =>
      _buildCacheOptions(maxStale: const Duration(minutes: 15));
```

移除 `mdCache` getter：

```dart
  @override
  Options get mdCache => mediumCacheOptions.toOptions();
```

- [ ] **Step 3: 修改抽象介面**

`lib/layer_data/repositories/abstract/abstract_cache_storage.dart` — 移除兩行：

```dart
  CacheOptions get mediumCacheOptions;
```

```dart
  Options get mdCache;
```

- [ ] **Step 4: 修改 mixin**

`lib/layer_business/network/api/cache_options_mixin.dart` — 移除一行：

```dart
  Options get mdCache => _cache.mdCache;
```

- [ ] **Step 5: 更新 3 個測試 stub**

`test/unit/services/auth_service_test.dart`、`test/unit/services/auth_interceptor_401_test.dart`、`test/auth/auth_interceptor_test.dart` — 各移除兩行 stub：

```dart
  @override
  CacheOptions get mediumCacheOptions => CacheOptions(store: MemCacheStore());
```

```dart
  @override
  Options get mdCache => Options();
```

- [ ] **Step 5b: 新增 cache 配置測試**

`test/unit/repositories/cache_storage_test.dart`：

```dart
// test/unit/repositories/cache_storage_test.dart
import 'dart:io';

import 'package:flutter_test/flutter_test.dart';
import 'package:hexagon_food_app/layer_data/repositories/cache_storage.dart';

import '../../auth/fake_path_provider.dart';

void main() {
  TestWidgetsFlutterBinding.ensureInitialized();
  registerFakePathProvider();

  late Directory tmpDir;
  late InCacheStorage storage;

  setUp(() async {
    tmpDir = Directory.systemTemp.createTempSync('cache_test_');
    storage = await InCacheStorage.create(tmpDir.path);
  });

  tearDown(() {
    tmpDir.deleteSync(recursive: true);
  });

  test('hitCacheOnErrorCodes excludes 400', () {
    final options = storage.defaultOptions;
    expect(options.hitCacheOnErrorCodes, isNot(contains(400)));
    expect(options.hitCacheOnErrorCodes, contains(404));
    expect(options.hitCacheOnErrorCodes, contains(500));
  });

  test('hitCacheOnNetworkFailure is enabled', () {
    expect(storage.defaultOptions.hitCacheOnNetworkFailure, isTrue);
  });

  test('smCache maxStale is 5 minutes', () {
    expect(storage.smallCacheOptions.maxStale, const Duration(minutes: 5));
  });

  test('lgCache maxStale is 8 hours', () {
    expect(storage.largeCacheOptions.maxStale, const Duration(hours: 8));
  });
}
```

> **注意**：`SembastCacheStore` 用 `databaseFactoryIo`（純 dart sembast，sembast_factory_io.dart），**不需 sqflite ffi init** — 只需要 `registerFakePathProvider()`（`InCacheStorage.create` 拿 storePath 用不到 path_provider，但 `CachedImageBase64Manager` 等相依可能觸發；若 compile/run 出 MissingPluginException 再加 fake path provider）。

- [ ] **Step 6: analyze + 全測試**

```bash
cd sales-order-app
fvm flutter analyze
fvm flutter test
```

預期：0 errors；全部測試通過。

- [ ] **Step 7: Commit**

```bash
git add lib/layer_data/repositories/ lib/layer_business/network/ test/
git commit -m "fix: correct cache error fallback codes, enable network-failure fallback, send X-App-Client header, drop unused mdCache"
```
---

## 階段 3：驗證

### Task 4: 端對端驗證

**Files:**
- Create: `test/integration/dio_cache_etag_test.dart`（dio 整合測試 — 已在 Task 3 建立）
- 無其他（驗證任務）

**Interfaces:**
- Consumes: Tasks 1-3
- Produces: 驗證紀錄（寫入 task report）

- [ ] **Step 1: 起後端 + 驗證 ETag/304 與 client 區分**

```bash
cd sales-order-backend
task infra:start   # docker-compose DB 等（若需要）
task dev           # air hot-reload，監聽 0.0.0.0:3080

# 驗證 client 區分：無 header → private, no-cache；X-App-Client: mobile → private, max-age=300
curl -s -D - -o /dev/null http://localhost:3080/api/v1/restricted/csrf
curl -s -D - -o /dev/null -H "X-App-Client: mobile" http://localhost:3080/api/v1/restricted/csrf

# 驗證 ETag：GET 兩次，第一次拿 ETag，第二次帶 If-None-Match 應得 304
curl -s -D - -o /dev/null http://localhost:3080/api/v1/restricted/csrf
curl -s -D - -H "If-None-Match: <上一步的 ETag>" http://localhost:3080/api/v1/restricted/csrf
```

預期：無 header → `Cache-Control: private, no-cache`；mobile → `private, max-age=300`；第一次 200 + `ETag`；第二次 304 無 body。

- [ ] **Step 2: dio 整合測試（不需後端，MemCacheStore）**

```bash
cd sales-order-app
fvm flutter test test/integration/dio_cache_etag_test.dart
```

預期：2 tests 全過 —
1. `max-age=300` response 寫入 cache，max-age 內第二次請求零網路（`extraFromNetworkKey == false`）
2. `no-cache` response（瀏覽器情境）每次 revalidate（frontend 零影響機制）

- [ ] **Step 3: App 端驗證 cache 行為**

```bash
cd sales-order-app
fvm flutter run --flavor dev --target lib/main_dev.dart
```

手動驗證（用 Talker logger 或網路監看）：
1. 登入後進客戶列表 → 第一次載入打 API（200 + ETag + max-age=300）
2. 離開再進（5min 內）→ 零網路請求（cache 直接命中）
3. 下拉刷新 → 一次網路請求（refreshForceCache）
4. 關掉後端 → 再進列表 → 顯示 cache 舊資料（hitCacheOnNetworkFailure）

- [ ] **Step 4: frontend 驗證**

```bash
cd sales-order-frontend
pnpm dev
```

手動開瀏覽器登入，確認各頁面資料正常、無錯誤、無行為變化（瀏覽器請求無 `X-App-Client` header → 拿 `no-cache`，行為與現況等價）。

- [ ] **Step 5: 紀錄驗證結果**

將各步驟的實際輸出寫入 task report（無 commit — 驗證任務；若 dio 整合測試在 Task 3 未建立，則在此建立並 commit）。

---

## 執行順序

```
Phase 1 (backend):  Task 1 → Task 2（依序）
Phase 2 (app):      Task 3（獨立，可與 Phase 1 並行）
Phase 3 (verify):   Task 4（依賴 Tasks 1-3）
```
