# Hexagon Food Web Application — Agent 指南

> 本文件為原 `sales-order-frontend/AGENTS.md` 之整合版，已搬移至 parent repo。內容以實際程式碼為準。

本指南提供給在 `hexagon-food-webapp` 專案上工作的 AI 程式開發 agent。閱讀本指南不需要事先了解此程式碼庫。

## 專案概覽

這是一個食品業銷售訂單 / ERP 風格系統的**單頁應用程式（SPA）**前端。它以 **SolidJS 1.9** 與 **TypeScript 5.9** 建構，使用 **Vite 6** 打包，並以 **TailwindCSS 3.4** 進行樣式設計。應用程式透過 REST API 與後端通訊（後端為以 Go 語言撰寫的 `hexagon-backend` 服務），並部署至 **Firebase Hosting**。

程式碼庫混合使用英文（識別字與大部分註解的主要語言）以及繁體中文 UI 標籤和部分行內註解。

## 技術堆疊

| 層級 | 技術 |
|------|------|
| 框架 | [SolidJS](https://www.solidjs.com/) 1.9 |
| 語言 | TypeScript 5.9（ES 模組，`type: "module"`） |
| 建置工具 | Vite 6 |
| 路由 | [TanStack Solid Router](https://tanstack.com/router/latest) 1.170（檔案式路由） |
| 狀態 / 伺服器快取 | [TanStack Solid Query](https://tanstack.com/query/latest) 5.100 |
| 表單 | `@modular-forms/solid`、`@tanstack/solid-form` |
| 驗證 | `valibot` 1.0.0-beta.9 |
| 樣式 | TailwindCSS 3.4 + `tailwindcss-animate` |
| UI 基礎元件 | `@kobalte/core`、`@ark-ui/solid`、`@corvu/*` |
| 圖示 | `solid-icons` |
| 圖表 | `chart.js` |
| 日期 / 時間 | `date-fns`、`@internationalized/date` |
| 拖曳 | `@atlaskit/pragmatic-drag-and-drop` |
| 認證加密 | `@node-rs/argon2`、`@oslojs/crypto`、`@oslojs/encoding` |
| 權限檢查 | `@casl/ability` |
| 通知 | `solid-sonner` |
| 動畫 | `animejs`、`solid-motionone` |
| QR Code | `qrcode-with-logos` |
| HTTP 客戶端 | 原生 `fetch`（於 `src/lib/requests` 提供輕量包裝） |
| 託管 / 部署 | Firebase Hosting（`firebase.json`） |
| 套件管理器 | pnpm（lockfile：`pnpm-lock.yaml`） |
| 任務執行器 | [Task](https://taskfile.dev/)（`Taskfile.yml`） |

## 專案結構

```
.
├── public/                    # 靜態資源，會原封不動複製
├── src/
│   ├── @types/               # 自訂 TypeScript 型別宣告（Google 認證、TanStack table 等）
│   ├── assets/               # 圖片與 .well-known app-link 檔案
│   ├── components/           # 可重複使用的 UI 與功能元件
│   │   ├── ui/              # 基礎元件（Button、Dialog、Table 等）
│   │   ├── form/            # 表單欄位包裝器
│   │   ├── datatable/       # TanStack table 包裝器與篩選器
│   │   ├── sidebar/         # 應用程式外殼側邊欄 / 麵包屑
│   │   └── ...
│   ├── constant/             # 常數、路由路徑、API 端點、CASL 規則、選單
│   ├── contexts/             # SolidJS context 提供者（Google OAuth 等）
│   ├── env/                  # 環境變數綱目與驗證
│   ├── hooks/                # SolidJS 自訂 hooks
│   ├── lib/                  # 領域專屬資料 / API 邏輯
│   │   ├── requests/        # HTTP 請求建構器與 fetch 包裝器（含 CSRF token 管理）
│   │   ├── auth/            # 認證查詢與輔助函式
│   │   ├── sales-order/     # 銷售訂單 API + TanStack 查詢
│   │   ├── customer/        # 客戶 API + 查詢
│   │   ├── item/            # 品項 API + 查詢
│   │   ├── metadict/        # NetSuite metadict / 清單值
│   │   └── ...（department、estimate-item、salesrep、tenant、user、role、article）
│   ├── models/               # TypeScript 介面 / 型別
│   │   ├── base.ts          # 共用基礎型別（篩選、分頁等）
│   │   ├── auth.ts          # 認證相關型別
│   │   ├── policy.ts        # CASL 原則型別
│   │   ├── traditional/     # 應用程式層級實體（article、role、tenant、user）
│   │   ├── base-types/      # 共用 NetSuite 實體結構
│   │   └── netsuite/        # NetSuite 專屬模型
│   ├── pages/                # 頁面層級元件，按路由區域分組
│   │   ├── admin/           # 受保護的應用程式頁面
│   │   ├── auth/            # 登入 / OTP 流程
│   │   ├── error/           # 錯誤頁面
│   │   ├── outside/         # 公開登陸頁
│   │   └── privacy-policy/  # 靜態法律頁面
│   ├── routes/               # TanStack 檔案式路由定義
│   │   ├── __root.tsx       # 根路由 / 版面配置
│   │   ├── (auth)/          # 認證路由群組
│   │   ├── admin/           # 受保護的管理路由
│   │   ├── (privacy)/       # 隱私權政策路由群組
│   │   └── (error)/         # 錯誤路由群組
│   ├── main.tsx              # 應用程式啟動器
│   ├── globals.css           # Tailwind 入口 + 淺色主題 CSS 變數
│   ├── index.css             # 替代 Tailwind 入口（目前由 main.tsx 匯入 globals.css）
│   └── routeTree.gen.ts      # 自動產生的 TanStack 路由樹
├── e2e/                      # Playwright E2E 測試規格（*.spec.cjs）
├── test_sqls/                # 臨時 SQL 草稿檔案（不屬於建置的一部分）
├── firebase.json             # Firebase Hosting 設定
├── package.json              # 相依套件與 npm 指令
├── pnpm-workspace.yaml       # pnpm workspace 設定（單一套件）
├── postcss.config.js         # PostCSS / Tailwind 設定
├── tailwind.config.ts        # Tailwind 主題與自訂顏色
├── tsconfig.json             # TypeScript 設定
├── ui.config.json            # solid-ui / shadcn 風格 CLI 設定
├── vite.config.ts            # Vite 外掛與路徑別名
├── vitest.config.ts          # Vitest 測試設定（jsdom、include 規則）
└── playwright.config.cjs     # Playwright E2E 測試設定（testDir: ./e2e）
```

## 路徑別名

- `~/*` 在 Vite 與 TypeScript 中都會解析為 `./src/*`。
- 跨頂層目錄匯入時，建議使用 `~/components/ui/button` 而非相對路徑。

## 建置與開發指令

所有指令都使用 pnpm。

```bash
# 安裝相依套件
pnpm install

# 啟動開發伺服器於 http://localhost:3000
pnpm run dev
# 或
pnpm start

# 正式建置（輸出至 dist/）
pnpm run build

# 在本機預覽正式建置
pnpm run serve

# 執行單元測試（Vitest + jsdom）
pnpm run test

# 執行 E2E 測試（Playwright；需先啟動前端與後端）
pnpm run test:e2e
pnpm run test:e2e:headed

# 移除 dist/
pnpm run clean
```

### Task（Taskfile.yml）捷徑

```bash
# 新增 solid-ui 元件
# 例如：task ui:add -- button
task ui:add -- <component-name>

# 開發 / 建置
task dev
task build

# 部署至 Firebase Hosting（遞增 patch 版本、建置、登入、部署）
task deploy
```

## 路由

應用程式使用 TanStack Solid Router 的**檔案式路由**。路由檔案位於 `src/routes/`，並由 `@tanstack/router-plugin/vite` 自動編譯為 `src/routeTree.gen.ts`。

### 慣例

- `src/routes/__root.tsx` — 根版面配置。
- `src/routes/admin/route.tsx` — 所有 `/admin/*` 路由的父版面配置；內含認證守衛。
- `src/routes/admin/sales-order.tsx` — `/admin/sales-order` 的路由定義（可定義 `loader`、`linkOptions` 等）。
- `src/routes/admin/sales-order.lazy.tsx` — `/admin/sales-order` 的懶載入元件。
- `(auth)/`、`(privacy)/`、`(error)/` 等路由群組不會影響 URL 路徑。

### 認證守衛

`src/routes/admin/route.tsx` 會在其 `loader` 中檢查持久化的認證狀態。若使用者未通過認證或 `/me` 查詢失敗，會重新導向至 `/signin` 並帶上 `redirect` 查詢參數。

## 資料取得模式

各領域模組遵循一致的分層：

1. **`src/lib/<domain>/<domain>.ts`** — 使用 `src/lib/requests` 的輔助函式建構 `Request` 物件。
2. **`src/lib/<domain>/index.ts`** — 匯出非同步函式與供元件使用的 TanStack `queryOptions` 物件。
3. **`src/constant/api.ts`** — 集中管理 API 路徑常數與 `HttpError` 類別。
4. **`src/lib/requests/utils.ts`** — 包裝 `fetch`、處理 JSON 主體、查詢參數，並為寫入方法附加 `X-CSRF-Token` 標頭（token 由 `src/lib/requests/csrf.ts` 從 `/restricted/csrf` 取得）。

在路由 loader 中的使用範例：

```tsx
import { salesOrdersQuery } from "~/lib/sales-order";

export const Route = createFileRoute("/admin/sales-order")({
  loader: async ({ context: { queryClient } }) => {
    await queryClient.ensureQueryData(salesOrdersQuery());
  },
});
```

## 狀態管理

- **伺服器狀態**：TanStack Solid Query（`queryClient` 建立於 `main.tsx`）。
- **認證狀態**：SolidJS store，透過 `src/pages/auth/context.tsx` 中的 `@solid-primitives/storage` 持久化至 `localStorage`。
- **UI 狀態**：元件內部的區域 SolidJS signals/stores；部分頁面層級 context 位於 `src/pages/admin/*/widgets/context.tsx`。
- **主題**：Kobalte `ColorModeProvider`，並持久化至 local storage。

## 元件慣例

- UI 基礎元件參考 shadcn/ui 設計，位於 `src/components/ui/`。它們使用 `class-variance-authority`（CVA）+ `tailwind-merge` + `clsx`（`cn()` 輔助函式）進行樣式設計。
- 大部分基礎元件包裝 Kobalte / Ark UI 無頭元件。
- 表單包裝器位於 `src/components/form/`，並整合 Modular Forms / TanStack Form。
- 頁面專屬 widgets 位於 `src/pages/<area>/<page>/widgets/`。

## 樣式

- TailwindCSS 3.4，自訂 HSL 顏色 token 定義於 `src/index.css`。
- 主題使用自訂藍/綠/青色調色盤（非預設的 shadcn slate）。
- 透過 `.dark` 或 `[data-kb-theme="dark"]` 支援深色模式。
- `globals.css` 包含較舊的主題檔案；`main.tsx` 匯入 `globals.css`，但 `index.css` 也存在並由 `ui.config.json` 引用。

## 環境變數

環境變數於 `src/env/index.ts` 使用 Valibot 進行驗證。瀏覽器使用的變數必須以 `VITE_` 為前綴。

主要變數（預設值請見 `src/env/index.ts`）：

- `NODE_ENV`
- `VITE_APP_VERSION`
- `VITE_SESSION_NAME`
- `VITE_GOOGLE_CLIENT_ID`
- `VITE_GOOGLE_OAUTH2_NONCE`
- `VITE_AUTH_SECRET`
- `VITE_AUTH_URL`
- `VITE_API_URL`
- `VITE_SELF_URL`
- `VITE_API_VERSION`（預設 `/api/v1`）
- `VITE_API_ACCESS_TOKEN`

`.env` 與 `.env.production` 檔案存在於專案根目錄，但可能包含機敏資訊，因此會阻擋直接讀取。請勿提交真實機密。

## 測試

- **單元測試**：執行器為 **Vitest 3.2**（`vitest.config.ts`），搭配 **jsdom** 與 `@solidjs/testing-library`；include 規則為 `src/**/*.test.{ts,tsx}`，使用 `*.test.ts` / `*.test.tsx` 命名慣例。
- 目前已有一個單元測試：`src/components/datatable/DepartmentFilter.test.tsx`。
- 使用 `pnpm run test` 執行單元測試。
- **E2E 測試**：使用 **Playwright**（`playwright.config.cjs`，`testDir: "./e2e"`）。規格檔位於 `e2e/`（目前有 `department-filter.spec.cjs`、`session-expiry.spec.cjs`），需先啟動前端 dev server（http://localhost:3000）與後端 API（http://localhost:3080），且 session-expiry 規格依賴後端 `feat/session-expiry` 分支與 seeder 建立的測試帳號。
- 使用 `pnpm run test:e2e`（或 `test:e2e:headed`）執行 E2E 測試；首次執行需 `pnpm exec playwright install chromium`。

## 部署

應用程式部署至專案 `hexagon-salesorder-platform` 的 **Firebase Hosting**。

- `firebase.json` 設定從 `dist/` 託管，並將 `/api/**` 與 `/customer_account_qrcode/**` 重寫至位於 `asia-east1` 的 Cloud Run 後端 `hexagon-backend`。
- 其他所有路徑都會回傳 `index.html`（SPA fallback）。
- 透過 `task deploy` 部署，它會遞增 patch 版本、執行 `pnpm run build`、重新以 Firebase 登入，並僅部署 hosting。

## 安全考量

- 認證仰賴後端 session cookie（`credentials: include`）加上寫入方法的 `X-CSRF-Token` 標頭（token 於執行期從 `/restricted/csrf` 取得，並在 401 時清除）。
- 認證狀態持久化於 `localStorage` 的 `auth_state`；請將其視為快取，而非安全邊界。
- `VITE_API_ACCESS_TOKEN` 與 `VITE_AUTH_SECRET` 是客戶端機密；請避免在記錄或 UI 中暴露它們。
- `/admin` 路由 loader 會強制將未認證的使用者重新導向至 `/signin`。
- CASL ability 規則定義於 `src/constant/casl.ts`，用於依角色控管 UI（`super`、`admin`、`acct`、`sales`）。

## Agent 常見任務

### 新增管理頁面

1. 在 `src/pages/admin/<feature>/` 下建立頁面元件。
2. 在 `src/routes/admin/<feature>.tsx` 與 `<feature>.lazy.tsx` 建立路由檔案。
3. 若頁面需要資料，在 `src/lib/<feature>/` 新增 API 建構器，並在 `src/lib/<feature>/index.ts` 中新增 `queryOptions`。
4. 視需要將型別新增至 `src/models/` 或 `src/models/netsuite/`。
5. 在 `src/constant/sidemenu.ts` 新增側邊欄選單項目。

### 新增 UI 基礎元件

使用 solid-ui CLI 捷徑：

```bash
task ui:add -- <component-name>
```

元件會根據 `ui.config.json` 產生至 `src/components/ui/`。

### 修改 API 端點

編輯 `src/constant/api.ts`（`ApiPathUrl`）以及 `src/lib/<domain>/<domain>.ts` 中對應的建構函式。

### 修改主題顏色

編輯 `src/index.css`（若 `globals.css` 仍被匯入也請一併檢查）中的 CSS 變數，或擴充 `tailwind.config.ts`。

## 注意事項與陷阱

- 應用程式同時匯入 `globals.css`，並依 Tailwind 設定使用 `index.css`。編輯主題前，請先確認 `main.tsx` 實際載入的是哪個檔案。
- 多個檔案包含無用 / 已註解的程式碼（dashboard widgets、dispatch boards、auth flows）。請驗證實際行為，不要假設註解內容仍然正確。
- 部分路由守衛使用 `window.location.href` 作為重新導向目標，可能包含完整 origin；若可行，請優先使用 loader 參數中的 `location.href`。
- `pnpm-workspace.yaml` 雖然存在，但此倉庫目前看起來是單一套件；除非新增其他套件，否則請視為單一包裝處理。
- `test_sqls/` 包含開發人員的臨時 SQL 草稿，不屬於建置的一部分。

---

最後更新：根據 2026-08-03 查證整理。
