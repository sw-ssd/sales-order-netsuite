## Why

業務員（Salesrep）若忘記密碼，目前無管理員強制重設機制。後端 repo 已支援 Salesrep 的 `ResetNullPassword`（清除密碼 → 強制下次登入重設），但 handler 層僅開放 Customer 類型，Salesrep 被阻擋。需補上此缺口，並在管理後台提供對應操作按鈕。

## What Changes

- **Backend**: 擴充 `POST /api/v1/authentication/reset-null-password/{id}/{utype}` handler，支援 `utype = salesrep`（及 `user`）
- **Frontend**: 業務員管理頁面資料列新增「重置密碼」按鈕（比照 Customer 管理頁面現有實作）

## Capabilities

### New Capabilities
（無）

### Modified Capabilities
- `auth-and-access`: 擴充 `reset-null-password` 端點角色支援範圍
- `salesrep-management`: 業務員管理頁面加入密碼重設操作
