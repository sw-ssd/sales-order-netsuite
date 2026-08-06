# Fastlane 截圖與上傳統一化設計

**Date:** 2026-08-06  
**Scope:** sales-order-app (Flutter)  
**Status:** Design approved, awaiting spec review

## 1. Goal

將目前分散在 Taskfile、Maestro、ButterKit、手動 `xcrun altool` 的截圖與 APP 上傳流程，統一到 fastlane 管理，並提供單一 root-level 入口。

覆蓋範圍：
- 截圖產出（iOS App Store + Android Google Play）
- 截圖上傳至 App Store Connect / Google Play Console
- APP binary 上傳（TestFlight beta + 商店正式版）

不在本次範圍：
- CI/GitHub Actions 自動化（純本地 fastlane）
- 憑證管理方式的結構性變更（維持現有模式）
- 新增商店語系或畫面內容

## 2. Current State

- **iOS：**
  - 截圖：Maestro flow → `integration_test/.maestro/test_output_directory/screenshots/store_ios/` → ButterKit 注入 → `appimg/export_app_store/`
  - 上傳：Taskfile `upload:ios` 直接用 `xcrun altool`
  - Fastfile：有 `beta`（TestFlight）與 `production`（App Store）lanes，`production` 已會帶截圖
- **Android：**
  - 截圖：Maestro flow → `integration_test/.maestro/test_output_directory/screenshots/android/`，無後續匯出處理
  - 上傳：Fastfile 有 `beta` / `production` / `metadata` lanes
  - 無統一的截圖上傳路徑
- **Root：** 無 root-level fastlane，使用者需記憶 Taskfile + 平台 Fastlane 指令

## 3. Architecture

新增 `sales-order-app/fastlane/` 作為統一入口。Root Fastfile 負責編排，平台 Fastfile（`ios/fastlane/`、`android/fastlane/`）負責平台特定動作。

```
sales-order-app/
├── fastlane/
│   ├── Fastfile          # 統一入口
│   ├── Appfile           # 共用基本資訊
│   └── README.md         # 使用說明
├── ios/fastlane/Fastfile # 擴充 screenshots / beta / production
├── android/fastlane/Fastfile # 擴充 screenshots / beta / production
└── Taskfile.yml          # 移除 xcrun altool，截圖由 fastlane 驅動
```

Root lanes 透過 `sh` 委派平台 lanes，例如：

```ruby
sh("cd ../ios && bundle exec fastlane ios screenshots")
```

## 4. Lanes

### 4.1 Root Fastfile lanes

| Lane | 功能 |
|---|---|
| `screenshots` | 同時產出 iOS + Android 商店截圖 |
| `screenshots_ios` | 只產 iOS 截圖 |
| `screenshots_android` | 只產 Android 截圖 |
| `beta` | 建置 prod IPA/AAB，上傳 TestFlight + Play Console Beta（不上截圖） |
| `production` | 建置 prod、產截圖、上傳 binary + 截圖 + metadata |
| `upload_metadata` | 只上傳 metadata + 截圖，不傳 binary |

### 4.2 iOS Fastfile lanes

| Lane | 功能 |
|---|---|
| `screenshots` | 呼叫 Maestro + ButterKit 產出 `appimg/export_app_store/` |
| `beta` | `build:release:ios` 後 `upload_to_testflight` |
| `production` | 先 `screenshots`，再 `upload_to_app_store`（帶截圖） |
| `upload_metadata` | 只執行 `upload_to_app_store(skip_binary_upload: true)` |

### 4.3 Android Fastfile lanes

| Lane | 功能 |
|---|---|
| `screenshots` | 呼叫 Maestro，將截圖整理到 Play Store metadata 路徑 |
| `beta` | `build:release:android` 後 `upload_to_play_store(track: 'beta')` |
| `production` | 先 `screenshots`，再 `upload_to_play_store(track: 'production', skip_upload_screenshots: false)` |
| `upload_metadata` | 只上傳 metadata + 截圖，不傳 AAB |

## 5. Screenshot Path Conventions

### 5.1 iOS

維持現有流程：

1. Maestro → `integration_test/.maestro/test_output_directory/screenshots/store_ios/`
2. ButterKit 注入 → `appimg/export_app_store/<locale>-<n>-<title>.png`
3. `upload_to_app_store` 讀取 `screenshots_path: '../../appimg/export_app_store'`（相對 `ios/fastlane/`）

### 5.2 Android

新增整理腳本：

1. Maestro → `integration_test/.maestro/test_output_directory/screenshots/android/`
2. 腳本將截圖複製/重命名到 `fastlane/metadata/android/<locale>/images/phoneScreenshots/`
3. `upload_to_play_store` 使用 `metadata_path: './fastlane/metadata/android'`

現有 `appimg/export_google_play/` 可作為中間產物保留，但最終上傳來源以 `fastlane/metadata/android/` 為準。

## 6. Credentials

- **iOS：** 維持 Appfile 中的 `team_id`、`itc_team_id`、`apple_id`；App Store Connect API key p8 檔繼續放 `.private_keys/`，透過 `api_key_path`、`api_key_id`、`api_key_issuer_id` 傳入 fastlane。
- **Android：** 維持 `PLAY_STORE_JSON_KEY` 環境變數傳入 `json_key`，Appfile 中的 `json_key_file` 作為本機備援。
- **Root Fastfile：** 從 ENV 讀取 `SALES_ORDER_APP_STORE_API_KEY_ID`、`SALES_ORDER_APP_STORE_ISSUER_ID`、`PLAY_STORE_JSON_KEY`，傳遞給平台 lanes。

## 7. Error Handling & Testing

### 7.1 Error handling

- Root lane 委派平台 lane 時，任一平台失敗即中斷整個 lane。
- `screenshots` lanes 先檢查後端 `localhost:3080` 是否可用，避免 Maestro 登入失敗。
- `production` lane 首次建議使用 `--dry_run` 或只建置不上傳；正式上傳時再帶 metadata / screenshots。
- 保留 `upload_metadata` lane，讓 metadata 更新可獨立於 binary 上傳。

### 7.2 Testing plan

1. 本地驗證 `bundle exec fastlane screenshots` 成功產出截圖到 `appimg/export_app_store/` 與 `fastlane/metadata/android/`。
2. 本地驗證 `bundle exec fastlane beta` 只建置上傳 binary，不上截圖。
3. 小版本 patch 實際上傳到 TestFlight / Play Console Beta 測試 end-to-end。
4. 檢查 App Store Connect / Play Console 截圖與 metadata 是否正確更新。

## 8. Risks & Open Questions

| Risk | Mitigation |
|---|---|
| Android 截圖尺寸與 Play Store 要求不一致 | 新增整理腳本時對照 Play Console 截圖規範 |
| `upload_to_app_store` 對截圖檔名/語系解析與現有 `appimg/export_app_store/` 命名不一致 | 測試後調整 `screenshots_path` 或截圖檔名 |
| Root Fastfile `sh` 委派在不同 shell 環境下失敗 | 使用絕對路徑或相對 `../ios`、`../android` 並測試 |
| 現有 `upload:ios` 直接 `xcrun altool` 被棄用 | 在 Taskfile 保留 alias 或移除並更新文件 |

## 9. Non-Goals

- 不引入 GitHub Actions CI。
- 不新增商店語系。
- 不改變憑證儲存位置或新增 secrets manager。
- 不重寫 Maestro 截圖流程，只由 fastlane 編排現有流程。
