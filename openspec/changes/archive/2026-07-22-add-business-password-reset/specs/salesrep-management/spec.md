## ADDED Requirements

### Requirement: 業務員密碼重設（管理員操作）

管理員可透過業務員管理頁面觸發密碼強制重設。

#### Scenario: 管理頁面密碼重設
- **WHEN** 管理員點選業務員資料列之「重置密碼」按鈕
- **THEN** 系統呼叫 PATCH `/api/v1/authentication/reset-null-password/{id}/salesrep`
- **THEN** 頁面顯示成功提示「已重置密碼」
- **THEN** 業務員登入時需先設定新密碼
