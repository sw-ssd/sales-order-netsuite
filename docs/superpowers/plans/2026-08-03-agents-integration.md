# AGENTS.md 整合 Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** 將三個 git submodule 內的 AGENTS.md 整合至 parent repo（重寫查證後放 `docs/AGENTS/`），刪除 submodule 內 AGENTS.md 並更新 parent gitlink。

**Architecture:** Parent `AGENTS.md` 保留 code-review-graph 內容並新增「Submodule 指引」索引，指向 `docs/AGENTS/{app,backend,frontend}.md`。三份文件以原 submodule AGENTS.md 為基礎重寫並查證。submodule 各本機 commit 刪除 AGENTS.md（不 push），parent 更新 gitlink 記錄。

**Tech Stack:** 文件整理任務 — git submodule 操作、Markdown。無程式碼建置。

**Spec:** `docs/superpowers/specs/2026-08-03-agents-integration-design.md`

## Global Constraints

- 文件語言：保留繁體中文（與 submodule 程式碼註解一致）。
- 內容以實際程式碼為準：版本號、指令、結構必須查證後更新，不照抄過時 claim。
- 修正過時路徑：`~/Documents/sales-order-netsuit/...` 一律改為相對路徑（如 `../sales-order-backend/`）。
- submodule commit 僅含 AGENTS.md 刪除，不得夾帶 dirty 檔案（app 的 `.vscode/sessions.json`、frontend 的 `.gitignore`）。
- **不 push** 任何 submodule 變更到 remote（`github.com/hexagon-maker/*`）。
- 三份文件檔頭註明：本文件為原 `<submodule>/AGENTS.md` 之整合版，以實際程式碼為準。
- 不處理 parent repo 其他未追蹤 agent 設定檔（`.cursorrules`、`.claude/` 等）。

---

### Task 1: 查證並建立 `docs/AGENTS/app.md`（Flutter App）

**Files:**
- Read: `sales-order-app/AGENTS.md`（來源內容）
- Read: `sales-order-app/.fvmrc`、`sales-order-app/pubspec.yaml`、`sales-order-app/Taskfile.yml`、`sales-order-app/integration_test/.maestro/`（查證）
- Create: `docs/AGENTS/app.md`

**Interfaces:**
- Consumes: 無（第一個文件任務）
- Produces: `docs/AGENTS/app.md` — Task 4 的索引指向它

- [ ] **Step 1: 查證 app 事實**

Run:
```bash
cd /Volumes/UTM2/Developer/sales-order-netsuite
cat sales-order-app/.fvmrc
grep -E '^(version:|environment:|  sdk:)' sales-order-app/pubspec.yaml
ls sales-order-app/Taskfile.yml
```

Expected: `.fvmrc` 為 `"flutter": "3.35.2"`；pubspec `version: 1.2.6+25`、`sdk: ">=3.9.0 <4.0.0"`；Taskfile.yml 存在。與原文一致則沿用；不一致以實際值更新。

- [ ] **Step 2: 寫 `docs/AGENTS/app.md`**

以 `sales-order-app/AGENTS.md` 全文為基礎，套用以下轉換後寫入 `docs/AGENTS/app.md`：

1. 檔頭新增（保留原標題後）：
   ```markdown
   > 本文件為原 `sales-order-app/AGENTS.md` 之整合版，已搬移至 parent repo。內容以實際程式碼為準。
   ```
2. 將所有 `~/Documents/sales-order-netsuit/sales-order-backend/` 改為 `../sales-order-backend/`（第 6 節測試前置條件處）。
3. 查證並更新（若 Step 1 有差異則修正；無差異保留原文）：
   - 第 1 節：Flutter `3.35.2`、Dart SDK、版本 `1.2.6+25`、`lib/` 檔案數（`glob sales-order-app/lib/**/*.dart` 數實際數量）。
   - 第 2 節：`lib/` 三層架構目錄樹與原文一致（對照實際目錄，刪除不存在的路徑、補上新增的）。
   - 第 4 節：Taskfile 任務清單對照 `sales-order-app/Taskfile.yml` 實際任務增刪。
   - 第 6 節：Maestro 流程檔名與 `integration_test/.maestro/` 實際內容核對。
   - 第 7 節：部署流程（fastlane lanes）對照 `sales-order-app/ios/fastlane/Fastfile` 與 `sales-order-app/android/fastlane/Fastfile`。
4. 更新文末「最後更新」行為「根據 2026-08-03 查證整理」。

- [ ] **Step 3: 驗證文件**

Run:
```bash
grep -n '~/Documents/sales-order-netsuit' docs/AGENTS/app.md
```
Expected: 無輸出（舊路徑已清除）。再以 read 讀 `docs/AGENTS/app.md:1-30` 確認檔頭與結構正常。

---

### Task 2: 查證並建立 `docs/AGENTS/backend.md`（Go Backend）

**Files:**
- Read: `sales-order-backend/AGENTS.md`（來源內容）
- Read: `sales-order-backend/go.mod`、`sales-order-backend/Taskfile.yml`、`sales-order-backend/internal/domain/`（查證）
- Create: `docs/AGENTS/backend.md`

**Interfaces:**
- Consumes: 無
- Produces: `docs/AGENTS/backend.md` — Task 4 的索引指向它

- [ ] **Step 1: 查證 backend 事實**

Run:
```bash
cd /Volumes/UTM2/Developer/sales-order-netsuite
head -8 sales-order-backend/go.mod
ls sales-order-backend/Taskfile.yml sales-order-backend/database/goose sales-order-backend/ent/schema
```

Expected: `go 1.25.0`；Taskfile.yml、`database/goose/`、`ent/schema/` 存在。與原文一致則沿用；不一致以實際值更新。

- [ ] **Step 2: 寫 `docs/AGENTS/backend.md`**

以 `sales-order-backend/AGENTS.md` 全文為基礎，套用以下轉換後寫入 `docs/AGENTS/backend.md`：

1. 檔頭新增：
   ```markdown
   > 本文件為原 `sales-order-backend/AGENTS.md` 之整合版，已搬移至 parent repo。內容以實際程式碼為準。
   ```
2. 查證並更新：
   - 第 1/2 節：Go 版本（Step 1 結果）、技術棧清單對照實際（`go.mod` require 區塊）。
   - 第 3 節：目錄樹對照 `sales-order-backend/` 實際結構（`cmd/`、`internal/domain/` 下實際網域清單）。
   - 第 4 節：環境變數前綴對照 `config/*.go` 檔名。
   - 第 5 節：Taskfile 任務對照 `sales-order-backend/Taskfile.yml` 實際任務。
   - 第 6 節：遷移機制（`database/goose/*.sql` 是否存在、`task goose:*` 任務）對照實際。
   - 第 7 節：認證/授權章節 — 確認 `third_party/authorization/` 存在。
   - 第 8 節：測試策略 — `internal/domain/*/*_test.go` 與 `*_mock.go` 是否存在（對照實際，更新「無測試」類 claim）。
   - 第 9 節：部署 — 對照 Dockerfile、docker-compose 檔名。
3. 檢查全文是否有指向 parent 路徑或本機絕對路徑的參照，有則改為相對/移除。

- [ ] **Step 3: 驗證文件**

Run:
```bash
ls docs/AGENTS/backend.md
```
Expected: 檔案存在。再以 read 讀 `docs/AGENTS/backend.md:1-20` 確認檔頭與結構正常。

---

### Task 3: 查證並建立 `docs/AGENTS/frontend.md`（SolidJS Frontend）

**Files:**
- Read: `sales-order-frontend/AGENTS.md`（來源內容）
- Read: `sales-order-frontend/package.json`、`sales-order-frontend/Taskfile.yml`、`sales-order-frontend/src/routes/`（查證）
- Create: `docs/AGENTS/frontend.md`

**Interfaces:**
- Consumes: 無
- Produces: `docs/AGENTS/frontend.md` — Task 4 的索引指向它

- [ ] **Step 1: 查證 frontend 事實**

Run:
```bash
cd /Volumes/UTM2/Developer/sales-order-netsuite
grep -E '"(version)"|"solid-js"|"typescript"|"vite"|"tailwindcss"|"@tanstack/solid-router"' sales-order-frontend/package.json
ls sales-order-frontend/Taskfile.yml sales-order-frontend/pnpm-lock.yaml
```

Expected: version `1.0.30`、solid-js `^1.9.14`、typescript `^5.9.3`、vite `^6.4.3`、tailwindcss `^3.4.19`、`@tanstack/solid-router` `^1.170.18`；Taskfile.yml 與 pnpm-lock.yaml 存在。與原文一致則沿用；不一致以實際值更新。

- [ ] **Step 2: 寫 `docs/AGENTS/frontend.md`**

以 `sales-order-frontend/AGENTS.md` 全文為基礎，套用以下轉換後寫入 `docs/AGENTS/frontend.md`：

1. 檔頭新增：
   ```markdown
   > 本文件為原 `sales-order-frontend/AGENTS.md` 之整合版，已搬移至 parent repo。內容以實際程式碼為準。
   ```
2. 查證並更新：
   - 技術堆疊表：版本號依 Step 1 結果。
   - 專案結構：`src/` 目錄樹對照實際（`src/routes/` 下路由群組、`src/lib/` 下領域清單）。
   - 建置指令：對照 `package.json` scripts 與 `Taskfile.yml` 實際任務。
   - 環境變數：對照 `src/env/index.ts` 實際變數清單。
   - 測試章節：確認「目前程式碼庫中沒有測試檔案」是否仍為真（`glob sales-order-frontend/src/**/*.test.*`），有則更新。
   - 部署：對照 `firebase.json` 與 `task deploy`。
3. 檢查全文是否有指向 parent 路徑或本機絕對路徑的參照，有則改為相對/移除。

- [ ] **Step 3: 驗證文件**

Run:
```bash
ls docs/AGENTS/frontend.md
```
Expected: 檔案存在。再以 read 讀 `docs/AGENTS/frontend.md:1-20` 確認檔頭與結構正常。

---

### Task 4: parent `AGENTS.md` 加索引並 commit

**Files:**
- Modify: `AGENTS.md`（檔尾追加）

**Interfaces:**
- Consumes: `docs/AGENTS/app.md`、`docs/AGENTS/backend.md`、`docs/AGENTS/frontend.md`（Tasks 1-3）
- Produces: 更新後的 parent `AGENTS.md`

- [ ] **Step 1: 追加索引章節**

在 `AGENTS.md` 檔尾追加（保留原有 code-review-graph 內容不動）：

```markdown
---

## Submodule 指引

本 repo 為 superproject，含三個 git submodule。submodule 內不再各自維護 AGENTS.md，指引統一由本 repo 提供。**在任一 submodule 目錄內工作前，先讀對應指引文件。**

| Submodule | 技術 | 指引文件 |
|---|---|---|
| `sales-order-app/` | Flutter / Dart 行動 App | `docs/AGENTS/app.md` |
| `sales-order-backend/` | Go REST API | `docs/AGENTS/backend.md` |
| `sales-order-frontend/` | SolidJS / TypeScript Web | `docs/AGENTS/frontend.md` |
```

- [ ] **Step 2: 驗證**

Run:
```bash
grep -n 'docs/AGENTS' AGENTS.md
```
Expected: 三行索引 + 說明文字存在。

- [ ] **Step 3: Commit（僅 parent 內文件變更）**

```bash
cd /Volumes/UTM2/Developer/sales-order-netsuite
git add AGENTS.md docs/AGENTS/
git commit -m "docs: consolidate submodule AGENTS guides into parent repo"
```
注意：此 commit 不得包含三個 submodule gitlink（`git add` 時勿加 `sales-order-*` 目錄）。

---

### Task 5: 刪除 submodule AGENTS.md 並更新 parent gitlink

**Files:**
- Delete: `sales-order-app/AGENTS.md`、`sales-order-backend/AGENTS.md`、`sales-order-frontend/AGENTS.md`（各自 repo 內）
- Modify: parent 三個 gitlink（`sales-order-app`、`sales-order-backend`、`sales-order-frontend`）

**Interfaces:**
- Consumes: Tasks 1-4（文件已入 parent，可安全刪除來源）
- Produces: submodule 本機 commit + parent gitlink commit

- [ ] **Step 1: 每個 submodule 刪除 AGENTS.md 並本機 commit**

對三個 submodule 各執行（以 app 為例，其餘同）：
```bash
cd /Volumes/UTM2/Developer/sales-order-netsuite/sales-order-app
git rm AGENTS.md
git commit -m "docs: remove AGENTS.md (moved to parent repo docs/AGENTS/)"
```
Commit 前用 `git status --short` 確認 staged 只有 `D AGENTS.md`（app 的 `.vscode/sessions.json`、frontend 的 `.gitignore` 等 dirty 檔案不得進 commit）。backend、frontend 相同步驟。

Expected: 三個 submodule 各產生一個僅含 AGENTS.md 刪除的 commit。

- [ ] **Step 2: parent 更新 gitlink 並 commit**

```bash
cd /Volumes/UTM2/Developer/sales-order-netsuite
git add sales-order-app sales-order-backend sales-order-frontend
git commit -m "chore: update submodule pointers (AGENTS.md removed from submodules)"
```
注意：parent 原本就有 `M sales-order-*`（gitlink 與 checkout 不一致）— 此 commit 記錄的是實際 checkout 的 commit（含 Step 1 的刪除 commit），屬預期行為。

---

### Task 6: 最終驗證

**Files:** 無（驗證任務）

- [ ] **Step 1: submodule 內已無 AGENTS.md**

Run:
```bash
cd /Volumes/UTM2/Developer/sales-order-netsuite
for s in sales-order-app sales-order-backend sales-order-frontend; do git -C $s ls-files | grep -c AGENTS.md; done
```
Expected: 三個 `0`。

- [ ] **Step 2: submodule 狀態與 gitlink 一致**

Run:
```bash
git submodule status
```
Expected: 三行皆無 `+`/`-` 前綴（index 記錄 = 實際 checkout commit）。

- [ ] **Step 3: parent 乾淨、無殘留舊路徑**

Run:
```bash
git status --short | grep -v '^??'
grep -rn '~/Documents/sales-order-netsuit' docs/AGENTS/ || true
```
Expected: 無未 commit 的追蹤檔變更（`??` 未追蹤檔可忽略）；舊路徑 grep 無輸出。

- [ ] **Step 4: 文件可讀**

以 read 讀 `docs/AGENTS/app.md`、`docs/AGENTS/backend.md`、`docs/AGENTS/frontend.md` 各開頭 20 行與 parent `AGENTS.md` 全文，確認索引表格與三份文件檔頭正常、無亂碼。

---

## Self-Review

- **Spec 覆蓋**：索引結構 → Task 4；docs/AGENTS/ 三份文件 → Tasks 1-3；submodule 本機 commit + gitlink → Task 5；驗證 → Task 6。無缺口。
- **Placeholder 掃描**：無 TBD/TODO；每個 doc 任務含具體查證指令與轉換清單。
- **一致性**：文件路徑 `docs/AGENTS/{app,backend,frontend}.md` 全篇一致；commit 訊息在 Task 4/5 明確；舊路徑取代規則統一為相對路徑。
