## 1. Replace Checkbox with shared Button component

- [x] 1.1 **salesrep** — Replace Checkbox+Label with `TableDeleteListButton` in `rud_acts` header
- [x] 1.2 **customer** — Replace Checkbox+TableColumnHeader with `TableDeleteListButton`
- [x] 1.3 **department** — Replace Checkbox+TableColumnHeader with `TableDeleteListButton`
- [x] 1.4 **estimate-item** — Replace Checkbox+TableColumnHeader with `TableDeleteListButton`
- [x] 1.5 **item** — Replace Checkbox+TableColumnHeader with `TableDeleteListButton`
- [x] 1.6 **metadict** — Replace Checkbox+TableColumnHeader with `TableDeleteListButton`
- [x] 1.7 **sales-order** — Replace Checkbox+TableColumnHeader with `TableDeleteListButton`
- [x] 1.8 **abstract** — Create `TableDeleteListButton` shared component in `~/components/datatable/`

## 2. Verification

- [x] 2.1 `pnpm build` — frontend builds without errors
- [x] 2.2 Spot-check: toggle button shows/hides soft-deleted records correctly in salesrep and customer pages
