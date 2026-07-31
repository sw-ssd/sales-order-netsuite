## 1. Add disabled rules to reset password buttons

- [x] 1.1 **salesrep** — `disabled={!isManager() || isSelf}`（管理員才可重置，且不能重置自己）
- [x] 1.2 **customer** — `disabled={!isManager() && !isAssignedSalesrep}`（所屬業務與管理員可操作）

## 2. Verification

- [x] 2.1 `pnpm build` — frontend builds without errors
- [x] 2.2 Visual check: reset password button is disabled for non-eligible users in salesrep/customer rows
