# Settings Frontend Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** 前端新增設定頁（`/admin/setting`）檢視/編輯系統設定（secret 遮罩、superadmin 限定），並將 `src/constant/options.ts` 的 11 個同步常數使用點改為 backend 執行期資料（`useSettings()`，loading 期間回退 `FALLBACK_SETTINGS`）。

**Architecture:** tygo 產出的 `SettingDTO` 對應 `src/models/settings.ts` 介面；`src/lib/setting/` 提供 request builders + TanStack Query `settingsQuery()`/`updateSettings()` + `useSettings()` hook（fallback）；設定頁為分組表單，secret 欄位遮罩顯示、null 語意提交；sidemenu 啟用「設定管理」。

**Tech Stack:** SolidJS 1.9, TanStack Solid Query 5, TailwindCSS, Kobalte/Ark UI 基礎元件（`~/components/ui/*`）。

## Global Constraints

- 所有新程式碼置於 `sales-order-frontend/`（submodule）。
- JSON key 一律 snake_case，與 backend DTO（spec §2 / `frontend_types/settings.ts`）一致；secret 欄位型別 `string | null`。
- 依賴 backend API 契約（backend plan Task 3/5/6/8 產出）：`GET/PUT /api/v1/settings`、遮罩格式 `••••<末4>`、PUT secret `null`=不變。
- loading 期間一律使用 `FALLBACK_SETTINGS`（值：vendor 807、department 6 — 修正後的種子值）。
- 產生/手寫型別入版控；`options.ts` 保留為 fallback 來源，不刪除。
- 每個 task 結束需 commit（`sales-order-frontend/` submodule 內）。

---

### Task 1: Settings 型別 + barrel

**Files:**
- Create: `sales-order-frontend/src/models/settings.ts`
- Modify: `sales-order-frontend/src/models/index.ts`

**Interfaces:**
- Produces: `Settings` interface（欄位與 backend `SettingDTO` 完全對應，secrets `string | null`）、`SecretKeys` 常數（6 個 secret key）。

- [ ] **Step 1: 建立 settings.ts**

```ts
// 對應 backend internal/domain/settings/model.go SettingDTO（tygo: frontend_types/settings.ts）
export interface Settings {
  default_department_id: number;
  approval: number;
  system_department_id: number;
  system_salesrep_id: number;
  system_test_salesrep_id: number;
  system_customer_id: number;
  system_customer_address_id: number;
  system_customer_address_entity_address_id: number;
  system_customer_contact_id: number;
  system_customer_entity_id: string;
  system_customer_name: string;
  default_vendor_id: number;
  default_vendor_name: string;
  default_salesrep_id: number;
  temp_car_number: number;
  company_admin_salesrep_id: number;
  about_url: string;
  support_email: string;
  default_timeout: number;
  frontend_url: string;
  // secrets（GET 為遮罩字串；PUT null = 不變）
  netsuite_account_id: string | null;
  netsuite_consumer_key: string | null;
  netsuite_consumer_secret: string | null;
  netsuite_token_id: string | null;
  netsuite_token_secret: string | null;
  email_password: string | null;
  // email 非 secret
  email_host: string;
  email_port: string;
  email_identity: string;
  email_username: string;
  email_from: string;
}

export const SECRET_KEYS = [
  "netsuite_account_id",
  "netsuite_consumer_key",
  "netsuite_consumer_secret",
  "netsuite_token_id",
  "netsuite_token_secret",
  "email_password",
] as const;
```

- [ ] **Step 2: barrel 匯出**

`src/models/index.ts` 最後一行後新增：

```ts
export * from "./settings";
```

- [ ] **Step 3: 驗證 typecheck**

Run: `cd sales-order-frontend && pnpm exec tsc --noEmit`
Expected: PASS。

- [ ] **Step 4: Commit**

```bash
cd sales-order-frontend && git add src/models && git commit -m "feat(models): add Settings types"
```

---

### Task 2: API 路徑 + request builders

**Files:**
- Modify: `sales-order-frontend/src/constant/api.ts`
- Create: `sales-order-frontend/src/lib/setting/setting.ts`

**Interfaces:**
- Consumes: `ApiPathUrl`（api.ts）、`Settings`（Task 1）。
- Produces: `getSettingsReq()` / `updateSettingsReq(body: Settings): Promise<Settings>`（`src/lib/setting/setting.ts`）。

- [ ] **Step 1: api.ts 加路徑**

`src/constant/api.ts` 的 `ApiPathUrl` 物件內（`putDispatchesUpdate` 之後）新增：

```ts
  // settings
  getSettings: apiVersion + "/settings",
  putSettings: apiVersion + "/settings",
```

- [ ] **Step 2: 建立 setting.ts**

```ts
import { ApiPathUrl } from "~/constant";
import { Settings } from "~/models";
import { apiReq, genParams } from "../requests";

const { getSettings, putSettings } = ApiPathUrl;

export const useSettingRequest = () => {
  const getSettingsReq = async (): Promise<Settings> => {
    const resp = await apiReq(getSettings, "GET");
    if (!resp.ok) throw await resp.json();
    return resp.json();
  };

  const updateSettingsReq = async (body: Settings): Promise<Settings> => {
    const resp = await apiReq(putSettings, "PUT", body as unknown as Record<string, unknown>);
    if (!resp.ok) throw await resp.json();
    return resp.json();
  };

  return { getSettingsReq, updateSettingsReq };
};
```

（`apiReq` 簽名依 `src/lib/requests/utils.ts` 實際匯出調整 — 需確認其參數順序與回傳型別；若 `apiReq` 不直接可用，改用既有 `fetchRequest` 模式，參照 `src/lib/customer/customer.ts`。）

- [ ] **Step 3: 驗證 typecheck**

Run: `cd sales-order-frontend && pnpm exec tsc --noEmit`
Expected: PASS。

- [ ] **Step 4: Commit**

```bash
cd sales-order-frontend && git add src/constant/api.ts src/lib/setting/setting.ts && git commit -m "feat(setting): add API request builders"
```

---

### Task 3: queryOptions + useSettings（fallback）

**Files:**
- Create: `sales-order-frontend/src/lib/setting/index.ts`
- Modify: `sales-order-frontend/src/constant/options.ts`（改為 fallback 常數，值修正）

**Interfaces:**
- Consumes: `useSettingRequest`（Task 2）、`Settings`/`SECRET_KEYS`（Task 1）、`clientEnv`。
- Produces: `settingsQuery()`、`useSettings(): () => Settings`（query 無資料時回傳 `FALLBACK_SETTINGS`）、`FALLBACK_SETTINGS`（options.ts）。

- [ ] **Step 1: options.ts 改為 fallback**

將 `sales-order-frontend/src/constant/options.ts` 整個內容替換為：

```ts
import { Settings } from "~/models";
import { clientEnv } from "~/env";

// 編譯期 fallback（backend settings API 不可用時使用；值與 backend seed 一致）
export const FALLBACK_SETTINGS: Settings = {
  default_department_id: 6,
  approval: 4,
  system_department_id: -16888,
  system_salesrep_id: -16888,
  system_test_salesrep_id: -17888,
  system_customer_id: -17888,
  system_customer_address_id: -17888,
  system_customer_address_entity_address_id: -17888,
  system_customer_contact_id: -17888,
  system_customer_entity_id: "SW17888",
  system_customer_name: "系統測試客戶",
  default_vendor_id: 807, // 修正：原 16 → backend seeder 實際建立之客戶 ID
  default_vendor_name: "樹森開發股份有限公司",
  default_salesrep_id: 119,
  temp_car_number: 1,
  company_admin_salesrep_id: -5,
  about_url: "https://www.hexagonty.com",
  support_email: "hexagon@hexagonty.com",
  default_timeout: 30,
  frontend_url: `https://${clientEnv().VITE_SELF_URL}`,
  netsuite_account_id: null,
  netsuite_consumer_key: null,
  netsuite_consumer_secret: null,
  netsuite_token_id: null,
  netsuite_token_secret: null,
  email_password: null,
  email_host: "smtp.gmail.com",
  email_port: "587",
  email_identity: "",
  email_username: "",
  email_from: "",
};
```

（`DEFAULT_DEPARTMENT` 等舊具名常數由各使用點改用 `useSettings()().<field>`；`DEEPLINK_COMPANY_URL` 由 `useSettings()().frontend_url` 取代。）

- [ ] **Step 2: 建立 index.ts**

```ts
import { useMutation, useQuery, useQueryClient } from "@tanstack/solid-query";
import { FALLBACK_SETTINGS } from "~/constant/options";
import { Settings } from "~/models";
import { useSettingRequest } from "./setting";

const { getSettingsReq, updateSettingsReq } = useSettingRequest();

export const settingsQuery = () => ({
  queryKey: ["settings"] as const,
  queryFn: getSettingsReq,
  staleTime: 5 * 60 * 1000,
});

// 元件內使用：const settings = useSettings(); settings().default_department_id
export const useSettings = (): (() => Settings) => {
  const query = useQuery(settingsQuery());
  return () => query.data ?? FALLBACK_SETTINGS;
};

export const useUpdateSettings = () => {
  const queryClient = useQueryClient();
  const mutation = useMutation({
    mutationFn: updateSettingsReq,
    onSuccess: (data) => {
      queryClient.setQueryData(["settings"], data);
    },
  });
  return mutation;
};
```

（依專案實際慣例：若既有 domain 使用 `createQuery` 而非 `useQuery`，請對照 `src/lib/department/index.ts` 的 queryOptions 型別調整 import。）

- [ ] **Step 3: 驗證 typecheck**

Run: `cd sales-order-frontend && pnpm exec tsc --noEmit`
Expected: PASS（此時舊使用點仍引用已刪除的具名常數 → 會報錯；先以 Task 6-8 的遷移消除。若需中途綠燈，可暫時保留舊 export 於 options.ts 註解標記 deprecated。）

- [ ] **Step 4: Commit**

```bash
cd sales-order-frontend && git add src/lib/setting/index.ts src/constant/options.ts && git commit -m "feat(setting): add query hook with fallback constants"
```

---

### Task 4: 設定頁（分組表單 + 遮罩 + superadmin gate）

**Files:**
- Modify: `sales-order-frontend/src/pages/admin/setting/Setting.tsx`（替換空 stub）
- Modify: `sales-order-frontend/src/routes/admin/setting.tsx`（loader 預取）

**Interfaces:**
- Consumes: `useSettings`/`useUpdateSettings`（Task 3）、`SECRET_KEYS`（Task 1）、`useAuth`（角色判斷）。
- Produces: 完整設定頁；`route.tsx` loader 加 `ensureQueryData(settingsQuery())`。

- [ ] **Step 1: 設定頁 loader**

`src/routes/admin/setting.tsx` 的 Route 定義改為：

```tsx
import { createFileRoute, linkOptions } from "@tanstack/solid-router";
import { settingsQuery } from "~/lib/setting";

export const Route = createFileRoute("/admin/setting")({
  loader: ({ context: { queryClient } }) => {
    queryClient.ensureQueryData(settingsQuery());
  },
});
```

（`context.queryClient` 存在性以 `src/routes/admin/department.tsx` 為準調整。）

- [ ] **Step 2: 實作 Setting.tsx**

```tsx
import { Component, For, Show, createMemo, createSignal } from "solid-js";
import { Button } from "~/components/ui/button";
import { Label } from "~/components/ui/label";
import { Input } from "~/components/ui/input";
import { toast } from "solid-sonner";
import { SECRET_KEYS, Settings } from "~/models";
import { FALLBACK_SETTINGS } from "~/constant/options";
import { useSettings, useUpdateSettings } from "~/lib/setting";
import { useAuth } from "~/pages/auth/context";

type FieldDef = { key: keyof Settings; label: string; secret?: boolean; type?: string };

const GROUPS: { title: string; fields: FieldDef[] }[] = [
  {
    title: "系統常數",
    fields: [
      { key: "default_department_id", label: "預設部門 ID", type: "number" },
      { key: "approval", label: "核准狀態", type: "number" },
      { key: "system_department_id", label: "系統管理員部門 ID", type: "number" },
      { key: "system_salesrep_id", label: "系統管理員業務 ID", type: "number" },
      { key: "system_test_salesrep_id", label: "系統測試業務 ID", type: "number" },
      { key: "system_customer_id", label: "系統測試客戶 ID", type: "number" },
      { key: "system_customer_address_id", label: "系統客戶地址 ID", type: "number" },
      { key: "system_customer_address_entity_address_id", label: "系統客戶實體地址 ID", type: "number" },
      { key: "system_customer_contact_id", label: "系統客戶聯絡人 ID", type: "number" },
      { key: "system_customer_entity_id", label: "系統客戶實體 ID" },
      { key: "system_customer_name", label: "系統測試客戶名稱" },
      { key: "default_vendor_id", label: "預設供應商 ID", type: "number" },
      { key: "default_vendor_name", label: "預設供應商名稱" },
      { key: "default_salesrep_id", label: "預設業務 ID", type: "number" },
      { key: "temp_car_number", label: "暫存車牌號碼", type: "number" },
    ],
  },
  {
    title: "App 設定",
    fields: [
      { key: "company_admin_salesrep_id", label: "公司管理員業務 ID", type: "number" },
      { key: "about_url", label: "關於我們 URL" },
      { key: "support_email", label: "客服 Email" },
      { key: "default_timeout", label: "預設逾時（秒）", type: "number" },
    ],
  },
  {
    title: "前端 URL",
    fields: [{ key: "frontend_url", label: "前端站台 URL" }],
  },
  {
    title: "NetSuite 憑證",
    fields: [
      { key: "netsuite_account_id", label: "Account ID", secret: true },
      { key: "netsuite_consumer_key", label: "Consumer Key", secret: true },
      { key: "netsuite_consumer_secret", label: "Consumer Secret", secret: true },
      { key: "netsuite_token_id", label: "Token ID", secret: true },
      { key: "netsuite_token_secret", label: "Token Secret", secret: true },
    ],
  },
  {
    title: "EMAIL 設定",
    fields: [
      { key: "email_host", label: "SMTP Host" },
      { key: "email_port", label: "SMTP Port" },
      { key: "email_identity", label: "SMTP Identity" },
      { key: "email_username", label: "SMTP Username" },
      { key: "email_password", label: "SMTP Password", secret: true },
      { key: "email_from", label: "寄件人 Email" },
    ],
  },
];

const Setting: Component<{}> = () => {
  const settings = useSettings();
  const updateSettings = useUpdateSettings();
  const auth = useAuth();

  // superadmin 判斷：依 authState 角色（欄位名以實際 SessionInfoMe 型別為準）
  const isSuper = createMemo(() => auth.authState.info?.role === "super");
  const [form, setForm] = createSignal<Settings>({ ...FALLBACK_SETTINGS, ...settings() });

  const setField = (key: keyof Settings, value: string | number | null) =>
    setForm((f) => ({ ...f, [key]: value }));

  const save = async () => {
    try {
      await updateSettings.mutateAsync(form());
      toast.success("設定已儲存");
    } catch (e: any) {
      toast.error(e?.detail ?? "儲存失敗");
    }
  };

  return (
    <div class="p-6 space-y-6">
      <h1 class="text-xl font-bold">系統設定</h1>
      <For each={GROUPS}>
        {(group) => (
          <section class="rounded-lg border p-4 space-y-3">
            <h2 class="font-semibold text-lg">{group.title}</h2>
            <Show
              when={group.fields.some((f) => f.secret) && !isSuper()}
              fallback={
                <For each={group.fields}>
                  {(field) => (
                    <div class="grid grid-cols-[220px_1fr] items-center gap-2">
                      <Label>{field.label}</Label>
                      {field.secret ? (
                        <Input
                          type="password"
                          placeholder={form()[field.key] ? String(form()[field.key]) : "（未設定）"}
                          value={""}
                          onInput={(e) => setField(field.key, (e.target as HTMLInputElement).value || null)}
                        />
                      ) : (
                        <Input
                          type={field.type ?? "text"}
                          value={String(form()[field.key] ?? "")}
                          onInput={(e) => setField(field.key, (e.target as HTMLInputElement).value)}
                        />
                      )}
                    </div>
                  )}
                </For>
              }
            >
              <p class="text-sm text-muted-foreground">僅 superadmin 可編輯憑證欄位。</p>
            </Show>
          </section>
        )}
      </For>
      <Button onClick={save} disabled={updateSettings.isPending}>
        {updateSettings.isPending ? "儲存中…" : "儲存"}
      </Button>
    </div>
  );
};

export default Setting;
```

（secret 欄位輸入框 placeholder 顯示遮罩值、輸入框本身留空 = 提交 null = 不變；`<Input>`/`<Label>`/`<Button>` 以 `src/components/ui/` 實際存在的元件名稱為準調整。`useAuth` 的存取方式參照 `src/pages/auth/context.tsx`。）

- [ ] **Step 3: 驗證 typecheck + 啟動**

Run: `cd sales-order-frontend && pnpm exec tsc --noEmit && pnpm run build`
Expected: PASS（build 前需 Task 6-8 完成，否則 options.ts 舊使用點報錯；此 task 完成時可先只驗證 tsc 於本頁面檔案。）

- [ ] **Step 4: Commit**

```bash
cd sales-order-frontend && git add src/pages/admin/setting/Setting.tsx src/routes/admin/setting.tsx && git commit -m "feat(setting): implement settings page with masked secrets"
```

---

### Task 5: sidemenu 啟用「設定管理」

**Files:**
- Modify: `sales-order-frontend/src/constant/sidemenu.ts`

**Interfaces:**
- Consumes: `settingLinkOptions`（`src/routes/admin/setting.tsx` 既有 export）。
- Produces: systemMenus 出現「設定管理」項目。

- [ ] **Step 1: 取消註解 + 補 import**

`sidemenu.ts`：
1. 取消註解 `systemMenus` 內的「設定管理」項目：

```ts
    {
      linkOpts: settingLinkOptions("設定管理"),
      tooltip: "設定管理",
      icon: RiSystemSettingsFill,
      iconColor: textColor,
      color: textColor,
    },
```

2. 補 import：`import { settingLinkOptions } from "~/routes/admin/setting";` 與 `import { RiSystemSettingsFill } from "solid-icons/ri";`（若該 icon 不存在，改用既有 `solid-icons/ri` 或 `solid-icons/bs` 的 settings icon，例如 `RiSettings3Fill`）。

- [ ] **Step 2: 驗證 typecheck**

Run: `cd sales-order-frontend && pnpm exec tsc --noEmit`
Expected: PASS。

- [ ] **Step 3: Commit**

```bash
cd sales-order-frontend && git add src/constant/sidemenu.ts && git commit -m "feat(sidemenu): enable settings menu item"
```

---

### Task 6: options.ts 遷移 — dispatch 區域（5 檔）

**Files:**
- Modify: `sales-order-frontend/src/pages/admin/dispatch/widgets/boards.tsx`
- Modify: `sales-order-frontend/src/pages/admin/dispatch/widgets/card.tsx`
- Modify: `sales-order-frontend/src/pages/admin/dispatch/widgets/column.tsx`
- Modify: `sales-order-frontend/src/pages/admin/dispatch/widgets/setting-context.tsx`
- Modify: `sales-order-frontend/src/pages/admin/dispatch/widgets/sub-board.tsx`

**Interfaces:**
- Consumes: `useSettings`（Task 3）。
- Produces: 上述檔案不再 import options.ts 具名常數。

- [ ] **Step 1: boards.tsx**

- 刪 `import { DEFAULT_DEPARTMENT } from "~/constant";`
- 加 `import { useSettings } from "~/lib/setting";` 並在元件內 `const settings = useSettings();`
- 第 339 行 `d.id === DEFAULT_DEPARTMENT` → `d.id === settings().default_department_id`

- [ ] **Step 2: card.tsx**

- 刪 `import { TEMP_CAR_NUMBER } from "~/constant";`，加 `useSettings` 同上
- 第 308 行 `` `column:${TEMP_CAR_NUMBER}` `` → `` `column:${settings().temp_car_number}` ``

- [ ] **Step 3: column.tsx**

- 同 card.tsx；第 290、297 行 `` `column:${TEMP_CAR_NUMBER}` `` → `` `column:${settings().temp_car_number}` ``

- [ ] **Step 4: setting-context.tsx**

- import 中移除 `SYSTEM_ADMIN_DEPARTMENT`、`TEMP_CAR_NUMBER`（來自 `~/constant`），加 `useSettings`
- 第 396 行 `m.id !== SYSTEM_ADMIN_DEPARTMENT` → `m.id !== settings().system_department_id`
- 第 423 行 `card.custbody_hf_car_number !== TEMP_CAR_NUMBER` → `card.custbody_hf_car_number !== settings().temp_car_number`
- 若該檔案非元件（context provider），改用 `createQuery(settingsQuery())` 取代 hook，取 `query.data ?? FALLBACK_SETTINGS`

- [ ] **Step 5: sub-board.tsx**

- 刪 `import { DEFAULT_DEPARTMENT } from "~/constant";`，加 `useSettings`
- 第 48 行 `department()?.id === DEFAULT_DEPARTMENT` → `department()?.id === settings().default_department_id`

- [ ] **Step 6: 驗證**

Run: `cd sales-order-frontend && pnpm exec tsc --noEmit`
Expected: PASS。

- [ ] **Step 7: Commit**

```bash
cd sales-order-frontend && git add src/pages/admin/dispatch && git commit -m "refactor(dispatch): read settings from backend"
```

---

### Task 7: options.ts 遷移 — 訂單/客戶/共用（5 檔）

**Files:**
- Modify: `sales-order-frontend/src/pages/admin/sales-order/widgets/create-vendorreturn-form.tsx`
- Modify: `sales-order-frontend/src/pages/admin/sales-order/widgets/form/main-from.tsx`
- Modify: `sales-order-frontend/src/pages/admin/sales-order/widgets/form/vendor-main-from.tsx`
- Modify: `sales-order-frontend/src/pages/admin/customer/widgets/qrcode-form.tsx`
- Modify: `sales-order-frontend/src/components/datatable/DepartmentFilter.tsx`

**Interfaces:**
- Consumes: `useSettings`（Task 3）。
- Produces: 上述檔案不再 import options.ts 具名常數。

- [ ] **Step 1: create-vendorreturn-form.tsx**

- 刪 `import { DEFAULT_VENDOR_ID } from "~/constant";`，加 `useSettings`
- 第 44 行 `entity: DEFAULT_VENDOR_ID` → `entity: settings().default_vendor_id`

- [ ] **Step 2: main-from.tsx**

- import 中移除 `SYSTEM_ADMIN_DEPARTMENT`、`SYSTEM_ADMIN_SALESREP_ID`、`SYSTEM_TEST_SALESREP_ID`，加 `useSettings`
- 第 69-71 行條件 `n.opt_id !== SYSTEM_ADMIN_SALESREP_ID && n.opt_id !== SYSTEM_TEST_SALESREP_ID && n.department !== SYSTEM_ADMIN_DEPARTMENT` → 對應 `settings().system_salesrep_id` / `settings().system_test_salesrep_id` / `settings().system_department_id`

- [ ] **Step 3: vendor-main-from.tsx**

- import 中移除 `DEFAULT_VENDOR_ID`、`DEFAULT_VENDOR_NAME`、`SYSTEM_ADMIN_DEPARTMENT`、`SYSTEM_ADMIN_SALESREP_ID`、`SYSTEM_TEST_SALESREP_ID`，加 `useSettings`
- 第 51-56 行 `defaultVendor` 物件的 `id/opt_id: DEFAULT_VENDOR_ID` → `settings().default_vendor_id`，`name/label: DEFAULT_VENDOR_NAME` → `settings().default_vendor_name`
- 第 74-78 行銷售代表排除條件 → 對應 `settings()` 欄位
- 第 91 行 `field().setValue(DEFAULT_VENDOR_ID)` → `field().setValue(settings().default_vendor_id)`

- [ ] **Step 4: qrcode-form.tsx**

- 刪 `import { DEEPLINK_COMPANY_URL } from "~/constant";`，加 `useSettings`
- 第 31 行 `` `${DEEPLINK_COMPANY_URL}/${rd.entity_id}` `` → `` `${settings().frontend_url}/customer_account_qrcode/${rd.entity_id}` ``

- [ ] **Step 5: DepartmentFilter.tsx**

- 刪 `import { SYSTEM_ADMIN_DEPARTMENT } from "~/constant/options";`，加 `useSettings`
- 第 39 行 `o.opt_id !== SYSTEM_ADMIN_DEPARTMENT` → `o.opt_id !== settings().system_department_id`
- 更新 `src/components/datatable/DepartmentFilter.test.tsx` 的 mock：加入 settings query mock（`vi.mock("~/lib/setting", ...)` 回傳 fallback；或將元件改為接受 `systemDepartmentId` prop 並於測試注入）— 以既有 `vi.hoisted` mock 模式為準。

- [ ] **Step 6: 驗證**

Run: `cd sales-order-frontend && pnpm exec tsc --noEmit && pnpm run test`
Expected: PASS（含 DepartmentFilter 既有測試）。

- [ ] **Step 7: Commit**

```bash
cd sales-order-frontend && git add src/pages/admin/sales-order src/pages/admin/customer src/components/datatable && git commit -m "refactor(sales-order,customer): read settings from backend"
```

---

### Task 8: options.ts 遷移 — auth context（1 檔）

**Files:**
- Modify: `sales-order-frontend/src/pages/auth/context.tsx`

**Interfaces:**
- Consumes: `useSettings`（Task 3）。
- Produces: auth context 不再 import options.ts 具名常數。

- [ ] **Step 1: context.tsx**

- 刪 `import { SYSTEM_ADMIN_SALESREP_ID } from "~/constant";`，加 `useSettings`
- 第 120 行 `id === SYSTEM_ADMIN_SALESREP_ID` → `id === settings().system_salesrep_id`
- 注意：此為 Provider 元件，`useSettings` 為 hook，確保呼叫位置合法（Provider 內層）。

- [ ] **Step 2: 驗證 + 全量檢查**

Run: `cd sales-order-frontend && grep -rn "SYSTEM_ADMIN_DEPARTMENT\|DEFAULT_DEPARTMENT\|TEMP_CAR_NUMBER\|DEFAULT_VENDOR_ID\|DEFAULT_VENDOR_NAME\|SYSTEM_ADMIN_SALESREP_ID\|SYSTEM_TEST_SALESREP_ID\|DEEPLINK_COMPANY_URL\|DEFAULT_SALESREP_ID" src --include=*.ts --include=*.tsx`
Expected: 僅 `src/constant/options.ts` 內殘留（FALLBACK_SETTINGS 內部無上述具名常數 → 預期 0 結果）。

Run: `pnpm exec tsc --noEmit && pnpm run build && pnpm run test`
Expected: 全 PASS。

- [ ] **Step 3: Commit**

```bash
cd sales-order-frontend && git add src/pages/auth/context.tsx && git commit -m "refactor(auth): read system salesrep id from settings"
```

---

### Task 9: 設定頁測試（vitest）

**Files:**
- Create: `sales-order-frontend/src/pages/admin/setting/Setting.test.tsx`

**Interfaces:**
- Consumes: `Setting` 元件（Task 4）、mock `~/lib/setting`。

- [ ] **Step 1: 寫測試**

```tsx
import { render, screen } from "@solidjs/testing-library";
import { beforeEach, describe, expect, it, vi } from "vitest";
import { FALLBACK_SETTINGS } from "~/constant/options";
import { Setting } from "./Setting";

vi.mock("~/lib/setting", () => ({
  useSettings: () => () => FALLBACK_SETTINGS,
  useUpdateSettings: () => ({
    mutateAsync: vi.fn().mockResolvedValue(FALLBACK_SETTINGS),
    isPending: false,
  }),
}));

vi.mock("~/pages/auth/context", () => ({
  useAuth: () => ({ authState: { info: { role: "super" } } }),
}));

describe("Setting page", () => {
  it("renders masked secret placeholders", async () => {
    render(() => <Setting />);
    // NetSuite Account ID 輸入框 placeholder 顯示遮罩值（fallback 為 null → 「（未設定）」）
    expect(await screen.findByText("NetSuite 憑證")).toBeTruthy();
    expect(screen.getByText("（未設定）")).toBeTruthy();
  });
});
```

（以 `@solidjs/testing-library` 既有用法調整；若元件為 default export 則 `import Setting from "./Setting"`。）

- [ ] **Step 2: 執行**

Run: `cd sales-order-frontend && pnpm run test`
Expected: PASS。

- [ ] **Step 3: Commit**

```bash
cd sales-order-frontend && git add src/pages/admin/setting/Setting.test.tsx && git commit -m "test(setting): add settings page test"
```

---

### Task 10: 文件 + 收尾

**Files:**
- Modify: `docs/AGENTS/frontend.md`（superproject root repo commit）

**Interfaces:**
- Produces: 文件說明設定頁與 options.ts 遷移。

- [ ] **Step 1: 更新 frontend 指引**

`docs/AGENTS/frontend.md`：於資料取得模式補 `src/lib/setting/`；「Agent 常見任務」補設定頁範例；說明 `src/constant/options.ts` 已改為 fallback 常數（`FALLBACK_SETTINGS`）。

- [ ] **Step 2: 冒煙驗證**

Run: `cd sales-order-frontend && pnpm run dev`（背景）→ browser 開啟 `/admin/setting`，確認登入後顯示分組表單、NetSuite 憑證區塊遮罩；非 superadmin 帳號登入時憑證區塊顯示「僅 superadmin 可編輯」。
（若無測試帳號，以 `pnpm run build && pnpm run test` 通過為準。）

- [ ] **Step 3: Commit**

```bash
git add docs/AGENTS/frontend.md && git commit -m "docs(frontend): add settings page guide"
```
（於 superproject root 執行。）
