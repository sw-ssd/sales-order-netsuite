## Why

`session-auth-improvements` 在後端與前端完成了 CSRF 保護、M2M API Key 系統、Sliding Session 等工作，但在 Mobile App（`sales-order-app/`）端有兩項未完成：
1. **靜態 API token** — `AuthInterceptor` 仍使用 `FlavorConfig.getApiAccessToken()` 的靜態 per-env token，應改為登入後取得的 session-bound token
2. **缺少啟動時 session 驗證** — App 啟動時若本機標記為已登入，未確認 server 端 session 是否仍有效

本次變更補上這兩項，讓 Mobile App 的認證行為與後端安全模型一致。

## What Changes

- 後端登入 response 加入 `accessToken` 欄位（session-bound JWT，24h expiry）
- Mobile `AuthProvider` 登入成功後從 response 讀取 token，存入 session info
- Mobile `AuthInterceptor` 從 session 取出動態 token 放入 `X-Sowinsoft-Token` header
- 移除 `FlavorConfig.getApiAccessToken()` 與對應 env 變數
- Mobile 啟動時驗證：若 `isAuth=true`，呼叫 `/restricted/me` 確認 server session 有效，失敗則清除 session 跳轉登入頁

## Capabilities

### Modified Capabilities

- `auth-and-access`: Mobile 端 token 管理（靜態 → 動態）+ 啟動 session 驗證

### New Capabilities

無

## Impact

- `sales-order-app/lib/layer_business/network/auth_interceptor.dart` — 改讀動態 token
- `sales-order-app/lib/layer_business/services/auth/auth_session_manager.dart` — 新增 token 儲存
- `sales-order-app/lib/layer_business/services/auth/provider.dart` — 登入時儲存 token
- `sales-order-app/lib/main.dart` — 啟動時加入 session 驗證
- `sales-order-app/lib/layer_business/utils/flavor_config.dart` — 移除 `getApiAccessToken()`
- 後端登入 handler — response body 新增 `accessToken` 欄位
