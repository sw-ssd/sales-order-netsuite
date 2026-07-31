## 1. Backend — Handler

- [x] 1.1 Expand `ResetNullPassword` handler switch to accept `constants.UTYPE_SALESREP` and `constants.UTYPE_USER`

## 2. Frontend — API Layer

- [x] 2.1 Add `resetSalesrepNullPassword` API call function in `lib/salesrep/` (pattern: `resetNullPasswordCustomerByIdQuery` in `lib/customer/`)

## 3. Frontend — Datatable Context

- [x] 3.1 Import/resolve `patchResetNullPasswordRequest` (already in `lib/auth/requests.ts`) for Salesrep datatable
- [x] 3.2 Add `resetNullPasswordMutater` to SalesrepDatatableContext (pattern: CustomerDatatableContext)

## 4. Frontend — Datatable Columns

- [x] 4.1 Add 「重置密碼」`TableSheetButton` in salesrep columns (pattern: customer columns, import `BiRegularReset` icon)
- [x] 4.2 Wire button click to `resetNullPasswordMutater.mutate(row.getValue("id"))`

## 5. Verification

- [x] 5.1 Run backend tests
- [x] 5.2 Run frontend build/lint
- [x] 5.3 Admin UI smoke test: click reset, verify salesrep can login → newset-password flow succeeds
