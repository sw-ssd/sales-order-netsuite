## ADDED Requirements

### Requirement: 根 Taskfile.yml
根目錄存在 `Taskfile.yml` 作為 monorepo 編排入口，以 `includes` 方式匯入三個子專案的 Taskfile。

#### Scenario: 目錄結構
- **WHEN** 開發者檢視根目錄
- **THEN** `Taskfile.yml` 存在且包含 `includes:`
- **THEN** `includes` 區塊包含 `frontend`、`backend`、`app` 三個 entries
- **THEN** 每個 `includes` 指向對應子目錄的 `Taskfile.yml`
- **THEN** 每個 `includes` 設有 `dir:` 讓 task 在正確目錄執行

### Requirement: 頂層 Tasks
根 Taskfile 定義 `dev`、`build`、`test` 三個頂層 task。

#### Scenario: task dev
- **WHEN** 開發者在根目錄執行 `task dev`
- **THEN** 啟動後端服務（`task: backend:dev`）
- **THEN** 啟動前端 dev server（`task: frontend:dev`）
- **THEN** 不啟動 mobile app（開發者手動執行 `task: app:dev`）

#### Scenario: task build
- **WHEN** 開發者在根目錄執行 `task build`
- **THEN** 依序執行 `task: frontend:build`、`task: backend:build`、`task: app:build`

#### Scenario: task test
- **WHEN** 開發者在根目錄執行 `task test`
- **THEN** 依序執行 `task: frontend:test`、`task: backend:test`、`task: app:test`

### Requirement: 向後相容
不修改任何子專案的既有 Taskfile.yml。

#### Scenario: 直接呼叫子專案
- **WHEN** 開發者進入 `sales-order-frontend/` 執行 `task dev`
- **THEN** 行為與變更前完全一致
- **THEN** 根 Taskfile 的 `includes` 僅是 proxy，不影響子專案獨立運作

### Requirement: Git 忽略規則
根目錄 `.gitignore` 確保各子專案的 build output 不被意外追蹤。

#### Scenario: 根 .gitignore
- **WHEN** 檢查根目錄 `.gitignore`
- **THEN** 包含各子專案的 build output 目錄（前端 `dist/`、後端 `bin/`、App `build/`）
- **THEN** 包含各語言的 dependencies 目錄（前端 `node_modules/`）
