## Context

Mobile App 目前認證流程：

```
Login → POST /login/{utype}
  → Set-Cookie: __session (由 CookieManager 自動處理)
  → X-Sowinsoft-Token: 靜態 per-env token (unchanged across users)

Every request → AuthInterceptor
  → headers['X-Sowinsoft-Token'] = FlavorConfig.getApiAccessToken()
  → headers['Accept'] = 'application/json'
```

問題：靜態 token 與使用者無關、無到期、無法撤銷。也缺少啟動時驗證 server session。

## Goals / Non-Goals

**Goals:**
- Mobile 使用 session-bound JWT token（取代靜態 per-env token）
- 後端登入 response 新增 `accessToken` 欄位
- App 啟動時若 `isAuth=true` 則驗證 server session 有效性
- 移除 `FlavorConfig.getApiAccessToken()` 及對應 env 變數

**Non-Goals:**
- 不改變後端 JWT 產生邏輯（沿用 HS256 + 24h expiry）
- 不重構 Mobile 整體 auth 架構
- 不改變現有 cookie-based session 機制

## Decisions

### 1. Token 存放位置
**選擇**：存入 `InSessionInfoStorage`（sembast KV），與 session info 一起持久化。

**理由**：
- 與 `AutherSessionInfo` 共用 storage，簡化清除邏輯
- App 重啟後仍可取得 token（直到 session 過期）
- `clearSession()` 自動清除 token

### 2. 啟動驗證時機
**選擇**：在 `main.dart` 的 `initializeMainApp()` 完成 setupLocator 之後，runApp 之前。

**流程**：
```
initializeMainApp()
  ├─ setupLocator()
  ├─ if (AuthSessionManager.isLoggedIn)
  │    try GET /restricted/me
  │    ├─ 200 → OK
  │    ├─ 401 → clearSession()
  │    └─ network error → keep session (offline mode)
  └─ runApp(App())
```

**理由**：
- 在 router 初始化之前完成，避免閃白或跳轉
- 不會干擾 UI 層的導航邏輯
- 不採用 lazy check（進入頁面後才驗證）避免短暫顯示錯誤內容

### 3. 後端 response 格式
**選擇**：在既有登入 response body 中加入 `accessToken` 欄位，不改變既有欄位。

**理由**：
- 向後相容 — 既有 Mobile App 無此欄位會忽略
- 新 Mobile App 有此欄位就使用
- 不須新增 API endpoint

### 4. 舊靜態 token 保留
**選擇**：暫時保留 `FlavorConfig.getApiAccessToken()` 與 env 變數，但 `AuthInterceptor` 不再使用。

**理由**：
- 避免一次性刪除導致其他未預期使用處崩潰
- 日後可另開 change 清理

## Risks / Trade-offs

| 風險 | 影響 | 緩解 |
|------|------|------|
| 啟動驗證延長啟動時間 | 多一次 HTTP round-trip | 僅在 `isAuth=true` 時執行（大多數情況使用者未登入）；離線模式跳過 |
| 網路錯誤導致登出 | 使用者明明有網路卻因暫時中斷被登出 | 網路錯誤保留 session，僅 401 才清除 |
| 新舊 App 版本並存 | 舊版 App 無 `accessToken` 欄位 | 後端回傳該欄位但舊 App 忽略；新版才取用 |
