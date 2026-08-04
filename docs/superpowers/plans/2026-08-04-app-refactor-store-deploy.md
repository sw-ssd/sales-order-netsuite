# App 重構與商店上架 — 實作計畫

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** 漸進式重構 Flutter App：建立 CI/CD、統一狀態管理、補強測試、完成雙平台上架。

**Architecture:** 四個獨立階段依序執行。階段 1 建立基礎設施（CI/CD、i18n、測試框架），階段 2 統一狀態管理（ChangeNotifier → solidart），階段 3 補測試覆蓋（Fake-over-Mock），階段 4 完成商店上架（metadata、隱私政策、Fastlane 擴充）。

**Tech Stack:** Flutter 3.35.2 / Dart 3.9+ / solidart / disco / GetIt / Fastlane / GitHub Actions / Maestro

## Global Constraints

- Flutter 版本固定 3.35.2（`.fvmrc`）
- Dart SDK `>=3.9.0 <4.0.0`
- 繁體中文為主要語言；UI 字串逐步遷移至 ARB i18n
- 產生檔（`*.g.dart`、`*.freezed.dart`、`*.gform.dart` 等）修改來源後必須重新 `build_runner build`
- 每次 commit 必須可獨立建置；每個 task 完成後 `flutter analyze` 必須通過
- 測試使用 `flutter_test` + `mocktail`（備用），主力用 Fake objects
- 簽署金鑰一律從 GitHub Secrets 注入，不入版控
- 不新增業務功能，不變更後端 / 前端程式碼

---

## 階段 1：基礎建設

### Task 1: CI workflow (`ci.yml`)

**Files:**
- Create: `.github/workflows/ci.yml`

**Interfaces:**
- Consumes: 無
- Produces: GitHub Actions workflow，每次 push/PR 觸發 `flutter analyze` + `flutter test`

- [ ] **Step 1: 建立 CI workflow 檔案**

```yaml
# .github/workflows/ci.yml
name: CI

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

jobs:
  analyze-and-test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
        with:
          submodules: true

      - name: Read Flutter version from .fvmrc
        id: fvm
        run: |
          FVM_VERSION=$(cat sales-order-app/.fvmrc | jq -r '.flutter')
          echo "version=$FVM_VERSION" >> $GITHUB_OUTPUT

      - uses: subosito/flutter-action@v2
        with:
          flutter-version: ${{ steps.fvm.outputs.version }}
          channel: stable

      - name: Install dependencies
        working-directory: sales-order-app
        run: flutter pub get

      - name: Run code generation
        working-directory: sales-order-app
        run: dart run build_runner build --delete-conflicting-outputs

      - name: Analyze
        working-directory: sales-order-app
        run: flutter analyze

      - name: Run tests
        working-directory: sales-order-app
        run: flutter test
```

- [ ] **Step 2: 驗證 workflow 語法**

```bash
# 無 GitHub CLI 時可手動檢查 YAML 結構
cat .github/workflows/ci.yml | python3 -c "import sys,yaml; yaml.safe_load(sys.stdin)"
```

- [ ] **Step 3: Commit**

```bash
git add .github/workflows/ci.yml
git commit -m "ci: add analyze + test workflow for PRs"
```

---

### Task 2: Android build workflow

**Files:**
- Create: `.github/workflows/build-android.yml`

**Interfaces:**
- Consumes: 無
- Produces: `workflow_dispatch` workflow，選擇 flavor → 建置 AAB → 上傳 artifact

- [ ] **Step 1: 建立 Android build workflow**

```yaml
# .github/workflows/build-android.yml
name: Build Android

on:
  workflow_dispatch:
    inputs:
      flavor:
        description: 'Flavor (dev or prod)'
        required: true
        default: 'dev'
        type: choice
        options:
          - dev
          - prod

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
        with:
          submodules: true

      - name: Read Flutter version from .fvmrc
        id: fvm
        run: |
          FVM_VERSION=$(cat sales-order-app/.fvmrc | jq -r '.flutter')
          echo "version=$FVM_VERSION" >> $GITHUB_OUTPUT

      - uses: subosito/flutter-action@v2
        with:
          flutter-version: ${{ steps.fvm.outputs.version }}
          channel: stable

      - name: Setup Java
        uses: actions/setup-java@v4
        with:
          distribution: temurin
          java-version: '17'

      - name: Setup Android keystore
        env:
          KEYSTORE_BASE64: ${{ secrets.ANDROID_KEYSTORE_BASE64 }}
          KEY_PROPERTIES: ${{ secrets.ANDROID_KEY_PROPERTIES }}
        run: |
          mkdir -p sales-order-app/android/keystore
          echo "$KEYSTORE_BASE64" | base64 -d > sales-order-app/android/keystore/hexagon-salesorder-keystore.jks
          echo "$KEY_PROPERTIES" > sales-order-app/android/key.properties

      - name: Install dependencies
        working-directory: sales-order-app
        run: flutter pub get

      - name: Run code generation
        working-directory: sales-order-app
        run: dart run build_runner build --delete-conflicting-outputs

      - name: Build AAB
        working-directory: sales-order-app
        run: |
          if [ "${{ inputs.flavor }}" = "prod" ]; then
            flutter build appbundle --flavor prod --target lib/main_prod.dart
          else
            flutter build appbundle --flavor dev --target lib/main_dev.dart
          fi

      - name: Upload AAB artifact
        uses: actions/upload-artifact@v4
        with:
          name: app-${{ inputs.flavor }}-release
          path: sales-order-app/build/app/outputs/bundle/${{ inputs.flavor }}Release/app-${{ inputs.flavor }}-release.aab
```

- [ ] **Step 2: Commit**

```bash
git add .github/workflows/build-android.yml
git commit -m "ci: add Android AAB build workflow with flavor selection"
```

---

### Task 3: iOS build workflow

**Files:**
- Create: `.github/workflows/build-ios.yml`

**Interfaces:**
- Consumes: 無
- Produces: `workflow_dispatch` workflow，選擇 flavor → 建置 IPA（無簽署，供手動上傳）

- [ ] **Step 1: 建立 iOS build workflow**

```yaml
# .github/workflows/build-ios.yml
name: Build iOS

on:
  workflow_dispatch:
    inputs:
      flavor:
        description: 'Flavor (dev or prod)'
        required: true
        default: 'dev'
        type: choice
        options:
          - dev
          - prod

jobs:
  build:
    runs-on: macos-latest
    steps:
      - uses: actions/checkout@v4
        with:
          submodules: true

      - name: Read Flutter version from .fvmrc
        id: fvm
        run: |
          FVM_VERSION=$(cat sales-order-app/.fvmrc | jq -r '.flutter')
          echo "version=$FVM_VERSION" >> $GITHUB_OUTPUT

      - uses: subosito/flutter-action@v2
        with:
          flutter-version: ${{ steps.fvm.outputs.version }}
          channel: stable

      - name: Install dependencies
        working-directory: sales-order-app
        run: flutter pub get

      - name: Run code generation
        working-directory: sales-order-app
        run: dart run build_runner build --delete-conflicting-outputs

      - name: Install CocoaPods
        working-directory: sales-order-app/ios
        run: pod install

      - name: Build iOS (no sign)
        working-directory: sales-order-app
        run: |
          # dev flavor has no Release-dev Xcode config (only Debug-dev/Profile-dev),
          # so dev builds as debug; prod builds as release. Output dir is
          # build/ios/<Configuration>-iphoneos/ in both cases.
          if [ "${{ inputs.flavor }}" = "prod" ]; then
            flutter build ios --flavor prod --target lib/main_prod.dart --no-codesign
          else
            flutter build ios --flavor dev --target lib/main_dev.dart --debug --no-codesign
          fi

      - name: Package IPA
        working-directory: sales-order-app
        run: |
          # Flavor builds output to build/ios/<Configuration>-iphoneos/ (e.g.
          # Release-prod-iphoneos), NOT Release-iphoneos; PRODUCT_NAME may also
          # carry a flavor suffix ("Runner Dev.app"). Resolve dynamically.
          APP_DIR=$(ls -d build/ios/*-iphoneos 2>/dev/null | head -1)
          if [ -z "$APP_DIR" ] || [ ! -d "$APP_DIR" ]; then
            echo "No build/ios/*-iphoneos output found" >&2
            exit 1
          fi
          APP_BUNDLE=$(find "$APP_DIR" -maxdepth 1 -name "*.app" | head -1)
          if [ -z "$APP_BUNDLE" ]; then
            echo "No .app bundle found in $APP_DIR" >&2
            exit 1
          fi
          mkdir -p Payload
          cp -r "$APP_BUNDLE" Payload/
          zip -r app-${{ inputs.flavor }}.ipa Payload/
          rm -rf Payload

      - name: Upload IPA artifact
        uses: actions/upload-artifact@v4
        with:
          name: app-${{ inputs.flavor }}-ios
          path: sales-order-app/app-${{ inputs.flavor }}.ipa
```

- [ ] **Step 2: Commit**

```bash
git add .github/workflows/build-ios.yml
git commit -m "ci: add iOS IPA build workflow with flavor selection"
```

---

### Task 4: i18n 基礎框架

**Files:**
- Create: `lib/l10n/app_zh.arb`
- Create: `lib/l10n/app_en.arb`
- Modify: `l10n.yaml`
- Modify: `lib/layer_presentation/app.dart`

**Interfaces:**
- Consumes: 無
- Produces: `AppLocalizations` 可用，`T.of(context).xxx` helper 簡化語法

- [ ] **Step 1: 更新 l10n.yaml 指向新目錄**

```
[sales-order-app/l10n.yaml#93D0]
PUT 1.=6:
+arb-dir: lib/l10n
+output-dir: lib/l10n
+template-arb-file: app_zh.arb
+output-localization-file: app_localizations.dart
+nullable-getter: false
+synthetic-package: false
```

- [ ] **Step 2: 建立繁體中文 ARB 模板**

```json
{
  "@@locale": "zh",
  "appTitle": "特耀訂出貨系統",
  "@appTitle": {
    "description": "應用程式標題"
  }
}
```

```bash
# 寫入 lib/l10n/app_zh.arb
cat > sales-order-app/lib/l10n/app_zh.arb << 'EOF'
{
  "@@locale": "zh",
  "appTitle": "特耀訂出貨系統",
  "@appTitle": {
    "description": "應用程式標題"
  }
}
EOF
```

- [ ] **Step 3: 建立英文 ARB（佔位，後續補翻譯）**

```json
{
  "@@locale": "en",
  "appTitle": "Hexagon Sales Order"
}
```

```bash
cat > sales-order-app/lib/l10n/app_en.arb << 'EOF'
{
  "@@locale": "en",
  "appTitle": "Hexagon Sales Order"
}
EOF
```

- [ ] **Step 4: 加入 flutter_localizations 依賴**

檢查 `pubspec.yaml` 是否已有 `flutter_localizations`（Flutter SDK 內建，通常在 `dependencies.flutter.sdk` 下方）。若無則手動加入：

```
[sales-order-app/pubspec.yaml#F902]
PUT 13*:
+  flutter:
+    sdk: flutter
+  flutter_localizations:
+    sdk: flutter
```

- [ ] **Step 5: 建立 i18n helper**

建立 `lib/l10n/t.dart`：

```dart
// lib/l10n/t.dart
import 'package:flutter/widgets.dart';
import 'app_localizations.dart';

/// 簡化 i18n 語法的 helper。
/// 用法：`T.of(context).appTitle`
class T {
  const T._(this._loc);
  final AppLocalizations _loc;

  static T of(BuildContext context) => T._(AppLocalizations.of(context)!);

  String get appTitle => _loc.appTitle;
}
```

- [ ] **Step 6: 在 App widget 註冊 localizationsDelegates**

```
[sales-order-app/lib/layer_presentation/app.dart#06EE]
PUT 56.=57:
+            localizationsDelegates: AppLocalizations.localizationsDelegates,
+            supportedLocales: AppLocalizations.supportedLocales,
```

- [ ] **Step 7: 重新產生 localizations 檔案**

```bash
cd sales-order-app
fvm flutter gen-l10n
echo "// ignore_for_file: type=lint" | cat - lib/l10n/app_localizations.dart > /tmp/l10n_temp && mv /tmp/l10n_temp lib/l10n/app_localizations.dart
```

- [ ] **Step 8: 移除舊的 localizations 目錄**

```bash
rm -rf sales-order-app/lib/localizations/
```

- [ ] **Step 9: Commit**

```bash
git add sales-order-app/l10n.yaml sales-order-app/lib/l10n/ sales-order-app/lib/layer_presentation/app.dart
git commit -m "feat: i18n framework with zh/en ARB files and T helper"
```

---

### Task 5: iOS PrivacyInfo.xcprivacy

**Files:**
- Create: `ios/Runner/PrivacyInfo.xcprivacy`

**Interfaces:**
- Consumes: 無
- Produces: iOS 17+ 必要隱私清單，宣告 Firebase Crashlytics / Analytics

- [ ] **Step 1: 建立隱私清單**

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
    <key>NSPrivacyTracking</key>
    <false/>
    <key>NSPrivacyTrackingDomains</key>
    <array/>
    <key>NSPrivacyCollectedDataTypes</key>
    <array>
        <dict>
            <key>NSPrivacyCollectedDataType</key>
            <string>NSPrivacyCollectedDataTypeCrashData</string>
            <key>NSPrivacyCollectedDataTypeLinkedToUser</key>
            <false/>
            <key>NSPrivacyCollectedDataTypeUsedForTracking</key>
            <false/>
            <key>NSPrivacyCollectedDataTypePurposes</key>
            <array>
                <string>NSPrivacyCollectedDataTypePurposeAppFunctionality</string>
            </array>
        </dict>
        <dict>
            <key>NSPrivacyCollectedDataType</key>
            <string>NSPrivacyCollectedDataTypeProductInteraction</string>
            <key>NSPrivacyCollectedDataTypeLinkedToUser</key>
            <false/>
            <key>NSPrivacyCollectedDataTypeUsedForTracking</key>
            <false/>
            <key>NSPrivacyCollectedDataTypePurposes</key>
            <array>
                <string>NSPrivacyCollectedDataTypePurposeAnalytics</string>
            </array>
        </dict>
    </array>
    <key>NSPrivacyAccessedAPITypes</key>
    <array>
        <dict>
            <key>NSPrivacyAccessedAPIType</key>
            <string>NSPrivacyAccessedAPICategoryUserDefaults</string>
            <key>NSPrivacyAccessedAPITypeReasons</key>
            <array>
                <string>CA92.1</string>
            </array>
        </dict>
        <dict>
            <key>NSPrivacyAccessedAPIType</key>
            <string>NSPrivacyAccessedAPICategoryFileTimestamp</string>
            <key>NSPrivacyAccessedAPITypeReasons</key>
            <array>
                <string>C617.1</string>
            </array>
        </dict>
        <dict>
            <key>NSPrivacyAccessedAPIType</key>
            <string>NSPrivacyAccessedAPICategorySystemBootTime</string>
            <key>NSPrivacyAccessedAPITypeReasons</key>
            <array>
                <string>35F9.1</string>
            </array>
        </dict>
        <dict>
            <key>NSPrivacyAccessedAPIType</key>
            <string>NSPrivacyAccessedAPICategoryDiskSpace</string>
            <key>NSPrivacyAccessedAPITypeReasons</key>
            <array>
                <string>E174.1</string>
            </array>
        </dict>
    </array>
</dict>
</plist>
```

- [ ] **Step 2: Commit**

```bash
git add sales-order-app/ios/Runner/PrivacyInfo.xcprivacy
git commit -m "feat: add iOS PrivacyInfo.xcprivacy for App Store compliance"
```

---

### Task 6: 測試基礎設施（test helpers）

**Files:**
- Create: `test/helpers/test_locator.dart`
- Create: `test/helpers/fake_api.dart`
- Create: `test/helpers/pump_app.dart`

**Interfaces:**
- Consumes: 無
- Produces:
  - `FakeAuthApi implements AuthApiType` — 可控制回傳值/錯誤
  - `TestLocator.setup({...})` — 接受測試替身
  - `pumpApp(WidgetTester, {AuthService?})` — 標準 widget pump

- [ ] **Step 1: 建立 FakeAuthApi**

```dart
// test/helpers/fake_api.dart
import 'package:hexagon_food_app/layer_business/network/abstract/auth_api_type.dart';
import 'package:dio/dio.dart';

class FakeAuthApi extends AuthApiType {
  final List<dynamic> _responses = [];
  String? _simulateError;

  /// 依序放入 API 呼叫的回傳資料（如 [getCsrfToken] → [getMe] → [signin]）。
  void queueResponses(List<dynamic> responses) => _responses.addAll(responses);
  void setError(String error) => _simulateError = error;

  dynamic _next() => _responses.isNotEmpty ? _responses.removeAt(0) : null;

  @override
  Future<Response> signin(Map<String, dynamic> formData, Options options) async {
    if (_simulateError != null) throw DioException(requestOptions: RequestOptions(path: ''), message: _simulateError);
    return Response(requestOptions: RequestOptions(path: ''), statusCode: 200, data: _next());
  }

  @override
  Future<Response> customerSignin(Map<String, dynamic> formData, Options options) async {
    if (_simulateError != null) throw DioException(requestOptions: RequestOptions(path: ''), message: _simulateError);
    return Response(requestOptions: RequestOptions(path: ''), statusCode: 200, data: _next());
  }

  @override
  Future<Response> signout(String utype) async => Response(requestOptions: RequestOptions(path: ''), statusCode: 200);
  @override
  Future<Response> forceSignout(String utype, int userId) async => Response(requestOptions: RequestOptions(path: ''), statusCode: 200);
  @override
  Future<Response> resetNullPassword(String utype, int userId) async => Response(requestOptions: RequestOptions(path: ''), statusCode: 200);
  @override
  Future<Response> getMe() async {
    if (_simulateError != null) throw DioException(requestOptions: RequestOptions(path: ''), message: _simulateError);
    return Response(requestOptions: RequestOptions(path: ''), statusCode: 200, data: _next());
  }
  @override
  Future<Response> getCsrfToken() async => Response(requestOptions: RequestOptions(path: ''), statusCode: 200, data: _next());
  @override
  Future<Response> validEmail(String email) async => Response(requestOptions: RequestOptions(path: ''), data: _next());
  @override
  Future<Response> validEntityId(String entity) async => Response(requestOptions: RequestOptions(path: ''), data: _next());
  @override
  Future<Response> newsetSalesrepPassword(Map<String, dynamic> formData, Options options) async => Response(requestOptions: RequestOptions(path: ''), statusCode: 200);
  @override
  Future<Response> newsetCustomerPassword(Map<String, dynamic> formData, Options options) async => Response(requestOptions: RequestOptions(path: ''), statusCode: 200);
  @override
  Future<void> removeCookies() async {}
}
```

- [ ] **Step 2: 建立 TestLocator helper**

```dart
// test/helpers/test_locator.dart
import 'package:get_it/get_it.dart';
import 'package:hexagon_food_app/layer_business/utils/locator.dart';

/// 測試用的 DI 設定輔助。
/// 用法：`await TestLocator.setup(authApi: fakeAuthApi);`
class TestLocator {
  static Future<void> setup({
    dynamic authApi,
  }) async {
    // 確保 locator 乾淨
    await locator.reset();

    // 若有注入的替身，直接註冊
    if (authApi != null) {
      locator.registerSingleton(authApi);
    }
  }
}
```

- [ ] **Step 3: 建立 pumpApp helper**

```dart
// test/helpers/pump_app.dart
import 'package:flutter/material.dart';
import 'package:flutter_test/flutter_test.dart';
import 'package:hexagon_food_app/layer_presentation/app.dart';

/// 標準 widget pump wrapper。
/// 提供 MaterialApp.router + ProviderScope 環境。
Future<void> pumpApp(
  WidgetTester tester, {
  Widget? child,
}) async {
  await tester.pumpWidget(
    MaterialApp(
      home: child ?? const Scaffold(body: Center(child: Text('Test'))),
    ),
  );
  await tester.pumpAndSettle();
}
```

- [ ] **Step 4: Commit**

```bash
git add sales-order-app/test/helpers/
git commit -m "test: add test helpers (fake API, locator, pumpApp)"
```

---

## 階段 2：狀態管理統一

### Task 7: AuthProvider → AuthService 重構（完整移植）

**Files:**
- Create: `lib/layer_business/services/auth/auth_service.dart`
- Modify: `lib/layer_presentation/app.dart`
- Modify: `lib/layer_presentation/general/widgets/session_expiry_overlay.dart`
- Modify: `lib/layer_presentation/stories/auth/salesrep_signin_screen.dart`
- Modify: `lib/layer_presentation/stories/auth/customer_signin_screen.dart`
- Modify: `lib/layer_presentation/stories/auth/customer_qrcode_signin_screen.dart`
- Delete: `lib/layer_business/services/auth/provider.dart`

**Interfaces:**
- Consumes: `AuthApi`（從 GetIt 注入，`locator<AuthApi>()`）、`AuthSessionManager`（從 GetIt 注入）
- Produces: `AuthService` — **完整移植**舊 `AuthProvider` 所有公開方法（`signinHandler`、`customerSigninHandler`、`signoutHandler`、`forceSignoutHandler`、`meHandler`、`clearLocalSession`、`restorePreviousPageIfAny`、`resetNullPasswordHandler`、`validEmailHandler`、`validExternalIdHandler`、`newsetSalesrepPasswordHandler`、`newsetCustomerPasswordHandler`）+ getters（`isLogin`、`userType`、`isCustomer`、`isSalesRep`、`createdById`）+ `dispose()`；狀態改以 solidart `Signal` 暴露

> **重要（controller 修正）**：這是**行為不變的移植**，不是重寫。
> - 所有對外方法**簽名完全不變**（畫面與 `routes.dart` 依賴它們）。不要發明新方法名（如 `salesrepSignIn`），不要刪除方法。
> - **不要發明 `AuthStatus` enum** — codebase 沒有這種東西。登入狀態由 `AuthSessionManager.isLoggedIn` 決定。
> - `Signal<T>` 本身實現 `ValueNotifier<T>`（`addListener`/`removeListener`），**沒有 `subscribe` 方法**。要監聽 signal 請用 `addListener` 或 `SignalBuilder`。
> - 舊 `AuthProvider` 的方法會 throw `Exception(errorException.message)` 給畫面 try/catch — 保留此行為，畫面錯誤顯示邏輯不需改。

- [ ] **Step 1: 建立 AuthService（完整移植，方法體與 AuthProvider 相同）**

```dart
// lib/layer_business/services/auth/auth_service.dart
import 'dart:async';

import 'package:awesome_flutter_extensions/awesome_flutter_extensions.dart';
import 'package:dio/dio.dart';
import 'package:disco/disco.dart';
import 'package:flutter_solidart/flutter_solidart.dart';
import 'package:hexagon_food_app/layer_business/network/abstract/auth_api_type.dart';
import 'package:hexagon_food_app/layer_business/network/api/auth_api.dart';
import 'package:hexagon_food_app/layer_business/network/error_exception.dart';
import 'package:hexagon_food_app/layer_business/router/paths.dart';
import 'package:hexagon_food_app/layer_business/router/routes.dart';
import 'package:hexagon_food_app/layer_business/services/auth/auth_session_manager.dart';
import 'package:hexagon_food_app/layer_business/utils/locator.dart';
import 'package:hexagon_food_app/layer_data/models/auther/auther_session_info.dart';
import 'package:hexagon_food_app/layer_data/models/auther/auther_status_check.dart';

final accountAuthProvider = Provider(
  (context) => AuthService(api: locator<AuthApi>(), sessionManager: locator<AuthSessionManager>()),
  dispose: (service) => service.dispose(),
);

/// 認證服務 — 取代 [ChangeNotifier] 型 [AuthProvider]。
///
/// 對外方法簽名與舊 [AuthProvider] 完全一致；畫面只需改 import 與
/// 型別參數（`AuthProvider` → `AuthService`），呼叫不變。
/// 狀態以 solidart [Signal] 暴露：[Signal] 本身是 [ValueNotifier]，
/// auto_route 的 `reevaluateListenable` 可直接吃 signal。
class AuthService {
  AuthService({required AuthApiType api, required AuthSessionManager sessionManager})
      : _api = api,
        _sessionManager = sessionManager {
    _sessionSubscription = _sessionManager.sessionStream.listen((info) {
      _sessionInfo.value = info;
    });
  }

  final AuthApiType _api;
  final AuthSessionManager _sessionManager;
  late final StreamSubscription<AutherSessionInfo?> _sessionSubscription;

  /// session 狀態 signal（ValueNotifier，可當 auto_route reevaluateListenable）。
  final _sessionInfo = Signal<AutherSessionInfo?>(null);
  Signal<AutherSessionInfo?> get sessionInfoSignal => _sessionInfo;

  bool get isLogin => _sessionManager.isLoggedIn;
  String get userType => _sessionManager.userType ?? "";
  bool get isCustomer => userType == "customer";
  bool get isSalesRep => userType == "salesrep";
  int? get createdById =>
      isSalesRep ? _sessionManager.inSessionInfo.salesrepInfo?.id : _sessionManager.inSessionInfo.customerInfo?.id;

  FutureOr<Response> meHandler(bool isAuth) async {
    try {
      final meResponse = await _api.getMe();
      if (meResponse.statusCode != 200) {
        ErrorException errorException = ErrorException.fromTypeError("無法取得使用者資料: ${meResponse.statusMessage}");
        throw Exception(errorException.message);
      }
      final si = AutherSessionInfo.fromJson(meResponse.data as Map<String, dynamic>);

      if (si.userType == "salesrep") {
        _sessionManager.inSessionInfo.write(si.copyWith(salesrep: si.salesrep, isAuth: isAuth, userType: si.userType));
      } else if (si.userType == "customer") {
        _sessionManager.inSessionInfo.write(si.copyWith(customer: si.customer, isAuth: isAuth, userType: si.userType));
      } else {
        ErrorException errorException = ErrorException.fromTypeError("無效的使用者類型: ${si.userType}");
        throw Exception(errorException.message);
      }
      return meResponse;
    } on DioException catch (dioError) {
      ErrorException errorException = ErrorException.fromDioError(dioError);
      throw Exception(errorException.message);
    }
  }

  FutureOr<Response> signinHandler(Map<String, dynamic> formData) async {
    try {
      final response = await _api.signin(formData, Options(extra: {"signinType": "salesrep"}));
      if (response.statusCode == 200) {
        await _completeSignIn();
      }
      return response;
    } on DioException catch (dioError) {
      ErrorException errorException = ErrorException.fromDioError(dioError);
      throw Exception(errorException.message);
    } catch (e) {
      throw Exception('登入失敗: $e');
    }
  }

  FutureOr<Response> customerSigninHandler(Map<String, dynamic> formData) async {
    try {
      final response = await _api.customerSignin(formData, Options(extra: {"signinType": "customer"}));
      if (response.statusCode == 200) {
        await _completeSignIn();
      }
      return response;
    } on DioException catch (dioError) {
      ErrorException errorException = ErrorException.fromDioError(dioError);
      throw Exception(errorException.message);
    } catch (e) {
      throw Exception('登入失敗: $e');
    }
  }

  /// 登入成功後統一取得使用者資訊；若失敗則清除已寫入的 cookies，避免狀態不一致。
  ///
  /// 失敗時只清 cookies 而不清 session；同時避免呼叫 [clearSession] 同步觸發
  /// reevaluateGuards，導致與進行中的 Dio / navigation 流程競爭。
  Future<void> _completeSignIn() async {
    try {
      await meHandler(true);
      // 登入後取得 CSRF token，供狀態變更請求使用（session 內可重複使用）
      await _fetchCsrfToken();
      // 不需手動通知；meHandler 寫入 session 時會透過 _sessionSubscription
      // 自動更新 signal，避免與進行中的導航競爭。
    } catch (_) {
      await _sessionManager.clearAuthCookies();
      rethrow;
    }
  }

  Future<void> _fetchCsrfToken() async {
    try {
      final resp = await _api.getCsrfToken();
      if (resp.statusCode == 200 && resp.data is Map<String, dynamic>) {
        final data = resp.data as Map<String, dynamic>;
        _sessionManager.csrfToken = data['csrf_token'] as String?;
      }
    } catch (_) {
      // CSRF 取得失敗不阻擋登入；若後續狀態變更請求因此 403，使用者重新登入即可
    }
  }

  FutureOr<Response> signoutHandler() async {
    try {
      final response = await _api.signout(userType);
      if (response.statusCode == 200) {
        await clearLocalSession();
      }
      return response;
    } on DioException catch (dioError) {
      ErrorException errorException = ErrorException.fromDioError(dioError);
      throw Exception(errorException.message);
    }
  }

  FutureOr<Response> forceSignoutHandler() async {
    late final int userId;
    if (isSalesRep) {
      userId = _sessionManager.inSessionInfo.salesrepInfo?.id ?? 0;
    } else {
      userId = _sessionManager.inSessionInfo.customerInfo?.id ?? 0;
    }

    try {
      final response = await _api.forceSignout(userType, userId);
      if (response.statusCode == 200) {
        await clearLocalSession();
      }
      return response;
    } on DioException catch (dioError) {
      ErrorException errorException = ErrorException.fromDioError(dioError);
      throw Exception(errorException.message);
    }
  }

  /// 清除本機 session/cookies/快取，不會呼叫遠端 signout API。
  Future<void> clearLocalSession() async {
    // 明確登出不還原頁面：清除 401 可能殘留的待還原路由。
    _sessionManager.pendingRestoreRoute = null;
    await _sessionManager.clearSession();
  }

  /// 重新登入後還原 401 前所在頁面（由各登入畫面導回首頁後呼叫）。
  Future<void> restorePreviousPageIfAny() async {
    final restore = _sessionManager.pendingRestoreRoute;
    _sessionManager.pendingRestoreRoute = null;
    if (restore == null || restore.isEmpty) {
      return;
    }
    if (!_isRestorableRoute(restore)) {
      return;
    }
    await locator<AppRouter>().pushPath(
      restore,
      onFailure: (failure) => '還原頁面失敗: $restore ($failure)'.log(),
    );
  }

  bool _isRestorableRoute(String path) {
    if (path.isEmpty || path == '/') {
      return false;
    }
    if (path.startsWith('/${RoutePath.authLayout.toValue()}')) {
      return false;
    }
    if (path.startsWith('/${RoutePath.salesorderItemLayout.toValue()}')) {
      return false;
    }
    if (path == locator<AppRouter>().currentPath) {
      return false;
    }
    return true;
  }

  FutureOr<Response> resetNullPasswordHandler(int userId) async {
    try {
      final response = await _api.resetNullPassword("customer", userId);
      return response;
    } on DioException catch (dioError) {
      ErrorException errorException = ErrorException.fromDioError(dioError);
      throw Exception(errorException.message);
    }
  }

  Future<AutherStatusCheck> validEmailHandler(String email) async {
    try {
      final resp = await _api.validEmail(email.trim());
      return AutherStatusCheck.fromJson(resp.data as Map<String, dynamic>);
    } on DioException catch (dioError) {
      ErrorException errorException = ErrorException.fromDioError(dioError);
      throw Exception(errorException.message);
    }
  }

  FutureOr<AutherStatusCheck> validExternalIdHandler(String externalId) async {
    try {
      final resp = await _api.validEntityId(externalId.trim());
      if (resp.statusCode != 200) {
        if (resp.statusCode == 404) {
          ErrorException errorException = ErrorException.fromTypeError("客戶帳號不存在");
          throw Exception(errorException.message);
        }
        ErrorException errorException = ErrorException.fromTypeError("無法驗證客戶帳號: ${resp.statusMessage}");
        throw Exception(errorException.message);
      }
      return AutherStatusCheck.fromJson(resp.data as Map<String, dynamic>);
    } on DioException catch (dioError) {
      ErrorException errorException = ErrorException.fromDioError(dioError);
      throw Exception(errorException.message);
    }
  }

  FutureOr<Response> newsetSalesrepPasswordHandler(Map<String, dynamic> formData) async {
    try {
      final resp = await _api.newsetSalesrepPassword(formData, Options(extra: {"passwordType": "salesrep"}));
      return resp;
    } on DioException catch (dioError) {
      ErrorException errorException = ErrorException.fromDioError(dioError);
      throw Exception(errorException.message);
    }
  }

  FutureOr<Response> newsetCustomerPasswordHandler(Map<String, dynamic> formData) async {
    try {
      final resp = await _api.newsetCustomerPassword(formData, Options(extra: {"passwordType": "customer"}));
      return resp;
    } on DioException catch (dioError) {
      ErrorException errorException = ErrorException.fromDioError(dioError);
      throw Exception(errorException.message);
    }
  }

  void dispose() {
    _sessionSubscription.cancel();
    _sessionInfo.dispose();
  }
}
```

> 移植後務必對照舊 `provider.dart` 全文（`lib/layer_business/services/auth/provider.dart`）逐一確認：方法清單、throw 行為、註解意圖（同步/競爭條件說明）都要保留。舊檔在 Task 7 最後才刪除。

- [ ] **Step 2: 更新 app.dart — reevaluateListenable 改用 signal**

`Signal` 是 `ValueNotifier`，直接當 `reevaluateListenable`：

```dart
// app.dart — 原：reevaluateListenable: accountAuthProvider.of(context),
// 改為：
reevaluateListenable: accountAuthProvider.of(context).sessionInfoSignal,
```

- [ ] **Step 3: 更新 session_expiry_overlay.dart — 改用 AuthService**

讀取 `session_expiry_overlay.dart` 原始碼，將 `AuthProvider? _authProvider` 換成 `AuthService? _authService`（型別來自 `accountAuthProvider.of(context)`）；原 `addListener(_onAuthChanged)` 改為 `_authService!.sessionInfoSignal.addListener(_onAuthChanged)`，`dispose` 時 `removeListener`。其餘邏輯不變。

- [ ] **Step 4: 更新登入畫面 — 只改型別**

`salesrep_signin_screen.dart`、`customer_signin_screen.dart`、`customer_qrcode_signin_screen.dart`：
- import 改為 `package:hexagon_food_app/layer_business/services/auth/auth_service.dart`
- 型別標註 `AuthProvider ua` → `AuthService ua`（含 helper 方法參數型別）
- `accountAuthProvider.of(context)` 用法不變；`ua.signinHandler(...)`、`ua.restorePreviousPageIfAny()`、`ua.newsetSalesrepPasswordHandler(...)` 等方法呼叫**一字不改**

- [ ] **Step 5: 更新路由 AuthGuard**

`routes.dart` 的 `AuthGuard` 檢查使用 `locator<AuthSessionManager>().isLoggedIn` — 保持不變（不依賴 AuthProvider）。若原始碼有其他 `AuthProvider` 引用（例如 `profile_screen.dart` 等），逐一改為 `AuthService`（import + 型別）。

- [ ] **Step 6: 全專案掃描剩餘 AuthProvider 引用**

```bash
cd sales-order-app
grep -rn "AuthProvider" lib/ --include="*.dart" | grep -v "provider.dart:"
```

逐一處理：`accountAuthProvider.of(context)` 不變；直接參考 `AuthProvider` 型別處改 `AuthService`。

- [ ] **Step 7: 執行 flutter analyze**

```bash
cd sales-order-app
fvm flutter analyze
```

預期：0 errors，0 warnings。

- [ ] **Step 8: 執行現有測試確認無回歸**

```bash
cd sales-order-app
fvm flutter test
```

預期：現有 auth 測試（`test/auth/`）全數通過。

- [ ] **Step 9: Commit**

```bash
git add lib/layer_business/services/auth/auth_service.dart \
        lib/layer_presentation/app.dart \
        lib/layer_presentation/general/widgets/session_expiry_overlay.dart \
        lib/layer_presentation/stories/auth/ \
        lib/layer_presentation/stories/admin/ \
        lib/layer_business/router/
git rm lib/layer_business/services/auth/provider.dart
git commit -m "refactor: AuthProvider (ChangeNotifier) → AuthService (solidart Signal)"
```

---

### Task 8: Provider 檔案改名與服務標準化

**Files:**
- Rename: `lib/layer_business/services/customer/provider.dart` → `customer_service.dart`
- Rename: `lib/layer_business/services/customer/modal_provider.dart` → `modal_service.dart`
- Rename: `lib/layer_business/services/salesorder/provider.dart` → `salesorder_service.dart`
- Rename: `lib/layer_business/services/article/provider.dart` → `article_service.dart`
- Rename: `lib/layer_business/services/metadict/provider.dart` → `metadict_service.dart`
- Rename: `lib/layer_business/services/profile/provider.dart` → `profile_service.dart`
- Rename: `lib/layer_business/services/scaffold/provider.dart` → `scaffold_service.dart`

**Interfaces:**
- Consumes: 無（僅改名，不對外介面變更）
- Produces: 統一的 `{domain}_service.dart` 命名

- [ ] **Step 1: 逐一改名**

```bash
cd sales-order-app/lib/layer_business/services

mv customer/provider.dart customer/customer_service.dart
mv customer/modal_provider.dart customer/modal_service.dart
mv salesorder/provider.dart salesorder/salesorder_service.dart
mv article/provider.dart article/article_service.dart
mv metadict/provider.dart metadict/metadict_service.dart
mv profile/provider.dart profile/profile_service.dart
mv scaffold/provider.dart scaffold/scaffold_service.dart
```

- [ ] **Step 2: 更新所有 import 路徑**

用 grep 找出所有 `services/customer/provider.dart`、`services/salesorder/provider.dart` 等 import，逐一更新為新路徑。

```bash
cd sales-order-app
# 列出所有需要更新的 import
grep -rn "services/customer/provider.dart" lib/ test/
grep -rn "services/customer/modal_provider.dart" lib/ test/
grep -rn "services/salesorder/provider.dart" lib/ test/
grep -rn "services/article/provider.dart" lib/ test/
grep -rn "services/metadict/provider.dart" lib/ test/
grep -rn "services/profile/provider.dart" lib/ test/
grep -rn "services/scaffold/provider.dart" lib/ test/
```

針對每個出現處，將 `services/<domain>/provider.dart` 改為 `services/<domain>/<domain>_service.dart`，`services/customer/modal_provider.dart` 改為 `services/customer/modal_service.dart`。

- [ ] **Step 3: 執行 flutter analyze 確認無誤**

```bash
cd sales-order-app
fvm flutter analyze
```

預期：0 errors。

- [ ] **Step 4: Commit**

```bash
git add sales-order-app/lib/layer_business/services/customer/ \
        sales-order-app/lib/layer_business/services/salesorder/ \
        sales-order-app/lib/layer_business/services/article/ \
        sales-order-app/lib/layer_business/services/metadict/ \
        sales-order-app/lib/layer_business/services/profile/ \
        sales-order-app/lib/layer_business/services/scaffold/
# 也加入所有引用這些檔案的其他檔案
git add sales-order-app/lib/
git add sales-order-app/test/
git commit -m "refactor: rename provider.dart → {domain}_service.dart for consistency"
```

---

## 階段 3：測試補強

### Task 9: AuthService 測試（P0）

**Files:**
- Create: `test/unit/services/auth_service_test.dart`

**Interfaces:**
- Consumes: `FakeAuthApi`（Task 6）、`AuthService`（Task 7）、`AuthSessionManager`
- Produces: 7 個測試案例覆蓋登入成功/失敗/登出

- [ ] **Step 1: 撰寫 AuthService 測試**

```dart
// test/unit/services/auth_service_test.dart
import 'dart:io';
import 'package:flutter_test/flutter_test.dart';
import 'package:hexagon_food_app/layer_business/services/auth/auth_service.dart';
import 'package:hexagon_food_app/layer_business/services/auth/auth_session_manager.dart';
import 'package:hexagon_food_app/layer_data/repositories/session_info_storage.dart';
import 'package:cookie_jar/cookie_jar.dart';
import 'package:cached_memory_image/cached_image_base64_manager.dart';
import 'package:dio/dio.dart';
import 'package:sqflite_common_ffi/sqflite_ffi.dart';
import '../../helpers/fake_api.dart';
import '../../auth/fake_path_provider.dart';

class StubCacheStorage extends CacheStorage {
  @override Future<void> clearCache() async {}
  @override CacheOptions get defaultOptions => CacheOptions(store: MemCacheStore());
  @override CacheOptions get smallCacheOptions => CacheOptions(store: MemCacheStore());
  @override CacheOptions get mediumCacheOptions => CacheOptions(store: MemCacheStore());
  @override CacheOptions get largeCacheOptions => CacheOptions(store: MemCacheStore());
  @override CacheOptions get noCacheOptions => CacheOptions(store: MemCacheStore());
  @override CacheOptions get refreshOptions => CacheOptions(store: MemCacheStore());
  @override CacheOptions get forceCacheOptions => CacheOptions(store: MemCacheStore());
  @override CacheOptions get refreshForceCacheOptions => CacheOptions(store: MemCacheStore());
  @override Options get smCache => Options();
  @override Options get mdCache => Options();
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

  late FakeAuthApi fakeApi;
  late AuthSessionManager sessionManager;
  late AuthService authService;
  late Directory tmpDir;

  setUp(() async {
    fakeApi = FakeAuthApi();
    tmpDir = Directory.systemTemp.createTempSync('auth_service_test_');
    final sessionStorage = await InSessionInfoStorage.create(tmpDir.path);
    sessionManager = AuthSessionManager(
      sessionInfo: sessionStorage,
      cookieJar: PersistCookieJar(ignoreExpires: true),
      cacheStorage: StubCacheStorage(),
      imageCache: CachedImageBase64Manager.instance(),
    );
    authService = AuthService(api: fakeApi, sessionManager: sessionManager);
  });

  tearDown(() {
    authService.dispose();
    tmpDir.deleteSync(recursive: true);
  });

  group('signinHandler', () {
    test('successful login updates session signal', () async {
      // 依序：getCsrfToken → signin → getMe
      fakeApi.queueResponses([
        {'csrf_token': 'test-csrf'},
        {'status': 'success'},
        {
          'id': 1,
          'name': '測試業務',
          'email': 'test@example.com',
          'utype': 'salesrep',
          'is_auth': true,
          'session_expires_at': '2026-12-31T23:59:59Z',
        },
      ]);

      await authService.signinHandler({'email': 'test@example.com', 'password': 'password'});
      expect(authService.isLogin, isTrue);
      expect(authService.sessionInfoSignal.value?.isAuth, isTrue);
    });

    test('wrong credentials throws with message', () async {
      fakeApi.setError('Invalid credentials');

      expect(
        () => authService.signinHandler({'email': 'bad@example.com', 'password': 'wrong'}),
        throwsException,
      );
    });

    test('network error throws', () async {
      fakeApi.setError('Connection refused');

      expect(
        () => authService.signinHandler({'email': 'test@example.com', 'password': 'password'}),
        throwsException,
      );
    });
  });

  group('signoutHandler', () {
    test('signout clears local session', () async {
      fakeApi.queueResponses([
        {'csrf_token': 'test-csrf'},
        {'status': 'success'},
        {
          'id': 1,
          'name': '測試業務',
          'email': 'test@example.com',
          'utype': 'salesrep',
          'is_auth': true,
          'session_expires_at': '2026-12-31T23:59:59Z',
        },
      ]);
      await authService.signinHandler({'email': 'test@example.com', 'password': 'password'});

      await authService.signoutHandler();
      expect(authService.isLogin, isFalse);
      expect(authService.sessionInfoSignal.value, isNull);
    });
  });

  group('initial state', () {
    test('starts not logged in', () {
      expect(authService.isLogin, isFalse);
      expect(authService.sessionInfoSignal.value, isNull);
    });
  });
}
```

- [ ] **Step 2: 執行測試，確認全部通過**

```bash
cd sales-order-app
fvm flutter test test/unit/services/auth_service_test.dart
```

預期：7 tests passed。

- [ ] **Step 3: Commit**

```bash
git add sales-order-app/test/unit/services/auth_service_test.dart
git commit -m "test: add AuthService unit tests (signin/signout/errors)"
```

---

### Task 10: AuthInterceptor 401 邏輯測試（P0）

**Files:**
- Create: `test/unit/services/auth_interceptor_401_test.dart`

**Interfaces:**
- Consumes: `AuthInterceptor`（現有）、`AuthSessionManager`
- Produces: 401 錯誤時清除 session 的測試

- [ ] **Step 1: 撰寫 401 測試**

```dart
// test/unit/services/auth_interceptor_401_test.dart
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
  @override CacheOptions get mediumCacheOptions => CacheOptions(store: MemCacheStore());
  @override CacheOptions get largeCacheOptions => CacheOptions(store: MemCacheStore());
  @override CacheOptions get noCacheOptions => CacheOptions(store: MemCacheStore());
  @override CacheOptions get refreshOptions => CacheOptions(store: MemCacheStore());
  @override CacheOptions get forceCacheOptions => CacheOptions(store: MemCacheStore());
  @override CacheOptions get refreshForceCacheOptions => CacheOptions(store: MemCacheStore());
  @override Options get smCache => Options();
  @override Options get mdCache => Options();
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
    tmpDir = Directory.systemTemp.createTempSync('401_test_');
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

  group('onError 401', () {
    test('clears csrfToken on 401', () async {
      sessionManager.csrfToken = 'should-be-cleared';

      final dioError = DioException(
        requestOptions: RequestOptions(path: '/api/v1/test'),
        response: Response(
          requestOptions: RequestOptions(path: '/api/v1/test'),
          statusCode: 401,
        ),
      );

      await interceptor.onError(dioError, ErrorInterceptorHandler());
      expect(sessionManager.csrfToken, isNull);
    });

    test('does NOT clear csrfToken on non-401 errors', () async {
      sessionManager.csrfToken = 'should-remain';

      final dioError = DioException(
        requestOptions: RequestOptions(path: '/api/v1/test'),
        response: Response(
          requestOptions: RequestOptions(path: '/api/v1/test'),
          statusCode: 500,
        ),
      );

      await interceptor.onError(dioError, ErrorInterceptorHandler());
      expect(sessionManager.csrfToken, equals('should-remain'));
    });
  });
}
```

- [ ] **Step 2: 執行測試**

```bash
cd sales-order-app
fvm flutter test test/unit/services/auth_interceptor_401_test.dart
```

預期：2 tests passed。

- [ ] **Step 3: Commit**

```bash
git add sales-order-app/test/unit/services/auth_interceptor_401_test.dart
git commit -m "test: add AuthInterceptor 401 session clearing tests"
```

---

### Task 11: ErrorException 錯誤訊息測試（P0）

**Files:**
- Create: `test/unit/utils/error_exception_test.dart`

**Interfaces:**
- Consumes: `ErrorException`（現有）
- Produces: 各種 HTTP status code 對應的繁體中文錯誤訊息測試

- [ ] **Step 1: 撰寫 ErrorException 測試**

```dart
// test/unit/utils/error_exception_test.dart
import 'package:flutter_test/flutter_test.dart';
import 'package:dio/dio.dart';
import 'package:hexagon_food_app/layer_business/network/error_exception.dart';

void main() {
  group('ErrorException', () {
    test('400 returns "錯誤的請求"', () {
      final dioError = DioException(
        requestOptions: RequestOptions(path: ''),
        response: Response(requestOptions: RequestOptions(path: ''), statusCode: 400),
      );
      final error = ErrorException.fromDioError(dioError);
      expect(error.message, equals('錯誤的請求'));
    });

    test('401 returns "未授權，請重新登入"', () {
      final dioError = DioException(
        requestOptions: RequestOptions(path: ''),
        response: Response(requestOptions: RequestOptions(path: ''), statusCode: 401),
      );
      final error = ErrorException.fromDioError(dioError);
      expect(error.message, equals('未授權，請重新登入'));
    });

    test('403 returns "禁止訪問"', () {
      final dioError = DioException(
        requestOptions: RequestOptions(path: ''),
        response: Response(requestOptions: RequestOptions(path: ''), statusCode: 403),
      );
      final error = ErrorException.fromDioError(dioError);
      expect(error.message, equals('禁止訪問'));
    });

    test('422 returns "驗證例外"', () {
      final dioError = DioException(
        requestOptions: RequestOptions(path: ''),
        response: Response(requestOptions: RequestOptions(path: ''), statusCode: 422),
      );
      final error = ErrorException.fromDioError(dioError);
      expect(error.message, equals('驗證例外'));
    });

    test('429 returns "嘗試次數過多，請稍後再試"', () {
      final dioError = DioException(
        requestOptions: RequestOptions(path: ''),
        response: Response(requestOptions: RequestOptions(path: ''), statusCode: 429),
      );
      final error = ErrorException.fromDioError(dioError);
      expect(error.message, equals('嘗試次數過多，請稍後再試'));
    });

    test('500 returns "伺服器內部錯誤"', () {
      final dioError = DioException(
        requestOptions: RequestOptions(path: ''),
        response: Response(requestOptions: RequestOptions(path: ''), statusCode: 500),
      );
      final error = ErrorException.fromDioError(dioError);
      expect(error.message, equals('伺服器內部錯誤'));
    });

    test('502 returns "錯誤的閘道"', () {
      final dioError = DioException(
        requestOptions: RequestOptions(path: ''),
        response: Response(requestOptions: RequestOptions(path: ''), statusCode: 502),
      );
      final error = ErrorException.fromDioError(dioError);
      expect(error.message, equals('錯誤的閘道'));
    });

    test('unknown status returns fallback message', () {
      final dioError = DioException(
        requestOptions: RequestOptions(path: ''),
        response: Response(requestOptions: RequestOptions(path: ''), statusCode: 418),
      );
      final error = ErrorException.fromDioError(dioError);
      expect(error.message, equals('糟糕，出了點問題，請稍後再試'));
    });

    test('connectionTimeout returns timeout message', () {
      final dioError = DioException(
        requestOptions: RequestOptions(path: ''),
        type: DioExceptionType.connectionTimeout,
      );
      final error = ErrorException.fromDioError(dioError);
      expect(error.message, equals('API 伺服器的連線逾時'));
    });

    test('cancel returns cancel message', () {
      final dioError = DioException(
        requestOptions: RequestOptions(path: ''),
        type: DioExceptionType.cancel,
      );
      final error = ErrorException.fromDioError(dioError);
      expect(error.message, equals('API 伺服器的請求已取消'));
    });
  });
}
```

- [ ] **Step 2: 執行測試**

```bash
cd sales-order-app
fvm flutter test test/unit/utils/error_exception_test.dart
```

預期：10 tests passed。

- [ ] **Step 3: Commit**

```bash
git add sales-order-app/test/unit/utils/error_exception_test.dart
git commit -m "test: add ErrorException Chinese error message tests"
```

---

### Task 12: Freezed model JSON 序列化測試（P1）

**Files:**
- Create: `test/unit/models/auther_session_info_test.dart`

**Interfaces:**
- Consumes: `AutherSessionInfo` freezed model（現有）
- Produces: round-trip 序列化測試

- [ ] **Step 1: 撰寫序列化測試**

```dart
// test/unit/models/auther_session_info_test.dart
import 'package:flutter_test/flutter_test.dart';
import 'package:hexagon_food_app/layer_data/models/auther/auther_session_info.dart';

void main() {
  group('AutherSessionInfo JSON round-trip', () {
    test('fromJson → toJson → fromJson produces identical object', () {
      final json = {
        'id': 1,
        'name': '測試業務',
        'email': 'test@hexagon.com',
        'utype': 'salesrep',
        'is_auth': true,
        'session_expires_at': '2026-12-31T23:59:59Z',
      };

      final session = AutherSessionInfo.fromJson(json);
      final roundTripped = AutherSessionInfo.fromJson(session.toJson());

      expect(roundTripped.id, equals(session.id));
      expect(roundTripped.name, equals(session.name));
      expect(roundTripped.email, equals(session.email));
      expect(roundTripped.utype, equals(session.utype));
      expect(roundTripped.isAuth, equals(session.isAuth));
      expect(roundTripped.sessionExpiresAt, equals(session.sessionExpiresAt));
    });

    test('fromJson handles null fields gracefully', () {
      final json = <String, dynamic>{
        'id': null,
        'name': null,
        'email': null,
        'utype': null,
        'is_auth': false,
        'session_expires_at': null,
      };

      final session = AutherSessionInfo.fromJson(json);
      expect(session.id, isNull);
      expect(session.name, isNull);
      expect(session.isAuth, isFalse);
    });

    test('toJson produces valid map', () {
      final json = {
        'id': 1,
        'name': 'Test',
        'email': 'test@test.com',
        'utype': 'salesrep',
        'is_auth': true,
        'session_expires_at': '2026-01-01T00:00:00Z',
      };
      final session = AutherSessionInfo.fromJson(json);
      final output = session.toJson();

      expect(output, isA<Map<String, dynamic>>());
      expect(output['id'], equals(1));
      expect(output['name'], equals('Test'));
    });
  });
}
```

- [ ] **Step 2: 執行測試**

```bash
cd sales-order-app
fvm flutter test test/unit/models/auther_session_info_test.dart
```

預期：3 tests passed。

- [ ] **Step 3: Commit**

```bash
git add sales-order-app/test/unit/models/auther_session_info_test.dart
git commit -m "test: add AutherSessionInfo JSON round-trip tests"
```

---

## 階段 4：商店上架

### Task 13: Fastlane Android 擴充

**Files:**
- Modify: `android/fastlane/Fastfile`
- Create: `android/fastlane/metadata/android/zh-Hant/short_description.txt`
- Create: `android/fastlane/metadata/android/zh-Hant/full_description.txt`
- Create: `android/fastlane/metadata/android/zh-Hant/title.txt`
- Create: `android/fastlane/metadata/android/zh-Hant/changelogs/default.txt`
- Create: `android/fastlane/metadata/android/en-US/short_description.txt`
- Create: `android/fastlane/metadata/android/en-US/full_description.txt`
- Create: `android/fastlane/metadata/android/en-US/title.txt`

**Interfaces:**
- Consumes: 現有 AAB 建置流程（Taskfile）
- Produces: `beta`、`production`、`metadata` lanes

- [ ] **Step 1: 擴充 Fastfile**

```ruby
# android/fastlane/Fastfile
default_platform(:android)

platform :android do
  desc "Runs all the tests"
  lane :test do
    gradle(task: "test")
  end

  desc "Submit a new Beta Build to Play Store"
  lane :beta do
    upload_to_play_store(
      track: 'beta',
      aab: '../build/app/outputs/bundle/prodRelease/app-prod-release.aab',
      release_status: 'draft',
    )
  end

  desc "Promote to Production"
  lane :production do
    upload_to_play_store(
      track: 'production',
      aab: '../build/app/outputs/bundle/prodRelease/app-prod-release.aab',
      release_status: 'completed',
    )
  end

  desc "Upload metadata and screenshots only"
  lane :metadata do
    upload_to_play_store(
      track: 'production',
      metadata_path: './fastlane/metadata/android',
      skip_upload_apk: true,
      skip_upload_aab: true,
      skip_upload_images: false,
      skip_upload_screenshots: false,
    )
  end
end
```

- [ ] **Step 2: 撰寫繁體中文商店描述**

`android/fastlane/metadata/android/zh-Hant/title.txt`：
```
特耀訂出貨系統
```

`android/fastlane/metadata/android/zh-Hant/short_description.txt`：
```
特耀餐飲訂出貨管理系統 — 讓訂出貨更簡單、更快速、更準確。業務員與客戶隨時隨地建立及追蹤銷售訂單。
```

`android/fastlane/metadata/android/zh-Hant/full_description.txt`：
```
特耀訂出貨系統是專為餐飲批發業設計的行動訂貨管理工具，
讓業務員與客戶隨時隨地透過手機完成訂單建立、查詢與管理。

主要功能：
• 業務/客戶雙身分登入 — 支援帳號密碼及 QR Code 快速登入
• 銷售訂單管理 — 建立、編輯、查詢訂單，即時同步至 NetSuite 後端
• 客戶管理 — 新增、搜尋客戶，快速查找歷史訂單
• 商品瀏覽 — 依分類瀏覽商品，快速加入訂單明細
• 公司快訊 — 接收公司最新公告
• 訂單歷史查詢 — 依日期、客戶、狀態篩選歷史訂單
• Cookie Session 認證 — 安全登入，自動續期

適用對象：
• 餐飲批發業務員外出拜訪時快速下單
• 餐廳客戶自主訂貨，減少電話來回確認
```

`android/fastlane/metadata/android/zh-Hant/changelogs/default.txt`：
```
- 新增 session 到期提醒與續期功能
- 優化部門篩選機制
- 修正多項穩定性問題
```

- [ ] **Step 3: 撰寫英文商店描述（精簡版）**

`android/fastlane/metadata/android/en-US/title.txt`：
```
Hexagon Sales Order
```

`android/fastlane/metadata/android/en-US/short_description.txt`：
```
Hexagon Food Service Order Management — simple, fast, and accurate order processing for sales reps and customers.
```

`android/fastlane/metadata/android/en-US/full_description.txt`：
```
Hexagon Sales Order is a mobile order management tool designed for the food service wholesale industry.

Key Features:
• Dual login (Sales Rep / Customer) with password and QR Code
• Sales order creation, editing, and real-time sync with NetSuite
• Customer management with search and order history
• Product browsing by category with quick add to order
• Company announcements
• Order history filtering by date, customer, and status
• Secure cookie-based session authentication with auto-renewal
```

- [ ] **Step 4: Commit**

```bash
git add sales-order-app/android/fastlane/
git commit -m "feat: expand Android Fastlane with production/metadata lanes and store descriptions"
```

---

### Task 14: Fastlane iOS 擴充

**Files:**
- Modify: `ios/fastlane/Fastfile`

**Interfaces:**
- Consumes: 現有 IPA 建置流程（Taskfile）
- Produces: `beta`、`production` lanes

- [ ] **Step 1: 擴充 iOS Fastfile**

```ruby
# ios/fastlane/Fastfile
default_platform(:ios)

platform :ios do
  desc "Push a new beta build to TestFlight"
  lane :beta do
    # 顯式指定 IPA 路徑（相對 fastlane 執行目錄 ios/）。
    # upload_to_testflight 預設 Dir["*.ipa"] 非遞迴，在 ios/ cwd 找不到
    # ../build/ios/ipa/ 的產物（與 Android lane 顯式傳 aab: 同模式）。
    ipa_path = Dir["../build/ios/ipa/*.ipa"].last
    UI.user_error!("找不到 IPA：../build/ios/ipa/*.ipa") if ipa_path.nil?
    upload_to_testflight(
      ipa: ipa_path,
      skip_waiting_for_build_processing: false,
    )
  end

  desc "Submit to App Store review"
  lane :production do
    upload_to_app_store(
      force: true,
      skip_metadata: false,
      skip_screenshots: false,
    )
  end
end
```

- [ ] **Step 2: Commit**

```bash
git add sales-order-app/ios/fastlane/Fastfile
git commit -m "feat: expand iOS Fastlane with production lane"
```

---

### Task 15: 隱私政策頁面

**Files:**
- Create: `docs/privacy/privacy-policy-zh.md`

**Interfaces:**
- Consumes: 無
- Produces: 公開隱私政策網頁（供 GitHub Pages 部署）

- [ ] **Step 1: 撰寫繁體中文隱私政策**

```markdown
# 隱私權政策

**最後更新日期：2026-08-04**

## 簡介

特耀訂出貨系統（以下簡稱「本應用程式」）重視您的隱私權。本隱私權政策說明我們如何收集、使用和保護您的資訊。

## 收集的資訊

本應用程式可能收集以下類型的資訊：

1. **帳號資訊**：登入時使用的電子郵件、公司代號等識別資訊，用於驗證您的身分及提供訂單服務。

2. **裝置資訊**：為了改善應用程式穩定性，我們會收集匿名的裝置資訊（如作業系統版本、應用程式版本）及崩潰報告（透過 Firebase Crashlytics）。

3. **使用資料**：我們使用 Firebase Analytics 收集匿名的使用統計資料，以了解功能使用情況並改善使用者體驗。此資料不會關聯到您的個人身分。

## 資訊用途

- 提供及維護本應用程式的核心功能（登入驗證、訂單處理）
- 偵測、預防及解決技術問題
- 改善應用程式功能與使用者體驗

## 第三方服務

本應用程式使用以下第三方服務：

- **Firebase Crashlytics**（Google）：收集崩潰報告
- **Firebase Analytics**（Google）：收集匿名使用統計

這些服務的隱私權政策請參閱 Google 的隱私權政策。

## 資料保留

您的帳號資訊僅在您使用本應用程式期間保留。崩潰報告與使用統計資料由 Firebase 依照其預設保留政策管理。

## 資料安全

我們使用業界標準的安全措施（HTTPS 加密傳輸、Cookie Session 認證）來保護您的資訊。

## 您的權利

您可以隨時要求刪除您的帳號資料。請聯絡我們（詳見下方聯絡方式）。

## 聯絡我們

如有任何隱私權相關問題，請聯絡：
- Email: support@hexagon.com.tw
```

- [ ] **Step 2: Commit**

```bash
git add docs/privacy/privacy-policy-zh.md
git commit -m "docs: add privacy policy for app store compliance"
```

---

### Task 16: CI/CD release workflows（擴充部署自動化）

**Files:**
- Modify: `.github/workflows/build-android.yml`（擴充 Fastlane 上傳步驟）
- Create: `.github/workflows/release-ios.yml`（從 build-ios 擴充 TestFlight 上傳）

**Interfaces:**
- Consumes: Tasks 2, 3（build workflows）、Tasks 13, 14（Fastlane）
- Produces: 一鍵部署至 Play Store / TestFlight

- [ ] **Step 1: 擴充 Android workflow — 加入 Play Store 上傳**

在 `build-android.yml` 的 build job 結尾加入（需 `secrets.PLAY_STORE_JSON_KEY`）：

```yaml
      - name: Setup Ruby and Fastlane
        uses: ruby/setup-ruby@v1
        with:
          ruby-version: '3.3'
          bundler-cache: true
          working-directory: sales-order-app/android

      - name: Upload to Play Store (beta)
        if: inputs.flavor == 'prod'
        env:
          PLAY_STORE_JSON_KEY: ${{ secrets.PLAY_STORE_JSON_KEY }}
        working-directory: sales-order-app/android
        run: bundle exec fastlane beta
```

- [ ] **Step 2: 建立 iOS release workflow（含 TestFlight 上傳）**

從 Task 3 的 `build-ios.yml` 為基礎，加入簽署與 Fastlane：

```yaml
# .github/workflows/release-ios.yml
name: Release iOS

on:
  workflow_dispatch:

jobs:
  release:
    runs-on: macos-latest
    steps:
      - uses: actions/checkout@v4
        with:
          submodules: true

      - name: Read Flutter version from .fvmrc
        id: fvm
        run: |
          FVM_VERSION=$(cat sales-order-app/.fvmrc | jq -r '.flutter')
          echo "version=$FVM_VERSION" >> $GITHUB_OUTPUT

      - uses: subosito/flutter-action@v2
        with:
          flutter-version: ${{ steps.fvm.outputs.version }}
          channel: stable

      - name: Install dependencies
        working-directory: sales-order-app
        run: flutter pub get

      - name: Run code generation
        working-directory: sales-order-app
        run: dart run build_runner build --delete-conflicting-outputs

      - name: Setup Ruby and Fastlane
        uses: ruby/setup-ruby@v1
        with:
          ruby-version: '3.3'
          bundler-cache: true
          working-directory: sales-order-app/ios

      - name: Setup iOS signing (requires secrets)
        # 需要 Apple Developer 憑證：distribution certificate + provisioning
        # profile。兩種方式擇一（依專案現況）：
        #   A) fastlane match（需 MATCH_* secrets 與 match repo）
        #   B) 手動 import：CERTIFICATE_BASE64 + CERT_PASSWORD + PROFILE_BASE64
        #      放入 .p12 / .mobileprovision 並寫入 keychain。
        # 未設定簽署時 build ipa 會失敗 — 這是部署前的必要前置。
        env:
          CERTIFICATE_BASE64: ${{ secrets.IOS_CERTIFICATE_BASE64 }}
          CERT_PASSWORD: ${{ secrets.IOS_CERT_PASSWORD }}
          PROFILE_BASE64: ${{ secrets.IOS_PROFILE_BASE64 }}
          KEYCHAIN_PASSWORD: ${{ secrets.IOS_KEYCHAIN_PASSWORD }}
        run: |
          if [ -z "$CERTIFICATE_BASE64" ] || [ -z "$PROFILE_BASE64" ]; then
            echo "::warning::iOS signing secrets not configured — TestFlight upload will fail."
            exit 0
          fi
          CERT_PATH=$RUNNER_TEMP/build_certificate.p12
          PP_PATH=$RUNNER_TEMP/build_pp.mobileprovision
          KEYCHAIN_PATH=$RUNNER_TEMP/app-signing.keychain-db
          echo "$CERTIFICATE_BASE64" | base64 --decode > $CERT_PATH
          echo "$PROFILE_BASE64" | base64 --decode > $PP_PATH
          security create-keychain -p "$KEYCHAIN_PASSWORD" $KEYCHAIN_PATH
          security set-keychain-settings -lut 21600 $KEYCHAIN_PATH
          security unlock-keychain -p "$KEYCHAIN_PASSWORD" $KEYCHAIN_PATH
          security import $CERT_PATH -P "$CERT_PASSWORD" -A -t cert -f pkcs12 -k $KEYCHAIN_PATH
          security list-keychain -d user -s $KEYCHAIN_PATH
          mkdir -p ~/Library/MobileDevice/Provisioning\ Profiles
          cp $PP_PATH ~/Library/MobileDevice/Provisioning\ Profiles/

      - name: Build and upload to TestFlight
        working-directory: sales-order-app
        env:
          APP_STORE_CONNECT_API_KEY_KEY_ID: ${{ secrets.APP_STORE_CONNECT_API_KEY_ID }}
          APP_STORE_CONNECT_API_KEY_ISSUER_ID: ${{ secrets.APP_STORE_CONNECT_API_KEY_ISSUER_ID }}
          APP_STORE_CONNECT_API_KEY_KEY: ${{ secrets.APP_STORE_CONNECT_API_KEY_KEY }}
        run: |
          # 先建置簽署的 prod IPA（Task 3 的 no-codesign 流程不適用上傳）
          flutter build ipa --flavor prod --target lib/main_prod.dart
          # fastlane beta 上傳 build/ios/ipa/*.ipa 至 TestFlight
          cd ios && bundle exec fastlane beta
```

- [ ] **Step 3: Commit**

```bash
git add .github/workflows/
git commit -m "ci: add Play Store and TestFlight deployment workflows"
```

---

### Task 17: 上線前最終檢查清單

**Files:**
- Create: `docs/store-launch-checklist.md`

**Interfaces:**
- Consumes: 所有前述 tasks
- Produces: 檢查清單文件，供手動逐項確認

- [ ] **Step 1: 建立檢查清單文件**

```markdown
# 商店上線前檢查清單

## Android (Google Play)

- [ ] keystore 備份確認（`android/keystore/hexagon-salesorder-keystore.jks`）
- [ ] `key.properties` 內容正確
- [ ] Prod flavor 指向正式 API endpoint
- [ ] Firebase Crashlytics 在 prod 正常收到 crash report
- [ ] Firebase Analytics 確認不記錄 PII
- [ ] 深層連結（customer QR code signin）release build 測試通過
- [ ] Play Store 隱私政策 URL 設定完成
- [ ] Data Safety 表單填寫完成
- [ ] 內容分級問卷完成
- [ ] 商店圖文（截圖、feature graphic、描述）上傳完成
- [ ] Closed track 內部測試通過（10+ 測試者）
- [ ] Open track beta 測試通過

## iOS (App Store)

- [ ] `PrivacyInfo.xcprivacy` 內容正確
- [ ] App Store Connect 隱私標籤填寫完成
- [ ] App Store 描述（繁中/英文）設定完成
- [ ] App Store 截圖上傳完成（含 6.9" artboard 成品）
- [ ] TestFlight 內部測試通過
- [ ] TestFlight 外部測試通過
- [ ] App Store 審查提交

## 通用

- [ ] 隱私政策網頁可公開存取
- [ ] 聯絡信箱可正常收信
- [ ] 後端 API prod endpoint 穩定運作
- [ ] 版本號正確（pubspec.yaml 及 git tag）
```

- [ ] **Step 2: Commit**

```bash
git add docs/store-launch-checklist.md
git commit -m "docs: add store launch pre-flight checklist"
```

---

## 執行順序

```
Phase 1: Tasks 1-6 (可部分並行)
Phase 2: Tasks 7-8 (依序，Task 8 依賴 Task 7)
Phase 3: Tasks 9-12 (可部分並行)
Phase 4: Tasks 13-17 (可部分並行)
```

---

## 後續迭代（本計畫範圍外，列為 follow-up）

| 項目 | 說明 |
|------|------|
| **P2 共用 Widget 測試** | search bar、form 元件 snapshot / interaction 測試 |
| **P2 路由 guard 測試** | AuthGuard 登入檢查 redirect 邏輯 |
| **P3 畫面 snapshot 測試** | 其餘畫面（Maestro E2E 已有部分覆蓋） |
| **CustomerService 完整測試** | 需 refactor 為不直接依賴 BuildContext 的設計 |
| **SalesOrderService 完整測試** | 同上 |
| **硬編碼字串全面 i18n 遷移** | 階段 1 僅建立 ARB 框架，現有字串逐步搬遷 |
| **英文版翻譯** | 商店上架需英文描述但 App 內 UI 英文版另案處理 |
| **Fat-free models** | `salesorder.gform.dart` (165KB)、`customer.gform.dart` (198KB) 等大檔拆分 |

---
 
 ## 執行順序
