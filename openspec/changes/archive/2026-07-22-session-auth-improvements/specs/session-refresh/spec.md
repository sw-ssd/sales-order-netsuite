## ADDED Requirements

### Requirement: 7 天 Session TTL 與 Sliding Extension
Session 有效時間為 7 天，使用者在活躍期間自動延長。

#### Scenario: Session 建立
- **WHEN** 使用者登入成功
- **THEN** session expiry 設為 7 天（604800 秒）
- **THEN** Set-Cookie header 含 `Max-Age=604800`

#### Scenario: Sliding extension（活躍延長）
- **WHEN** 已登入使用者發起 API 請求
- **THEN** 後端 middleware 檢查 session 剩餘時間
- **THEN** 若剩餘時間 < 24h，執行 `session.SetExpiry(7d)` 並更新 DB
- **THEN** 回應含更新後的 Set-Cookie header

#### Scenario: 閒置過期
- **WHEN** 使用者連續 7 天無請求
- **THEN** DB session expiry < now()
- **THEN** cleanup goroutine（每 30min）刪除過期 rows
- **THEN** 下一次請求回傳 401

### Requirement: Session 清理與監控
系統定期清理過期 session，並提供管理查詢。

#### Scenario: 定時清理
- **WHEN** cleanup goroutine 執行（每 30 分鐘）
- **THEN** 刪除所有 `expiry < now()` 的 session rows

#### Scenario: 活躍 session 查詢（管理用）
- **WHEN** 管理者透過 DB 查詢特定 user 的 session
- **THEN** 系統列出該 user 的所有活躍 session（建立時間、到期時間、user_type）
- **THEN** 管理者可手動刪除特定 session row 達成遠端撤銷
