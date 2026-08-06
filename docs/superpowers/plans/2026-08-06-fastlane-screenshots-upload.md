# Fastlane 截圖與上傳統一化 Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** 將 sales-order-app 的截圖產出、截圖上傳與 APP binary 上傳統一到 fastlane，並以 root-level Fastfile 作為單一入口。

**Architecture：** 新增 `sales-order-app/fastlane/` 作為編排入口，Root Fastfile 透過 `sh` 委派到 `ios/fastlane/` 與 `android/fastlane/`。平台 Fastfile 負責實際的 Maestro 截圖、ButterKit 處理與 `upload_to_*` 上傳。Android 新增截圖整理腳本，將 Maestro 輸出對應到 Play Store metadata 結構。

**Tech Stack：** Fastlane (Ruby)、Flutter、Maestro、ButterKit (iOS)、Taskfile、shell。

## Global Constraints

- Root fastlane 入口位於 `sales-order-app/fastlane/`
- 平台 fastlane 分別位於 `sales-order-app/ios/fastlane/` 與 `sales-order-app/android/fastlane/`
- 截圖產出仍使用既有 Maestro flow：`integration_test/.maestro/screenshots_store/Flow.yaml`（Android）與 `screenshots_store_ios/Flow.yaml`（iOS）
- iOS 截圖成品目錄：`appimg/export_app_store/`
- Android 截圖成品暫存：`integration_test/.maestro/test_output_directory/screenshots/android/`，上傳前整理到 `android/fastlane/metadata/android/<locale>/images/phoneScreenshots/`
- 不引入 CI/GitHub Actions
- 維持現有憑證模式：iOS API key p8 放 `.private_keys/`，Android Play Store JSON key 透過 `PLAY_STORE_JSON_KEY` 環境變數

---

## File Structure

| File | Action | Responsibility |
|---|---|---|
| `sales-order-app/fastlane/Fastfile` | Create | Root entry lanes: `screenshots`, `screenshots_ios`, `screenshots_android`, `beta`, `production`, `upload_metadata` |
| `sales-order-app/fastlane/Appfile` | Create | Shared app/package identifiers |
| `sales-order-app/fastlane/README.md` | Create | Usage instructions for root fastlane |
| `sales-order-app/ios/fastlane/Fastfile` | Modify | Add `screenshots`, update `beta`/`production`, add `upload_metadata` |
| `sales-order-app/android/fastlane/Fastfile` | Modify | Add `screenshots`, update `beta`/`production`, add `upload_metadata` |
| `sales-order-app/scripts/arrange_android_screenshots.sh` | Create | Copy/rename Maestro Android screenshots into Play Store metadata structure |
| `sales-order-app/scripts/arrange_ios_screenshots.sh` | Create | Reorganize ButterKit iOS export into fastlane deliver locale subdirectories |
| `sales-order-app/Taskfile.yml` | Modify | Remove/deprecate `upload:ios`, add fastlane helper tasks |

---

### Task 1: Create Root Fastfile Entry Point

**Files:**
- Create: `sales-order-app/fastlane/Fastfile`
- Create: `sales-order-app/fastlane/Appfile`
- Create: `sales-order-app/fastlane/README.md`

**Interfaces:**
- Consumes: iOS/Android platform fastlane lanes via `sh`
- Produces: `screenshots`, `screenshots_ios`, `screenshots_android`, `beta`, `production`, `upload_metadata` root lanes

- [ ] **Step 1: Create root Appfile**

Create `sales-order-app/fastlane/Appfile`:

```ruby
# Root Appfile for sales-order-app
# Platform-specific credentials remain in ios/fastlane/Appfile and android/fastlane/Appfile

app_identifier("com.hexagonty.salesorder.app")
```

- [ ] **Step 2: Create root Fastfile with delegating lanes**

Create `sales-order-app/fastlane/Fastfile`:

```ruby
default_platform(:ios)

platform :ios do
  desc "Generate iOS and Android store screenshots"
  lane :screenshots do
    screenshots_ios
    screenshots_android
  end

  desc "Generate iOS store screenshots via Maestro + ButterKit"
  lane :screenshots_ios do
    sh("cd ../ios && bundle exec fastlane ios screenshots")
  end

  desc "Generate Android store screenshots via Maestro"
  lane :screenshots_android do
    sh("cd ../android && bundle exec fastlane android screenshots")
  end

  desc "Upload beta builds to TestFlight and Play Console"
  lane :beta do
    sh("cd ../ios && bundle exec fastlane ios beta")
    sh("cd ../android && bundle exec fastlane android beta")
  end

  desc "Upload production builds with screenshots and metadata"
  lane :production do
    sh("cd ../ios && bundle exec fastlane ios production")
    sh("cd ../android && bundle exec fastlane android production")
  end

  desc "Upload only metadata and screenshots"
  lane :upload_metadata do
    sh("cd ../ios && bundle exec fastlane ios upload_metadata")
    sh("cd ../android && bundle exec fastlane android upload_metadata")
  end
end
```

- [ ] **Step 3: Create README.md**

Create `sales-order-app/fastlane/README.md`:

```markdown
# sales-order-app fastlane

Root-level fastlane entry point.

## Requirements

- Ruby / bundler
- `bundle install` in `sales-order-app/`
- iOS: `.private_keys/` with App Store Connect API key p8 file
- Android: `PLAY_STORE_JSON_KEY` env var with Play Store service account JSON

## Common lanes

```bash
bundle exec fastlane screenshots        # iOS + Android screenshots
bundle exec fastlane screenshots_ios    # iOS only
bundle exec fastlane screenshots_android # Android only
bundle exec fastlane beta               # TestFlight + Play Console Beta
bundle exec fastlane production         # App Store + Play Store production
bundle exec fastlane upload_metadata    # Only metadata + screenshots
```
```

- [ ] **Step 4: Commit**

```bash
git add sales-order-app/fastlane/
git commit -m "feat(fastlane): add root fastlane entry point"
```

---

### Task 2: Create iOS Screenshot Arrangement Script and Extend iOS Fastfile

**Files:**
- Create: `sales-order-app/scripts/arrange_ios_screenshots.sh`
- Modify: `sales-order-app/ios/fastlane/Fastfile`

**Interfaces:**
- Consumes: `appimg/export_app_store/` flat screenshots (e.g. `zh-hant-1-foo.png`)
- Produces: `appimg/export_app_store/<locale>/` subdirectories for fastlane deliver

- [ ] **Step 1: Create iOS arrangement script**

Create `sales-order-app/scripts/arrange_ios_screenshots.sh`:

```bash
#!/bin/bash
set -euo pipefail

SOURCE_DIR="${1:-appimg/export_app_store}"

echo "Arranging iOS screenshots in $SOURCE_DIR"

if [ ! -d "$SOURCE_DIR" ]; then
  echo "ERROR: Source directory not found: $SOURCE_DIR"
  exit 1
fi

# Move locale-prefixed flat files into locale subdirectories
# e.g. zh-hant-1-foo.png -> zh-Hant/1_foo.png
for file in "$SOURCE_DIR"/*.png; do
  [ -e "$file" ] || continue
  basename=$(basename "$file")
  # Extract locale prefix before first hyphen
  locale=$(echo "$basename" | cut -d'-' -f1)
  rest=$(echo "$basename" | cut -d'-' -f2-)
  # Normalize locale for fastlane deliver: zh-hant -> zh-Hant, en-US stays en-US
  normalized="$locale"
  if [ "$locale" = "zh-hant" ]; then normalized="zh-Hant"; fi
  target_dir="$SOURCE_DIR/$normalized"
  mkdir -p "$target_dir"
  cp "$file" "$target_dir/$rest"
done

echo "Arranged screenshots:"
find "$SOURCE_DIR" -mindepth 2 -name '*.png' | sort
```

- [ ] **Step 2: Make executable**

```bash
chmod +x sales-order-app/scripts/arrange_ios_screenshots.sh
```

- [ ] **Step 3: Commit iOS arrangement script**

```bash
git add sales-order-app/scripts/arrange_ios_screenshots.sh
git commit -m "feat(fastlane): add iOS screenshot arrangement script"
```

- [ ] **Step 4: Add helper to resolve screenshot path**

At the top of `sales-order-app/ios/fastlane/Fastfile`, add:

```ruby
def screenshots_dir
  File.expand_path("../../../appimg/export_app_store", __dir__)
end

def latest_ipa
  ipas = Dir["../build/ios/ipa/*.ipa"]
  UI.user_error!("找不到 IPA：../build/ios/ipa/*.ipa") if ipas.empty?
  ipas.last
end
```

- [ ] **Step 5: Add iOS screenshots lane**

Add inside `platform :ios do`:

```ruby
  desc "Generate store screenshots using Maestro + ButterKit"
  lane :screenshots do
    # Run from ios/ directory so Taskfile relative paths resolve correctly
    sh("cd .. && task screenshots:ios")
    sh("cd .. && task screenshots:inject:ios")
    sh("cd .. && task screenshots:export:ios")
    sh("cd .. && ./scripts/arrange_ios_screenshots.sh")
    UI.success("iOS screenshots exported to #{screenshots_dir}")
  end
```

- [ ] **Step 6: Update iOS beta lane**

Replace existing `beta` lane with:

```ruby
  desc "Push a new beta build to TestFlight"
  lane :beta do
    sh("cd .. && task build:release:ios CLI_ARGS='ipa'")
    upload_to_testflight(
      ipa: latest_ipa,
      skip_waiting_for_build_processing: false,
      api_key_path: ENV["APP_STORE_CONNECT_API_KEY_PATH"] || "./.private_keys/AuthKey_G496UPT5WY.p8",
      api_key_id: ENV["APP_STORE_CONNECT_API_KEY_ID"] || "G496UPT5WY",
      api_key_issuer_id: ENV["APP_STORE_CONNECT_API_ISSUER_ID"] || "d5270d10-579c-4ac2-a52e-c46caccde830"
    )
  end
```

- [ ] **Step 7: Update iOS production lane**

Replace existing `production` lane with:

```ruby
  desc "Submit to App Store review with screenshots and metadata"
  lane :production do
    screenshots
    sh("cd .. && task build:release:ios CLI_ARGS='ipa'")
    upload_to_app_store(
      ipa: latest_ipa,
      force: true,
      skip_metadata: false,
      skip_screenshots: false,
      screenshots_path: screenshots_dir,
      api_key_path: ENV["APP_STORE_CONNECT_API_KEY_PATH"] || "./.private_keys/AuthKey_G496UPT5WY.p8",
      api_key_id: ENV["APP_STORE_CONNECT_API_KEY_ID"] || "G496UPT5WY",
      api_key_issuer_id: ENV["APP_STORE_CONNECT_API_ISSUER_ID"] || "d5270d10-579c-4ac2-a52e-c46caccde830"
    )
  end
```

- [ ] **Step 8: Add iOS upload_metadata lane**

Add inside `platform :ios do`:

```ruby
  desc "Upload App Store screenshots and metadata only"
  lane :upload_metadata do
    upload_to_app_store(
      force: true,
      skip_metadata: false,
      skip_screenshots: false,
      screenshots_path: screenshots_dir,
      skip_binary_upload: true,
      api_key_path: ENV["APP_STORE_CONNECT_API_KEY_PATH"] || "./.private_keys/AuthKey_G496UPT5WY.p8",
      api_key_id: ENV["APP_STORE_CONNECT_API_KEY_ID"] || "G496UPT5WY",
      api_key_issuer_id: ENV["APP_STORE_CONNECT_API_ISSUER_ID"] || "d5270d10-579c-4ac2-a52e-c46caccde830"
    )
  end
```

- [ ] **Step 9: Validate syntax**

```bash
cd sales-order-app/ios
bundle exec fastlane lanes
```

Expected: lanes list includes `screenshots`, `beta`, `production`, `upload_metadata`.

- [ ] **Step 10: Commit**

```bash
git add sales-order-app/ios/fastlane/Fastfile
git commit -m "feat(fastlane): extend iOS lanes for screenshots and upload"
```

---

### Task 3: Create Android Screenshot Arrangement Script

**Files:**
- Create: `sales-order-app/scripts/arrange_android_screenshots.sh`

**Interfaces:**
- Consumes: `integration_test/.maestro/test_output_directory/screenshots/android/*.png`
- Produces: `android/fastlane/metadata/android/<locale>/images/phoneScreenshots/*.png`

- [ ] **Step 1: Create arrangement script**

Create `sales-order-app/scripts/arrange_android_screenshots.sh`:

```bash
#!/bin/bash
set -euo pipefail

SOURCE_DIR="${1:-integration_test/.maestro/test_output_directory/screenshots/android}"
TARGET_DIR="${2:-android/fastlane/metadata/android}"
LOCALE="${3:-zh-TW}"
PHONE_DIR="$TARGET_DIR/$LOCALE/images/phoneScreenshots"

echo "Arranging Android screenshots from $SOURCE_DIR to $PHONE_DIR"

if [ ! -d "$SOURCE_DIR" ]; then
  echo "ERROR: Source directory not found: $SOURCE_DIR"
  exit 1
fi

mkdir -p "$PHONE_DIR"
rm -f "$PHONE_DIR"/*.png

# Copy and sort screenshots by numeric prefix if present
find "$SOURCE_DIR" -maxdepth 1 -name '*.png' -print0 | sort -z | \
  while IFS= read -r -d '' file; do
    basename=$(basename "$file")
    cp "$file" "$PHONE_DIR/$basename"
  done

echo "Copied $(find "$PHONE_DIR" -maxdepth 1 -name '*.png' | wc -l) screenshots"
```

- [ ] **Step 2: Make executable**

```bash
chmod +x sales-order-app/scripts/arrange_android_screenshots.sh
```

- [ ] **Step 3: Test script locally**

```bash
cd sales-order-app
task screenshots:android
./scripts/arrange_android_screenshots.sh
ls android/fastlane/metadata/android/zh-TW/images/phoneScreenshots/
```

Expected: PNG files copied.

- [ ] **Step 4: Commit**

```bash
git add sales-order-app/scripts/arrange_android_screenshots.sh
git commit -m "feat(fastlane): add Android screenshot arrangement script"
```

---

### Task 4: Extend Android Fastfile for Screenshots and Upload

**Files:**
- Modify: `sales-order-app/android/fastlane/Fastfile`

**Interfaces:**
- Consumes: Maestro Android screenshots, `../build/app/outputs/bundle/prodRelease/app-prod-release.aab`
- Produces: `screenshots`, `beta`, `production`, `upload_metadata` Android lanes

- [ ] **Step 1: Add helper for AAB path**

At the top of `sales-order-app/android/fastlane/Fastfile`, add:

```ruby
def aab_path
  "../build/app/outputs/bundle/prodRelease/app-prod-release.aab"
end
```

- [ ] **Step 2: Add Android screenshots lane**

Add inside `platform :android do`:

```ruby
  desc "Generate store screenshots using Maestro and arrange for Play Store"
  lane :screenshots do
    sh("cd .. && task screenshots:android")
    sh("cd .. && ./scripts/arrange_android_screenshots.sh")
    UI.success("Android screenshots arranged in fastlane/metadata/android/")
  end
```

- [ ] **Step 3: Update Android beta lane**

Replace existing `beta` lane with:

```ruby
  desc "Submit a new Beta Build and Upload to Play Store"
  lane :beta do
    sh("cd .. && task build:release:android CLI_ARGS='appbundle'")
    upload_to_play_store(
      track: 'beta',
      aab: aab_path,
      json_key: ENV['PLAY_STORE_JSON_KEY'],
      release_status: 'draft',
      skip_upload_apk: true,
      skip_upload_metadata: true,
      skip_upload_changelogs: true,
      skip_upload_images: true,
      skip_upload_screenshots: true,
    )
  end
```

- [ ] **Step 4: Update Android production lane**

Replace existing `production` lane with:

```ruby
  desc "Submit production build with screenshots and metadata"
  lane :production do
    screenshots
    sh("cd .. && task build:release:android CLI_ARGS='appbundle'")
    upload_to_play_store(
      track: 'production',
      aab: aab_path,
      json_key: ENV['PLAY_STORE_JSON_KEY'],
      release_status: 'draft',
      skip_upload_apk: true,
      skip_upload_metadata: false,
      skip_upload_changelogs: false,
      skip_upload_images: false,
      skip_upload_screenshots: false,
    )
  end
```

- [ ] **Step 5: Update Android upload_metadata lane**

Replace existing `metadata` lane (rename to `upload_metadata` for consistency):

```ruby
  desc "Upload Play Store metadata and screenshots only"
  lane :upload_metadata do
    upload_to_play_store(
      track: 'production',
      metadata_path: './fastlane/metadata/android',
      skip_upload_apk: true,
      skip_upload_aab: true,
      skip_upload_images: false,
      skip_upload_screenshots: false,
    )
  end
```

- [ ] **Step 6: Validate syntax**

```bash
cd sales-order-app/android
bundle exec fastlane lanes
```

Expected: lanes list includes `screenshots`, `beta`, `production`, `upload_metadata`.

- [ ] **Step 7: Commit**

```bash
git add sales-order-app/android/fastlane/Fastfile
git commit -m "feat(fastlane): extend Android lanes for screenshots and upload"
```

---

### Task 5: Update Taskfile and Remove Legacy Upload

**Files:**
- Modify: `sales-order-app/Taskfile.yml`

**Interfaces:**
- Consumes: Root fastlane lanes
- Produces: Convenience tasks `fastlane:screenshots`, `fastlane:beta`, `fastlane:production`, `fastlane:upload_metadata`

- [ ] **Step 1: Remove legacy upload:ios task**

Remove from `sales-order-app/Taskfile.yml`:

```yaml
  upload:ios:
    cmds:
      - xcrun altool --upload-app --type ios -f build/ios/ipa/*.ipa --p8-file-path ./.private_keys --apiKey G496UPT5WY --apiIssuer d5270d10-579c-4ac2-a52e-c46caccde830
```

- [ ] **Step 2: Add convenience fastlane tasks**

Add under existing `fastlane:ios` / `fastlane:android` tasks:

```yaml
  fastlane:screenshots:
    cmds:
      - '{{.FLB}} screenshots'

  fastlane:screenshots:ios:
    cmds:
      - '{{.FLB}} screenshots_ios'

  fastlane:screenshots:android:
    cmds:
      - '{{.FLB}} screenshots_android'

  fastlane:beta:
    cmds:
      - '{{.FLB}} beta'

  fastlane:production:
    cmds:
      - '{{.FLB}} production'

  fastlane:upload_metadata:
    cmds:
      - '{{.FLB}} upload_metadata'
```

- [ ] **Step 3: Verify Taskfile**

```bash
cd sales-order-app
task -l | grep fastlane
```

Expected: new tasks appear in list.

- [ ] **Step 4: Commit**

```bash
git add sales-order-app/Taskfile.yml
git commit -m "chore(taskfile): remove legacy xcrun altool and add fastlane convenience tasks"
```

---

### Task 6: End-to-End Validation

**Files:**
- No file changes

**Interfaces:**
- Consumes: All new lanes and scripts

- [ ] **Step 1: Validate root fastlane lanes list**

```bash
cd sales-order-app
bundle exec fastlane lanes
```

Expected: root lanes `screenshots`, `screenshots_ios`, `screenshots_android`, `beta`, `production`, `upload_metadata` listed.

- [ ] **Step 2: Run iOS screenshots lane (requires backend + simulator)**

```bash
cd sales-order-app
bundle exec fastlane screenshots_ios
```

Expected: `appimg/export_app_store/` updated with new screenshots.

- [ ] **Step 3: Run Android screenshots lane (requires backend + emulator)**

```bash
cd sales-order-app
bundle exec fastlane screenshots_android
```

Expected: `android/fastlane/metadata/android/zh-TW/images/phoneScreenshots/` populated.

- [ ] **Step 4: Build beta binaries without upload (dry build validation)**

For iOS:

```bash
cd sales-order-app
task build:release:ios CLI_ARGS='ipa'
```

For Android:

```bash
cd sales-order-app
task build:release:android CLI_ARGS='appbundle'
```

Expected: `../build/ios/ipa/*.ipa` and `../build/app/outputs/bundle/prodRelease/app-prod-release.aab` exist.

- [ ] **Step 5: Beta upload (real, with patch version bump)**

```bash
cd sales-order-app
task build:version CLI_ARGS='patch'
bundle exec fastlane beta
```

Expected: New build appears in App Store Connect TestFlight and Play Console Beta.

- [ ] **Step 6: Production metadata/screenshot upload (real)**

After beta success:

```bash
cd sales-order-app
bundle exec fastlane upload_metadata
```

Expected: App Store Connect and Play Console screenshots/metadata updated.

- [ ] **Step 7: Production binary + screenshots upload (real, final verification)**

```bash
cd sales-order-app
task build:version CLI_ARGS='patch'
bundle exec fastlane production
```

Expected: Production releases created in draft status with screenshots.

---

## Self-Review Checklist

- [ ] **Spec coverage:**
  - Root Fastfile entry point → Task 1
  - iOS screenshot arrangement + iOS upload lanes → Task 2
  - Android screenshot arrangement script → Task 3
  - Android upload lanes → Task 4
  - Taskfile cleanup → Task 5
  - End-to-end validation → Task 6
- [ ] **Placeholder scan:** No TBD, TODO, or vague instructions.
- [ ] **Type consistency:**
  - `upload_to_app_store` uses `skip_binary_upload: true`
  - `upload_to_play_store` uses `skip_upload_apk/aab/metadata/changelogs/images/screenshots`
  - API key env vars: `APP_STORE_CONNECT_API_KEY_PATH`, `APP_STORE_CONNECT_API_KEY_ID`, `APP_STORE_CONNECT_API_ISSUER_ID`
- [ ] **Path consistency:**
  - iOS screenshots resolved via `File.expand_path("../../../appimg/export_app_store", __dir__)`
  - Android metadata path `./fastlane/metadata/android` relative to `android/`
- [ ] **No CI scope creep:** No GitHub Actions tasks included.

---

## Execution Handoff

Plan complete and saved to `docs/superpowers/plans/2026-08-06-fastlane-screenshots-upload.md`.

**Two execution options:**

1. **Subagent-Driven (recommended)** - Dispatch a fresh subagent per task, review between tasks, fast iteration.
2. **Inline Execution** - Execute tasks in this session using executing-plans, batch execution with checkpoints.

Which approach?
