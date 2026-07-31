## Context

業務員（Salesrep）密碼若遺忘，需由管理員強制重設。後端 repo 層 `ResetNullPassword` 已完整支援三種角色：

- `UTYPE_CUSTOMER` → clear password + set is_verified=false
- `UTYPE_SALESREP` → clear password + set is_verified=false
- `UTYPE_USER` → clear password + set is_verified=false

但 handler `ResetNullPassword` 的 utype switch 僅接受 `customer`，其餘回傳 `ErrInvalidUserType`。

前端 Customer 管理頁面已有「重置密碼」按鈕（`TableSheetButton` + `resetNullPasswordMutater`），但 Salesrep 管理頁面無對應功能。

## Goals / Non-Goals

**Goals:**
- 管理員可對業務員執行密碼強制重設（清除密碼 → 業務員下次以 `newset-password` 流程設定新密碼）
- 管理員後台業務員列表資料列顯示「重置密碼」按鈕

**Non-Goals:**
- 不新增自助密碼重設流程（forgot-password / OTP / Email）
- 不變更 Customer 或 User 現有行為
- 不新建 API 端點（重用既有 `reset-null-password`）

## Decisions

### Decision: Handler 直接擴充 utype switch

現有 handler 增加 `constants.UTYPE_SALESREP` 與 `constants.UTYPE_USER` case。repo 與 usecase 已有完整實作，無需修改。

**Alternatives considered:**
- 新增獨立 Salesrep 專用端點 → 重複路由、偏離既有設計。捨棄。

### Decision: Frontend 比照 Customer 實作

`reset-null-password` 的 PATCH 請求已在 `lib/auth/requests.ts` 的 `patchResetNullPasswordRequest` 存在。只需：
1. 在 `lib/salesrep.ts`（或類似模組）加入 API query
2. 在 `SalesrepDatatableContext` 加入 mutation
3. 在 `columns.tsx` 加入「重置密碼」按鈕

## Risks / Trade-offs

- **操作後業務員需立即設定新密碼**：重設密碼後 credential 的 `password` 被清空、`is_verified` 設為 false。業務員下次登入時會進入 `newset-password` 流程，若不知 email 或系統異常則無法恢復。
- **無確認對話框**：比照 Customer 實作，點擊即觸發。可考慮後續加入二次確認。
