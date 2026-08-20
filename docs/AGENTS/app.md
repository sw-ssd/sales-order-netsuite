# Hexagon Food App — AI Agent 指引

> 專案名稱：`hexagon_food_app`  
> 說明：特耀訂出貨系統 — 讓訂出貨更簡單、更快速、更準確！  
> 類型：跨平台行動應用程式（Flutter / Dart）

> 本文件為原 `sales-order-app/AGENTS.md` 之整合版，已搬移至 parent repo。內容以實際程式碼為準。

本文件供 AI coding agent 在協助維護此專案時參考。內容皆根據目前工作目錄中的實際檔案查證撰寫；若發現與實作不符，以實際程式碼為準並更新本文件。

---

## 1. 專案概覽

- **框架**：Flutter `3.35.2`（`.fvmrc` 固定，透過 FVM 管理；VS Code workspace 指向 `.fvm/versions/3.35.2`），Dart SDK `>=3.9.0 <4.0.0`。
- **目標平台**：Android、iOS（無 web / desktop 設定）。
- **業務**：業務員（salesrep）與客戶（customer）用來建立、查詢、管理銷售訂單（sales order）的 App；後端為 NetSuite 相關服務（REST API 前綴 `/api/v1`）。
- **主要語言**：程式碼註解與 UI 字串以繁體中文為主（UI 字串多為硬編碼）；程式碼識別字、檔案名稱、相依套件名稱保持英文原文。
- **版本**：`pubspec.yaml` 中為 `1.2.6+25`。
- **規模**：`lib/` 223 個 `.dart` 檔（47 個為產生檔，176 個為手寫來源檔）。
- **README.md**：僅為 Flutter 預設模板，無實際內容；本文件是唯一完整的專案說明。

### 關鍵設定檔

| 檔案 | 用途 |
|------|------|
| `pubspec.yaml` | Flutter 套件、版本、flutter_gen、launcher icons、native splash 設定 |
| `analysis_options.yaml` | Linter：繼承 `flutter_lints`，啟用 `solidart_lint: ^3.0.0` analyzer plugin |
| `Taskfile.yml` | [Task](https://taskfile.dev) 任務定義：建置、產生碼、測試、fastlane、firebase 等日常指令入口 |
| `l10n.yaml` | 本地化設定（template 為 `lib/localizations/app_en.arb`；實際 UI 字串多為硬編碼繁體中文） |
| `.fvmrc` | 固定 Flutter 版本 `3.35.2` |
| `.dev.env` / `.prod.env` | 後端 URL、Session 名稱、API Access Token、Crashlytics 開關（`*.env` 已加入 `.gitignore`） |
| `dev.env.hexagon` | env 檔的範本/備份（含實際值且在版控內，注意勿外洩） |
| `firebase.json` | FlutterFire 多 flavor 設定（dev / prod 兩組 Firebase 專案） |
| `firebase_flavor.sh` | 重新產生 `firebase_options_*.dart` 與平台專用 Google 設定檔的腳本 |
| `Gemfile` / `Gemfile.lock` | Ruby 相依（fastlane、xcodeproj 等）；`bin/`、`fastlane_bin/` 為 bundler binstub 目錄，非專案原始碼 |
| `~/.kimi/mcp.json` | Kimi CLI 的 MCP server 設定（butterkit / XcodeBuildMCP / android-studio / github），詳見第 9 節 |
| `.xcodebuildmcp/config.yaml` | XcodeBuildMCP 專案層級預設值（workspace、scheme prod、模擬器 iPhone 17 Pro Max、Debug-dev） |

---

## 2. 程式碼組織

`lib/` 採三層式架構（layer_business / layer_data / layer_presentation）：

```
lib/
├── main.dart                 # 共用初始化：edgeToEdge、GetStorage→Sembast 遷移、setupLocator()、Firebase/Crashlytics
├── main_dev.dart             # dev flavor 進入點（設定 Flavor.dev 後呼叫 initializeMainApp()）
├── main_prod.dart            # prod flavor 進入點
├── firebase_options_dev.dart / firebase_options_prod.dart
├── env/                      # envied 環境讀取類別（DevEnv / ProdEnv + 產生的 *.g.dart）
├── gen/                      # flutter_gen 產生（assets / colors / fonts）
├── localizations/            # app_en.arb + 產生的 app_localizations*.dart
├── layer_business/           # 業務邏輯層
│   ├── network/              # dio_client、auth_interceptor、error_exception、
│   │                         #   api/（各 API 類別 + endpoints + cache_options_mixin）、abstract/（介面）
│   ├── router/               # auto_route 路由（routes.dart）、RoutePath 列舉（paths.dart）、route_observer
│   ├── services/             # Providers：auth/、customer/、salesorder/、article/、metadict/、profile/、
│   │                         #   settings/（SettingsService + SettingsStore）、scaffold/ + refreshable_resource.dart
│   ├── utils/                # locator（GetIt）、flavor_config、tools
│   ├── extensions/           # BuildContext / DateTime 格式化 / QR 圖片匯出等擴充
│   └── firebase/             # crashlytics_talker_observer（Talker → Crashlytics 橋接）
├── layer_data/               # 資料層
│   ├── models/               # Freezed + json_serializable 模型
│   │                         #   （article、auther、base、customer、department、estimate、metadict、salesorder、setting、validations）
│   ├── repositories/         # Sembast 本機儲存（sembast_kv_storage、session_info、cookies、cache）+ abstract/ 介面
│   ├── constants/            # system_constants（編譯期 fallback）、metadict_tables、text_style_const、theme_data
│   ├── converts/             # JsonConverter（session info、Netsuite 日期、salesorder record type）
│   └── enums/                # SalesOrderRecordType
└── layer_presentation/       # 表現層
    ├── app.dart              # App 入口 widget：disco ProviderScope、router config、deepLinkBuilder、主題
    ├── stories/              # 畫面：admin/（dashboard tabs、customer、salesorder_item）、auth/、error/、logger/
    ├── widgets/              # 共用 widget（appbar、bottom_nav、form、pinput、refresh、search_bar、tiles...）
    ├── general/              # Base widget、alert/snack mixin、主題 extension、prompt 訊息、Utils
    └── utils/                # 畫面輔助（dropdown helper、salesorder item helpers）
```

### 主要模組職責

- **Flavor / 環境**：`FlavorConfig`（單例，`lib/layer_business/utils/flavor_config.dart`）依 `Flavor.dev` / `Flavor.prod` 讀取 `DevEnv` / `ProdEnv`（envied 產生），提供 `getBaseUrl()`、`getFrontendUrl()`、`getSessionName()`、`getApiAccessToken()` 及 Crashlytics 各開關。Android 上 `localhost` URL 會自動改寫為 `10.0.2.2`。
- **相依注入（兩層）**：
  - `GetIt`（`lib/layer_business/utils/locator.dart` 的 `setupLocator()`）註冊基礎設施：`DioClient`、`AuthApi`、`AppRouter`、Sembast storages（session / cookies / cache / settings）、`PersistCookieJar`、`AuthSessionManager`、`SettingsService`（lazy singleton），以及 dev 專屬的 `Talker`。
  - `disco` 的 `ProviderScope`（於 `lib/layer_presentation/app.dart`）注入：`accountAuthProvider`、`customerApi`、`departmentApi`、`estimateApi`、`metadictApi`、`salesorderApi`、`articleApi`；各 story 內另有局部 ProviderScope（如 `customerProvider`、`metadictProvider`）。
- **狀態管理**：
  - `AuthProvider`（`services/auth/provider.dart`）是唯一的 `ChangeNotifier`，訂閱 `AuthSessionManager.sessionStream`，並作為 router 的 `reevaluateListenable`。
  - 其餘業務 provider（customer、salesorder、article、metadict、scaffold）使用 `flutter_solidart` 的 `Signal` / `ListSignal` / `Resource`，搭配自訂 `RefreshableResource<T, F>`（`services/refreshable_resource.dart`）封裝「filter signal + resource + refresh」模式。
  - `SettingsService`（`services/settings/settings_service.dart`）以 `Signal<SettingsModel?>`（`settingsSignal`）暴露 backend 設定；`current` 取值順序為 Signal → Sembast 快取 → 編譯期 fallback。
- **路由**：`auto_route`，定義於 `lib/layer_business/router/routes.dart`（`replaceInRouteName: 'Screen|Page,Route'`）；路徑常數為 `paths.dart` 的 `@MappableEnum() enum RoutePath`（dart_mappable）。登入檢查使用 `AuthGuard`（未登入 `redirectUntil(SelectSigninRoute())`）。路由樹：
  - `/dashboardLayout`（AuthGuard，initial）→ `HomeRoute`、`SalesorderRoute`、`OrderHistoryRoute`、`ProfileRoute`
  - `/customerLayout`（AuthGuard）→ `CustomerRoute`
  - `/salesorderItemLayout`（AuthGuard）→ `SalesorderItemRoute`
  - `/authLayout`（無 guard）→ `SelectSigninRoute`、`SalesrepSigninRoute`、`CustomerSigninRoute`、`customerQrcodeSignin/:customerAccount`
  - `/networkLogger` → `TalkerRoute`；`*` → `UndefinedRoute`
  - auth 另有 forget_password / otp / register / reset_password 四個畫面，路由已註解停用（檔案位於 `stories/auth/widget/unuse/`）。
- **網路**：`Dio`（`network/dio_client.dart`）+ `CookieManager`（PersistCookieJar）+ 自訂 `AuthInterceptor` + `dio_cache_interceptor`（Sembast store）；連線 timeout 30s、收發 60s。
  - `AuthInterceptor`：`onRequest` 加上 `X-Sowinsoft-Token: <API_ACCESS_TOKEN>` 等標頭；已登入但目標 URI 無 cookies 時主動 `clearSession()`（狀態不一致防護）；`onError` 收到 HTTP 401 時 `clearSession()`。
  - 各 API 類別（`network/api/*_api.dart`：auth、customer、salesorder、department、estimate、metadict、article、settings）皆有對應抽象介面（`network/abstract/*_api_type.dart`）並混入 `CacheOptionsMixin`（sm 5 分鐘 / md 15 分鐘 / lg 8 小時等快取預設，`hitCacheOnErrorCodes: [400, 404, 500]`）。
  - 僅 dev flavor 且 debug/profile 模式才掛 `TalkerDioLogger`（不印 request/response data）。
  - 錯誤統一由 `ErrorException`（`network/error_exception.dart`）轉為繁體中文訊息，含 400/401/403/422/429/500/502 fallback。
- **表單**：`reactive_forms` + `reactive_forms_generator`，表單模型與資料模型同檔（`@Rf()` / `@RfGroup()`）。僅 4 個模型有 `*.gform.dart`：`customer/customer.dart`、`salesorder/salesorder.dart`、`auther/salesrep/signin_form_model.dart`、`auther/customer/customer_signin_form_model.dart`。
- **資料模型**：以 `freezed` + `json_serializable` 為主（`abstract class ... with _$X`）；僅路由 `RoutePath` 使用 `dart_mappable`。通用 wrapper：`PaginatedModel<T, R>`、`DefaultMetaModel`（`models/base/data_wrapper.dart`）。
- **本機持久化**：Sembast 單一資料庫 `app_storage.db`（`repositories/sembast_kv_storage.dart`），存放 session info（`session_info_store`）、cookies（`cookie_store`，作為 `PersistCookieJar` 後端）、HTTP 快取（`cache_store`，`http_cache_sembast_store`）、系統設定（`settings_store`，key `settings`）。首次啟動從舊版 GetStorage 做一次性遷移（旗標 `_sembast_session_migrated_v1`）。圖片快取使用 `cached_memory_image`（git fork 相依）。
- **系統設定**：`SettingsService`（`services/settings/settings_service.dart`，GetIt lazy singleton）於 app 啟動時 `loadCache()`（`main.dart`）讀取 Sembast 快取，登入成功時 `refresh()`（`services/auth/auth_service.dart`）由 backend `GET /api/v1/settings`（`SettingsApi`，`network/api/settings_api.dart`，sm 快取）重新取得並寫回快取；網路失敗保留既有值。`SystemConstants`（`layer_data/constants/system_constants.dart`）已改為編譯期 fallback：`defaultDepartment = 6`（已修正，與 backend seed 一致）、`frontendUrl = ''`（執行期由 backend settings 覆寫，`deeplinkCompanyLink` 已標記 deprecated）。設定實際用於：customer 部門篩選排除系統部門（`services/customer/customer_service.dart`）、訂單表單系統業務員/部門替換為公司管理員/預設部門（`services/salesorder/salesorder_service.dart`、`stories/admin/salesorder_item/`）、客戶 QR Code / 分享連結組裝 frontendUrl（`getCustomerDeepLinkUrl()`）、關於頁 aboutUrl（`services/profile/profile_service.dart`）。

---

## 3. Flavor 與環境

| Flavor | Dart 進入點 | Android applicationId | iOS bundle ID | Firebase 專案 |
|--------|-------------|-----------------------|---------------|---------------|
| `dev`  | `lib/main_dev.dart` | `com.hexagonty.salesorder.app.dev` | `com.hexagonty.salesorder.app.dev` | `hexagon-salesorder-pf-dev` |
| `prod` | `lib/main_prod.dart` | `com.hexagonty.salesorder.app` | `com.hexagonty.salesorder.app` | `hexagon-salesorder-platform` |

- **Android**：product flavors 定義於 `android/app/build.gradle`（dimension `"app"`；dev 加 `applicationIdSuffix ".dev"`，app_name 分別為 `hexagon_food_app` / `Dev hexagon_food_app`）。release 簽章讀自 `android/key.properties`（未入版控），keystore 位於 `android/keystore/`。輸出檔命名為 `hexagon_food_app-${versionName}.${versionCode}`。
- **iOS**：build configurations 為 `Debug-dev` / `Profile` / `Profile-dev` / `Release-prod`（無 `Debug` / `Release`，Runner / RunnerTests / RunnerUITests 皆同），另有 `dev`、`prod` 兩個 shared scheme（`ios/Runner.xcodeproj/xcshareddata/xcschemes/`）；`ios/Podfile` platform 為 iOS `15.6`。
- **VS Code**：啟動設定內建於 `.vscode/launch.json`（`dev` / `prod` 兩組）。
- **平台工具鏈版本**（供參考）：AGP `9.2.1`、Kotlin `2.2.0`、google-services `4.4.4`、firebase-crashlytics gradle plugin `3.0.7`、`compileSdk = 37`、Java 17、NDK `28.2.13676358`；minSdk / targetSdk 沿用 Flutter 預設值。

---

## 4. 常用建置與執行指令

> Flutter 指令建議透過 FVM（`fvm flutter ...`）或已設定的 VS Code workspace。專案另有 [Task](https://taskfile.dev)（`Taskfile.yml`）封裝常用指令；注意 Task 內直接呼叫 `flutter` / `dart`，若系統 PATH 中沒有 Flutter，請改用 FVM 的 flutter 手動執行對應指令。

```bash
# 安裝/更新相依套件
fvm flutter pub get

# 開發模式執行（dev / prod）
fvm flutter run --flavor dev --target lib/main_dev.dart
fvm flutter run --flavor prod --target lib/main_prod.dart

# 產生程式碼（Freezed、json_serializable、reactive_forms、auto_route、envied、flutter_gen、dart_mappable）
fvm dart run build_runner build --delete-conflicting-outputs   # 等同 task gen
fvm dart run build_runner watch --delete-conflicting-outputs   # 開發時持續監看

# flutter_gen（assets / colors / fonts）
fluttergen -c ./pubspec.yaml   # 等同 task gen:flutter（需先 dart pub global activate flutter_gen）

# 產生本地化檔（模板幾乎為空，鮮少使用）
fvm flutter gen-l10n

# 產生啟動圖示與啟動畫面
fvm dart run flutter_launcher_icons        # 等同 task gen:icons
fvm dart run flutter_native_splash:create  # 等同 task splash
```

### Taskfile 常用任務

| 指令 | 說明 |
|------|------|
| `task gen` / `task gen:clean` | build_runner build / clean |
| `task gen:all` | build_runner + flutter_gen + launcher icons |
| `task gen:icons` | 產生 launcher icons（`fvm flutter pub run flutter_launcher_icons`） |
| `task splash` | 產生 native splash（`fvm flutter pub run flutter_native_splash:create`） |
| `task build` | 完整 release 流程：clean → gen → 安裝 flutterfire_cli → build 號 +1（pub_version_plus）→ 建置 prod appbundle 與 ipa |
| `task build:version` | 以 `pub_version_plus` 遞增 build 號（`task build` 內部使用） |
| `task build:release:android -- appbundle` / `task build:release:ios -- ipa` | prod release 建置（`--` 後為 flutter build 目標） |
| `task build:debug:android -- apk` / `task build:debug:ios -- ios` | dev debug 建置 |
| `task clean` / `task clean:all` | flutter clean + pub get / 含 Pods、build、pub cache 的深層清理 |
| `task iospod` | 重建 iOS Pods（刪 Pods、Podfile.lock、.symlinks 後 pod install） |
| `task test:maestro` | 執行 Maestro 整合測試 |
| `task test` | 執行全部單元測試（`fvm flutter test`） |
| `task analyze` | 執行靜態分析（`fvm flutter analyze`） |
| `task screenshots:android` | 以 Maestro 擷取 Android 各分頁商店截圖（需先啟動模擬器並安裝 dev app） |
| `task screenshots:ios` | 以 Maestro + iOS 模擬器擷取 App Store 商店截圖 |
| `task screenshots:inject:ios` | 將 `store_ios/` 截圖注入 ButterKit iPhone 6.9″ artboards |
| `task screenshots:export:ios` | 匯出 ButterKit iPhone 6.9″ 成品到 `appimg/export_app_store/` |
| `task firebase:flavor -- dev|prod` | 執行 `firebase_flavor.sh` 重新產生 Firebase 設定 |
| `task fastlane:android -- <lane>` / `task fastlane:ios -- <lane>` | 透過 `fastlane_bin/` binstub 執行 fastlane |
| `task deeplink_test` | 以 adb 觸發客戶 QR Code deep link 測試 |
| `task crashlytics:mode -- DEBUG|INFO` / `task crashlytics:cat` | 調整 / 觀察 Android 上 Crashlytics log |

### 平台建置範例（不使用 Task 時）

```bash
fvm flutter build apk --flavor prod --target lib/main_prod.dart
fvm flutter build appbundle --flavor prod --target lib/main_prod.dart
fvm flutter build ios --flavor prod --target lib/main_prod.dart   # 需 macOS + Xcode
```

### Firebase 設定重新產生

若更動 Firebase 專案或 flavor，執行：

```bash
./firebase_flavor.sh dev    # 或 task firebase:flavor -- dev
./firebase_flavor.sh prod
```

此腳本（`flutterfire config`）會產生：
- `lib/firebase_options_dev.dart` / `lib/firebase_options_prod.dart`
- `ios/dev/GoogleService-Info.plist` / `ios/prod/GoogleService-Info.plist`
- `android/app/src/dev/google-services.json` / `android/app/src/prod/google-services.json`

---

## 5. 程式碼風格與慣例

- **Linter**：繼承 `package:flutter_lints/flutter.yaml`，並在 `analysis_options.yaml` 啟用 `solidart_lint: ^3.0.0` analyzer plugin；`invalid_annotation_target` 設為 ignore（freezed / reactive_forms 註解需要）。下列規則已關閉：
  - `use_late_for_private_fields_and_variables: false`
  - `implicit_call_tearoffs: false`
  - `specify_nonobvious_property_types: false`
- **行寬**：`120`（`.vscode/settings.json` 的 `dart.lineLength`；flutter_gen 輸出亦設 120）。
- **產生檔入版控**：`*.g.dart`、`*.freezed.dart`、`*.gform.dart`、`*.mapper.dart`、`routes.gr.dart`、`lib/gen/*`、`lib/env/*.g.dart`、`lib/localizations/app_localizations*.dart` 皆已加入版控；修改來源檔後務必重新執行 `build_runner` 並一併提交產生檔。
- **命名**：
  - 資料模型：`lib/layer_data/models/{domain}/{model}.dart`，搭配 `*.freezed.dart` / `*.g.dart`；查詢 filter 模型統一命名為 `filter.dart`。
  - 表單模型：與資料模型同檔，加 `@Rf()` / `@RfGroup()` 註解後產生 `*.gform.dart`。
  - API 類別：`lib/layer_business/network/api/*_api.dart`，抽象介面在 `lib/layer_business/network/abstract/*_api_type.dart`。
  - Provider/Service：`lib/layer_business/services/{domain}/provider.dart`（以 disco `Provider` 暴露）。
  - 畫面：`lib/layer_presentation/stories/{story}/{screen}.dart`，`@RoutePage()` 畫面類別以 `Screen` 結尾。
- **註解**：以繁體中文撰寫；interceptor、session 清除、登入流程等同步/競爭條件相關處，請保留並維護原有說明註解。
- **UI 字串**：多為硬編碼繁體中文；`lib/localizations/app_en.arb` 僅為模板。新增使用者可見文字請維持繁體中文，或先與團隊確認多語系策略。

---

## 6. 測試

- **單元 / Widget 測試**：`test/` 目錄已有單元與 widget 測試（`services/settings_service_test.dart`、auth、表單 widget、快取、`integration/` 等）；若新增測試請放在 `test/` 並使用 `flutter_test`：
  ```bash
  fvm flutter test
  ```
- **整合測試**：使用 [Maestro](https://maestro.mobile.dev/)（可用 `task install:maestro` 安裝）。
  - 設定檔：`integration_test/.maestro/config.yaml`（測試輸出至 `test_output_directory/`）
  - 主要流程：`integration_test/.maestro/hexagon.yaml`（針對 dev app `com.hexagonty.salesorder.app.dev`，`clearState` 啟動後執行業務登入子流程）
  - 子流程：`integration_test/.maestro/salesrep_login/Flow.yaml`
  - 執行：
    ```bash
    task test:maestro
    # 或：maestro test --config integration_test/.maestro/config.yaml integration_test/.maestro/hexagon.yaml
    ```
  - 注意：`salesrep_login/Flow.yaml` 與 `screenshots/Flow.yaml` 內含真實測試帳號密碼（明碼），請勿對外洩漏。
  - `integration_test/samples/` 是 Maestro 官方範例素材，與本專案測試無關。
- **Android 商店截圖流程**：`integration_test/.maestro/screenshots/Flow.yaml` 會登入業務帳號後，依序截取首頁 / 商品 / 訂單歷史 / 功能四個分頁，輸出至 `integration_test/.maestro/test_output_directory/screenshots/android/`：
  ```bash
  task screenshots:android
  ```
  **前置**：後端需在線（截圖需登入並載入資料；task 會先以 `curl http://localhost:3080/` 檢查，任何 HTTP 回應皆視為在線，root 回 404 屬正常）。後端專案位於 `../sales-order-backend/`（Go），啟動方式：
  ```bash
  cd ../sales-order-backend
  task infra:start   # 啟動 docker-compose 基礎服務（DB 等）
  task dev           # air hot-reload 啟動 API server（長駐，監聽 0.0.0.0:3080）
  ```
  選擇器策略（實測教訓，修改 flow 前必讀）：
  - App 的 Flutter Semantics identifier 會成為 Android resource-id，請優先用 `id:` 選擇器（如 `select_signin_screen_salesrep_button`、`auth_form_email_field`）。
  - 純文字選擇器不可靠：部分元件文字位於 `accessibilityText`（如 `我是業務\n我是業務`）而非 text 屬性，比對會失敗。
  - 底部導覽列選取中的 tab `accessibilityText` 為 `Tab 1 of 4\n首頁`，未選取的僅有 `Tab N of 4`（無標題）→ 切 tab 用 `tapOn: "Tab N of 4"`，斷言用 regex `(?s).*首頁`（`.` 不跨換行，需 `(?s)`）。
  - 不要在 flow 內用 `launchApp: clearState: true`（實測會導致 app 退到背景 / semantics 競態）；清狀態改由 Taskfile 在執行前 `adb shell pm clear com.hexagonty.salesorder.app.dev`。
  - 輸入密碼後需先 `hideKeyboard` 再點登入鈕；每步截圖前加 `waitForAnimationToEnd`。
  - 失敗排查：看 `test_output_directory/<timestamp>/screenshot-❌-*.png` 與 `adb shell uiautomator dump` 的 accessibilityText（或 `maestro hierarchy`）。
- **Play 商店截圖流程**：`integration_test/.maestro/screenshots_store/Flow.yaml` 擷取 9 張商店用截圖（身分選擇 / 業務登入 / 客戶登入 / 公司快訊 / 功能 / 客戶列表 / 新增客戶 / 訂單商品編輯 / 訂單明細），輸出至 `test_output_directory/screenshots/store/`。執行：
  ```bash
  adb shell pm clear com.hexagonty.salesorder.app.dev
  maestro test --config integration_test/.maestro/config.yaml integration_test/.maestro/screenshots_store/Flow.yaml
  ```
  訂單段實測教訓（修改前必讀）：
  - **Maestro `inputText` 不支援 Unicode**（issue #146）→ 中文搜尋改用 ASCII 關鍵字：客戶搜尋輸入代號 `C000026`（後端 `company_name` 參數會同時比對 entity_id）；商品下拉為 client-side `name.contains`（區分大小寫），輸入 `AU` 後點 regex `(?s).*腱子心.*`。
  - 客戶卡片 semantics 合併為 `公司名\n代號`（如 `測試用客戶-禮\nC000026`），點卡用 `(?s).*測試用客戶-禮\nC000026.*`（分店01/02 的 `\n` 前多 `-分店0N` 故不匹配）。
  - **Material `ReactiveDropdownField` 的 overlay menu 無法用 `scrollUntilVisible` 捲動** → 用 `swipe`（start/end 百分比座標落在選單範圍 x≈72% 內）拖曳：派車編號開啟時停在尾段（選項順序 = metadicts API 回傳序），向下拖（start y 小 → end y 大）回前段選 `3車`；業務下拉開啟停在首部，向上拖兩次選 `周禮`（index 9）。
  - 訂單明細卡片（ExpansionTile）收起態 merged text 為 `品名\n切法: …\n商品編號: …`；展開要点 `(?s).*商品編號.*`（勿用品名 regex，會誤點上方同品名的下拉欄位而把下拉重新打開）。
  - 明細卡內的切法欄位 `readOnly: true`（`salesorder_item_card.dart`），值由估價品項預設帶入，無法手選。
  - 從新增訂單項目頁返回時單次 `back` 有時無效（僅收合卡片）→ 用 `runFlow: { when: { notVisible: "Tab 3 of 4" }, commands: [back] }` 補一次。
- **iOS 商店截圖流程**：`integration_test/.maestro/screenshots_store_ios/Flow.yaml` 擷取 9 張 App Store 用截圖，輸出至 `integration_test/.maestro/test_output_directory/screenshots/store_ios/`。執行：
  ```bash
  task screenshots:ios
  ```
  與 Android 版的差異：
  - 輸出路徑為 `store_ios/0X_*`，避免覆蓋 Android 截圖。
  - 登入後使用 `hideKeyboard` 自動提交，不點明確 submit button。
  - 新增客戶為 WoltModalSheet，關閉鈕 semantics 為 `Dismiss`，關閉後下層頁面 title 為 `Back`，flow 以 `tapOn point: 95%,14%` 關 modal 後再點 `Back` 返回。
  - 商品下拉選擇後需先點商品列本體（`50%,30%`）讓 `selectedItem` 生效，再點「+」按鈕（`91%,16%`）加入明細。
  - 返回訂單編輯頁後點 `Back`，斷言 `Tab 2 of 4` 可見後切換到 `Tab 3 of 4` 取明細。
- **Android 16 KB 對齊檢查**：`scripts/check_elf_alignment.sh`（Google 官方腳本）可檢查 APK 內 `.so` 的 ELF page alignment（對應 Android 16 KB page-size 要求）。

---

## 7. 部署流程

無 CI/CD 服務設定（無 `.github/workflows`、codemagic、bitrise 等）；發布以本機 Fastlane + Taskfile 為主。Ruby 相依定義於根目錄 `Gemfile`（fastlane、xcodeproj 等），`bin/`、`fastlane_bin/` 為 bundler binstub。

### Android / iOS

部署由 superproject root Taskfile 統一編排（於 repo root 執行）：

```bash
task fastlane:beta -- auto                    # 版本遞增 + 建置 + 上傳 Beta（純 binary）
task fastlane:production -- auto              # 版本遞增 + 截圖 + 建置 + 上傳正式版（含 metadata/截圖）
task fastlane:upload_build:production -- auto # 版本遞增 + 建置 + 上傳正式版（純 binary）
```

- 上述三個 task 皆需 `-- auto`（自增 patch+build）或 `-- <build_number>`（patch+1 並設 build number）；不帶參數會直接報錯，避免以上一版版本號重複上傳。

- 各平台 lane 定義於 `ios/fastlane/Fastfile`、`android/fastlane/Fastfile`；app 層不再維護 combined Fastfile。
- Android `beta` lane 上傳 Play Store beta track（draft）；`production` 另含截圖與 metadata。
- iOS `beta` lane 上傳 TestFlight；`production` 另含截圖與 metadata。
- 平台層亦可單獨執行：`task fastlane:android -- <lane>` / `task fastlane:ios -- <lane>`（於 sales-order-app/）。

**Android**：
- `android/fastlane/Appfile`：package `com.hexagonty.salesorder.app`，`json_key_file("./keystore/hexagon-salesorder-platform.json")`（Google Play API key，未入版控）。
- release 簽章需 `android/key.properties` 與 `android/keystore/hexagon-salesorder-keystore.jks`（皆未入版控）。

**iOS**：
- `ios/fastlane/Appfile`：bundle id `com.hexagonty.salesorder.app`、team_id `3YU8K7FQ69`、itc_team_id `124040207`。
- `ios/fastlane/Snapfile`：snapshot 截圖設定（裝置 iPhone 17 Pro Max / 17 Pro、語系 `zh-Hant`、`dev` scheme、`configuration("Debug-dev")`，輸出至 `ios/fastlane/screenshots/`）。已建立 `RunnerUITests` UI Testing target（4 個 config 與 Runner 一致，已加入 dev / prod scheme 的 TestAction）；執行前尚需加入 SnapshotHelper 與截圖導覽碼（詳見 Snapfile 檔頭註解）。注意：Flutter 的 release/profile 僅能建置實機，模擬器截圖只能用 Debug-dev（產物為 dev app）。
- 完整一鍵 release 流程見 `task build`（clean → 產生碼 → build 號 +1 → 建置 appbundle + ipa）。

### 商店截圖管線（ButterKit）

[ButterKit](https://butterkit.app) 負責截圖後製（裝置框、3D、文案、50 語言 AI 翻譯）與 **App Store Connect** 上傳；**不支援 Google Play**。整體流程：

1. **擷取原始截圖**：
   - iOS：`task screenshots:ios`（Maestro + iPhone 17 Pro Max 模擬器）→ 輸出 `integration_test/.maestro/test_output_directory/screenshots/store_ios/`。
   - Android：`task screenshots:android`（Maestro）→ 輸出 `integration_test/.maestro/test_output_directory/screenshots/android/`。
2. **注入 ButterKit**：
   - iOS：`task screenshots:inject:ios`（對應 widthPx == 1290 的 iPhone 6.9″ artboards）。
   - Android：`python3 scripts/butterkit/butterkit_inject.py integration_test/.maestro/test_output_directory/screenshots/store --platform android`（預設即 android）。
3. **後製**：在 ButterKit 專案 `../appimg/screenshots.butterkit` 中套用裝置框 / 背景 / 文案。
4. **匯出成品**：
   - iOS：`task screenshots:export:ios` → `../appimg/export_app_store/`（20 張 = 10 畫面 × en-US/zh-hant）。
   - Android：`python3 scripts/butterkit/mcp_call.py call design_export_artboards ...` → `../appimg/export_google_play/`。
5. **上傳**：
   - App Store：由 ButterKit 上傳（GUI 或 MCP 工具 `asc_upload_screenshots` / `asc_upload_metadata` / `asc_create_version`；前置：ButterKit Settings > MCP 啟用、ASC API 憑證於 ButterKit 內設定）。
   - Google Play：ButterKit 不支援，須將後製圖放入 `android/fastlane/metadata/android/zh-TW/images/phoneScreenshots/`，搭配 supply（`upload_to_play_store`）上傳；現有 `beta` lane 已 skip images/screenshots，需另行調整或手動上傳。

> 注意：AI agent 呼叫 ButterKit MCP 工具須重啟 Kimi session 後才會掛載（見第 9 節）；且實際上傳前請先與團隊確認版本與素材。

**截圖注入自動化（已實跑）**：`scripts/butterkit/` 內三支腳本以 JSON-RPC stdio 直接驅動 `butterkit-mcp`（不經 Kimi MCP 掛載）：
```bash
# 註：`<repo 根目錄>` 為本機絕對路徑，依機器而異；script 需要絕對 file:// URI，執行前請替換為實際路徑
# 1. 重新抓取文件 artboard / device modelId 對照（base 與 zh-hant variant 的 modelId 各自不同）
python3 scripts/butterkit/butterkit_devices.py "file://<repo 根目錄>/appimg/screenshots.butterkit/" scripts/butterkit/butterkit_devices.json

# 2. 注入（--dry-run 先預覽；會把 PNG 複製到 group container 的 MCPAssets 再逐裝置設置）
# iOS
python3 scripts/butterkit/butterkit_inject.py integration_test/.maestro/test_output_directory/screenshots/store_ios --platform ios --dry-run
python3 scripts/butterkit/butterkit_inject.py integration_test/.maestro/test_output_directory/screenshots/store_ios --platform ios
# Android
python3 scripts/butterkit/butterkit_inject.py integration_test/.maestro/test_output_directory/screenshots/store --platform android --dry-run
python3 scripts/butterkit/butterkit_inject.py integration_test/.maestro/test_output_directory/screenshots/store --platform android

# 3. 匯出驗證（outputDir 必須在 ~/Library/Group Containers/group.app.butterkit/ 內，/tmp 無權限）
python3 scripts/butterkit/mcp_call.py call design_export_artboards '{"documentId":"file://<repo 根目錄>/appimg/screenshots.butterkit/","outputDir":"~/Library/Group Containers/group.app.butterkit/MCPAssets/export"}'
```
- 對應文件：`../appimg/screenshots.butterkit`（10 畫面 × 2 尺寸 × 2 語系 = 40 artboards）；截圖→artboard 對照表寫在 `butterkit_inject.py` 的 `MAPPING`。
- `登錄頁面` 為雙機構圖：
  - Android：`Pixel 10 Pro` ← 業務登入、`Pixel 10 Pro 2` ← 客戶登入。
  - iPhone 6.9″：第一裝置（`iPhone 15 Pro Max`）← 業務登入、第二裝置（`iPhone 17 Pro Max`）← 客戶登入。
- `訂單商品選擇-2`、`訂單查詢-2` 無裝置（沿用 -1 構圖）會自動略過。
- 最新成品匯出於：
  - `../appimg/export_app_store/`（20 張 iPhone 6.9″，en-US/zh-hant）。
  - `../appimg/export_google_play/`（20 張，en-US/zh-hant）。

---

## 8. 安全與機密注意事項

- **環境變數檔**：`.dev.env`、`.prod.env` 已被 `.gitignore` 排除（`*.env`）；但 `dev.env.hexagon`（env 範本/備份，含實際值）與 `lib/env/*.g.dart`（混淆後的值）在版控內，請勿外洩。需修改環境變數時請向團隊索取原始 `.env` 檔。
- **API Access Token**：於 `AuthInterceptor` 以 `X-Sowinsoft-Token` 標頭送出，請勿寫入 log 或截圖。
- **App Store Connect API key**：`ios/fastlane/Fastfile` 的 `app_store_connect_api_key` helper 內嵌 key ID 與 issuer；私鑰（`.p8`）放在 `.private_keys/`（已 gitignore）。
- **Maestro 測試帳號**：`integration_test/.maestro/salesrep_login/Flow.yaml`、`integration_test/.maestro/screenshots/Flow.yaml` 與 `integration_test/.maestro/screenshots_store_ios/Flow.yaml` 含明碼測試帳密，請勿對外洩漏。
- **認證資料**：session info、cookies、HTTP 快取皆以 Sembast 存於應用程式快取目錄（`app_storage.db`），未額外加密；測試或審查時請注意資料殘留。
- **登出與清除**：`AuthSessionManager.clearSession()` 會清除 session、cookies、HTTP 快取、圖片快取（idempotent）；`clearAuthCookies()` 僅清 cookies 且不發通知，用於登入流程失敗時避免狀態不一致與導航競爭。
- **Deep Link**：App 處理 `/customer_account_qrcode/:customerAccount` 深度連結，導向客戶 QR Code 登入頁；實作位於 `lib/layer_presentation/app.dart` 的 `deepLinkBuilder`。客戶 QR Code / 分享連結由 `CustomerService.getCustomerDeepLinkUrl()` 以 `SettingsService.current.frontendUrl`（執行期由 backend settings 提供）組裝，不再使用 `SystemConstants.deeplinkCompanyLink`（已標記 `@Deprecated`，`SystemConstants.frontendUrl` 為空字串）。
- **Firebase Crashlytics**：dev / prod 預設皆啟用並上傳 fatal error；開關由 `.env` 控制（`ENABLE_CRASHLYTICS`、`ENABLE_CRASHLYTICS_TALKER`、`ENABLE_FATAL_ERROR_RECORDING`），dev 可額外啟用 Talker 觀察者（`CrashlyticsTalkerObserver`）。

---

## 9. AI 開發工具（MCP Server）

Kimi CLI 自 `~/.kimi/mcp.json` 載入 MCP server；**新增或修改 server 後須重啟 Kimi session 才會掛載工具**。注意 CLI 版本：具備 `kimi mcp list` / `kimi mcp test <name>` 子命令的是 **v1.43+**（位於 VS Code 擴充套件目錄 `~/Library/Application Support/Code/User/globalStorage/moonshot-ai.kimi-code/bin/kimi/kimi`）；`~/.kimi-code/bin/kimi`（0.24.x）無 `mcp` 子命令，勿混用。目前設定：

| Server | 啟動方式 | 用途 | 前置條件 |
|--------|----------|------|----------|
| `github` | `npx @modelcontextprotocol/server-github` | GitHub issue / PR / 檔案操作 | GitHub token（環境既有） |
| `butterkit` | `/Applications/ButterKit.app/Contents/MacOS/butterkit-mcp`（stdio，會自動喚醒 ButterKit） | 截圖後製與 ASC 上傳（40 個工具，僅 App Store） | ButterKit Settings > MCP 啟用；ASC 憑證於 ButterKit 內設定 |
| `XcodeBuildMCP` | `npx xcodebuildmcp@latest mcp`（stdio） | Xcode 建置 / 測試 / 模擬器截圖（讀取 `.xcodebuildmcp/config.yaml` 預設值） | macOS 14.5+、Xcode 16+ |
| `android-studio` | Streamable HTTP `http://127.0.0.1:64342/stream` | Android Studio IDE 操作（27 個工具：build_project、get_file_problems、search_symbol、execute_run_configuration 等） | Android Studio（2026.1）未內建 IDE-as-MCP-server 模組，需 JetBrains「MCP Server」plugin（id 26071，`com.intellij.mcpServer`，已安裝於 IDE plugins 目錄）+ `options/mcpServer.xml` 設 `enableMcpServer=true`（已配置），重啟 IDE 後生效。MCP 埠為自動指派（自 64342 起，有機會變動；可用 `lsof -nP -iTCP -sTCP:LISTEN -a -p <studio pid>` 查，異動時需同步此處 url）。注意：Settings 裡 Gemini/Studio Bot 的「MCP Servers」是 **client 端**設定（讓 IDE 連外部 MCP server），與此無關 |

---

## 10. 其他注意事項

- `packages/custom_refresh_indicator/` 是殘留的上游套件 checkout（無 `pubspec.yaml`、無 `.dart` 原始碼）；專案實際使用 pub.dev 版 `custom_refresh_indicator: ^4.0.1`，請勿修改該目錄。
- `cached_memory_image` 使用 git 相依（fork：`github.com/sowiner/cached_memory_image`，ref `add_file_extension`），更換或升級時請注意。
- `pubspec.yaml` dev_dependencies 中 `custom_lint`、`solidart_lint`、`golden_screenshot` 目前被註解停用；`solidart_lint` plugin 版本改於 `analysis_options.yaml` 指定。
- `lib/layer_presentation/general/enums/enums.dart` 為空檔案（保留中）。
- `.fvm/` 已 gitignore；若本機 FVM 無 `3.35.2`，請先 `fvm install` 再使用 VS Code workspace 設定。

---

## 11. 給 AI Agent 的快速檢查清單

開始修改前，建議確認：

1. 是否使用正確的 flavor 與 `main_*.dart` 進入點（dev → `--flavor dev --target lib/main_dev.dart`）。
2. 修改 `lib/layer_data/models/` 的 Freezed / json_serializable / reactive_forms 來源檔後，是否已重新執行 `build_runner`（`task gen`）。
3. 修改 `lib/layer_business/router/paths.dart` 或 `routes.dart` 後，是否已重新產生 `paths.mapper.dart` / `routes.gr.dart`。
4. 修改 `.env` 後，是否已重新產生 `lib/env/*.g.dart`（envied 由 build_runner 一併處理）。
5. 新增資源（圖片、`assets/colors/*.xml`）後，是否已重新執行 `flutter_gen`（`task gen:flutter`）。
6. 畫面字串多為硬編碼繁體中文；若新增使用者可見文字，請維持繁體中文或與團隊確認多語系策略。
7. 若更動了本文件提及的架構、指令或流程，請同步更新本文件。

---

最後更新：根據 2026-08-21 查證整理（版本 `1.2.6+25`）。
