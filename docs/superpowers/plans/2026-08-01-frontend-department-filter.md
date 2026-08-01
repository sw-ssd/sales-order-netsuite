# Customer / Sales-Order 部門下拉過濾 Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Add a department dropdown filter to the customer and sales-order admin pages: non-managers get a disabled dropdown locked to their own department (list always filtered to it), managers get an enabled dropdown with 「全部部門」+ all active departments (default = own department).

**Architecture:** A shared `DepartmentFilter` component (ark-ui `Select`, driven by `useAuth()`'s `isManager`/`infoDepartment` + metadict `departments` options) is added to each page's `TableToolbar`. Selection changes trigger the pages' existing server-search mutation (`smu` → `fetchQuery(xxxQuery({...params}))`), sending `department_id` (customers) or `department` (sales-orders) — both backend filter params already exist. Initial page queries carry the department param derived from auth so non-managers never see all data.

**Tech Stack:** SolidJS, ark-ui (`~/components/ui/select.tsx`), TanStack Solid Query, vitest + @solidjs/testing-library (unit), Playwright (e2e).

## Global Constraints

- Work happens inside the `sales-order-frontend` git submodule. All `git` commands in tasks run with `cwd = sales-order-frontend`.
- Backend has NO changes (filter params `department_id` / `department` already exist and are implemented).
- Frontend-only UX (user-confirmed): API-level bypass is accepted for non-managers.
- Design spec: `docs/superpowers/specs/2026-08-01-frontend-department-filter-design.md`.
- Behavior contract (user-confirmed):
  - Non-manager: dropdown `disabled`, value locked to own department (`infoDepartment()`), `onChange` never fires; list query always carries the department param.
  - Manager: dropdown enabled, options = 「全部部門」(id 0) + active departments (exclude `is_inactive`), default = own department; selecting 全部 omits the param.
  - `infoDepartment()` missing → non-manager shows 「無部門」 disabled and no param; manager defaults to 全部.
- `isManager()` / `infoDepartment()` come from `useAuth()` actions (`src/pages/auth/context.tsx`).
- There are currently NO vitest tests in the repo — Task 1 creates the scaffold (vitest config + first test). Do not add new dependencies (jsdom and @solidjs/testing-library are already in devDependencies).
- The `Select` API follows the existing pattern in `src/components/form/FormSelectOptField.tsx` (props `options`, `optionValue`, `optionTextValue`, `optionDisabled`, `value` = option object, `onChange(option|null)`, `itemComponent`).
- Every task ends with a commit inside the submodule.

---

### Task 1: vitest scaffold + shared `DepartmentFilter` component + unit tests

**Files:**
- Create: `vitest.config.ts`
- Create: `src/components/datatable/DepartmentFilter.tsx`
- Create: `src/components/datatable/DepartmentFilter.test.tsx`

**Interfaces:**
- Produces: `DepartmentFilter` component with props `{ options: MetadictOption[]; value: number; onChange: (id: number) => void; showAll?: boolean }` (value `0` = 全部; `showAll` default `true`, `false` omits the 全部 option; inactive departments are shown but disabled/反白) — consumed by Task 2/3/4 page toolbars.

- [ ] **Step 1: Create the vitest config**

Create `vitest.config.ts`:

```ts
import { defineConfig } from "vitest/config";

export default defineConfig({
  test: {
    environment: "jsdom",
    include: ["src/**/*.test.{ts,tsx}"],
  },
});
```

- [ ] **Step 2: Write the failing component tests**

Create `src/components/datatable/DepartmentFilter.test.tsx`:

```tsx
import { describe, expect, it, vi } from "vitest";
import { render, screen } from "@solidjs/testing-library";
import { fireEvent } from "@solidjs/testing-library";
import { MetadictOption } from "~/models";
import { DepartmentFilter } from "./DepartmentFilter";

const mocks = vi.hoisted(() => ({
  isManager: vi.fn<() => boolean>(() => false),
  infoDepartment: vi.fn<() => number | undefined>(() => -16888),
}));

vi.mock("~/pages/auth/context", () => ({
  useAuth: () => [
    {},
    {
      isManager: mocks.isManager,
      infoDepartment: mocks.infoDepartment,
      isSystemAdmin: () => false,
    },
  ],
}));

const depts: MetadictOption[] = [
  { id: 1, opt_id: -16888, name: "Sowinsoft", is_inactive: false, table_name: "departments" },
  { id: 2, opt_id: 6, name: "Sales", is_inactive: false, table_name: "departments" },
  { id: 3, opt_id: 7, name: "Inactive Dept", is_inactive: true, table_name: "departments" },
];

describe("DepartmentFilter", () => {
  it("non-manager: disabled, locked to own department, no onChange", () => {
    const onChange = vi.fn();
    render(() => (
      <DepartmentFilter options={depts} value={-16888} onChange={onChange} />
    ));

    const trigger = screen.getByRole("button", { name: "Sowinsoft" });
    expect(trigger).toHaveProperty("disabled", true);
    expect(screen.queryByText("全部部門")).toBeNull();
    expect(onChange).not.toHaveBeenCalled();
  });

  it("non-manager without own department: shows 無部門 placeholder", () => {
    mocks.infoDepartment.mockReturnValue(undefined);
    render(() => <DepartmentFilter options={depts} value={0} onChange={vi.fn()} />);
    expect(screen.getByRole("button", { name: "無部門" })).toBeTruthy();
  });

  it("manager: default own department, 全部 + departments listed, onChange fires", async () => {
    mocks.isManager.mockReturnValue(true);
    mocks.infoDepartment.mockReturnValue(-16888);
    const onChange = vi.fn();
    render(() => (
      <DepartmentFilter options={depts} value={-16888} onChange={onChange} />
    ));

    // default selection shows own department
    expect(screen.getByRole("button", { name: "Sowinsoft" })).toBeTruthy();

    // open the select: 全部部門 + all departments (inactive shown too)
    await fireEvent.click(screen.getByRole("button", { name: "Sowinsoft" }));
    expect(screen.getByText("全部部門")).toBeTruthy();
    expect(screen.getByText("Sales")).toBeTruthy();
    expect(screen.getByText("Inactive Dept")).toBeTruthy(); // inactive visible

    // selecting 全部部門 fires onChange(0)
    await fireEvent.click(screen.getByText("全部部門"));
    expect(onChange).toHaveBeenCalledWith(0);
  });

  it("manager with showAll=false: no 全部部門 option", async () => {
    mocks.isManager.mockReturnValue(true);
    mocks.infoDepartment.mockReturnValue(-16888);
    render(() => (
      <DepartmentFilter options={depts} value={-16888} onChange={vi.fn()} showAll={false} />
    ));

    await fireEvent.click(screen.getByRole("button", { name: "Sowinsoft" }));
    expect(screen.queryByText("全部部門")).toBeNull();
    expect(screen.getByText("Sales")).toBeTruthy();
  });

  it("inactive department item is disabled (not selectable)", async () => {
    mocks.isManager.mockReturnValue(true);
    mocks.infoDepartment.mockReturnValue(-16888);
    const onChange = vi.fn();
    render(() => (
      <DepartmentFilter options={depts} value={-16888} onChange={onChange} />
    ));

    await fireEvent.click(screen.getByRole("button", { name: "Sowinsoft" }));
    const inactiveItem = screen.getByText("Inactive Dept");
    expect(inactiveItem).toHaveProperty("aria-disabled", "true");
    await fireEvent.click(inactiveItem);
    expect(onChange).not.toHaveBeenCalled();
  });
});
```

Note: ark-ui renders `SelectContent` in a portal — if `getByRole` queries race, use `await`/`waitFor` from `@solidjs/testing-library` around the open assertions.

- [ ] **Step 3: Run tests to verify they fail**

```bash
npm test -- DepartmentFilter
```

Expected: FAIL — `DepartmentFilter` module not found (component doesn't exist yet).

- [ ] **Step 4: Implement the component**

Create `src/components/datatable/DepartmentFilter.tsx`:

```tsx
import { Component, createMemo } from "solid-js";
import {
  Select,
  SelectContent,
  SelectItem,
  SelectTrigger,
  SelectValue,
} from "~/components/ui/select";
import { MetadictOption } from "~/models";
import { useAuth } from "~/pages/auth/context";

const ALL_OPTION: MetadictOption = {
  id: 0,
  opt_id: 0,
  name: "全部部門",
  is_inactive: false,
  table_name: "departments",
};

interface DepartmentFilterProps {
  /** departments metas（table_name === "departments"；inactive 也會顯示但不可選） */
  options: MetadictOption[];
  /** 目前選取的部門 id；0 = 全部 */
  value: number;
  onChange: (id: number) => void;
  /** 是否顯示「全部部門」選項；dispatch 頁傳 false */
  showAll?: boolean;
}

export const DepartmentFilter: Component<DepartmentFilterProps> = (props) => {
  const [, { isManager, infoDepartment }] = useAuth();

  const departments = () =>
    props.options.filter((o) => o.table_name === "departments");

  const manager = () => isManager();
  const ownDeptId = () => infoDepartment();

  const selectOptions = () => {
    if (manager()) {
      return props.showAll === false
        ? departments()
        : [ALL_OPTION, ...departments()];
    }
    const own = departments().find((o) => o.opt_id === ownDeptId());
    return own ? [own] : [];
  };

  const selected = createMemo(() => {
    if (manager()) {
      return selectOptions().find((o) => o.opt_id === props.value) ??
        (props.showAll === false ? selectOptions()[0] ?? null : ALL_OPTION);
    }
    return selectOptions()[0] ?? null;
  });

  const handleChange = (val: MetadictOption | null) => {
    if (manager() && val && !val.is_inactive) {
      props.onChange(val.opt_id);
    }
  };

  return (
    <Select
      value={selected()}
      options={selectOptions()}
      optionValue="opt_id"
      optionTextValue="name"
      optionDisabled="is_inactive"
      placeholder={manager() ? "全部部門" : "無部門"}
      onChange={handleChange}
      disabled={!manager()}
      itemComponent={(p) => (
        <SelectItem item={p.item} class={p.item.rawValue.is_inactive ? "opacity-50" : undefined}>
          {p.item.rawValue.name}
        </SelectItem>
      )}
    >
      <SelectTrigger aria-label="部門過濾" class="w-[180px]">
        <SelectValue<MetadictOption>>
          {(state) => state.selectedOption()?.name ?? (manager() ? "全部部門" : "無部門")}
        </SelectValue>
      </SelectTrigger>
      <SelectContent class="max-h-[calc(100svh-30rem)] overflow-y-auto" />
    </Select>
  );
};

export default DepartmentFilter;
```

- [ ] **Step 5: Run tests to verify they pass**

```bash
npm test -- DepartmentFilter
```

Expected: PASS (all 3 cases). If the ark-ui portal interaction is flaky in jsdom, adjust the tests to assert the trigger's `disabled` state and the `value` label only, and assert the manager options via the component's DOM after opening — but keep the `onChange` call assertion.

- [ ] **Step 6: Commit**

```bash
git add vitest.config.ts src/components/datatable/DepartmentFilter.tsx src/components/datatable/DepartmentFilter.test.tsx
git commit -m "feat: shared DepartmentFilter component with unit tests"
```

---

### Task 2: Customer page wiring

**Files:**
- Modify: `src/models/netsuite/customer.ts` (CustomerFilter)
- Modify: `src/routes/admin/customer.tsx` (CUSTOMER_METADICT_KEYS)
- Modify: `src/pages/admin/customer/widgets/context.tsx` (signal + smu wiring + expose via tableExActions)
- Modify: `src/pages/admin/customer/widgets/data-table.tsx` (render DepartmentFilter in toolbar)
- Modify: `src/pages/admin/customer/Customer.tsx` (initial query params from auth)

**Interfaces:**
- Consumes: `DepartmentFilter` (Task 1); `useAuth()` actions `infoDepartment`; backend param `department_id` (exists).
- Produces: customer list server-search with `department_id` param; initial list query filtered for non-managers.

- [ ] **Step 1: Add `department_id` to CustomerFilter**

In `src/models/netsuite/customer.ts`, after the `CustomerFilter` interface:

```ts
export interface CustomerFilter extends BaseFilter, Partial<Customer> {
  /** 部門過濾（後端 Nsfilter query=department_id） */
  department_id?: number;
}
```

- [ ] **Step 2: Add departments to metadict keys**

In `src/routes/admin/customer.tsx`, change `CUSTOMER_METADICT_KEYS` to include `"departments"`:

```ts
export const CUSTOMER_METADICT_KEYS: NsTableNames = [
  "currency",
  "term",
  "customlist_hf_invoice_type",
  "customlist_hf_cat_number",
  "customlist_hf_checkout_method",
  "customlist_hf_payment_methods",
  "customlist_hf_customer_type",
  "departments",
];
```

- [ ] **Step 3: Wire the filter in the customer context**

In `src/pages/admin/customer/widgets/context.tsx`:

1. The `Actions` interface — add:

```ts
interface Actions {
  getCustomerColumns: () => ColumnDef<any, any>[];
  getCustomerTable: () => Table<any>;
  debounceSetGlobalFilter: Scheduled<[value: string]>;
  departmentFilter: {
    value: () => number;
    setValue: (id: number) => void;
    options: () => MetadictOption[];
  };
}
```

2. In the provider, extend the existing `useAuth()` destructure and add the signal + handler (place near `smu`):

```tsx
  const [, { isSystemAdmin, infoDepartment }] = useAuth();

  const [departmentId, setDepartmentId] = createSignal<number>(
    infoDepartment() ?? 0,
  );

  const handleDepartmentChange = (id: number) => {
    setDepartmentId(id);
    smu.mutate(id === 0 ? {} : { department_id: id });
  };
```

3. In `FormActionsFeature.createTable`, expose it via `tableExActions` (next to the existing `getSalesrepOptions`):

```tsx
      table.tableExActions = {
        getSalesrepOptions,
        getMetadictForTableName,
        departmentFilter: {
          value: departmentId,
          setValue: handleDepartmentChange,
          options: () => props.metadictOptions?.data ?? [],
        },
      };
```

(If `getMetadictForTableName` is not already in `tableExActions`, add it alongside — it already exists in the provider scope.)

- [ ] **Step 4: Render the filter in the customer toolbar**

In `src/pages/admin/customer/widgets/data-table.tsx`:

1. Destructure `departmentFilter`:

```tsx
  const {
    createMutater,
    syncMutater,
    onChangeDate,
    tableStore: [store, setStore],
  } = table.tableActions!;
  const { departmentFilter } = table.tableExActions!;
```

2. Add the component to `TableToolbar` children (before the global-filter `TextField`):

```tsx
          <DepartmentFilter
            options={departmentFilter.options()}
            value={departmentFilter.value()}
            onChange={departmentFilter.setValue}
          />
```

3. Add the import at the top:

```tsx
import DepartmentFilter from "~/components/datatable/DepartmentFilter";
```

- [ ] **Step 5: Initial query carries the department param**

In `src/pages/admin/customer/Customer.tsx`:

1. Add imports: `useAuth` and `CustomerFilter`:

```tsx
import { CustomerFilter } from "~/models";
import { useAuth } from "~/pages/auth/context";
```

2. Replace the query creation (REVISED 2026-08-02: manager defaults to 全部 (0); only non-managers carry their own department — the original own-department default made system-admin users see a near-empty list):

```tsx
const [, { isManager, infoDepartment }] = useAuth();

const initialDepartmentId = () =>
  isManager() ? 0 : (infoDepartment() ?? 0);

const [departmentId, setDepartmentId] = createSignal<number>(
  initialDepartmentId(),
);

const query = useQuery(() =>
  customersQuery(departmentId() ? { department_id: departmentId() } : {}),
);
```

- [ ] **Step 6: Typecheck + unit tests still pass**

```bash
npx tsc --noEmit
npm test -- DepartmentFilter
```

Expected: no type errors; DepartmentFilter tests still PASS.

- [ ] **Step 7: Commit**

```bash
git add src/models/netsuite/customer.ts src/routes/admin/customer.tsx src/pages/admin/customer/
git commit -m "feat: department dropdown filter on customer page"
```

---

### Task 3: Sales-order page wiring

**Files:**
- Modify: `src/routes/admin/sales-order.tsx` (SALES_ORDER_METADICT_KEYS; drop unfiltered prefetch)
- Modify: `src/pages/admin/sales-order/widgets/context.tsx` (props + tableExActions exposure; NO internal signal/smu — see Task 2 fix)
- Modify: `src/pages/admin/sales-order/widgets/data-table.tsx` (render DepartmentFilter)
- Modify: `src/pages/admin/sales-order/SalesOrder.tsx` (department signal + reactive query + provider props)

**Interfaces:**
- Consumes: `DepartmentFilter` (Task 1); backend param `department` (exists on `SalesOrderFilter`).
- Produces: sales-order list query REACTIVE to the department signal (key change → refetch); initial list filtered for non-managers; no unfiltered loader prefetch.

**CORRECTED WIRING (follows the Task 2 Critical fix — do NOT use the smu-cache-write pattern):** the page component owns the department signal and the rendered query reads it, so a dropdown change refetches the table.

- [ ] **Step 1: Add departments to metadict keys**

In `src/routes/admin/sales-order.tsx`, uncomment `"departments"` in `SALES_ORDER_METADICT_KEYS` (and `"salesreps"` if the page needs it — check the page's existing usage; default: only `"departments"`).

- [ ] **Step 2: Drop the unfiltered prefetch in the loader**

In `src/routes/admin/sales-order.tsx`, remove `await queryClient.ensureQueryData(salesOrdersQuery());` (keep the deferred prefetches; drop the `salesOrdersQuery` import if it becomes unused).

- [ ] **Step 3: Lift the department signal into the page**

In `src/pages/admin/sales-order/SalesOrder.tsx` (mirror Task 2's fix — read `src/pages/admin/customer/Customer.tsx` for the exact committed pattern):

```tsx
const [, { infoDepartment }] = useAuth();

const [departmentId, setDepartmentId] = createSignal<number>(
  infoDepartment() ?? 0,
);

const query = useQuery(() =>
  salesOrdersQuery(departmentId() ? { department: departmentId() } : {}),
);
```

and pass `departmentValue={departmentId()}` + `onDepartmentChange={setDepartmentId}` to `SalesOrderDatatableProvider`. (Add imports: `createSignal` from solid-js, `useAuth`, `SalesOrderFilter` if referenced.)

- [ ] **Step 4: Wire the context (props-driven, no internal signal)**

In `src/pages/admin/sales-order/widgets/context.tsx` (mirror Task 2's fix — read the committed customer context):

1. Add `departmentValue: number` and `onDepartmentChange: (id: number) => void` to `DataTableProps`.
2. Add to the `Actions` interface:

```ts
  departmentFilter: {
    value: () => number;
    setValue: (id: number) => void;
    options: () => MetadictOption[];
  };
```

3. Expose `departmentFilter` in both the provider's `actions` object and `tableExActions`:

```tsx
      departmentFilter: {
        value: () => props.departmentValue,
        setValue: props.onDepartmentChange,
        options: () => props.metadictOptions?.data ?? [],
      },
```

Do NOT add a department signal or smu wiring in the context — the page's query already reacts to the signal.

- [ ] **Step 5: Render the filter in the sales-order toolbar**

In `src/pages/admin/sales-order/widgets/data-table.tsx` (mirror Task 2 Step 4): destructure `departmentFilter` from `table.tableExActions!`, add `<DepartmentFilter options={...} value={...} onChange={...} />` inside `TableToolbar` children, add the import.

- [ ] **Step 6: Typecheck**

```bash
npx tsc --noEmit
```

Expected: no NEW errors in touched files (9 pre-existing errors exist elsewhere on the branch).

- [ ] **Step 7: Commit**

```bash
git add src/routes/admin/sales-order.tsx src/pages/admin/sales-order/
git commit -m "feat: department dropdown filter on sales-order page"
```

---

### Task 4: Dispatch page conversion (tabs → dropdown)

**Files:**
- Modify: `src/pages/admin/dispatch/widgets/boards.tsx` (replace department Tabs with DepartmentFilter)
- Modify: `src/pages/admin/dispatch/widgets/setting-context.tsx` (departmentMetas: stop excluding inactive; keep SYSTEM_ADMIN_DEPARTMENT exclusion)
- Reference: `src/pages/admin/dispatch/Dispatch.tsx` (no change — data flow stays client-side)

**Interfaces:**
- Consumes: `DepartmentFilter` with `showAll={false}` (Task 1); existing `departmentMetas`, `departmentTabDefault`, `selectedDepartment`/`department` board state (unchanged semantics).
- Produces: dispatch page department selection via dropdown instead of tabs; inactive departments visible but 反白/disabled; non-manager locked to own department.

- [ ] **Step 1: Keep inactive departments in `departmentMetas`**

In `src/pages/admin/dispatch/widgets/setting-context.tsx`, change `departmentMetas` to keep `is_inactive` items (the component renders them 反白/disabled):

```tsx
  const departmentMetas = () =>
    metadictOptions?.data?.filter(
      (m) =>
        m.table_name === "departments" &&
        m.id !== SYSTEM_ADMIN_DEPARTMENT,
    ) || [];
```

(Removed `!m.is_inactive`; kept `m.id !== SYSTEM_ADMIN_DEPARTMENT`.)

- [ ] **Step 2: Replace the department Tabs with the dropdown**

First read `src/pages/admin/dispatch/widgets/boards.tsx` end-to-end (it is ~390 lines) to identify the exact `Tabs`/`TabsList`/`TabsTrigger` department-tab region, the `selectedDepartment`/`department` memos, and where `departmentMetas`/`infoDepartment`/`isManager`/`DEFAULT_DEPARTMENT`/`SYSTEM_ADMIN_DEPARTMENT` come from (some are provided by the settings context; import what is missing in `boards.tsx`).

Then replace the department Tabs block with a single dropdown. The board body and `department` memo stay untouched:

```tsx
  const departmentTabDefault = departmentMetas().find((d) =>
    infoDepartment() ? d.id === infoDepartment() : d.id === DEFAULT_DEPARTMENT,
  ) ?? departmentMetas()[0];

  const [selectedDepartmentId, setSelectedDepartmentId] = createSignal(
    departmentTabDefault?.id ?? 0,
  );

  const selectedDepartment = () =>
    departmentMetas().find((d) => d.id === selectedDepartmentId()) ??
    departmentTabDefault;

  return (
    <div class="flex items-center gap-2">
      <DepartmentFilter
        options={departmentMetas()}
        value={selectedDepartmentId()}
        onChange={(id) => setSelectedDepartmentId(id)}
        showAll={false}
      />
      …existing board content driven by selectedDepartment()…
    </div>
  );
```

Note: `DEFAULT_DEPARTMENT`, `SYSTEM_ADMIN_DEPARTMENT`, `infoDepartment`, and `isManager` must be reachable from `boards.tsx` (they already are in `setting-context.tsx` — import what `boards.tsx` lacks, or hoist the metas/defaults from the settings context as the existing code does). Remove the now-unused Tabs imports if nothing else uses them in this file. Add the import:

```tsx
import DepartmentFilter from "~/components/datatable/DepartmentFilter";
```

- [ ] **Step 3: Typecheck**

```bash
npx tsc --noEmit
```

Expected: no type errors.

- [ ] **Step 4: Commit**

```bash
git add src/pages/admin/dispatch/
git commit -m "feat: dispatch page department tabs to unified dropdown"
```

---

### Task 5: E2E + full verification

**Files:**
- Create: `e2e/department-filter.spec.cjs`
- No other changes unless fallout fixes are needed.

**Interfaces:** n/a — end-to-end contract check.

- [ ] **Step 1: Write the e2e spec**

Create `e2e/department-filter.spec.cjs` (mirrors the login pattern of `e2e/session-expiry.spec.cjs`):

```js
// @ts-check
const { test, expect } = require("playwright/test");

/**
 * 部門下拉過濾 E2E（docs/superpowers/specs/2026-08-01-frontend-department-filter-design.md）
 * 管理者（ssd@sowinsoft.com）場景完整驗證；非管理者場景需 fixture 帳號（見檔尾註解）。
 */

const BASE = "http://localhost:3000";
const API = "http://localhost:3080";

const MANAGER_EMAIL = "ssd@sowinsoft.com";
const MANAGER_PASSWORD = "sowinsoft#29157352";
// ssd 的部門：-16888 "Sowinsoft"
const MANAGER_DEPT_ID = -16888;
const MANAGER_DEPT_NAME = "Sowinsoft";

async function apiLogin(page, email, password) {
  const r = await page.request.post(
    `${API}/api/v1/authentication/login/salesrep`,
    {
      headers: { "Content-Type": "application/json" },
      data: { email, password },
    },
  );
  expect(r.status()).toBe(200);
}

/** 用 /me 的真實回傳注入 auth_state，讓前端 useAuth 讀得到 salesrep.department / is_manager */
async function seedFrontendAuth(page) {
  const me = await page.request.get(`${API}/api/v1/authentication/me`);
  expect(me.status()).toBe(200);
  const info = await me.json();
  await page.goto(`${BASE}/signin`);
  await page.evaluate(
    (info) => {
      localStorage.setItem(
        "auth_state",
        JSON.stringify({ isAuth: true, info }),
      );
    },
    info,
  );
  await page.reload();
  await page.waitForTimeout(1500);
}

test.describe("manager: department dropdown filter", () => {
  test.beforeEach(async ({ page }) => {
    await apiLogin(page, MANAGER_EMAIL, MANAGER_PASSWORD);
    await seedFrontendAuth(page);
  });

  test("customer page: dropdown enabled, defaults to own department, filters by department_id", async ({
    page,
  }) => {
    await page.goto(`${BASE}/admin/customer`);

    // 下拉預設 = 自己部門
    const trigger = page.getByRole("button", { name: MANAGER_DEPT_NAME });
    await expect(trigger).toBeVisible();
    await expect(trigger).toBeEnabled();

    // 開啟後有「全部部門」+ 部門選項
    await trigger.click();
    await expect(page.getByText("全部部門")).toBeVisible();

    // 切到「全部部門」→ 列表請求不含 department_id（攔截最後一次 list 請求）
    const listReq = page.waitForRequest(
      (r) =>
        r.url().includes("/api/v1/customers") && r.method() === "GET",
    );
    await page.getByText("全部部門").click();
    const req = await listReq;
    expect(req.url()).not.toContain("department_id=");
  });

  test("sales-order page: dropdown enabled, defaults to own department, filters by department", async ({
    page,
  }) => {
    await page.goto(`${BASE}/admin/sales-orders`);

    const trigger = page.getByRole("button", { name: MANAGER_DEPT_NAME });
    await expect(trigger).toBeVisible();
    await expect(trigger).toBeEnabled();

    await trigger.click();
    await expect(page.getByText("全部部門")).toBeVisible();
  });

  test("dispatch page: dropdown replaces tabs, no 全部 option, switches board", async ({
    page,
  }) => {
    await page.goto(`${BASE}/admin/dispatch`);

    // 下拉取代 tab：預設顯示部門（ssd 自己部門；若被 SYSTEM_ADMIN_DEPARTMENT 排除則為 DEFAULT_DEPARTMENT）
    const trigger = page.getByRole("button", { name: "部門過濾" });
    await expect(trigger).toBeVisible();
    await expect(trigger).toBeEnabled();

    await trigger.click();
    // dispatch 無「全部部門」選項
    await expect(page.getByText("全部部門")).toHaveCount(0);
  });
});

test.describe("non-manager: dropdown locked to own department", () => {
  // 需要 fixture 帳號：非管理者 salesrep（department = 自己的部門）。
  // 建立方式（一次性，於 dev DB 執行）：
  //   INSERT INTO salesreps (internal_id, last_modified_date, created_at, updated_at,
  //                          email, salesrep_account, is_inactive, is_salesrep, department_id)
  //   VALUES (90001, now(), now(), now(), 'dept-filter-user@example.com',
  //           'dept-filter-user@example.com', false, true, -16888);
  //   -- + credential（密碼雜湊，沿用 seeder 的 getHashPw 流程）→ 登入後 is_manager=false
  // 設定環境變數 DEPT_FILTER_USER / DEPT_FILTER_PASS 後此組測試才會執行。
  const email = process.env.DEPT_FILTER_USER;
  const password = process.env.DEPT_FILTER_PASS;

  test.skip(!email || !password, "DEPT_FILTER_USER/PASS 未設定");

  test("customer page: dropdown disabled and list filtered to own department", async ({
    page,
  }) => {
    await apiLogin(page, email, password);
    await seedFrontendAuth(page);
    await page.goto(`${BASE}/admin/customer`);

    const trigger = page.getByRole("button", { name: "部門過濾" });
    await expect(trigger).toBeVisible();
    await expect(trigger).toBeDisabled();
  });
});
```

- [ ] **Step 2: Run the full unit suite + typecheck + build**

Requires backend + frontend dev servers running for e2e; the first three commands are local:

```bash
npm test
npx tsc --noEmit
npm run build
```

Expected: all PASS (vitest), no type errors, build succeeds.

- [ ] **Step 3: Run the e2e spec** (servers must be up: backend on :3080, frontend on :3000)

```bash
npm run test:e2e -- e2e/department-filter.spec.cjs
```

Expected: manager scenarios PASS. The non-manager scenario is skipped unless `DEPT_FILTER_USER`/`DEPT_FILTER_PASS` are set.

- [ ] **Step 4: Commit**

```bash
git add e2e/department-filter.spec.cjs
git commit -m "test: e2e for department dropdown filter"
```

- [ ] **Step 5: Report**

Summarize: component behavior, both pages' wiring, filter params (`department_id` vs `department`), test results, and the documented fixture setup for the non-manager e2e.
