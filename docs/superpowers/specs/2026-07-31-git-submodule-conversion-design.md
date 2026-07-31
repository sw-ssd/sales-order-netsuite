# Git Submodule 轉換設計

- **日期**: 2026-07-31
- **主題**: 將 sales-order-backend、sales-order-frontend、sales-order-app 改為 Git submodule
- **狀態**: 已核准，待實作

## 1. 背景與目標

目前 `/Volumes/UTM2/Developer/sales-order-netsuite` 的根目錄不是 git repo，但底下三個子目錄已經是各自獨立的 git repo，且都有對應的遠端：

| 子專案 | 遠端 URL | 目前分支 |
|---|---|---|
| sales-order-backend | `https://github.com/hexagon-maker/sales-order-backend.git` | `master` |
| sales-order-frontend | `https://github.com/hexagon-maker/sales-order-frontend.git` | `master` |
| sales-order-app | `https://github.com/hexagon-maker/sales-order-app.git` | `master` |

本設計要將根目錄變成父倉庫（parent repo），並把這三個子目錄正式登錄為 git submodule，讓未來可以用 `git submodule update --remote` 統一更新，並在父倉庫層級保留共用設定與 orchestration。

## 2. 範圍與非範圍

**範圍內：**
- 初始化根目錄為新的 git repo。
- 把三個子目錄轉換成 submodule，追蹤各自遠端的 `master` 分支。
- 保留根目錄現有檔案：`.gitignore`、`Taskfile.yml`、`openspec/`、`appimg/`、`.omp/` 等。
- 備份並復原本地未入版控檔案（`.env`、keystore 等）。

**範圍外：**
- 不修改三個遠端 repo 的內容。
- 不重新命名 submodule 路徑。
- 不為父倉庫設定遠端 origin（本次僅在本地建立）。
- 不對 CI/CD、部署腳本或程式碼邏輯做額外調整。

## 3. 最終狀態

轉換完成後，根目錄的結構如下（示意）：

```text
sales-order-netsuite/          <- parent repo
├── .git/
├── .gitmodules
├── .gitignore
├── Taskfile.yml
├── openspec/
├── appimg/
├── .omp/
├── docs/
│   └── superpowers/
│       └── specs/
│           └── 2026-07-31-git-submodule-conversion-design.md
├── sales-order-backend/       <- submodule (gitlink)
├── sales-order-frontend/      <- submodule (gitlink)
└── sales-order-app/           <- submodule (gitlink)
```

父倉庫不儲存 submodule 的原始碼內容，只記錄：
- `.gitmodules` 的 submodule 名稱、路徑、URL、branch
- 各 submodule 目錄在 index 中的 gitlink（commit hash）

## 4. 選擇的方案

評估過三種做法：

1. **原地轉換（本設計採用）**：備份子目錄、刪除後用 `git submodule add -b master` 重新 clone。步驟標準、風險可控。
2. **吸收現有 `.git`**：手動把既有的 nested git repo 轉為 submodule，不用重新 clone。步驟複雜、容易遺漏 gitlink 或 `.gitmodules` 設定。
3. **全新 parent repo 再置換**：在別處建立 parent repo，完成後覆蓋原目錄。最安全但需要額外空間與搬移步驟。

選擇方案 1，因為三個子專案的遠端內容與本地一致，重新 clone 的代價小，且最符合 git submodule 的標準用法。

## 5. 轉換步驟

1. **備份子目錄**
   - 把 `sales-order-backend`、`sales-order-frontend`、`sales-order-app` 整個搬到父目錄外的暫存備份區，例如：
     `../sales-order-netsuite-backup-2026-07-31/`。
   - 搬移（move）而非複製，避免重複佔用大量空間（特別是 frontend 的 `node_modules` 與 app 的 `build/`）。

2. **確認父倉庫已初始化**
   - 為了提交本設計文件，根目錄已經先執行過 `git init` 並把 spec 加入第一個 commit。
   - 若實作時發現尚未初始化，再補執行 `git init`。
   - 實際轉換 submodule 時，再把 `.gitmodules`、submodule gitlink 與其他根目錄 orchestration 檔案一起提交。

3. **新增 Submodule**
   - 依序執行：
     ```bash
     git submodule add -b master https://github.com/hexagon-maker/sales-order-backend.git sales-order-backend
     git submodule add -b master https://github.com/hexagon-maker/sales-order-frontend.git sales-order-frontend
     git submodule add -b master https://github.com/hexagon-maker/sales-order-app.git sales-order-app
     ```
   - `git submodule add` 會自動建立 `.gitmodules` 並把 submodule HEAD 登錄為 gitlink。

4. **復原本地未入版控檔案**
   - 從備份區把下列檔案複製回對應位置：
     - `sales-order-backend/.env`
     - `sales-order-backend/cmd/sw8/.env`
     - `sales-order-frontend/.env`
     - `sales-order-frontend/.env.production`
     - `sales-order-app/android/keystore/hexagon-salesorder-keystore.jks`
   - 若還有其他未入版控檔案需要保留，可在此時從備份區手動補回。

5. **提交父倉庫**
   - `git add .gitmodules` 與三個 submodule 路徑。
   - 提交，訊息例如：`chore: convert backend, frontend and app to git submodules`

6. **清理備份**
   - 確認 `git status` 乾淨、submodule 狀態正確、本地檔案已復原後，刪除暫存備份區。

## 6. 本地未入版控檔案處理

以下檔案目前在各自的 `.gitignore` 中，不會出現在遠端，必須在轉換前後特別處理：

| 檔案路徑 | 說明 |
|---|---|
| `sales-order-backend/.env` | 後端環境變數 |
| `sales-order-backend/cmd/sw8/.env` | 後端 sw8 指令環境變數 |
| `sales-order-frontend/.env` | 前端開發環境變數 |
| `sales-order-frontend/.env.production` | 前端正式環境變數 |
| `sales-order-app/android/keystore/hexagon-salesorder-keystore.jks` | Android release keystore |

處理原則：
- 備份時整個子目錄搬走，所以這些檔案自然保留。
- 復原時只把需要的檔案貼回 submodule 路徑；其餘 build artifact（`node_modules/`、`build/`、`.dart_tool/` 等）由 submodule 重新產生，不復原。

## 7. 錯誤處理與復原

- **隨時可回滾**：在刪除備份區之前，只要將三個子目錄從備份區搬回原路徑，即可回到轉換前狀態。
- **clone 失敗**：若 `git submodule add` 因網路或權限失敗，保留備份區、刪除已產生的半成品目錄，再搬回備份即可。
- **父倉庫 commit 錯誤**：若父倉庫 commit 有誤，在尚未推送遠端前可用 `git reset --soft` 或 `git reset --hard` 修正；父倉庫本次無遠端，風險更低。
- **不動遠端**：整個流程只對本地與 submodule 的遠端做 read-only clone，不會推送任何東西到三個子專案的 repo。

## 8. 驗證

轉換後必須確認：

- `git submodule status` 顯示三個 submodule，且各自有對應的 commit hash。
- 每個 submodule 內執行 `git status --short` 為乾淨（復原的未入版控檔案會顯示 untracked，屬預期）。
- 根目錄 `git status` 沒有未預期的未追蹤檔案。
- `Taskfile.yml` 的 include 路徑仍然有效（可執行 `task -l` 檢查是否能列出任務）。
- 復原的本地檔案存在於正確位置：
  - `sales-order-backend/.env`
  - `sales-order-frontend/.env`
  - `sales-order-app/android/keystore/hexagon-salesorder-keystore.jks`

## 9. 團隊工作流

未來其他成員 clone 時：

```bash
git clone --recurse-submodules <parent-repo-url>
```

若已經 clone 但沒有 submodule：

```bash
git submodule update --init --recursive
```

更新 submodule 到最新 `master`：

```bash
git submodule update --remote
```

## 10. 決策紀錄

| 決策 | 選項 | 理由 |
|---|---|---|
| 轉換方式 | 原地轉換 | 步驟標準、風險由完整備份控制 |
| 追蹤分支 | `master` | 與現有子專案分支一致，且用戶選擇追蹤分支 |
| 父倉庫遠端 | 暫不設定 | 用戶表示本地 parent repo 即可 |
| 本地未入版控檔案 | 全部保留並復原 | 用戶選擇全部保留 |
| 是否保留根目錄 orchestration | 保留 | 用戶選擇保留 Taskfile、openspec、appimg 等 |

---

**下一步**：實作本設計前，請先確認此 spec 無誤，再由 AI 產生詳細實作計畫。