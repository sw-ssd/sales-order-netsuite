# 特耀訂出貨系統 — 本地開發總覽

> 這份文件是給本地開發、修改、新增功能、除錯時的初始參考。內容涵蓋 parent repo 與三個 submodule 的職責、技術棧、資料流、常用指令與常見地雷。

## 1. 專案定位

這是一套食品業銷售訂單 / ERP 風格的系統，主要能力：

- 訂單（Sales Order）與出貨調度（Dispatch）管理
- 客戶、商品、部門、業務代表資料維護
- 與 NetSuite 雙向同步（讀 SuiteQL、寫 Record API）
- 郵件通知、排程任務、SSE / WebSocket 即時看板
- 業務員與客戶使用的跨平台行動 App

## 2. 倉庫架構

```
/sales-order-netsuite          <- parent repo（僅放 orchestration）
├── .gitmodules
├── Taskfile.yml               <- 統一任務入口
├── sales-order-netsuite.code-workspace   <- VS Code 多根 workspace
├── openspec/                  <- OpenSpec 變更與規格
├── appimg/                    <- 商店圖片、宣傳素材
├── .omp/                      <- superpowers harness 設定
├── docs/superpowers/specs/    <- 設計文件
├── sales-order-backend/       <- Go 後端（submodule）
├── sales-order-frontend/      <- SolidJS 前端（submodule）
└── sales-order-app/           <- Flutter App（submodule）
```

Parent repo 不儲存 submodule 程式碼，只記錄 `.gitmodules` 與 gitlink。

## 3. 系統互動圖

```
┌─────────────────┐     REST / SSE / WS      ┌─────────────────────┐
│  Browser (Web)  │ ◀──────────────────────▶ │ sales-order-backend │
└─────────────────┘                          │   Go 1.25 / chi     │
                                             │   PostgreSQL        │
┌─────────────────┐     REST + Cookie        │   Redis (optional)  │
│  Flutter App    │ ◀──────────────────────▶ │   NetSuite client   │
└─────────────────┘                          └──────────┬──────────┘
                                                        │
                                                        ▼
                                                  ┌──────────┐
                                                  │ NetSuite │
                                                  └──────────┘
```

- Web 與 App 都透過 REST API 與後端溝通。
- 後端使用 session cookie + `X-Sowinsoft-Token` API token 兩種認證。
- 後端與 NetSuite 同步資料，資料庫以 PostgreSQL 為主。

## 4. 各 Submodule 一覽

### 4.1 sales-order-backend（Go）

| 項目 | 內容 |
|---|---|
| 語言 / 框架 | Go 1.25、go-chi/chi/v5、entgo.io/ent |
| 資料庫 | PostgreSQL（MySQL 保留彈性） |
| 遷移 | Goose + Atlas |
| Session | scs + 自訂 Postgres/Ent Store |
| 授權 | Casbin RBAC with domain（tenant） |
| 進入點 | `cmd/sw8/main.go` |
| 常用指令 | `task dev`、`task build`、`task infra:start`、`task test` |

**關鍵目錄：**

```
cmd/              # 可執行入口：sw8、migrate、seed、route、token
config/           # 環境變數結構（envconfig）
ent/              # Ent schema 與產生檔
internal/domain/  # 各網域 DDD 分層
third_party/      # 授權、資料庫、NetSuite client、Redis 等
scripts/          # k6 / locust 等負載測試
taskfiles/        # Task 子任務拆分
database/         # Goose 遷移檔
```

**新增後端功能的常見切入點：**

1. 在 `internal/domain/<plural>/` 建立：
   - `model.go` — request/response/filter DTO
   - `repository.go` — 資料存取
   - `usecase.go` — 業務邏輯
   - `handler.go` — HTTP handler
   - `register.go` — 路由註冊
   - `transformation.go` — DTO ↔ ent 轉換
2. 在 `ent/schema/` 新增 entity 後執行 `task ent:gen`。
3. 若需要 migration，使用 `task db:migrate -- <name>` 產生 Goose 遷移檔。
4. 用 `task routes` 檢查路由是否掛上。

**環境變數前綴：** `API_`、`DB_`、`CORS_`、`SESSION_`、`EMAIL_`、`REDIS_`、`NETSUITE_`、`OTEL_`、`OAUTH2_`。

### 4.2 sales-order-frontend（SolidJS）

| 項目 | 內容 |
|---|---|
| 框架 | SolidJS 1.9 + TypeScript 5.9 |
| 建置 | Vite 6 |
| 路由 | TanStack Solid Router（file-based） |
| 狀態 | TanStack Solid Query |
| 表單 | Modular Forms / TanStack Form |
| 驗證 | valibot |
| 樣式 | TailwindCSS 3.4 + Kobalte/Ark UI |
| 套件管理 | pnpm |
| 常用指令 | `pnpm install`、`pnpm run dev`、`pnpm run build`、`pnpm run test` |

**關鍵目錄：**

```
src/routes/        # 檔案式路由
src/pages/         # 頁面元件
src/components/ui/ # 基礎 UI 元件
src/lib/<domain>/  # 各領域 API + query options
src/models/        # TypeScript 型別
src/constant/      # API 路徑、CASL 規則、選單
```

**新增前端功能的常見切入點：**

1. 在 `src/pages/admin/<feature>/` 建立頁面。
2. 在 `src/routes/admin/` 建立路由檔案（`*.tsx` + `*.lazy.tsx`）。
3. 在 `src/lib/<feature>/` 新增 API builder 與 `queryOptions`。
4. 在 `src/constant/sidemenu.ts` 新增側邊欄選單。

**環境變數：** 瀏覽器可用變數需以 `VITE_` 開頭，例如 `VITE_API_URL`、`VITE_AUTH_SECRET`。`.env` 與 `.env.production` 已從 frontend submodule 的版控中移除，目前為本地未追蹤檔案。

### 4.3 sales-order-app（Flutter）

| 項目 | 內容 |
|---|---|
| 框架 | Flutter 3.35.2（FVM 鎖定） |
| 語言 | Dart >=3.9.0 |
| 路由 | auto_route |
| 狀態 | flutter_solidart Signal + 自訂 RefreshableResource |
| 網路 | Dio + CookieManager + AuthInterceptor |
| 表單 | reactive_forms |
| 本地儲存 | Sembast |
| Flavor | dev / prod |
| 常用指令 | `fvm flutter pub get`、`task gen`、`task build` |

**關鍵目錄：**

```
lib/main.dart                  # 共用初始化
lib/main_dev.dart              # dev flavor 進入點
lib/main_prod.dart             # prod flavor 進入點
lib/layer_business/            # 業務邏輯、路由、API、services
lib/layer_data/                # 模型、repository、constants、enums
lib/layer_presentation/        # widgets、stories（畫面）
android/keystore/              # release keystore（未入版控）
integration_test/.maestro/     # Maestro E2E 測試
```

**新增 App 功能的常見切入點：**

1. 在 `lib/layer_data/models/<domain>/` 建立 Freezed 模型。
2. 在 `lib/layer_business/network/api/` 新增 API 類別與抽象介面。
3. 在 `lib/layer_business/services/<domain>/` 新增 provider。
4. 在 `lib/layer_presentation/stories/<story>/<screen>.dart` 建立畫面。
5. 在 `lib/layer_business/router/routes.dart` 註冊路由。

**環境：** `DevEnv` / `ProdEnv` 透過 envied 產生；`.dev.env`、`.prod.env` 為本地未入版控檔案。`dev.env.hexagon` 與 `prod.env.hexagon` 是入版控的範例/備份，注意敏感資訊。

## 5. 本地開發工作流

### 5.1 初次 Clone

```bash
git clone --recurse-submodules <parent-repo-url>
cd sales-order-netsuite
```

若已 clone 但 submodule 未初始化：

```bash
git submodule update --init --recursive
```

### 5.2 啟動後端

```bash
cd sales-order-backend
task infra:start   # Postgres + Valkey + Mailpit
task dev           # air hot reload
```

常用：

```bash
task routes        # 列出所有路由
task apitoken      # 產生 API JWT token
task seeder -- up ns
task test
```

### 5.3 啟動前端

```bash
cd sales-order-frontend
pnpm install
pnpm run dev        # http://localhost:3000
```

### 5.4 啟動 App

```bash
cd sales-order-app
fvm flutter pub get
fvm flutter run --flavor dev --target lib/main_dev.dart
```

### 5.5 從 Parent 統一執行

```bash
task -l              # 列出所有任務
task dev             # 後端 + 前端一起啟動
task build           # 依序建置 frontend → backend → app
task test            # 執行所有子專案測試
```

## 6. 環境與憑證管理

**轉換成 submodule 後，以下檔案為本地未入版控，需自行準備或從安全備份還原：**

| 檔案 | 說明 |
|---|---|
| `sales-order-backend/.env` | 後端環境變數 |
| `sales-order-backend/cmd/sw8/.env` | 後端 sw8 指令環境變數 |
| `sales-order-frontend/.env` | 前端開發環境變數 |
| `sales-order-frontend/.env.production` | 前端正式環境變數 |
| `sales-order-app/android/keystore/hexagon-salesorder-keystore.jks` | Android release keystore |
| `sales-order-app/.dev.env` / `.prod.env` | App 環境變數 |

**注意事項：**

- `sales-order-backend/hexagon.env` 目前仍被追蹤，內含範例/真實憑證，請避免寫入生產密鑰。
- `sales-order-frontend/.env` 與 `.env.production` 已從版控移除並加入 `.gitignore`，但舊 commit 中仍可看到歷史內容；若曾含真實憑證，需評估是否重寫 history。
- 請勿將 `token.export`、API key、keystore 密碼等提交到任何 submodule。

## 7. OpenSpec / Superpowers 慣例

- 專案使用 `openspec` 作為變更管理流程：`openspec/config.yaml` 設定為 `schema: spec-driven`。
- 文件預設使用繁體中文；程式碼、變數名稱、API 介面維持英文。
- 常用指令在 `.omp/commands/opsx-*.md`。
- 已完成的 change：`fix-dispatch-sub-board-sent-filter`。
- 新增功能建議走 OpenSpec：`openspec new change` → 產生 proposal → design → tasks → spec。

## 8. 除錯與常見地雷

### 後端

- `task routes` 可快速檢查路由是否註冊成功。
- `task apitoken` 產生無到期日的 JWT，供中台或 App 使用。
- `config.New(filenames...)` 會用 `godotenv.Overload` 載入 env 檔，再用 `envconfig` 讀環境變數。
- NetSuite client 目前 `InsecureSkipVerify: true`，生產環境應移除。

### 前端

- 路由使用 TanStack file-based routing；新增/刪除路由後 Vite plugin 會自動產生 `routeTree.gen.ts`。
- `pnpm-lock.yaml` 雖在 `.gitignore` 中，但因早期已追蹤，目前仍由 git 管理。
- 部署到 Firebase Hosting 時，`/api/**` 會 rewrite 到 Cloud Run 後端。

### App

- Flutter 版本由 `.fvmrc` 鎖定，建議使用 FVM：`fvm flutter ...`。
- `lib/env/*.g.dart` 等產生檔已入版控；修改來源後要重新跑 `task gen`。
- iOS / Android 建置 flavor 分 dev / prod；Firebase 設定透過 `firebase_flavor.sh` 重新產生。
- Maestro 測試流程在 `integration_test/.maestro/`，截圖流程依賴後端已啟動。

### Parent repo

- `git submodule status` 可查看三個 submodule 目前指向的 commit。
- 更新 submodule 到最新 master：`git submodule update --remote`。
- 設定 `submodule.recurse=true` 與 `push.recurseSubmodules=check` 已加入 local git config。

## 9. 修改功能時的快速對照

| 想做的事 | 從哪裡開始 |
|---|---|
| 新增後端 API | `internal/domain/<plural>/` + `ent/schema/` |
| 新增前端頁面 | `src/pages/admin/<feature>/` + `src/routes/admin/` |
| 新增前端 API | `src/lib/<feature>/` |
| 新增 App 畫面 | `lib/layer_presentation/stories/<story>/` |
| 新增 App API | `lib/layer_business/network/api/` |
| 新增選單項目 | `src/constant/sidemenu.ts`（前端） |
| 調整 NetSuite 同步 | `third_party/netsuite/`（後端） |
| 調整權限規則 | `third_party/authorization/`（後端） + `src/constant/casl.ts`（前端） |

---

**最後更新：** 2026-07-31（submodule 轉換完成後）
