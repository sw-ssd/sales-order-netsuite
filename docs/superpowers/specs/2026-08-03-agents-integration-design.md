# AGENTS.md 整合設計 — git submodule 指引併入 parent repo

日期：2026-08-03

## 背景與動機

Parent repo `sales-order-netsuite` 含三個 git submodule，每個 submodule 內各有一份獨立的 `AGENTS.md`（給 AI agent 的專案導覽）：

| Submodule | 技術 | 原 AGENTS.md | 內容 |
|---|---|---|---|
| `sales-order-app/` | Flutter / Dart 行動 App | `sales-order-app/AGENTS.md` | ~396 行，繁中 |
| `sales-order-backend/` | Go REST API | `sales-order-backend/AGENTS.md` | ~348 行，繁中 |
| `sales-order-frontend/` | SolidJS / TS Web | `sales-order-frontend/AGENTS.md` | ~150 行，繁中 |

Parent 自身的 `AGENTS.md` 僅含 code-review-graph MCP 工具說明，未涵蓋 submodule 指引。

目標：將三份 submodule 內的 AGENTS.md 整合至 parent repo，刪除 submodule 內的 AGENTS.md，使在 parent repo 工作的 agent 能從單一入口取得全部指引。

## 決策（已與使用者確認）

1. **結構**：parent `AGENTS.md` 作為索引，指向 repo 內個別文件（不併成單一巨型檔）。
2. **位置**：個別文件放 `docs/AGENTS/`：`app.md`、`backend.md`、`frontend.md`。
3. **submodule 處理**：三個 submodule 各本機 commit「刪除 AGENTS.md」，**不 push** 到 remote（`github.com/hexagon-maker/*`）；parent 更新 gitlink。
4. **內容**：搬移時**重寫並查證**（版本號、指令、目錄結構、測試/部署流程），修正過時參照；保留繁體中文。
5. **執行策略**：直接讀取三個 submodule 關鍵檔案查證後重寫（不委派 scout）。

## 設計

### 1. Parent `AGENTS.md`（索引）

保留現有 code-review-graph 章節，新增「Submodule 指引」章節：

- 表格列出三個 submodule：路徑、技術、對應指引文件。
- 規則：在任一 submodule 目錄內工作前，先讀 `docs/AGENTS/<name>.md`（submodule 內已無 AGENTS.md，統一由 parent 提供）。

### 2. `docs/AGENTS/app.md` / `backend.md` / `frontend.md`

以原文件為基礎重寫，並逐份查證更新：

- **app.md**（Flutter）：核對 `.fvmrc`（Flutter 版本）、`pubspec.yaml`（版本、Dart SDK）、`Taskfile.yml`、目錄結構、Maestro 測試流程、部署。
- **backend.md**（Go）：核對 `go.mod`（Go 版本）、`Taskfile.yml`、目錄結構、環境變數前綴、遷移機制、測試。
- **frontend.md**（SolidJS）：核對 `package.json`（版本）、`Taskfile.yml`、路由、環境變數、部署。
- 修正過時參照：例如 app 文件內 `~/Documents/sales-order-netsuit/sales-order-backend/` 改為相對路徑 `../sales-order-backend/`。
- 檔頭註明：本文件為原 `<submodule>/AGENTS.md` 之整合版，以實際程式碼為準。
- 保留繁體中文。

### 3. Submodule 刪除與 gitlink

- 每個 submodule：`git rm AGENTS.md` → 本機 commit，訊息 `docs: remove AGENTS.md (moved to parent repo docs/AGENTS/)`；commit 僅含此刪除，不夾帶其他 dirty 檔案。
- Parent：`git add` 三個 gitlink + `AGENTS.md` + `docs/AGENTS/*` → commit。

### 4. 風險與注意事項

- submodule 目前有 dirty 檔案（app 的 `.vscode/sessions.json`、frontend 的 `.gitignore`）：commit 時只 stage AGENTS.md 刪除。
- submodule checkout commit 與 parent 記錄的 gitlink 不一致（`+` 前綴）：parent commit 時記錄實際 checkout 的 commit。
- 不 push → remote clone 仍保有 submodule AGENTS.md；此為已知且已確認的取捨。
- 重寫時若發現原文 claim 與實際程式碼不符，以實際程式碼為準並更新。

## 驗證

1. 三個 submodule 內 `AGENTS.md` 已刪除，刪除已本機 commit。
2. Parent `git status` 顯示：`docs/AGENTS/*`、`AGENTS.md`、三個 gitlink 變更已 commit。
3. `git submodule status` 無 `+`/`-` 前綴（gitlink 與 checkout 一致，或記錄預期 commit）。
4. grep 確認 `docs/AGENTS/` 內無殘留舊路徑（`~/Documents/sales-order-netsuit`）或已更新。
5. 三份文件 Markdown 結構完整可讀。

## 非目標

- 不處理 parent repo 其他未追蹤的 agent 設定檔（`.cursorrules`、`.claude/` 等）。
- 不 push submodule 變更到 remote。
- 不翻譯文件語言。
