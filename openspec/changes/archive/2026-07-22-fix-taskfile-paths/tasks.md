## 1. App Taskfile — 替換硬編碼路徑

- [x] 1.1 新增 `PROJECT_ROOT` 變數：`PROJECT_ROOT: "{{.TASKFILE_DIR}}/.."` 於 app Taskfile vars 區段
- [x] 1.2 替換 `screenshots:android` 中 `~/Documents/sales-order-netsuit/sales-order-backend` 為 `{{.PROJECT_ROOT}}/sales-order-backend`
- [x] 1.3 替換 `screenshots:ios` 中相同硬編碼路徑
- [x] 1.4 替換 `screenshots:export:ios` 中 `~/Documents/sales-order-netsuit/appimg/...` 路徑為 `{{.PROJECT_ROOT}}/appimg/...`（第 197、198、199 行）
- [x] 1.5 替換 `screenshots:export:ios` 中 ButterKit `file://` URL 為動態建構（第 196 行）

- [x] 1.6 移除 backend Taskfile 的 `dotenv:` 區段（root Taskfile 已載入 `.env`）

## 2. 驗證

- [x] 2.1 確認 app Taskfile 無殘留 `~/Documents/sales-order-netsuit` 路徑
- [x] 2.2 執行 `task info` 確認路徑變數正確
- [x] 2.3 從 repo root 執行 `task app:info` 確認 included Taskfile 路徑解析正確
