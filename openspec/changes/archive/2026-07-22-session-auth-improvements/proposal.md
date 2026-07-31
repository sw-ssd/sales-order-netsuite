## Why

目前認證系統存在多項安全與維護問題：無 CSRF 保護、API token 永不過期且所有使用者共用、三端 401 行為不一致、force logout 功能不完整、以及 session TTL 預設過短（24h）。需整頓以提升安全性與使用者體驗。

## What Changes

- **新增 CSRF 保護** — 啟用既有 `/api/v1/restricted/csrf` endpoint，前端寫操作帶 CSRF token；SameSite 留 `none` 不動
- **建立機器對機器 API 金鑰系統** — 取代現有靜態 X-Sowinsoft-Token；DB 儲存、scope-based、可撤銷 **BREAKING**
- **統一三端 401 處理** — 後端標準化 401 response body，前端 fetch 層全域攔截，Mobile 啟動時驗證 session
- **Session TTL 調整** — 24h → 7 天 + sliding session（活躍時自動延長）
- **Force Logout 補齊** — 支援 User / Salesrep / Customer 三種 user type
- **移除已註解的 OAuth/OTP stub 程式碼**

**BREAKING**: 舊的靜態 `X-Sowinsoft-Token` env 變數與對應的 HS256 JWT middleware 已移除，改用新的 API Key 認證 middleware。但 `X-Sowinsoft-Token` header 名稱仍保留，現在用於 (1) Mobile 登入後取得的動態 JWT（24h 到期）或 (2) 機器對機器 API Key。現有靜態 env token 將失效，需申請新的 API key。

## Capabilities

### New Capabilities

- `api-key-management`: 機器對機器 API 金鑰管理系統 — DB 儲存、scope-based 權限、可撤銷與輪換
- `session-refresh`: Session 延長機制，7 天 TTL + sliding（活躍請求自動延長）

### Modified Capabilities

- `auth-and-access`: CSRF 保護、401 response 統一格式、session TTL 7d、force logout 補齊、移除 dead code
