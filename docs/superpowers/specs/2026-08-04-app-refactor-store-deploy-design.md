# App 重構與商店上架 — 設計規格

> 日期：2026-08-04  
> 狀態：設計核准  
> 關聯：sales-order-app（Flutter 行動應用，v1.2.7+26）

---

## 目標

漸進式重構 sales-order-app，統一狀態管理、補強測試、建立 CI/CD、完成 Google Play 與 App Store 雙平台上架。無硬性截止日，品質優先。

## 策略

四個獨立階段，每階段完成後皆可獨立驗收與上線：

| 階段 | 主題 | 產出 |
|------|------|------|
| 1 | 基礎建設 | CI/CD、i18n 框架、測試基礎設施、商店合規骨架 |
| 2 | 狀態管理統一 | ChangeNotifier → solidart、Provider 標準化、命名統一 |
| 3 | 測試補強 | 分層測試（unit/widget）、Fake-over-Mock、P0→P3 優先級 |
| 4 | 商店上架 | Play Store + App Store 上架、隱私政策、CI/CD 部署自動化 |

---

## 階段 1：基礎建設

### CI/CD Pipeline（GitHub Actions）

```
.github/workflows/
├── ci.yml              # push/PR：flutter analyze + flutter test
├── build-android.yml   # workflow_dispatch：建置 AAB
└── build-ios.yml       # workflow_dispatch：建置 IPA
```

- **`ci.yml`**：`flutter analyze` + `flutter test`，PR 阻擋合併
- **`build-android.yml`**：手動選擇 flavor → 建置 AAB → 上傳 GitHub Release artifacts
- **`build-ios.yml`**：macOS runner，建置 IPA → 可上傳 App Store Connect
- 簽署金鑰皆由 GitHub Secrets 注入
- Flutter 版本從 `.fvmrc` 讀取（`subosito/flutter-action`）

### i18n 基礎框架

```
lib/l10n/
├── app_zh.arb        # 繁體中文（主要語言）
└── app_en.arb        # 英文（後續補）
```

- 加入 `flutter_localizations`（Flutter SDK 內建）
- `l10n.yaml` 指向 `lib/l10n/`
- App 端註冊 delegates
- 建立 `T.of(context).xxx` helper 簡化語法
- **第一階段不搬移現有硬編碼字串**，只建立基礎設施；新字串走 i18n

### 商店合規骨架

- **Android**：`android/app/src/main/play/release-notes.xml`、確認 targetSdkVersion / 權限、Data Safety 資料整理
- **iOS**：`PrivacyInfo.xcprivacy`（iOS 17+ 必要）、`Info.plist` 隱私宣告確認

### 測試基礎設施

```
test/
├── helpers/
│   ├── test_locator.dart     # 可替換 DI setup
│   ├── fake_api.dart         # Fake API 實作
│   └── pump_app.dart         # 標準 widget pump wrapper
├── unit/                     # 純邏輯測試
├── widget/                   # Widget 測試
└── integration/              # 保持現有 Maestro，不變
```

- 加入 `mocktail` 作為 mock 框架（備用，主力用 Fake）
- `test_locator.dart`：重構 `setupLocator()` 使其接受測試替身
- `pump_app.dart`：標準化 `MaterialApp.router` + `ProviderScope` 的 pump 流程

---

## 階段 2：狀態管理統一

### 現狀問題

| 機制 | 用途 | 處理 |
|------|------|------|
| GetIt | 全域基礎設施 | ✅ 保留（合理） |
| disco | DI 容器 | ✅ 保留（與 solidart 不衝突） |
| ChangeNotifier | 僅 AuthProvider | ❌ 消滅，統一用 solidart |
| solidart Signal | 業務 provider | ✅ 主力，標準化 |

### AuthProvider → AuthService

```dart
// 現狀
class AuthProvider with ChangeNotifier {
  AuthStatus _status = AuthStatus.unknown;
  Future<void> signIn(...) async { …; notifyListeners(); }
}

// 目標
class AuthService {
  final _status = Signal<AuthStatus>(AuthStatus.unknown);
  final _sessionInfo = Signal<AuthSessionInfo?>(null);
  Future<void> signIn(...) async { …; _status.value = newStatus; }
}
```

- `reevaluateListenable` 改用 solidart computed signal 或直接 observe `_status`
- disco `accountAuthProvider` 提供 `AuthService`
- `SessionExpiryOverlay` 改用 `SignalBuilder` / `context.watch`

### Provider 標準化

每個 domain 一個 service class，暴露 signals + methods：

```dart
class CustomerService {
  final CustomerApi _api;
  final customers = ListSignal<Customer>([]);
  final selectedCustomer = Signal<Customer?>(null);
  final filter = Signal<CustomerFilter>(CustomerFilter());
  Future<List<Customer>> search(String keyword) async { … }
  Future<void> create(CustomerForm form) async { … }
}
```

不強制每次都用 `RefreshableResource` — 只有需要 refresh/filter pattern 的場景才用。

### 檔案組織

`provider.dart` → `{domain}_service.dart`：

```
lib/layer_business/services/
├── auth/
│   └── auth_service.dart
├── customer/
│   ├── customer_service.dart
│   └── modal_service.dart
├── salesorder/
│   └── salesorder_service.dart
├── article/
│   └── article_service.dart
├── metadict/
│   └── metadict_service.dart
├── profile/
│   └── profile_service.dart
└── scaffold/
    └── scaffold_service.dart
```

disco provider 定義放各 service 檔末或獨立的 `providers.dart`。

### disco / GetIt 分工

- **GetIt**：無生命週期的全域單例（DioClient、Router、Sembast storages）
- **disco**：需要 widget tree context 的 service 實例、需要 scope 隔離的 provider

---

## 階段 3：測試補強

### 測試分層

```
test/
├── helpers/          # 階段 1 已建立
├── unit/             # 純邏輯，無 Flutter 依賴
│   ├── services/     # AuthService, CustomerService …
│   ├── models/       # Freezed model JSON 序列化
│   └── utils/        # 擴充方法、格式化
├── widget/           # Widget 測試
│   ├── auth/         # 登入畫面
│   ├── admin/        # 首頁、訂單、客戶 tabs
│   └── widgets/      # 共用元件
└── integration/      # 現有 Maestro，不變
```

### 優先級

| 優先級 | 測試對象 | 理由 |
|--------|----------|------|
| P0 | AuthService 登入/登出/續期 | 所有功能入口 |
| P0 | AuthInterceptor 401 邏輯 | 曾修過 bug，易 regression |
| P0 | ErrorException 錯誤轉換 | 繁體中文訊息，手改易壞 |
| P1 | CustomerService 搜尋/CRUD | 核心業務 |
| P1 | SalesOrderService 建立/編輯 | 核心業務 |
| P1 | Freezed model JSON 序列化 | schema 相容性 |
| P2 | 共用 Widget | 多處複用 |
| P2 | 路由 guard 邏輯 | 登入 redirect |
| P3 | 其餘畫面 snapshot | 低優先（Maestro 已有 E2E） |

### Mock 策略：Fake over Mock

實作抽象介面的簡單假物件，不 record/verify：

```dart
class FakeAuthApi implements AuthApiType {
  String? _simulateError;
  AuthSessionInfo? _fakeSession;
  @override
  Future<AuthSessionInfo> signIn(...) async {
    if (_simulateError != null) throw Exception(_simulateError);
    return _fakeSession!;
  }
}
```

### 覆蓋率目標

關鍵路徑必須有測試：
- Auth 流程：登入成功 / 失敗 / 重開 / 401 登出 / session 到期
- 核心 CRUD：每個 service happy path + 一個 error path
- 模型序列化：每個 freezed model round-trip

---

## 階段 4：商店上架

### Google Play

| 項目 | 現狀 | 行動 |
|------|------|------|
| 簽署 | keystore 已就緒 | 確認上傳用與本地的一致 |
| AAB 建置 | `task build:release:android` | ✅ 已就緒 |
| Fastlane | beta lane 存在 | 擴充：metadata、截圖、production lane |
| 截圖 | `appimg/export_google_play/` 18 張 | ✅ 已就緒 |
| Feature graphic | 無 | 建立 1024×500 |
| 商店描述 | 無 | 撰寫繁中+英文 |
| 隱私政策 | 無 | 建立公開網頁 |
| Data Safety | 無 | 填寫表單 |
| 內容分級 | 無 | 完成問卷 |

**Fastlane lanes**：

```ruby
lane :beta do
  upload_to_play_store(track: 'beta', aab: '...', release_status: 'draft')
end

lane :production do
  upload_to_play_store(track: 'production', aab: '...', release_status: 'completed')
end

lane :metadata do
  upload_to_play_store(metadata_path: './fastlane/metadata/android',
    skip_upload_apk: true, skip_upload_aab: true,
    skip_upload_images: false, skip_upload_screenshots: false)
end
```

**Metadata 目錄**：

```
android/fastlane/metadata/android/
├── zh-Hant/              # 繁體中文
│   ├── short_description.txt
│   ├── full_description.txt
│   ├── title.txt
│   └── changelogs/default.txt
├── en-US/                # 英文
│   ├── short_description.txt
│   ├── full_description.txt
│   └── title.txt
└── images/
    ├── feature-graphic.png   # 1024×500
    └── phoneScreenshots/     # 現有截圖複製
```

### App Store

| 項目 | 現狀 | 行動 |
|------|------|------|
| 簽署 | Xcode 自動管理 | ✅ 已就緒 |
| IPA 建置 | `task build:release:ios` + `task upload:ios` | ✅ 已就緒 |
| Fastlane | beta lane 存在 | 擴充 metadata 上傳 |
| 截圖 | `appimg/export_app_store/` 20 張 | ✅ 已就緒 |
| PrivacyInfo | 無 | 建立 `PrivacyInfo.xcprivacy` |
| 隱私標籤 | 無 | 填寫 App Store Connect |
| App 描述 | 無 | 撰寫繁中+英文 |

### 隱私政策

建立公開網頁（建議 GitHub Pages），內容：
- 收集資料：帳號、裝置資訊、crash log
- 用途：登入驗證、訂單處理、App 改善
- 第三方：Firebase Crashlytics / Analytics
- 資料保留與刪除
- 聯絡方式

### CI/CD 部署自動化

從階段 1 CI/CD 擴充 release workflows：

```yaml
# release-android.yml: workflow_dispatch → AAB → Play Store beta
# release-ios.yml:     workflow_dispatch → IPA  → TestFlight
# Secrets: PLAY_STORE_JSON_KEY, KEYSTORE_BASE64, APP_STORE_CONNECT_API_KEY
```

### 上架分階段發布

1. **內部測試**：Play Store closed track + TestFlight internal（團隊）
2. **封閉 Beta**：Play Store open track + TestFlight external（擴大到 100 人）
3. **正式上架**：Play Store production + App Store review提交

每階段間隔 1–2 週，收集 crash report 和回饋。

### 上線前檢查清單

- [ ] Android keystore 備份確認
- [ ] Prod flavor server endpoint 確認
- [ ] Crashlytics prod 正常收到 crash report
- [ ] Firebase Analytics 不記錄 PII
- [ ] 深層連結 release build 測試
- [ ] Play Store closed track 內部測試
- [ ] TestFlight 內部測試
- [ ] 隱私政策法律審查

---

## 不納入範圍

以下項目刻意排除，不在本次重構範圍：

- **硬編碼字串全面搬移至 i18n**：階段 1 僅建立基礎設施，全面搬移是長期目標
- **多語系翻譯**：英文版翻譯不在本次範圍，僅建立 ARB 框架
- **backend / frontend 重構**：本設計只針對 sales-order-app（Flutter）
- **功能新增**：不新增業務功能，只做架構改善
- **Flutter 版本升級**：保持 Flutter 3.35.2
- **Netsuite 模組變更**：不碰 `solidart_lint` custom lint 與現有 linter 設定

---

## 風險與緩解

| 風險 | 影響 | 緩解 |
|------|------|------|
| AuthService 重構引入登入 bug | 全部功能掛掉 | P0 測試先寫，重構後立刻驗證 |
| solidart 與 auto_route reevaluateListenable 整合 | 路由 guard 失效 | 先 spike 確認可行性，階段 2 最前面驗證 |
| CI/CD secrets 外洩 | 安全事件 | 僅使用 GitHub Secrets，不入版控任何金鑰 |
| App Store 審查被拒 | 延遲上架 | 內部測試階段先檢查合規、隱私清單 |
