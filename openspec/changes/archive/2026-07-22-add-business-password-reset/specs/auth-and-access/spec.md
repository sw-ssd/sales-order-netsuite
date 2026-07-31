## MODIFIED Requirements

### Requirement: 密碼設定與重設

擴充 `reset-null-password` 端點支援角色範圍。

#### Scenario: 管理員重設業務員密碼
- **WHEN** 管理員 PATCH `/api/v1/authentication/reset-null-password/{id}/salesrep`
- **THEN** 系統清除該業務員 Credential 密碼
- **THEN** 系統設定 `is_verified = false`
- **THEN** 業務員下次登入需執行 `newset-password` 流程

#### Scenario: 管理員重設系統使用者密碼
- **WHEN** 管理員 PATCH `/api/v1/authentication/reset-null-password/{id}/user`
- **THEN** 系統清除該使用者 Credential 密碼
- **THEN** 系統設定 `is_verified = false`
