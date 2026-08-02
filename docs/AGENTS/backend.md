# 特耀訂出貨 API 後台服務 — AI Agent 指引

> 本文件是給 AI coding agent 的專案導覽。若你要修改程式碼、新增功能或除錯，請先讀完本文件。

> 本文件為原 `sales-order-backend/AGENTS.md` 之整合版，已搬移至 parent repo。內容以實際程式碼為準。

## 1. 專案概述

這是 **特耀訂出貨 API 後台服務**（`github.com/hexagon-maker/sales-order-backend`），一套以 RESTful API 為主的後台系統，主要功能包括：

- 訂單（Sales Order）與出貨調度（Dispatch）查詢、建立、更新、刪除與復原
- 客戶（Customer）、商品（Item）、部門（Department）、銷售代表（Salesrep）資料管理
- 報價單項目（Estimate Item）與中繼字典（Metadict）管理
- 使用者（User）、角色（Role）、租戶（Tenant）與權限（Policy）管理
- 與 NetSuite 進行雙向同步（讀取 SuiteQL、寫入 Record API）
- 郵件通知（Notify）與排程任務（Cron / Cloud Scheduler）觸發
- Server-Sent Events（SSE）與 WebSocket 即時派工看板

本專案採用 **Go 1.25**，以模組化網域分層組織，資料庫使用 PostgreSQL（MySQL 在程式碼中保留彈性）。

## 2. 技術棧

| 分類 | 主要套件 / 工具 |
|------|----------------|
| 語言與執行 | Go 1.25 |
| Web 框架 / Router | [go-chi/chi/v5](https://github.com/go-chi/chi) |
| ORM / DB 產生 | [entgo.io/ent](https://entgo.io/) + Atlas |
| 資料庫 Driver | `pgx/v5`、`lib/pq`、`go-sql-driver/mysql` |
| 遷移工具 | [pressly/goose/v3](https://github.com/pressly/goose) + Atlas `migrate diff` |
| Session | [alexedwards/scs/v2](https://github.com/alexedwards/scs)，自訂 Postgres/Ent Store |
| 權限 | [casbin/casbin/v2](https://github.com/casbin/casbin) + 自訂 Ent Adapter |
| 驗證 / 密碼 | `golang-jwt/jwt/v5`（僅遺留的 middleware / `cmd/token` 使用，見 §7.1）、`alexedwards/argon2id` |
| 驗證器 | `go-playground/validator/v10` |
| HTTP 輸入綁定 | `ggicci/httpin` |
| 快取（可選） | Redis / Valkey（`go-redis/v9`） |
| 事件 | `gookit/event` |
| 排程 | `go-co-op/gocron/v2` |
| 外部 API | NetSuite REST API（`oapi-codegen` 產生）+ SuiteQL |
| 日誌 | 標準 `log/slog` + 自訂 trace handler |
| 容器 | Docker / Podman、docker-compose |
| 部署 | GCP Cloud Run + Cloud SQL（Unix socket） |
| 任務執行 | [Task](https://taskfile.dev/)（`Taskfile.yml`） |
| 熱重載 | [air](https://github.com/air-verse/air)（`.air.toml`） |

## 3. 專案結構

```text
.
├── cmd/                    # 可執行入口
│   ├── sw8/                # 主 HTTP server（cmd/sw8/main.go）
│   ├── migrate/            # 執行 Goose up/down
│   ├── seed/               # 資料種子（up/down ns|user）
│   ├── route/              # 列出所有註冊路由
│   ├── token/              # 產生 API JWT token（遺留工具，見 7.1）
│   └── auto_increment/     # 自動遞增 API_VERSION patch
├── config/                 # 環境變數與設定（envconfig + godotenv）
├── database/               # Goose 遷移、seeder、Atlas diff 工具
├── ent/                    # Ent schema、產生後的 gen/、privacy rule、viewer
├── internal/               # 應用程式核心
│   ├── domain/             # 各網域（DDD 分層）
│   ├── global/             # 跨域通用結構（如 MailInfo）
│   ├── middleware/         # HTTP middleware
│   ├── server/             # Server 初始化、路由註冊、Swagger docs
│   └── utility/            # 小工具（respond、filter、param、nsstmt、csrf…）
├── logger/                 # 自訂 slog trace handler（logger/withTrace.go）
├── third_party/            # 與外部服務整合與通用套件
│   ├── authorization/      # Casbin Enforcer / Adapter / Model
│   ├── database/           # sql/sqlx/ent/CloudSQL 連線包裝
│   ├── netsuite/           # NetSuite API client 與模型
│   ├── postgres_entstore/  # scs 的 Postgres session store
│   ├── redis/              # Redis/Cluster 連線包裝
│   ├── tools/              # 字串、指標、JSON、路由 walk 工具
│   ├── validate/           # validator 包裝
│   └── constants/          # 常數與 context key
├── testcontainers/         # dockertest 容器測試輔助
├── scripts/                # shell 腳本、load test（k6/locust）
├── taskfiles/              # Task 子任務拆分
├── docker-compose-*.yml    # 本地基礎設施 / app / CloudSQL proxy
├── Dockerfile
├── Taskfile.yml
├── .air.toml
├── tygo.yaml               # 產生前端 TypeScript 型別
└── hexagon.env             # 範例/開發環境變數檔（注意內含敏感資訊）
```

### 3.1 網域分層慣例

每個 `internal/domain/<domain>` 通常包含：

- `model.go`：request/response/filter DTO，通常會內嵌 `*gen.Xxx` 作為 schema 層
- `handler.go`：HTTP handler，使用 `internal/utility/respond` 回傳 JSON
- `repository.go`：資料存取，使用 `*gen.Client`（Ent）
- `usecase.go`：業務邏輯，可呼叫 repository 與 NetSuite client
- `register.go`：註冊路由，通常會掛上 `middleware.Authenticate(session)`
- `transformation.go`：DTO 與 ent 模型之間的轉換
- `*_test.go`：單元 / 整合測試
- `*_mock.go`：由 `moq` / `mockery` 產生的 mock

網域名稱多為複數英文，例如 `sales_orders`、`customers`、`departments`。API 路徑前綴統一為 `/api/v1`。

## 4. 環境與設定

設定採用 **env file + 環境變數** 混合模式：

1. `config.New(filenames...)` 會先用 `godotenv.Overload(filenames...)` 載入 `.env` 檔。
2. 再用 `envconfig.MustProcess("<PREFIX>", &cfg)` 從環境變數讀取值。

主程式可透過 `-env <path>` 指定額外 env 檔：

```bash
go run cmd/sw8/main.go -env hexagon.env
```

### 4.1 主要環境變數前綴

| 前綴 | 檔案 / 結構 | 說明 |
|------|------------|------|
| `API_*` | `config/api.go` | 埠號、secret、版本、swagger、graceful timeout |
| `DB_*` | `config/database.go` | 資料庫連線、pool、Cloud SQL instance |
| `CORS_*` | `config/cors.go` | 允許來源清單 |
| `SESSION_*` | `config/cookie.go` | Cookie 名稱、HttpOnly、Secure、SameSite、Lifetime |
| `EMAIL_*` | `config/email.go` | SMTP 帳號密碼 |
| `REDIS_*` | `config/cache.go` | Redis/Cluster 設定（目前 `config.New` 預設未啟用） |
| `NETSUITE_*` | `config/netsuite.go` | NetSuite Token Based Authentication |
| `OTEL_*` | `config/opentelemetry.go` | OpenTelemetry（目前 `config.New` 預設未啟用） |
| `OAUTH2_*` | `config/provider.go` | Google OAuth2（目前 `config.New` 預設未啟用） |

> **注意**：`hexagon.env` 目前被追蹤在 repo 中且包含範例/真實憑證。請避免將生產環境祕鑰提交到版本控制。

## 5. 常用建置與執行指令

本專案使用 [Task](https://taskfile.dev/) 作為主要任務執行器。執行前請先安裝 `task`。

```bash
# 列出所有可用任務
task -l

# 開發執行（ent 產生 + go run）
task run

# 熱重載開發（air，使用 dev build tag）
task dev

# 連接 Cloud SQL 的熱重載開發
task cloudsql:dev

# 建置二進位到 ./bin/sw8
task build

# 啟動 / 停止本地基礎設施（Postgres + Valkey + Mailpit）
task infra:start
task infra:stop
task infra:stop:volumes

# 列出所有路由
task routes

# 產生 API JWT token（寫入 token.export，遺留工具，見 7.1）
task apitoken

# 執行資料種子
task seeder -- up ns
task seeder -- up user
```

### 5.1 程式碼產生

```bash
# 產生 Ent 程式碼（ent/gen/*）
task ent:gen        # 等同 go generate ./ent

# 產生 Swagger 文件到 internal/server/docs
task swagger

# 產生 NetSuite OpenAPI client
task oapigen

# 產生前端 TypeScript 型別到 frontend_types/
task typego

# 執行所有 //go:generate
task generate
```

## 6. 資料庫與遷移

### 6.1 兩套遷移機制

1. **Goose SQL 遷移**：實際在 runtime 執行。
   - 檔案位於 `database/goose/*.sql`。
   - `database/migrate.go` 使用 `//go:embed goose/*.sql` 嵌入，並在 server 啟動時呼叫 `goose.Up()`。
   - `cmd/migrate/main.go` 可獨立執行 `goose.Up()` / `goose.Down()`。

2. **Atlas / Ent diff**：用來產生新的 Goose 格式遷移檔。
   - `database/migrate_goose/main.go`（build tag `ignore`）使用 Atlas `migrate.NamedDiff` 將 schema 差異寫入 `database/goose/`。
   - `task goose:migrate -- <name>` 會產生新的遷移檔。
   - `task goose:diff` 與 `task goose:apply` 則是直接呼叫 Atlas CLI。

> 命名注意：`task db:migrate` 目前的行為是「先 `ent:gen` 再產生 Atlas diff」，不是直接對資料庫執行 up。要套用 migration 請用 `task goose:up` 或啟動 server。

### 6.2 常用遷移指令

```bash
# 產生新的 Goose 遷移檔（Atlas diff）
task db:migrate -- add_new_field

# 使用 Goose CLI 直接對資料庫執行 up
task goose:up

# 執行單步 up / down
task goose:step
task goose:rollback

# 查看目前狀態
task goose:status

# 清空資料庫（危險）
task goose:reset
```

### 6.3 Ent 與隱私規則

- Schema 定義在 `ent/schema/`。
- `ent/generate.go` 定義了產生指令，啟用許多 feature：versioned migration、upsert、execquery、modifier、bidi edges、named edges、intercept、privacy、entql、snapshot。
- `ent/setorclear/` 是自訂 template extension。
- `ent/viewer/` 提供 tenant / admin / view 的 viewer context。
- `ent/rule/` 實作 privacy rule：`DenyIfNoViewer`、`AllowIfAdmin`、`FilterTenantRule`、`FilterDepartmentRule`。
- 通用 mixin 包括 `mixin_base`（建立/更新時間）、`mixin_tenant`、`mixin_hooks_softdelete` 等。

## 7. 認證、授權與安全

### 7.1 兩種認證方式

1. **Session Cookie**：透過 `scs` 管理，store 是自訂的 `postgres_entstore`，以 Postgres 儲存 session。
   - 路由使用 `middleware.Authenticate(session)` 保護。
   - `middleware.LoadAndSave(session)` 會將登入使用者 ID 寫入 context。

2. **API Key**：`middleware.ApiKeyMiddleware` 檢查 `X-Sowinsoft-Token` header，以 SHA-256 hash 對照 `api_keys` table 驗證；此 middleware 已取代舊的 HS256 JWT-based `ApiTokenMiddleware`（舊 JWT token 已不再被接受）。
   - API key 由 `/api/v1/api-keys`（`internal/domain/apikeys`）發放，支援啟用狀態、到期時間與 scopes，每次請求會更新 `last_used_at`。
   - 舊的 `cmd/token/main.go`（HS256 JWT、無 `exp` claim）仍存在，但其產生的 token 已無法通過新的 `ApiKeyMiddleware`，屬遺留工具。

### 7.2 授權

- 使用 **Casbin** RBAC with domain（tenant）。
- Model 位於 `third_party/authorization/model.go`。
- Adapter 位於 `third_party/authorization/adapter.go`，直接操作 `ent/gen/casbinrule`。
- `third_party/authorization/enforcer.go` 提供 `ICasbinEnforcer`。
- 以下路徑在授權檢查例外清單：`/version`、`/swagger`、`/api/v1/netsuite/*`、`/api/v1/casl_resources`、`/api/v1/permissions`。
- matcher 中 hardcode 允許 `r.sub == "sowinsoft"` 超級管理員。

### 7.3 密碼與敏感資料

- 密碼使用 `argon2id.CreateHash` 雜湊。
- 生產環境請設定 `SESSION_SECURE=true`、`SESSION_HTTP_ONLY=true`、`SESSION_SAME_SITE=lax`（或更嚴格）。
- `NETSUITE_*` 與 `EMAIL_*` 等憑證僅透過環境變數載入，**不要寫死在程式碼中**。
- NetSuite client 目前設定 `InsecureSkipVerify: true`，在生產環境應移除或改為正確的 TLS 設定。
- `cmd/token` 屬遺留工具：其產生的 HS256 JWT 沒有 `exp` claim，且已無法通過新的 `ApiKeyMiddleware`；需要 API 存取請改用 `/api/v1/api-keys` 發放的 API key。

## 8. 測試策略

### 8.1 單元測試

- 使用 `testify` 與 `suite`。
- Repository / UseCase / Handler 皆有對應的單元測試。
- Mock 透過 `moq`（`//go:generate moq ...`）或 `mockery` 產生，例如 `usecase_mock.go`、`repository_mock.go`、`enforcer_mock.go`。

```bash
task test:unit     # go test -short ./...
task test          # go test ./...
```

### 8.2 整合測試

- `testcontainers/local_containers.go` 使用 `ory/dockertest` 啟動 PostgreSQL container。
- 整合測試會建立 container、跑 `client.Schema.Create(...)`、建立 session 與 Casbin enforcer，再測試 handler。
- 範例：`internal/domain/users/handler_integration_test.go`、`internal/domain/tenants/handler_integration_test.go`、`internal/domain/roles/handler_intefration_test.go`。

```bash
task test:integration   # go test -run Integration ./...
```

### 8.3 E2E / 負載測試

- `task test:e2e` 會執行 `e2e/docker-compose.yml`（目前檔案可能不存在，使用前請確認）。
- `scripts/k6.js` 與 `scripts/locustfile.py` 提供負載測試腳本。

## 9. 部署與維運

### 9.1 本地容器

```bash
# 啟動基礎設施
task infra:start

# 啟動 app 容器（docker-compose-sw8.yml，透過 dc: 命名空間）
task dc:sw8:start
```

### 9.2 建置映像檔

```bash
task build
# 或手動
docker build --platform linux/amd64 -t sw8/server .
```

### 9.3 GCP 部署

部署流程定義在 `taskfiles/GCP.yml`：

```bash
task gcp:deploy   # 自動遞增版本、build、push、deploy 到 Cloud Run
```

- 目標專案：`hexagon-salesorder-platform`
- Artifact Registry：`asia-east1-docker.pkg.dev/hexagon-salesorder-platform/hexagon-salesorder-docker/hexagon-backend`
- Cloud Run 會掛載 Cloud SQL instance，並設定 Unix socket：`/cloudsql/<INSTANCE_CONNECTION_NAME>`。
- `cmd/auto_increment/main.go` 會自動將 `API_VERSION` patch +1 並寫回 `.env` 與 `hexagon.env`。

## 10. 程式碼風格與慣例

- 遵循標準 Go 格式：`task fmt`（`go fmt ./...`）。
- 靜態檢查：`task lint`（`golangci-lint run`）、`task vet`（`go vet ./...`）。
- 漏洞掃描：`task vuln`（`govulncheck ./...`）。
- race 檢測：`task race`。
- 註解風格：許多 domain model 欄位會同時保留英文與中文說明，例如 `// Password 密碼`。新增業務欄位時建議維持此慣例。
- 錯誤處理：handler 統一透過 `internal/utility/respond` 回傳 JSON 錯誤。
- 過濾器：list API 使用 `internal/utility/filter.Filter` + `ggicci/httpin` 綁定 query string。
- NetSuite 相關欄位命名常直接使用 NetSuite 原始欄位名稱（如 `custentity_hf_car_number`）。

## 11. 常見入口與除錯

| 入口 | 用途 |
|------|------|
| `cmd/sw8/main.go` | 主 HTTP server |
| `cmd/route/main.go` | 印出所有註冊路由 |
| `cmd/migrate/main.go` | 執行 Goose 遷移 |
| `cmd/seed/main.go` | 資料種子與測試資料清除 |
| `cmd/token/main.go` | 產生 API JWT token（遺留工具，見 7.1） |
| `cmd/auto_increment/main.go` | 遞增 `API_VERSION` patch |
| `internal/server/server.go` | Server 初始化、middleware 鏈、資源關閉 |
| `internal/server/initDomains.go` | 註冊所有 domain 路由 |
| `third_party/netsuite/netsuite_client.go` | NetSuite client 初始化 |

## 12. 注意事項與已知限制

- `config.New` 目前將 `Cache`、`Elasticsearch`、`OpenTelemetry`、`Provider` 註解掉，未預設載入；啟用前需取消註解並確保對應基礎設施就緒。
- Redis/Valkey 目前是選配；server 只在 `REDIS_ENABLE=true` 時建立連線。
- SSE / WebSocket endpoint 位於 `/api/v1/sales_orders/dispatches/sse` 與 `/api/v1/sales_orders/dispatches/ws`。
- `task test:e2e` 相依的 `e2e/docker-compose.yml` 目前可能不存在，使用前請檢查。
- `.env` 與 `token.export` 已被 `.gitignore` 排除；`hexagon.env` 目前仍被追蹤，請注意不要寫入正式密鑰。

最後更新：根據 2026-08-03 查證整理。
