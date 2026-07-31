## Why

專案由三個獨立子專案（前端 SolidJS、後端 Go、App Flutter）組成，各有各自的 Taskfile.yml，但缺少根層級的統一編排。開發者需手動切換目錄執行指令，CI/CD 也無統一的觸發點。

本次變更在根目錄建立一層 Taskfile.yml，以 `includes` 方式整合三個子專案的既有 Taskfile，實現一鍵 dev/build/test。

## What Changes

- 建立根目錄 `Taskfile.yml` — 以 `includes` 匯入三個子專案的 Taskfile
- 定義頂層 tasks：`dev`、`build`、`test`
- 各子專案的 Taskfile 不須修改（保持向後相容）
- 更新根目錄 `.gitignore` 加入各子專案的 build output（如有缺失）
- 更新 `AGENTS.md` 或新增根層級說明文件

**無 Breaking Change** — 各子專案原本的 `task dev` / `task build` 仍可直接執行。

## Capabilities

### New Capabilities

- `monorepo-toolchain`: 根 Taskfile 編排層 — `includes` 三個子專案，定義跨語言 pipeline

### Modified Capabilities

無

## Impact

- 根目錄新增 `Taskfile.yml`
- 開發流程可選用根目錄 `task dev` 一鍵啟動，也可維持 `cd subproject && task dev`
- CI：單一 `task build` / `task test` 觸發所有專案
