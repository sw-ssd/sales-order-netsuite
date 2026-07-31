## Context

三個子專案各有獨立的 Taskfile.yml，但缺乏根層級編排：

```
sales-order-netsuite/
├── Taskfile.yml              ← 不存在（本次新增）
├── sales-order-frontend/
│   ├── Taskfile.yml          ← 已有 dev, build, deploy
│   └── ...
├── sales-order-backend/
│   ├── Taskfile.yml          ← 已有 run, dev, build, test, lint...
│   └── ...
├── sales-order-app/
│   ├── Taskfile.yml          ← 已有 dev, build, gen, test...
│   └── ...
```

## Goals / Non-Goals

**Goals:**
- 根 Taskfile.yml 以 `includes` 整合三個子專案
- 定義頂層 `dev`、`build`、`test` 三個 task
- 不修改子專案的既有 Taskfile
- 向後相容：各子專案仍可獨立執行 `task dev`

**Non-Goals:**
- 不改動各子專案內部的 build 邏輯
- 不導入新的 CI/CD 系統（但 CI 可改用 `task build`）
- 不處理 workspace 套件共用（非必要）

## Decisions

### 1. includes 模式

**選擇**：使用 Taskfile v3 的 `includes` + `dir` 指向子專案。

```yaml
version: '3'

includes:
  frontend:
    taskfile: ./sales-order-frontend/Taskfile.yml
    dir: ./sales-order-frontend
  backend:
    taskfile: ./sales-order-backend/Taskfile.yml
    dir: ./sales-order-backend
  app:
    taskfile: ./sales-order-app/Taskfile.yml
    dir: ./sales-order-app
```

**理由**：
- `dir:` 確保 task 在子專案目錄執行，無需 `cd`
- 子專案的 Taskfile 完全不需要修改
- Taskfile 命名衝突？不會 — 透過前綴呼叫：`task: frontend:dev`

### 2. 頂層 task 設計（dev）

```yaml
dev:
  desc: 啟動後端 + 前端開發伺服器
  cmds:
    - task: backend:dev
  deps:
    - frontend:dev
```

**不採用平行啟動的理由**：`deps` 執行完才回傳，但 `frontend:dev` 和 `backend:dev` 都是持久行程。實際上其中一個需在背景執行。改用 `cmds` 順序執行 + `&` background。

**修正方案**：
```yaml
dev:
  desc: 啟動後端 + 前端開發伺服器
  cmds:
    - task: backend:dev &
    - sleep 3
    - task: frontend:dev
```

### 3. 不修改子專案 Taskfile

**選擇**：零改動。

**理由**：
- Taskfile `includes` 完全透明 — 不入侵
- 子專案獨立開發流程不變
- CI 可直接 `task build`，也可精準 `task: frontend:build`
- 將來的變更只需改根 Taskfile

## Risks / Trade-offs

| 風險 | 影響 | 緩解 |
|------|------|------|
| 背景行程管理 | `dev` 用 `&` background，終止時需手動 `kill` | 使用 `task -p dev` 或 tmux 分窗 |
| 跨語言 test 順序相依 | 後端 test 跑完才跑前端 test 不一定對 | 頂層 `test` 依序執行，各子專案獨立快取 |
| 根 Taskfile 版本與子專案不一致 | 子專案可能使用不同 Taskfile 版本 | 統一鎖定 `version: '3'` |
