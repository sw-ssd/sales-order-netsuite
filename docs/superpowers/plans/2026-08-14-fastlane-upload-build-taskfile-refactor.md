# Fastlane upload-build lanes + Taskfile 整理重構 Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** 新增 iOS/Android production「建置 + 上傳純 binary」流程（含版本號遞增編排），並將 superproject 四個 Taskfile 分組整理、刪除冗餘 fastlane 中間層。

**Architecture:** Fastlane 行為只存在 platform Fastfiles（`ios/`、`android/`）；跨平台編排只在 superproject root `Taskfile.yml`；app Taskfile 只留 per-platform passthrough。Spec：`docs/superpowers/specs/2026-08-14-fastlane-upload-build-taskfile-refactor-design.md`。

**Tech Stack:** fastlane (Ruby)、go-task (YAML)、Flutter/FVM。

## Global Constraints

- 三層 git repo：root superproject、`sales-order-app`、`sales-order-backend`、`sales-order-frontend` 各自為獨立 repo（submodule）；commit 要在**對應的 repo** 內執行（`cwd` 指定）。
- 不實際執行上傳（需真實 artifact 與商店憑證）。
- 保留 task / lane 名稱一律不改（僅新增與刪除列明者）。
- Fastfile 註解/UI 字串維持繁體中文。
- 版本號遞增編排僅存在 root Taskfile；platform lanes 不碰版本號。

---

### Task 1: iOS Fastfile 新增 `upload_build_production` lane

**Files:**
- Modify: `sales-order-app/ios/fastlane/Fastfile`（在 `lane :beta` 結束後、`lane :production` 之前插入）

**Interfaces:**
- Produces: lane `ios upload_build_production`（root Taskfile Task 4 以 `bundle exec fastlane ios upload_build_production` 呼叫）。

- [ ] **Step 1: 插入 lane**

在 `ios/fastlane/Fastfile` 的 `lane :beta do ... end` 區塊之後插入：

```ruby
  desc "Build and upload production binary only (no screenshots/metadata)"
  lane :upload_build_production do
    sh_unbundled("cd .. && task build:release:ios CLI_ARGS='ipa'")
    upload_to_app_store(
      ipa: latest_ipa,
      force: true,
      skip_metadata: true,
      skip_screenshots: true,
      precheck_include_in_app_purchases: false,
      api_key: app_store_connect_api_key
    )
  end
```

- [ ] **Step 2: 驗證語法與 lane 註冊**

```bash
cd sales-order-app/ios && ruby -c fastlane/Fastfile && bundle exec fastlane lanes
```

Expected: `Syntax OK`；lanes 清單出現 `ios upload_build_production`。

- [ ] **Step 3: Commit（sales-order-app repo）**

```bash
cd sales-order-app && git add ios/fastlane/Fastfile && git commit -m "feat(fastlane): add ios upload_build_production lane (binary only)"
```

---

### Task 2: Android Fastfile 新增 `upload_build_production` lane

**Files:**
- Modify: `sales-order-app/android/fastlane/Fastfile`（在 `lane :beta` 結束後、`lane :production` 之前插入）

**Interfaces:**
- Produces: lane `android upload_build_production`（root Taskfile Task 4 呼叫）。

- [ ] **Step 1: 插入 lane**

```ruby
  desc "Build and upload production binary only (no screenshots/metadata)"
  lane :upload_build_production do
    sh("cd .. && task build:release:android CLI_ARGS='appbundle'")
    upload_to_play_store(
      track: 'production',
      aab: aab_path,
      json_key: play_store_json_key_path,
      release_status: 'draft',
      skip_upload_apk: true,
      skip_upload_metadata: true,
      skip_upload_changelogs: true,
      skip_upload_images: true,
      skip_upload_screenshots: true,
    )
  end
```

- [ ] **Step 2: 驗證語法與 lane 註冊**

```bash
cd sales-order-app/android && ruby -c fastlane/Fastfile && bundle exec fastlane lanes
```

Expected: `Syntax OK`；lanes 清單出現 `android upload_build_production`。

- [ ] **Step 3: Commit（sales-order-app repo）**

```bash
cd sales-order-app && git add android/fastlane/Fastfile && git commit -m "feat(fastlane): add android upload_build_production lane (binary only)"
```

---

### Task 3: 刪除 app combined fastlane 層

**Files:**
- Delete: `sales-order-app/fastlane/`（整目錄：`Fastfile`、`Appfile`、`README.md`、`report.xml`）
- Modify: `sales-order-app/.gitignore`（若無 `report.xml` 規則則加入）

**Interfaces:**
- Consumes: 無。
- Produces: app 根目錄不再有 `fastlane/Fastfile`；Task 5 刪除依賴它的 Taskfile tasks。

- [ ] **Step 1: 確認 platform Appfile 自足**

```bash
ls sales-order-app/ios/fastlane/Appfile sales-order-app/android/fastlane/Appfile
```

Expected: 兩個檔案都存在（root Appfile 只有 `app_identifier`，平台層不依賴它——platform lanes 皆從 `ios/`、`android/` 目錄執行，讀各自的 Appfile）。

- [ ] **Step 2: 刪除目錄**

```bash
cd sales-order-app && git rm -r fastlane/
```

- [ ] **Step 3: .gitignore 加 report.xml**

檢查：

```bash
cd sales-order-app && grep -n "report.xml" .gitignore || echo "MISSING"
```

若輸出 `MISSING`，在 `.gitignore` 末尾加一行：

```gitignore
report.xml
```

- [ ] **Step 4: 驗證 fastlane 仍能從平台目錄運作**

```bash
cd sales-order-app/ios && bundle exec fastlane lanes
```

Expected: 正常列出 lanes（證明不需要 app 根目錄的 Appfile/Fastfile）。

- [ ] **Step 5: Commit（sales-order-app repo）**

```bash
cd sales-order-app && git add -A fastlane/ .gitignore && git commit -m "chore(fastlane): remove redundant combined Fastfile layer"
```

---

### Task 4: Root Taskfile 新增 `fastlane:upload_build:production:*`

**Files:**
- Modify: `Taskfile.yml`（root repo；在 `fastlane:production:` task 區塊之後插入，並為 fastlane 區段加分組註解）

**Interfaces:**
- Consumes: Task 1/2 的 `upload_build_production` lanes；既有 `build:version`、`build:version:patch-and-set` root tasks。

- [ ] **Step 1: 插入三個 tasks**

在 root `Taskfile.yml` 的 `fastlane:production:` 區塊（結束於 `task: fastlane:production:android`）之後插入：

```yaml
  fastlane:upload_build:production:ios:
    desc: 建置並上傳 iOS 正式版 build（純 binary，不含截圖/metadata）到 App Store
    cmds:
      - cd sales-order-app/ios && bundle exec fastlane ios upload_build_production

  fastlane:upload_build:production:android:
    desc: 建置並上傳 Android 正式版 build（純 binary，不含截圖/metadata）到 Play Store production track
    cmds:
      - cd sales-order-app/android && bundle exec fastlane android upload_build_production

  fastlane:upload_build:production:
    desc: 建置並上傳 iOS + Android 正式版 build（純 binary；-- auto 自增 patch+build；-- <number> patch+1 並設 build number）
    cmds:
      - |
        MODE="{{.CLI_ARGS}}"
        if [ "$MODE" = "auto" ]; then
          task build:version
        elif [ -n "$MODE" ]; then
          task build:version:patch-and-set -- "$MODE"
        fi
      - task: fastlane:upload_build:production:ios
      - task: fastlane:upload_build:production:android
```

並在現有 `# App Store / Play Store release orchestration` 註解下方、`fastlane:beta:ios:` 之前的區段邊界維持原樣（該分組註解已存在，不需新增）。

- [ ] **Step 2: 驗證解析**

```bash
task -l | grep upload_build
```

Expected: 列出 `fastlane:upload_build:production`、`fastlane:upload_build:production:ios`、`fastlane:upload_build:production:android` 三行。

- [ ] **Step 3: Commit（root repo）**

```bash
git add Taskfile.yml && git commit -m "feat(taskfile): add fastlane:upload_build:production tasks (binary only)"
```

---

### Task 5: App Taskfile 分組整理

**Files:**
- Modify: `sales-order-app/Taskfile.yml`

**Interfaces:**
- Consumes: Task 3（combined Fastfile 已刪）。
- Produces: 重整後的 app Taskfile；root Taskfile 以 `app:<task>` 引用的 tasks（`app:build`、`app:test`、`app:build:version`、`app:build:number`、`app:build:version:patch-and-set`、`app:screenshots:*`）名稱與行為完全不變。

- [ ] **Step 1: 刪除下列 tasks（整段移除）**

- `fastlane:screenshots`
- `fastlane:screenshots:ios`
- `fastlane:screenshots:android`
- `fastlane:beta`
- `fastlane:production`
- `fastlane:upload_metadata`
- `fastlane:android:init`
- `fastlane:ios:init`
- `fastlane:init`
- `fastlane:android:supply:init`
- `build:dev:android`（檔尾，與 `build:debug:android` 重複）

- [ ] **Step 2: 重新分組並補齊 desc**

將保留的 tasks 依下列順序重新排列，每組前加一行註解標題；cmds/vars/dir/env/interactive/requires 等內容**逐字保留**，僅移動位置並為缺 `desc` 的 task 補上下列文字：

```yaml
# ── 預設 ──
# default

# ── 程式碼產生 ──
# gen / gen:clean / gen:flutter / gen:icons / gen:all / splash / install:gen:flutter

# ── 建置 ──
# build / build:debug / build:version / build:number / build:version:patch-and-set
# build:release:ios / build:release:android / build:debug:ios / build:debug:android

# ── 執行與環境 ──
# dev / emulator / iospod / clean / clean:all

# ── 商店截圖 ──
# ensure:android:emulator / screenshots:android / screenshots:ios
# screenshots:inject:ios / screenshots:inject:android
# screenshots:export:android / screenshots:export:ios

# ── fastlane ──
# fastlane:android / fastlane:ios

# ── Firebase / Crashlytics ──
# firebase:flavor / firebase:reauth / install:flutterfire / crashlytics:mode / crashlytics:cat

# ── 測試與分析 ──
# test / test:unit / test:auth / analyze / test:maestro / test:maestro:record / test:adb:install

# ── 安裝工具 ──
# install:maestro / install:version:plus / install:bundler / install:xcodeproj

# ── 其他 ──
# kstore / deeplink_test / info
```

需補的 `desc`（task: 補上的文字）：

| task | desc |
|---|---|
| `default` | `執行程式碼產生（build_runner）` |
| `gen` | `執行 build_runner 產生程式碼` |
| `gen:clean` | `清除 build_runner 產出` |
| `gen:flutter` | `執行 flutter_gen（assets / colors / fonts）` |
| `gen:icons` | `產生 launcher icons` |
| `gen:all` | `build_runner + flutter_gen + launcher icons` |
| `splash` | `產生 native splash` |
| `build` | `完整 release 流程：clean → gen → 版本遞增 → 建置 prod AAB 與 IPA` |
| `build:debug` | `建置 dev debug APK 與 iOS` |
| `build:release:ios` | `建置 prod release iOS（-- 後為 flutter build 目標，如 ipa）` |
| `build:release:android` | `建置 prod release Android（-- 後為 flutter build 目標，如 appbundle）` |
| `build:debug:ios` | `建置 dev debug iOS` |
| `build:debug:android` | `建置 dev debug Android（-- 後為 flutter build 目標，如 apk）` |
| `dev` | `以 dev flavor debug 模式執行` |
| `emulator` | `啟動 Android 模擬器（Small_Phone）` |
| `iospod` | `重建 iOS Pods（刪 Pods、Podfile.lock、.symlinks 後 pod install）` |
| `clean` | `flutter clean + pub get` |
| `clean:all` | `深層清理（含 Pods、build、pub cache）` |
| `install:maestro` | `安裝 Maestro（整合測試工具）` |
| `install:version:plus` | `安裝 pub_version_plus` |
| `install:bundler` | `安裝 bundler 與 fastlane binstub` |
| `install:gen:flutter` | `安裝 flutter_gen（dart pub global）` |
| `install:flutterfire` | `安裝 flutterfire_cli` |
| `install:xcodeproj` | `安裝 xcodeproj gem` |
| `test:maestro` | `執行 Maestro 整合測試` |
| `test:maestro:record` | `以 Maestro 錄製整合測試` |
| `test:adb:install` | `adb 安裝 dev debug APK` |
| `fastlane:android` | `在 android/ 目錄執行 fastlane（-- 後為 lane 或 action）` |
| `crashlytics:cat` | `觀察 Android 上 Crashlytics log` |
| `firebase:reauth` | `Firebase 重新登入` |
| `deeplink_test` | `以 adb 觸發客戶 QR Code deep link 測試` |
| `info` | `顯示 Taskfile 目錄資訊（除錯用）` |
| `kstore` | `產生 Android release keystore` |

（`fastlane:ios`、`ensure:android:emulator`、`screenshots:*`、`crashlytics:mode`、`firebase:flavor`、`test*`、`analyze`、`build:version*` 已有 desc，保留原文。）

- [ ] **Step 3: 驗證解析與 root 引用**

```bash
cd sales-order-app && task -l
cd .. && task -l && task app:build:version -- dry-run 2>/dev/null || task -l | grep "app:build"
```

Expected: `task -l` 兩處皆正常列出、無 YAML 錯誤；root 可見 `app:build`、`app:test`、`app:build:version`、`app:screenshots:ios` 等。

- [ ] **Step 4: Commit（sales-order-app repo）**

```bash
cd sales-order-app && git add Taskfile.yml && git commit -m "refactor(taskfile): group tasks, add descs, drop duplicate fastlane lanes"
```

---

### Task 6: Backend Taskfile 分組整理

**Files:**
- Modify: `sales-order-backend/Taskfile.yml`

**Interfaces:**
- Produces: 重整後的 backend Taskfile；root 引用的 `backend:dev`、`backend:build`、`backend:test`、`backend:test:auth` 名稱與行為不變。

- [ ] **Step 1: 刪除兩處**

1. `list` task（與 `default` 同為 `task -l`）整段移除：

```yaml
  list:
    desc: Lists available commands
    cmds:
      - task -l
```

2. 註解掉的 `goose:init` 區塊（`# goose:init:` 到 `#     - task: seeder` 共 10 行）整段移除。

- [ ] **Step 2: 分組**

tasks 依下列順序排列，每組前加註解標題；內容逐字保留（所有 task 已有 desc，不補）：

```yaml
# ── 預設 ──
# default

# ── 開發與執行 ──
# run / dev / cloudsql:dev / routes / apitoken

# ── 建置與檢查 ──
# build / check / clean / tidy / vet / lint / vuln / fmt

# ── 測試 ──
# test / test:unit / test:integration / test:auth / test:verbose / test:coverage
# test:slow / test:e2e / test:e2e:down / test:clean / race

# ── 程式碼產生 ──
# generate / swagger / oapigen / typego

# ── 基礎設施與部署 ──
# docker_stop / docker_start / infra:start / infra:stop / infra:logs / infra:stop:volumes
# gcp:deploy / db:migrate
```

- [ ] **Step 3: 驗證**

```bash
cd sales-order-backend && task -l
cd .. && task -l | grep "backend:"
```

Expected: 正常列出、無 `list`；root 可見 `backend:dev`、`backend:build`、`backend:test`、`backend:test:auth`。

- [ ] **Step 4: Commit（sales-order-backend repo）**

```bash
cd sales-order-backend && git add Taskfile.yml && git commit -m "refactor(taskfile): group tasks, drop list alias and dead goose:init block"
```

---

### Task 7: Frontend Taskfile 分組整理

**Files:**
- Modify: `sales-order-frontend/Taskfile.yml`

**Interfaces:**
- Produces: 重整後的 frontend Taskfile；root 引用的 `frontend:dev`、`frontend:build`、`frontend:test` 名稱與行為不變（注意：目前無 `test` task，root `task test` 引用 `frontend:test`——見 Step 3 驗證；若不存在則屬既有問題，本計畫不新增）。

- [ ] **Step 1: 刪除 `j2t` 內兩行註解死碼**

移除：

```yaml
      # - json2ts --no-declareExternallyReferenced --no-bannerComment --no-additionalProperties --cwd=src -i 'schemas/' -o dist/
      # - json2ts --cwd=src -i 'schemas/*.json' -o dist/tmp/ && cat dist/tmp/*.schema.d.ts > dist/generated/schema.d.ts
```

- [ ] **Step 2: 分組並補 desc**

依下列順序排列，組前加註解標題；為每個 task 補上對應 `desc`：

```yaml
# ── 程式碼產生 ──
# ui:add        → desc: 以 solidui-cli 新增 UI 元件
# gen:golang    → desc: 由 webrpc schema 產生 Go server
# gen:typescript → desc: 由 webrpc schema 產生 TypeScript client
# gen:openapi   → desc: 由 webrpc schema 產生 OpenAPI 文件
# j2t           → desc: 由 JSON schema 產生 TypeScript 型別

# ── 開發與建置 ──
# dev           → desc: 啟動前端 dev server
# build         → desc: 建置前端 production bundle

# ── 部署 ──
# deploy        → desc: 版號遞增、建置並部署到 Firebase Hosting
# firebase:tools:update → desc: 更新 firebase-tools
```

- [ ] **Step 3: 驗證**

```bash
cd sales-order-frontend && task -l
cd .. && task -l | grep "frontend:"
```

Expected: 正常列出；root 可見 `frontend:dev`、`frontend:build`。

- [ ] **Step 4: Commit（sales-order-frontend repo）**

```bash
cd sales-order-frontend && git add Taskfile.yml && git commit -m "refactor(taskfile): group tasks and add descs"
```

---

### Task 8: 更新 docs/AGENTS/app.md

**Files:**
- Modify: `docs/AGENTS/app.md`（root repo；「Taskfile 常用任務」表與「7. 部署流程」段）

**Interfaces:**
- Consumes: Task 4/5 的最終 task 名稱。

- [ ] **Step 1: 更新 Task 表**

在 `docs/AGENTS/app.md` 的 Task 表中：
- 移除 `task fastlane:android -- <lane>` 與 `task fastlane:ios -- <lane>` 相關列若提及已刪 tasks；保留 passthrough 說明。
- 移除提及 `task upload:ios` 的內容（該 task 已不存在於 Taskfile）。

- [ ] **Step 2: 更新部署段落**

將「7. 部署流程」的 Android / iOS 小節改寫為：

```markdown
### Android / iOS

部署由 superproject root Taskfile 統一編排（於 repo root 執行）：

```bash
task fastlane:beta                    # 版本遞增 + 建置 + 上傳 Beta（純 binary）
task fastlane:production              # 版本遞增 + 截圖 + 建置 + 上傳正式版（含 metadata/截圖）
task fastlane:upload_build:production # 版本遞增 + 建置 + 上傳正式版（純 binary）
```

- 各平台 lane 定義於 `ios/fastlane/Fastfile`、`android/fastlane/Fastfile`；app 層不再維護 combined Fastfile。
- Android `beta` lane 上傳 Play Store beta track（draft）；`production` 另含截圖與 metadata。
- iOS `beta` lane 上傳 TestFlight；`production` 另含截圖與 metadata。
- 平台層亦可單獨執行：`task fastlane:android -- <lane>` / `task fastlane:ios -- <lane>`（於 sales-order-app/）。
```

（併入時保留該節關於簽章、Appfile、`key.properties` 的既有說明；僅替換 lane 敘述。）

- [ ] **Step 3: 更新文件版本戳**

將文件末尾「最後更新」日期改為 `2026-08-14`。

- [ ] **Step 4: Commit（root repo）**

```bash
git add docs/AGENTS/app.md && git commit -m "docs(app): update fastlane deployment tasks after refactor"
```

---

## Self-Review 結果

- **Spec 覆蓋**：§1 → Task 1/2；§2 → Task 3；§3 → Task 4（beta 變體已存在 `fastlane:beta`，不重複新增）；§4 → Task 5；§5 → Task 6；§6 → Task 7；§7 → Task 8。無缺口。
- **Placeholder 掃描**：無 TBD/TODO；所有新增程式碼完整給出。
- **名稱一致性**：lane 名 `upload_build_production` 在 Task 1/2/4 一致；root task 名 `fastlane:upload_build:production(:ios|:android)` 在 Task 4/8 一致；保留 task 名（`app:build`、`backend:dev`、`frontend:build` 等）與現況一致。
- **驗證限制**：`fastlane lanes` 與 `task -l` 只驗證語法/註冊；實際上傳行為沿用既有 lane 的相同 action 參數，不在本計畫執行。
