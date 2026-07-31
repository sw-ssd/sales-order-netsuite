## Context

此專案為食品業進銷存系統（特耀食品），採用三端架構：

```
┌──────────────────────────────────────────────────────────────┐
│                      sales-order-netsuite                    │
├──────────────────────────────────────────────────────────────┤
│                     ┌──────────────────┐                     │
│                     │   NetSuite ERP    │                     │
│                     │ SuiteQL / RecordAPI│                    │
│                     └────────┬─────────┘                     │
│                              │ (sync)                        │
│              ┌───────────────┼───────────────┐               │
│              │               │               │               │
│     ┌────────▼──┐   ┌───────▼───────┐  ┌────▼────────┐      │
│     │ Backend   │   │  Frontend      │  │  Mobile     │      │
│     │ Go + Ent  │   │  SolidJS SPA   │  │  Flutter    │      │
│     │ PostgreSQL│   │  Firebase Host │  │  iOS/Android│      │
│     │ REST API  │   │  TanStack Qry  │  │  Dio +      │      │
│     │ 35 ents   │   │  15+ pages     │  │  Sembast    │      │
│     │ 20 domains│   │  Tailwind      │  │  3-layer    │      │
│     └───────────┘   └────────────────┘  └─────────────┘      │
│                                                              │
│     Auth: Session cookie + X-Sowinsoft-Token (JWT)           │
│     RBAC: Casbin (multi-tenant)                              │
│     Sync: gocron scheduler + gookit/event bus                │
└──────────────────────────────────────────────────────────────┘
```

目前現狀：所有規格為零散存放在各端 AGENTS.md 與程式碼中，無集中式規格文件。

## Goals / Non-Goals

**Goals:**
- 為 11 個領域建立正式規格文件於 `openspec/specs/<domain>/spec.md`
- 定義每個領域的：實體模型、API 端點、業務規則、NetSuite 同步行為
- 定義跨端（Frontend/Mobile）應一致遵循的行為規範
- 為後續開發提供可引用的規格基礎

**Non-Goals:**
- 不修改任何程式碼
- 不引入新功能或重構
- 不建立測試
- 不修改部署流程
- 不建立架構圖以外的文件格式

## Decisions

### 1. 領域切割方式：按業務概念，而非按技術層

**選擇**：將 35 個 ent 實體按業務領域分組為 11 個 spec 目錄，而非按後端內部分層（如 auth、entity、infra）。

**理由**：
- 業務團隊與利害關係人關心的是「客戶管理」、「訂單」等概念，而非「ent schema 層」
- 跨端溝通時，領域名稱是共同語言
- 未來功能開發按領域展開，spec 目錄自然對應到實作範圍

### 2. 規格層級：行為規格，非實作細節

**選擇**：spec 著重於 WHAT（系統行為與 API 合約），不記錄 HOW（實作細節如 ent mixin、UseCase 內部邏輯）。

**理由**：
- 實作細節變化頻率高，放在 spec 會產生大量維護成本
- 行為規格足以作為前後端對接的約定
- 設計決策記錄在 design.md，實作細節留在程式碼

### 3. 同步邏輯集中描述

**選擇**：將 NetSuite 同步機制獨立為一個 spec（netsuite-sync），而非分散在各領域 spec。

**理由**：
- 同步機制（SuiteQL、Record API、gocron、event bus）是跨領域的基礎設施
- 各領域 spec 中僅標註「支援 NetSuite 同步」，細節統一在 netsuite-sync 描述
- 避免多處維護同一份同步規格

### 4. 規格文件語言：繁體中文

**選擇**：spec 內容使用繁體中文描述業務行為，專業術語保留英文。

**理由**：
- 業務團隊（非工程）可能參與規格審查
- 中文能降低業務理解的認知負擔
- API 端點名、實體欄位名保持英文以對應程式碼

### 5. 跨端行為一致性的處理方式

**選擇**：spec 中定義「系統行為」，不區分 Frontend vs Mobile 實作差異。若某行為僅限單端，特別標註。

**理由**：
- 規格應描述「這個系統做了什麼」，而非「Web App 做了什麼、App 做了什麼」
- 前後端實作差異是設計決策，記錄在設計文件中，不應出現在行為規格

## Risks / Trade-offs

| 風險 | 影響 | 緩解方式 |
|------|------|----------|
| Spec 與實作脫節 | spec 可能過時，失去參考價值 | 後續開發時要求更新對應 spec；Code Review 時檢核 spec 一致性 |
| 領域邊界模糊 | 部分跨域行為（如 Dispatch 跨 SalesOrder + Inventory）不易歸類 | 若跨域邏輯複雜，建立 cross-cutting spec 或設計文件 |
| spec 粒度不一致 | 有的領域寫太細、有的太粗 | 以「可做為前後端對接契約」為粒度標準 |
| 規格與 AGENTS.md 重疊 | 各端已有 AGENTS.md 描述專案結構 | AGENTS.md 屬於 onboarding 文件，spec 屬於行為定義，兩者定位不同 |
