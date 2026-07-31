## ADDED Requirements

### Requirement: 業務員 (Salesrep) 管理
系統維護業務員主檔，資料源為 NetSuite 同步。

#### Scenario: 查詢業務員列表
- **WHEN** 使用者 GET `/api/v1/salesreps` 傳入篩選條件
- **THEN** 系統回傳業務員列表（email, title, initials, alias 等）

#### Scenario: 建立業務員
- **WHEN** 使用者 POST `/api/v1/salesreps`
- **THEN** 系統建立 Salesrep 記錄
- **THEN** 系統自動建立對應 Credential 記錄（for auth login）

#### Scenario: 編輯業務員
- **WHEN** 使用者 PATCH `/api/v1/salesreps/{id}`
- **THEN** 系統更新業務員資料

#### Scenario: 業務員軟刪除與復原
- **WHEN** 使用者 DELETE `/api/v1/salesreps/{id}`
- **THEN** 系統執行軟刪除
- **WHEN** 使用者 PUT `/api/v1/salesreps/recover/{id}`
- **THEN** 系統復原業務員

#### Scenario: 強制同步
- **WHEN** 使用者 GET `/api/v1/salesreps/force-sync`
- **THEN** 系統從 NetSuite 拉取最新業務員資料

### Requirement: 業務員密碼重設（管理員操作）
管理員可透過業務員管理頁面觸發密碼強制重設。

#### Scenario: 管理頁面密碼重設
- **WHEN** 管理員點選業務員資料列之「重置密碼」按鈕
- **THEN** 系統呼叫 PATCH `/api/v1/authentication/reset-null-password/{id}/salesrep`
- **THEN** 頁面顯示成功提示「已重置密碼」
- **THEN** 業務員登入時需先設定新密碼
