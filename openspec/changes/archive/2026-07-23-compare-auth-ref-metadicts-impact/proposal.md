# 比對 master 與 auth_ref 對 metadicts 資料取得的影響

## 目標

比對 `sales-order-backend` 與 `sales-order-frontend` 的 `master` 與 `auth_ref` 分支，找出導致 `metadicts` 資料取得異常的根因，並決定修復方向。

## 背景

`auth_ref` 是認證強化分支，包含：

- 後端：新增 CSRF middleware、API key 機制、sliding session、調整 `nsstmt` 的 metadicts 查詢。
- 前端：重構 `apiReq` 請求層，加入 CSRF token 管理與 401 自動導回登入頁。

`metadicts` 是銷售訂單系統的字典表（客戶、業務、部門、計量單位等），後端提供 `GET /api/v1/metadicts`、`DELETE /api/v1/metadicts/{id}/{table_name}`、`PUT /api/v1/metadicts/recover/{id}/{table_name}`、`GET /api/v1/metadicts/force-sync`，前端透過 `metadictsQuery` 與 `metadictOptionsQuery` 取用。

## 預期產出

1. 明確指出 `auth_ref` 中哪些改動影響了 metadicts 資料取得。
2. 給出具體修復步驟（後端或前端）。
3. 更新 `design.md` 與 `tasks.md` 作為實作依據。
