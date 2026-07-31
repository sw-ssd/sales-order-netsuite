## ADDED Requirements

### Requirement: Email 通知
系統透過 email 發送特定業務通知。

#### Scenario: 發送通知
- **WHEN** 特定業務事件觸發（如訂單狀態變更）
- **THEN** 系統透過 nikoksr/notify 發送 email 通知相關人員

#### Scenario: 測試發送
- **WHEN** POST `/api/v1/email-test`
- **THEN** 系統發送測試郵件確認設定正常

### Requirement: 系統事件（Event Bus）
系統使用 gookit/event 實現內部事件發布與訂閱。

#### Scenario: 領域事件發布
- **WHEN** 建立/修改關鍵實體
- **THEN** 系統透過 Event Bus 發布領域事件
- **THEN** 訂閱者（NetSuite sync, Email notification）非同步處理
