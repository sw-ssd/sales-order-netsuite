## ADDED Requirements

### Requirement: Sliding Session 單元測試
測試 `internal/middleware/authentication.go` 中的 sliding session 邏輯。

#### Scenario: Session 延長
- **SHALL** 當上次 touch 時間 > 24h 時，更新 `session_last_touch` 時間戳
- **SHALL** 當上次 touch 時間 < 24h 時，不更新時間戳
- **SHALL** 首次請求時初始化 `session_last_touch` 時間戳

### Requirement: Force Logout 單元測試（repository 層）
測試 `internal/domain/authentication/repository.go` 的 `SignoutUser`。

#### Scenario: Force logout 不區分 utype
- **SHALL** `SignoutUser(userID)` 刪除所有 `info_id = userID` 的 session rows
- **SHALL** 不檢查 user type（User / Salesrep / Customer 皆適用）
