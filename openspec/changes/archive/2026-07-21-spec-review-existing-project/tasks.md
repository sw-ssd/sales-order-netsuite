## 1. Initialize Spec Directory

- [x] 1.1 Create `openspec/specs/` directory structure mirroring the 11 capability groups
- [x] 1.2 Define symlink/copy convention: spec source-of-truth lives under `openspec/specs/` (project-level), not change-level

## 2. Document Core Business Specs

- [x] 2.1 Write `openspec/specs/core-sales-order/spec.md` — SalesOrder, SalesOrderItem, Dispatch entities
- [x] 2.2 Write `openspec/specs/customer-management/spec.md` — Customer, AddressBook, Contact
- [x] 2.3 Write `openspec/specs/product-catalog/spec.md` — Item, EstimateItem
- [x] 2.4 Write `openspec/specs/salesrep-management/spec.md` — Salesrep

## 3. Document Auth & Infrastructure Specs

- [x] 3.1 Write `openspec/specs/auth-and-access/spec.md` — User, Role, Tenant, Credential, OTP, Provider, Session, CasbinRule
- [x] 3.2 Write `openspec/specs/dictionary-lists/spec.md` — 13 List* dictionary tables
- [x] 3.3 Write `openspec/specs/department-management/spec.md` — Department
- [x] 3.4 Write `openspec/specs/netsuite-sync/spec.md` — SuiteQL read, Record API write, gocron, event bus
- [x] 3.5 Write `openspec/specs/infrastructure-cron/spec.md` — Cron job tracking
- [x] 3.6 Write `openspec/specs/cms-article/spec.md` — Article CMS
- [x] 3.7 Write `openspec/specs/notification/spec.md` — Email notification, event bus

## 4. Create Design Document

- [x] 4.1 Write `design.md` — architecture overview, domain boundaries, cross-cutting sync logic, key decisions

## 5. Promote Specs to Project Level

- [x] 5.1 Copy final spec files from change-level `specs/` to project-level `openspec/specs/`
- [x] 5.2 Verify `openspec schemas spec-driven` lists the specs correctly
- [x] 5.3 Archive the change
