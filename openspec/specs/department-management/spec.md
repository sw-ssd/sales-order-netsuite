## ADDED Requirements

### Requirement: 部門 (Department) 管理
系統維護部門主檔，資料源為 NetSuite 同步。

#### Scenario: 查詢部門列表
- **WHEN** 使用者 GET `/api/v1/departments` 傳入篩選條件
- **THEN** 系統回傳部門列表（name, full_name, is_inactive, subsidiary）

#### Scenario: 強制同步
- **WHEN** 使用者 GET `/api/v1/departments/force-sync`
- **THEN** 系統從 NetSuite 拉取最新部門資料

#### Scenario: 部門軟刪除與復原
- **WHEN** 使用者 DELETE `/api/v1/departments/{id}`
- **THEN** 系統執行軟刪除
- **WHEN** 使用者 PUT `/api/v1/departments/recover/{id}`
- **THEN** 系統復原部門
