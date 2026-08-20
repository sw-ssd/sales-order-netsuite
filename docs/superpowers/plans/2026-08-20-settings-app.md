# Settings App (Flutter) Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** App 由編譯期 `SystemConstants` 改為執行期 backend settings：登入後 fetch → Sembast 快取 → `Signal`，離線/失敗回退快取或編譯期 fallback；9 個使用點遷移。

**Architecture:** freezed model（snake_case ↔ camelCase via `@JsonKey`）+ `SettingsApi`（Dio，`CacheOptionsMixin`）+ GetIt 註冊的 `SettingsService`（`Signal<SettingsModel?>` + Sembast `settings_store` + `SystemConstants` fallback）；登入後由 AuthProvider 觸發 `refresh()`；服務經建構注入、widget 經 `locator<SettingsService>()` 同步讀取。

**Tech Stack:** Flutter 3.35.2, freezed/json_serializable, dio, disco, flutter_solidart, Sembast, GetIt。

## Global Constraints

- 所有新程式碼置於 `sales-order-app/`（submodule）。
- JSON key 為 snake_case（backend DTO 契約），model 欄位為 camelCase；secret 欄位解析為可空但 app 不使用。
- 依賴 backend API 契約（backend plan Task 3/5/6/8）：`GET /api/v1/settings` 回傳遮罩 secret；app 僅 GET（唯讀）。
- 回退順序：Sembast 快取 → `SystemConstants`（fallback 數值更新：`defaultDepartment` 6、新增 `frontendUrl`）。
- `SystemConstants` 保留為 fallback 類別（更名與否以最小改動為準），不刪除。
- 產生檔（`*.freezed.dart`、`*.g.dart`）入版控；改 model 後執行 `fvm dart run build_runner build --delete-conflicting-outputs`。
- app 目前無 `test/` 目錄 — 本計畫建立 `test/`，以 `fvm flutter test` 執行。
- 每個 task 結束需 commit（`sales-order-app/` submodule 內）。

---

### Task 1: freezed model + 產生碼

**Files:**
- Create: `sales-order-app/lib/layer_data/models/setting/setting.dart`
- Generate: `setting.freezed.dart`、`setting.g.dart`

**Interfaces:**
- Produces: `SettingsModel`（欄位 = backend DTO 非 secret 全部 + secret 可空）；`SettingsModel.fromJson`。

- [ ] **Step 1: 建立 model**

```dart
import 'package:freezed_annotation/freezed_annotation.dart';

part 'setting.freezed.dart';
part 'setting.g.dart';

@freezed
abstract class SettingsModel with _$SettingsModel {
  const factory SettingsModel({
    @JsonKey(name: 'default_department_id') int? defaultDepartmentId,
    @JsonKey(name: 'approval') int? approval,
    @JsonKey(name: 'system_department_id') int? systemDepartmentId,
    @JsonKey(name: 'system_salesrep_id') int? systemSalesrepId,
    @JsonKey(name: 'system_test_salesrep_id') int? systemTestSalesrepId,
    @JsonKey(name: 'system_customer_id') int? systemCustomerId,
    @JsonKey(name: 'system_customer_address_id') int? systemCustomerAddressId,
    @JsonKey(name: 'system_customer_address_entity_address_id')
    int? systemCustomerAddressEntityAddressId,
    @JsonKey(name: 'system_customer_contact_id') int? systemCustomerContactId,
    @JsonKey(name: 'system_customer_entity_id') String? systemCustomerEntityId,
    @JsonKey(name: 'system_customer_name') String? systemCustomerName,
    @JsonKey(name: 'default_vendor_id') int? defaultVendorId,
    @JsonKey(name: 'default_vendor_name') String? defaultVendorName,
    @JsonKey(name: 'default_salesrep_id') int? defaultSalesrepId,
    @JsonKey(name: 'temp_car_number') int? tempCarNumber,
    @JsonKey(name: 'company_admin_salesrep_id') int? companyAdminSalesrepId,
    @JsonKey(name: 'about_url') String? aboutUrl,
    @JsonKey(name: 'support_email') String? supportEmail,
    @JsonKey(name: 'default_timeout') int? defaultTimeout,
    @JsonKey(name: 'frontend_url') String? frontendUrl,
    // secrets（遮罩值；app 不使用）
    @JsonKey(name: 'netsuite_account_id') String? netsuiteAccountId,
    @JsonKey(name: 'netsuite_consumer_key') String? netsuiteConsumerKey,
    @JsonKey(name: 'netsuite_consumer_secret') String? netsuiteConsumerSecret,
    @JsonKey(name: 'netsuite_token_id') String? netsuiteTokenId,
    @JsonKey(name: 'netsuite_token_secret') String? netsuiteTokenSecret,
    @JsonKey(name: 'email_password') String? emailPassword,
    // email 非 secret
    @JsonKey(name: 'email_host') String? emailHost,
    @JsonKey(name: 'email_port') String? emailPort,
    @JsonKey(name: 'email_identity') String? emailIdentity,
    @JsonKey(name: 'email_username') String? emailUsername,
    @JsonKey(name: 'email_from') String? emailFrom,
  }) = _SettingsModel;

  factory SettingsModel.fromJson(Map<String, dynamic> json) => _$SettingsModelFromJson(json);
}
```

- [ ] **Step 2: 產生碼**

Run: `cd sales-order-app && fvm dart run build_runner build --delete-conflicting-outputs`
Expected: `setting.freezed.dart`、`setting.g.dart` 產生。

- [ ] **Step 3: 驗證**

Run: `cd sales-order-app && fvm flutter analyze lib/layer_data/models/setting`
Expected: No issues found。

- [ ] **Step 4: Commit**

```bash
cd sales-order-app && git add lib/layer_data/models/setting && git commit -m "feat(models): add SettingsModel"
```

---

### Task 2: SettingsApi + Endpoints + 抽象介面

**Files:**
- Create: `sales-order-app/lib/layer_business/network/api/settings_api.dart`
- Create: `sales-order-app/lib/layer_business/network/abstract/settings_api_type.dart`
- Modify: `sales-order-app/lib/layer_business/network/api/endpoints.dart`

**Interfaces:**
- Consumes: `DioClient`、`CacheOptionsMixin`（既有）、`Endpoints.getSettings`。
- Produces: `Endpoints.getSettings = "$_apiVersion/settings"`；`SettingsApiType`（`Future<Response> get({Options? options})`、`removeCookies()`）；`SettingsApi`；`settingsApi = Provider(...)`。

- [ ] **Step 1: endpoints.dart 加路徑**

於 `static const String _article = "articles";` 之後加：

```dart
  static const String _setting = "settings";
```

於既有 endpoint 常數區（`getArticles` 附近）加：

```dart
  static const String getSettings = "$_apiVersion/$_setting";
```

- [ ] **Step 2: 抽象介面**

```dart
import 'package:dio/dio.dart';

abstract class SettingsApiType {
  Future<Response> get({Options? options});
  Future<void> removeCookies();
}
```

- [ ] **Step 3: 實作 SettingsApi**

```dart
import 'package:dio/dio.dart';
import 'package:disco/disco.dart';
import 'package:hexagon_food_app/layer_business/network/abstract/settings_api_type.dart';
import 'package:hexagon_food_app/layer_business/network/api/cache_options_mixin.dart';
import 'package:hexagon_food_app/layer_business/network/api/endpoints.dart';
import 'package:hexagon_food_app/layer_business/utils/locator.dart';

import '../dio_client.dart';

final settingsApi = Provider((context) => SettingsApi(dioClient: locator<DioClient>()));

class SettingsApi extends SettingsApiType with CacheOptionsMixin {
  SettingsApi({required this.dioClient});

  final DioClient dioClient;

  Options get defaultCache => smCache;

  @override
  Future<Response> get({Options? options}) async {
    try {
      return await dioClient.get(
        Endpoints.getSettings,
        options: options ?? defaultCache,
      );
    } catch (e) {
      rethrow;
    }
  }

  @override
  Future<void> removeCookies() async {
    dioClient.removeCookiesJar();
    await cleanCache();
  }
}
```

（參照 `department_api.dart`；`smCache` 5 分鐘 — 設定頁異動後 app 不致過久取不到新值。）

- [ ] **Step 4: 驗證**

Run: `cd sales-order-app && fvm flutter analyze lib/layer_business/network`
Expected: No issues found。

- [ ] **Step 5: Commit**

```bash
cd sales-order-app && git add lib/layer_business/network && git commit -m "feat(network): add settings API"
```

---

### Task 3: SettingsService（Signal + Sembast 快取 + fallback）

**Files:**
- Create: `sales-order-app/lib/layer_business/services/settings/settings_service.dart`
- Modify: `sales-order-app/lib/layer_business/utils/locator.dart`（註冊 `settingsStorage` + `SettingsService`）
- Modify: `sales-order-app/lib/layer_data/constants/system_constants.dart`（fallback 數值更新）

**Interfaces:**
- Consumes: `SettingsApi`（Task 2）、`SembastKvStorage`（既有）、`SettingsModel`（Task 1）。
- Produces: `SettingsStore`（`Future<Map<String, dynamic>?> readJson()`、`Future<void> writeJson(Map<String, dynamic>)`）＋ `SembastSettingsStore` 實作；`SettingsService`（`final Signal<SettingsModel?> settingsSignal`、`Future<void> refresh()`、`Future<void> loadCache()`、`SettingsModel get current`、`SettingsModel get fallback`）；`SystemConstants` fallback 數值更新（`defaultDepartment = 6`、新增 `frontendUrl`）。

- [ ] **Step 1: 更新 SystemConstants fallback**

`lib/layer_data/constants/system_constants.dart`：

- `defaultDepartment` 1 → 6（與 backend seed 一致）。
- 新增 `static const String frontendUrl = '';`（執行期由 settings 覆寫；deeplink/manuals 用）。
- `deeplinkCompanyLink` 保留但標註 deprecated（由 `frontendUrl` 組裝）。

```dart
class SystemConstants {
  static const String aboutUrl = 'https://www.hexagonty.com';
  static const String supportEmail = 'hexagon@hexagonty.com';
  static const int defaultTimeout = 30; // seconds
  static const int defaultDepartment = 6; // 修正：與 backend seed 一致
  static const int systemDepartmentId = -16888; // Default system department ID
  static const int systemSalesrepId = -16888; // Default system sales representative ID
  static const int defaultTestCustomerId = -17888; // Default test customer ID
  static const int defaultTestSalesrepId = -17888; // Default test sales representative ID
  static const int companyAdminSalesrepId = -5; // Default company sales representative admin user ID
  static const String frontendUrl = ''; // 執行期由 backend settings 覆寫
  @Deprecated('由 settings.frontendUrl 組裝')
  static String get deeplinkCompanyLink => '$frontendUrl/customer_account_qrcode';
}
```

- [ ] **Step 2: 建立 SettingsStore 介面 + 實作**

```dart
// 獨立檔案：lib/layer_business/services/settings/settings_store.dart
import 'package:hexagon_food_app/layer_data/repositories/sembast_kv_storage.dart';

/// 設定快取介面（測試可注入 in-memory fake）。
abstract class SettingsStore {
  Future<Map<String, dynamic>?> readJson();
  Future<void> writeJson(Map<String, dynamic> value);
}

/// Sembast 實作（單一 key 'settings'）。
class SembastSettingsStore implements SettingsStore {
  SembastSettingsStore(this._storage);

  final SembastKvStorage _storage;
  static const String _cacheKey = 'settings';

  @override
  Future<Map<String, dynamic>?> readJson() => _storage.readJson(_cacheKey);

  @override
  Future<void> writeJson(Map<String, dynamic> value) => _storage.writeJson(_cacheKey, value);
}
```

- [ ] **Step 3: 建立 SettingsService**

```dart
import 'package:flutter_solidart/flutter_solidart.dart';
import 'package:hexagon_food_app/layer_business/network/api/settings_api.dart';
import 'package:hexagon_food_app/layer_business/services/settings/settings_store.dart';
import 'package:hexagon_food_app/layer_data/constants/system_constants.dart';
import 'package:hexagon_food_app/layer_data/models/setting/setting.dart';

/// 系統設定服務：登入後由 backend 取得並快取；失敗回退快取 → 編譯期 fallback。
class SettingsService {
  SettingsService({required SettingsApi api, required SettingsStore store})
      : _api = api,
        _store = store;

  final SettingsApi _api;
  final SettingsStore _store;

  final Signal<SettingsModel?> settingsSignal = Signal<SettingsModel?>(null);

  SettingsModel? _cachedValue;

  /// 目前有效設定：Signal 值 → 快取 → 編譯期 fallback。
  SettingsModel get current => settingsSignal.value ?? _cachedValue ?? fallback;

  SettingsModel get fallback => SettingsModel(
        defaultDepartmentId: SystemConstants.defaultDepartment,
        systemDepartmentId: SystemConstants.systemDepartmentId,
        systemSalesrepId: SystemConstants.systemSalesrepId,
        systemTestSalesrepId: SystemConstants.defaultTestSalesrepId,
        companyAdminSalesrepId: SystemConstants.companyAdminSalesrepId,
        aboutUrl: SystemConstants.aboutUrl,
        supportEmail: SystemConstants.supportEmail,
        defaultTimeout: SystemConstants.defaultTimeout,
        frontendUrl: SystemConstants.frontendUrl,
      );

  /// 讀取快取（app 啟動時呼叫一次）。
  Future<void> loadCache() async {
    final json = await _store.readJson();
    if (json != null) {
      _cachedValue = SettingsModel.fromJson(json);
      settingsSignal.value = _cachedValue;
    }
  }

  /// 由 backend 重新取得並寫入快取；失敗時保留既有值。
  Future<void> refresh() async {
    try {
      final resp = await _api.get();
      final json = resp.data as Map<String, dynamic>;
      final model = SettingsModel.fromJson(json);
      _cachedValue = model;
      settingsSignal.value = model;
      await _store.writeJson(json);
    } catch (_) {
      // 網路失敗：保留快取/Signal；離線時以 current 回退
    }
  }
}
```

- [ ] **Step 4: locator 註冊**

`lib/layer_business/utils/locator.dart` 的 `setupLocator()`：參照既有 Sembast storages（session/cookies/cache）註冊方式新增：

```dart
    final settingsStorage = SembastKvStorage(storePath: appDocDir.path, storeName: 'settings_store');
    locator.registerSingleton(settingsStorage);
    locator.registerLazySingleton<SettingsService>(
      () => SettingsService(
        api: SettingsApi(dioClient: locator<DioClient>()),
        store: SembastSettingsStore(settingsStorage),
      ),
    );
```

（確切寫法依 locator.dart 現有註冊慣例調整 — 參照 `PersistCookieJar`/`AuthSessionManager` 的註冊方式；`appDocDir` 以 locator 內既有變數為準。）

- [ ] **Step 5: 驗證**

Run: `cd sales-order-app && fvm flutter analyze lib/layer_business/services/settings lib/layer_business/utils/locator.dart lib/layer_data/constants/system_constants.dart`
Expected: No issues found。

- [ ] **Step 6: Commit**

```bash
cd sales-order-app && git add lib/layer_business/services/settings lib/layer_business/utils/locator.dart lib/layer_data/constants/system_constants.dart && git commit -m "feat(settings): add SettingsService with cache and fallback"
```

---

### Task 4: 登入後觸發 refresh + 啟動載入快取

**Files:**
- Modify: `sales-order-app/lib/layer_business/services/auth/provider.dart`（session 訂閱處）

**Interfaces:**
- Consumes: `SettingsService`（Task 3）、`AuthSessionManager.sessionStream`（既有）。
- Produces: 登入成功（session 生效）時 `unawaited(settingsService.refresh())`；app 啟動時 `loadCache()`。

- [ ] **Step 1: AuthProvider 觸發 refresh**

於 `lib/layer_business/services/auth/provider.dart` 中訂閱 `AuthSessionManager.sessionStream` 的既有 listener 內，偵測 session 由無變有時（登入成功）加入：

```dart
        if (session != null) {
          unawaited(locator<SettingsService>().refresh());
        }
```

（`locator` 已 import；`unawaited` 自 `dart:async`。）

- [ ] **Step 2: 啟動載入快取**

於 `lib/main.dart` 初始化流程（`setupLocator()` 之後、`runApp` 前）加入：

```dart
  await locator<SettingsService>().loadCache();
```

（若 main 初始化為同步，改為 `unawaited(locator<SettingsService>().loadCache())` — 以不阻塞首幀為原則。）

- [ ] **Step 3: 驗證**

Run: `cd sales-order-app && fvm flutter analyze lib/main.dart lib/layer_business/services/auth/provider.dart`
Expected: No issues found。

- [ ] **Step 4: Commit**

```bash
cd sales-order-app && git add lib/main.dart lib/layer_business/services/auth/provider.dart && git commit -m "feat(settings): refresh on login and load cache on startup"
```

---

### Task 5: 遷移 — services（3 檔）

**Files:**
- Modify: `sales-order-app/lib/layer_business/services/customer/customer_service.dart`
- Modify: `sales-order-app/lib/layer_business/services/profile/profile_service.dart`
- Modify: `sales-order-app/lib/layer_business/services/salesorder/salesorder_service.dart`

**Interfaces:**
- Consumes: `SettingsService`（Task 3，`locator<SettingsService>().current`）。
- Produces: 上述 service 不再直接讀 `SystemConstants.*`（fallback 類別除外）。

- [ ] **Step 1: customer_service.dart**

- 建構時取得 `final _settings = locator<SettingsService>().current;`（或在需要處直接 `locator<SettingsService>().current`）。
- 第 82 行 `currentDepartmentId == SystemConstants.systemDepartmentId` → `== _settings.systemDepartmentId`
- 第 88 行 `"${SystemConstants.deeplinkCompanyLink}/${customer?.entityId}"` → `"${_settings.frontendUrl}/customer_account_qrcode/${customer?.entityId}"`
- 第 125 行 `e.id != SystemConstants.systemDepartmentId` → `e.id != _settings.systemDepartmentId`
- 移除 `SystemConstants` import（若無其他使用）。

- [ ] **Step 2: profile_service.dart**

- 第 130 行 `onOpenUrl(SystemConstants.aboutUrl)` → `onOpenUrl(locator<SettingsService>().current.aboutUrl ?? SystemConstants.aboutUrl)`

- [ ] **Step 3: salesorder_service.dart**

- 第 231-232 行：

```dart
    final s = locator<SettingsService>().current;
    form.salesrepControl.value =
        (salesrep == s.systemSalesrepId || salesrep == s.systemTestSalesrepId)
        ? (s.companyAdminSalesrepId ?? SystemConstants.companyAdminSalesrepId)
        : salesrep ?? 0;
```

- [ ] **Step 4: 驗證**

Run: `cd sales-order-app && fvm flutter analyze lib/layer_business/services`
Expected: No issues found。

- [ ] **Step 5: Commit**

```bash
cd sales-order-app && git add lib/layer_business/services && git commit -m "refactor(services): read settings from SettingsService"
```

---

### Task 6: 遷移 — stories（6 檔）

**Files:**
- Modify: `sales-order-app/lib/layer_presentation/stories/admin/customer/customer_create_screen.dart`
- Modify: `sales-order-app/lib/layer_presentation/stories/admin/customer/form/main_edit_form.dart`
- Modify: `sales-order-app/lib/layer_presentation/stories/admin/customer/form/main_form.dart`
- Modify: `sales-order-app/lib/layer_presentation/stories/admin/salesorder_item/salesorder_item_screen.dart`
- Modify: `sales-order-app/lib/layer_presentation/stories/admin/tabs/profile/widgets/profile_list.dart`
- Modify: `sales-order-app/lib/layer_presentation/stories/admin/tabs/salesorder/form/salesorder_cards.dart`

**Interfaces:**
- Consumes: `SettingsService`（Task 3，`locator<SettingsService>().current`）。
- Produces: 上述 widget 不再直接讀 `SystemConstants.*`。

- [ ] **Step 1: customer_create_screen.dart**

- 第 26 行 `cp.salesrepInfo?.id == SystemConstants.systemSalesrepId ? -5 : ...` → 以 `final s = locator<SettingsService>().current;` 取 `s.systemSalesrepId`；`-5` 改 `s.companyAdminSalesrepId ?? -5`。

- [ ] **Step 2: main_edit_form.dart / main_form.dart**

- 第 29/28 行 `e.id != SystemConstants.systemSalesrepId` → `e.id != s.systemSalesrepId`（`s` 於 build 前取得）。

- [ ] **Step 3: salesorder_item_screen.dart**

- 第 85-90 行：

```dart
      final s = locator<SettingsService>().current;
      if (form.salesrepControl.value == s.systemSalesrepId) {
        form.salesrepControl.value = s.companyAdminSalesrepId ?? SystemConstants.companyAdminSalesrepId;
      }
      if (form.departmentControl.value == s.systemDepartmentId) {
        form.departmentControl.value = s.defaultDepartmentId ?? SystemConstants.defaultDepartment;
      }
```

- [ ] **Step 4: profile_list.dart**

- 第 23 行 `cp.salesrepInfo?.id == SystemConstants.systemSalesrepId` → `== locator<SettingsService>().current.systemSalesrepId`

- [ ] **Step 5: salesorder_cards.dart**

- 第 43-44 行 `e.id != SystemConstants.systemSalesrepId && e.id != SystemConstants.defaultTestSalesrepId` → 對應 `s.systemSalesrepId` / `s.systemTestSalesrepId`

- [ ] **Step 6: 驗證 + 殘留檢查**

Run: `cd sales-order-app && fvm flutter analyze lib`
Expected: No issues found。

Run: `grep -rn "SystemConstants\." lib/layer_business lib/layer_presentation`
Expected: 僅剩 `SystemConstants.aboutUrl`/`companyAdminSalesrepId`/`defaultDepartment` 等 fallback 使用（若為 null 時的回退）— 或已全數改為 `SettingsService` 取得；移除所有 `@Deprecated` 標記成員的使用（`deeplinkCompanyLink`）。

- [ ] **Step 7: Commit**

```bash
cd sales-order-app && git add lib/layer_presentation && git commit -m "refactor(stories): read settings from SettingsService"
```

---

### Task 7: 單元測試（fallback 邏輯）

**Files:**
- Create: `sales-order-app/test/services/settings_service_test.dart`

**Interfaces:**
- Consumes: `SettingsService`、`SettingsStore`（Task 3）。
- Produces: 測試涵蓋：無快取且 API 失敗 → fallback；有快取 → 快取值優先；refresh 成功 → Signal 更新 + 寫入快取。

- [ ] **Step 1: 寫測試**

```dart
import 'package:dio/dio.dart';
import 'package:flutter_test/flutter_test.dart';
import 'package:hexagon_food_app/layer_business/network/api/settings_api.dart';
import 'package:hexagon_food_app/layer_business/services/settings/settings_service.dart';
import 'package:hexagon_food_app/layer_business/services/settings/settings_store.dart';
import 'package:hexagon_food_app/layer_data/models/setting/setting.dart';

class _FakeStore implements SettingsStore {
  _FakeStore(this.json);
  Map<String, dynamic>? json;

  @override
  Future<Map<String, dynamic>?> readJson() async => json;

  @override
  Future<void> writeJson(Map<String, dynamic> value) async => json = value;
}

class _FakeApi {
  _FakeApi(this.onGet);
  Future<Response> Function() onGet;

  Future<Response> get({Options? options}) => onGet();
}

SettingsService _service(_FakeStore store, _FakeApi api) =>
    SettingsService(api: api as SettingsApi, store: store);

void main() {
  test('無快取且 API 失敗 → current 回退 fallback', () async {
    final store = _FakeStore(null);
    final api = _FakeApi(() async => throw Exception('network down'));
    final svc = _service(store, api as SettingsApi);

    await svc.loadCache();
    await svc.refresh();

    expect(svc.current.defaultDepartmentId, SystemConstants.defaultDepartment);
    expect(svc.settingsSignal.value, isNull);
  });

  test('有快取 → current 使用快取值', () async {
    final store = _FakeStore({
      'default_department_id': 6,
      'frontend_url': 'https://frontend.example.com',
    });
    final api = _FakeApi(() async => throw Exception('network down'));
    final svc = _service(store, api as SettingsApi);

    await svc.loadCache();

    expect(svc.current.defaultDepartmentId, 6);
    expect(svc.current.frontendUrl, 'https://frontend.example.com');
  });

  test('refresh 成功 → Signal 更新且寫入快取', () async {
    final store = _FakeStore(null);
    final api = _FakeApi(
      () async => Response<Map<String, dynamic>>(
        requestOptions: RequestOptions(path: '/api/v1/settings'),
        data: {'default_department_id': 6, 'system_salesrep_id': -16888},
      ),
    );
    final svc = _service(store, api as SettingsApi);

    await svc.refresh();

    expect(svc.settingsSignal.value?.systemSalesrepId, -16888);
    expect(store.json?['system_salesrep_id'], -16888);
  });
}
```

（`SettingsService` 建構參數為 `SettingsApi` 與 `SettingsStore` — `_FakeApi` 需以 `implements SettingsApi`（含 `SettingsApiType` 的 `removeCookies`）取代 cast；或將 `SettingsService` 建構參數改為 `SettingsApiType` 介面。以 `fvm flutter analyze` 結果調整，保持測試與服務解耦。`SystemConstants` 需 import `package:hexagon_food_app/layer_data/constants/system_constants.dart`。）

- [ ] **Step 2: 執行**

Run: `cd sales-order-app && fvm flutter test`
Expected: PASS（建立 `test/` 後 `fvm flutter test` 全綠）。

- [ ] **Step 3: Commit**

```bash
cd sales-order-app && git add test lib/layer_business/services/settings && git commit -m "test(settings): add fallback logic tests"
```

---

### Task 8: 文件 + 收尾

**Files:**
- Modify: `docs/AGENTS/app.md`（superproject root repo commit）

**Interfaces:**
- Produces: 文件說明 settings service 與 SystemConstants 角色轉變。

- [ ] **Step 1: 更新 app 指引**

`docs/AGENTS/app.md`：於資料層/服務層補 `SettingsService`（登入後 fetch、Sembast 快取、fallback）；說明 `SystemConstants` 已改為編譯期 fallback（`defaultDepartment = 6`、`frontendUrl` 由 backend 提供）。

- [ ] **Step 2: 冒煙驗證**

Run: `cd sales-order-app && fvm flutter run --flavor dev --target lib/main_dev.dart`（需 backend + 模擬器；若環境不可用，以 `fvm flutter test` + `fvm flutter analyze` 通過為準）— 登入後確認 customer 篩選（系統部門排除）、訂單表單業務員排除邏輯正常。

- [ ] **Step 3: Commit**

```bash
git add docs/AGENTS/app.md && git commit -m "docs(app): add settings service guide"
```
（於 superproject root 執行。）
