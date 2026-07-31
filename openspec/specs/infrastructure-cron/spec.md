## ADDED Requirements

### Requirement: 排程任務管理
系統記錄與監控所有定時任務的執行狀況。

#### Scenario: 查詢任務列表
- **WHEN** 使用者 GET `/api/v1/crons`
- **THEN** 系統回傳所有排程任務記錄（name, crontab, last_run, run_count, error_message）

#### Scenario: 任務執行監控
- **WHEN** 排程任務執行
- **THEN** 系統記錄 last_run_start/end 時間戳
- **THEN** 執行失敗時記錄 error_message
