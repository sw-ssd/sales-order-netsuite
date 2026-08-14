# Fastlane upload-build lanes + Taskfile 整理重構 — 設計

日期：2026-08-14
範圍：superproject root `Taskfile.yml`、`sales-order-app`、`sales-order-backend`、`sales-order-frontend`

## 目標

1. 新增「build only」流程：iOS + Android、beta + production，**遞增版本號 → 重新建置 → 只上傳 binary**（跳過截圖產生與 metadata/截圖上傳）。
2. 四個 Taskfile 分組整理：補齊 `desc`、刪死碼與重複 task、統一結構。

## 已確認的決策（使用者核准）

- Build-only 語意：含版本號遞增與重新建置；上傳時 beta 與 production 皆不含 metadata/截圖。
  （版本號在建置時包進 binary，故必須重建；純上傳現有 artifact 不在本次範圍。）
- 採方案 A（單層編排）：刪除 `sales-order-app/fastlane/` combined Fastfile，fastlane 跨平台編排統一由 superproject root Taskfile 負責。
- 刪除 app Taskfile 的 combined-lane tasks（由 root 接管）。
- 刪除 backend Taskfile 的 `list` alias。

## 現況與問題

- `fastlane/Fastfile`（app 根目錄）：combined lanes 以 `sh("cd ../ios && bundle exec fastlane ios …")` 轉呼叫 platform Fastfiles。root Taskfile 的 `fastlane:*` 已直接 `cd sales-order-app/ios|android && bundle exec fastlane`，**繞過此層** → 中間層純冗餘。
- `sales-order-app/fastlane/Appfile` 只設 `app_identifier`；platform 各有自己的 Appfile → 可整目錄刪除。`report.xml` 為執行產物，未入 `.gitignore`。
- App `Taskfile.yml`（435 行）：fastlane 區依賴 combined Fastfile；`build:dev:android` 與 `build:debug:android` 完全重複；大量 task 缺 `desc`；無分組。
- Backend `Taskfile.yml`（277 行 + 6 includes）：`default` 與 `list` 同為 `task -l`；有註解掉的 `goose:init` 死碼。
- Frontend `Taskfile.yml`（44 行）：全部缺 `desc`；`j2t` 內有註解死碼。
- Root `Taskfile.yml`（159 行）：結構尚可，`fastlane:*` 含版本號遞增編排（`-- auto` / `-- <number>`）。

## 設計

### 1. Platform Fastfiles 新增 lanes

現有 `beta` lanes 已是「建置 + 傳純 binary」，不需新增；真正缺口是 production 的 build-only 變體。

**`sales-order-app/ios/fastlane/Fastfile`**：

- `upload_build_production` — 建置 IPA（`sh_unbundled("cd .. && task build:release:ios CLI_ARGS='ipa'")`）後 `upload_to_app_store(ipa: latest_ipa, force: true, skip_metadata: true, skip_screenshots: true, precheck_include_in_app_purchases: false, api_key: app_store_connect_api_key)`。不產生截圖、不傳 metadata/截圖。

**`sales-order-app/android/fastlane/Fastfile`**：

- `upload_build_production` — 建置 AAB（`sh("cd .. && task build:release:android CLI_ARGS='appbundle'")`）後 `upload_to_play_store(track: 'production', aab: aab_path, json_key: play_store_json_key_path, release_status: 'draft', skip_upload_apk: true, skip_upload_metadata: true, skip_upload_changelogs: true, skip_upload_images: true, skip_upload_screenshots: true)`。不產生截圖、不傳 metadata/截圖。

皆複用既有 helper（`latest_ipa`、`aab_path`、`app_store_connect_api_key`、`play_store_json_key_path`），不新增抽象。

### 2. 刪除 app combined fastlane 層

- `git rm -r sales-order-app/fastlane/`（Fastfile、Appfile、README.md、report.xml）。
- `sales-order-app/.gitignore` 加入 `report.xml`（防平台層再產生入版控——先確認是否已有對應規則，無則加）。

### 3. Root `Taskfile.yml`

新增（沿用現有 `cd sales-order-app/<platform> && bundle exec fastlane <platform> <lane>` 風格；版本號遞增編排比照現有 `fastlane:beta|production`：`-- auto` 自增 patch+build、`-- <number>` patch+1 並設 build number）：

| Task | 行為 |
|---|---|
| `fastlane:upload_build:production:ios` | `fastlane ios upload_build_production` |
| `fastlane:upload_build:production:android` | `fastlane android upload_build_production` |
| `fastlane:upload_build:production` | 版本號遞增後依序呼叫上述兩者 |

Beta 不新增 root task：現有 `fastlane:beta` 已是「版本遞增 + 建置 + 傳純 binary」（platform `beta` lanes 本來就不含截圖/metadata），再加 `upload_build:beta` 會是完全重複。

其餘 tasks 維持；加分組註解（release orchestration / screenshots / version）。

### 4. App `Taskfile.yml` 整理

- 分組（註解標題）：`gen` / `build` / `screenshots` / `fastlane` / `firebase` / `crashlytics` / `test` / `install` / `misc`；每個 task 補 `desc`。
- **刪除**：`fastlane:beta`、`fastlane:production`、`fastlane:upload_metadata`、`fastlane:screenshots`、`fastlane:screenshots:ios`、`fastlane:screenshots:android`、`fastlane:init`、`fastlane:android:init`、`fastlane:ios:init`、`fastlane:android:supply:init`（combined Fastfile 已刪；`init` 類為一次性歷史任務）；`build:dev:android`（與 `build:debug:android` 重複）。
- **保留**：`fastlane:android` / `fastlane:ios` per-platform passthrough。
- 其餘 task 行為不變。

### 5. Backend `Taskfile.yml` 整理

- 分組 + 補齊 `desc`；刪除註解掉的 `goose:init` 區塊；刪除 `list`（與 `default` 重複）。
- **不改任何保留 task 的名稱與行為**（root Taskfile 與既有流程引用之）。

### 6. Frontend `Taskfile.yml` 整理

- 補齊 `desc`；刪 `j2t` 內兩行註解死碼；分組（`gen` / `build` / `deploy`）。
- 不改 task 名稱與行為。

### 7. 文件

- 更新 `docs/AGENTS/app.md`：Task 表移除已刪 task、fastlane 部署段落改述 root 編排（build+upload：`task fastlane:beta|production`；upload-only：`task fastlane:upload_build:beta|production`）。

## 驗證

- 各目錄執行 `task -l` 正常解析並列出（root、app、backend、frontend）。
- `cd sales-order-app/ios && bundle exec fastlane lanes` 與 `cd sales-order-app/android && bundle exec fastlane lanes` 顯示新 lanes 且無語法錯誤。
- 不實際上傳（需真實 artifact 與商店憑證；upload 行為沿用既有 lane 的相同 action 參數）。

## 非目標

- 不改 backend/frontend 任何保留 task 的名稱或行為。
- 不改 root `fastlane:beta|production` 的版本號遞增邏輯。
- 不新增 CI/CD；部署仍為本機 fastlane + Taskfile。
- 不處理 `latest_ipa` 以字典序取最後一個 IPA 的既有脆弱性（沿用現況）。
