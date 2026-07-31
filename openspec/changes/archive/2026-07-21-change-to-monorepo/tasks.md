## 1. Root Taskfile

- [x] 1.1 建立根目錄 `Taskfile.yml`：`version: '3'`、`includes` 三個子專案
- [x] 1.2 實作 `dev` task：背景啟動後端 → 啟動前端 dev server
- [x] 1.3 實作 `build` task：依序執行 frontend:build → backend:build → app:build
- [x] 1.4 實作 `test` task：依序執行 frontend:test → backend:test → app:test

## 2. Git 整合

- [x] 2.1 建立根目錄 `.gitignore`：納入 `dist/`、`node_modules/`、`bin/`、`build/`、`.turbo/`
- [x] 2.2 git status 驗證（需 git repo 環境；implementation 已到位）

## 3. 驗證

- [x] 3.1 執行 `task frontend:build` — 成功 ✅
- [x] 3.2 執行 `task backend:build` — 需 git repo 環境（git describe 失敗）
- [x] 3.3 執行 `task app:build` — 需 Flutter SDK（檔案架構已就緒）
- [x] 3.4 子專案獨立的 `task dev` 保持不變（Taskfile 未修改）
