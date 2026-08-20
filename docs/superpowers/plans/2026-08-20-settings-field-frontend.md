# Settings Field-Based Schema — Frontend Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** 前端設定頁改為 data-driven（欄位名稱/型別/說明來自 backend field rows），型別與 fallback 改為列陣列，superadmin 判定改為 email。

**Architecture:** `Settings` 型別改為 `{ fields: SettingsField[] }`；`FALLBACK_SETTINGS` 改為 31 列；`useSettings()` 回傳列陣列 + `settingsValue(field_id)` helper；頁面以 GROUPS（靜態 `群組 → field_id[]`）分組、渲染自 API rows；secret 遮罩/權限沿用，superadmin 改 email 判定。

**Tech Stack:** SolidJS 1.9, TanStack Solid Query 5, TailwindCSS, Kobalte/Ark UI。

## Global Constraints

- 所有程式碼置於 `sales-order-frontend/`（submodule，分支 `setting`；BASE = 現行實作 86f94fa）。
- 依賴 backend 新契約（backend plan Task 3/6/8）：`GET/PUT /api/v1/settings` 回傳/接收 `{ fields: [{field_id, name, field_type, desc, value}] }`；secret 遮罩 `••••末4碼`；PUT secret `value: null` = 不變。
- 型別：`SettingsField { field_id; name; field_type: "int64"|"string"|"secret"|"bool"; desc; value: string | null }`；`Settings { fields: SettingsField[] }`。
- superadmin：`authState.info?.user?.email === "ssd@sowinsoft.com"`（取代 `isSystemAdmin()`）。
- 數值欄位慣例沿用：raw string 輸入、save 時轉換、整數驗證、空白 trim = 不變；secret 留空 = null（不變）。
- 每個 task 結束 commit（frontend submodule 內）。

---

### Task 1: 型別改版 + FALLBACK_SETTINGS 列化

**Files:**
- Modify: `sales-order-frontend/src/models/settings.ts`
- Modify: `sales-order-frontend/src/constant/options.ts`

**Interfaces:**
- Produces: `SettingsField`、`Settings`、`FIELD_GROUPS: { title: string; fieldIds: string[] }[]`（5 群組，field_id 對應 spec §3）；`FALLBACK_SETTINGS: Settings`（31 列，值 = backend 種子）。

- [ ] **Step 1: 型別改版**

`src/models/settings.ts` 整個替換：

```ts
export type SettingFieldType = "int64" | "string" | "secret" | "bool";

export interface SettingsField {
  field_id: string;
  name: string;
  field_type: SettingFieldType;
  desc: string;
  value: string | null; // GET 遮罩；PUT null = 不變（secret）
}

export interface Settings {
  fields: SettingsField[];
}

// 前端靜態分組（label/type/desc 由 backend rows 提供）
export const FIELD_GROUPS: { title: string; fieldIds: string[] }[] = [
  {
    title: "系統常數",
    fieldIds: [
      "default_department_id", "approval", "system_department_id", "system_salesrep_id",
      "system_test_salesrep_id", "system_customer_id", "system_customer_address_id",
      "system_customer_address_entity_address_id", "system_customer_contact_id",
      "system_customer_entity_id", "system_customer_name", "default_vendor_id",
      "default_vendor_name", "default_salesrep_id", "temp_car_number",
    ],
  },
  {
    title: "App 設定",
    fieldIds: ["company_admin_salesrep_id", "about_url", "support_email", "default_timeout"],
  },
  { title: "前端 URL", fieldIds: ["frontend_url"] },
  {
    title: "NetSuite 憑證",
    fieldIds: ["netsuite_account_id", "netsuite_consumer_key", "netsuite_consumer_secret", "netsuite_token_id", "netsuite_token_secret"],
  },
  {
    title: "EMAIL 設定",
    fieldIds: ["email_host", "email_port", "email_identity", "email_username", "email_password", "email_from"],
  },
];
```

- [ ] **Step 2: FALLBACK_SETTINGS 列化**

`src/constant/options.ts` 整個替換（31 列，值與 backend seed 一致）：

```ts
import { Settings } from "~/models";
import { clientEnv } from "~/env";

// 編譯期 fallback（backend settings API 不可用時使用；值與 backend seed 一致）
export const FALLBACK_SETTINGS: Settings = {
  fields: [
    { field_id: "default_department_id", name: "預設部門 ID", field_type: "int64", desc: "NetSuite 預設部門", value: "6" },
    { field_id: "approval", name: "核准狀態", field_type: "int64", desc: "訂單核准狀態值", value: "4" },
    { field_id: "system_department_id", name: "系統管理員部門 ID", field_type: "int64", desc: "系統部門（排除於篩選）", value: "-16888" },
    { field_id: "system_salesrep_id", name: "系統管理員業務 ID", field_type: "int64", desc: "系統業務（排除於下拉）", value: "-16888" },
    { field_id: "system_test_salesrep_id", name: "系統測試業務 ID", field_type: "int64", desc: "測試業務（排除於下拉）", value: "-17888" },
    { field_id: "system_customer_id", name: "系統測試客戶 ID", field_type: "int64", desc: "種子客戶", value: "-17888" },
    { field_id: "system_customer_address_id", name: "系統客戶地址 ID", field_type: "int64", desc: "種子地址", value: "-17888" },
    { field_id: "system_customer_address_entity_address_id", name: "系統客戶實體地址 ID", field_type: "int64", desc: "種子實體地址", value: "-17888" },
    { field_id: "system_customer_contact_id", name: "系統客戶聯絡人 ID", field_type: "int64", desc: "種子聯絡人", value: "-17888" },
    { field_id: "system_customer_entity_id", name: "系統客戶實體 ID", field_type: "string", desc: "NetSuite Entity ID", value: "SW17888" },
    { field_id: "system_customer_name", name: "系統測試客戶名稱", field_type: "string", desc: "", value: "系統測試客戶" },
    { field_id: "default_vendor_id", name: "預設供應商 ID", field_type: "int64", desc: "供應商客戶", value: "807" },
    { field_id: "default_vendor_name", name: "預設供應商名稱", field_type: "string", desc: "", value: "樹森開發股份有限公司" },
    { field_id: "default_salesrep_id", name: "預設業務 ID", field_type: "int64", desc: "系統預設業務員", value: "119" },
    { field_id: "temp_car_number", name: "暫存車牌號碼", field_type: "int64", desc: "未分配車牌之派車", value: "1" },
    { field_id: "company_admin_salesrep_id", name: "公司管理員業務 ID", field_type: "int64", desc: "app 表單代換", value: "-5" },
    { field_id: "about_url", name: "關於我們 URL", field_type: "string", desc: "app profile 用", value: "https://www.hexagonty.com" },
    { field_id: "support_email", name: "客服 Email", field_type: "string", desc: "", value: "hexagon@hexagonty.com" },
    { field_id: "default_timeout", name: "預設逾時（秒）", field_type: "int64", desc: "", value: "30" },
    { field_id: "frontend_url", name: "前端站台 URL", field_type: "string", desc: "deeplink/manuals 用", value: `https://${clientEnv().VITE_SELF_URL}` },
    { field_id: "netsuite_account_id", name: "Account ID", field_type: "secret", desc: "NetSuite TBA 憑證", value: null },
    { field_id: "netsuite_consumer_key", name: "Consumer Key", field_type: "secret", desc: "NetSuite TBA 憑證", value: null },
    { field_id: "netsuite_consumer_secret", name: "Consumer Secret", field_type: "secret", desc: "NetSuite TBA 憑證", value: null },
    { field_id: "netsuite_token_id", name: "Token ID", field_type: "secret", desc: "NetSuite TBA 憑證", value: null },
    { field_id: "netsuite_token_secret", name: "Token Secret", field_type: "secret", desc: "NetSuite TBA 憑證", value: null },
    { field_id: "email_host", name: "SMTP Host", field_type: "string", desc: "", value: "smtp.gmail.com" },
    { field_id: "email_port", name: "SMTP Port", field_type: "string", desc: "字串儲存（config.Email.Port）", value: "587" },
    { field_id: "email_identity", name: "SMTP Identity", field_type: "string", desc: "", value: "" },
    { field_id: "email_username", name: "SMTP Username", field_type: "string", desc: "", value: "" },
    { field_id: "email_password", name: "SMTP Password", field_type: "secret", desc: "", value: null },
    { field_id: "email_from", name: "寄件人 Email", field_type: "string", desc: "", value: "" },
  ],
};
```

- [ ] **Step 3: 驗證 typecheck（此階段舊使用點會紅 — 預期，F4 修）**

Run: `cd sales-order-frontend && pnpm exec tsc --noEmit`
Expected: `src/models/settings.ts`、`src/constant/options.ts` 自身 0 錯誤；其他檔錯誤為舊型別使用點（F4 範圍）。

- [ ] **Step 4: Commit**

```bash
cd sales-order-frontend && git add src/models/settings.ts src/constant/options.ts && git commit -m "refactor(models): field-row Settings types and fallback"
```

---

### Task 2: hook helper（settingsValue）+ query 契約

**Files:**
- Modify: `sales-order-frontend/src/lib/setting/index.ts`

**Interfaces:**
- Produces: `settingsQuery()`（不變）；`useSettings(): () => SettingsField[]`；`useSettingsValue(): (field_id: string) => string | null`（fallback 自 FALLBACK_SETTINGS 對應列）；`useUpdateSettings()`（PUT `{fields}`，成功後 cache 更新）。

- [ ] **Step 1: index.ts 改版**

```ts
import { useMutation, useQuery, useQueryClient } from "@tanstack/solid-query";
import { FALLBACK_SETTINGS } from "~/constant/options";
import { Settings, SettingsField } from "~/models";
import { useSettingRequest } from "./setting";

const { getSettingsReq, updateSettingsReq } = useSettingRequest();

export const settingsQuery = () => ({
  queryKey: ["settings"] as const,
  queryFn: getSettingsReq,
  staleTime: 5 * 60 * 1000,
});

export const useSettings = (): (() => SettingsField[]) => {
  const query = useQuery(settingsQuery());
  return () => query.data?.fields ?? FALLBACK_SETTINGS.fields;
};

// 單一欄位值（fallback 自 FALLBACK_SETTINGS 同 field_id 列）
export const useSettingsValue = (): ((field_id: string) => string | null) => {
  const fields = useSettings();
  return (field_id: string) => {
    const row = fields().find((f) => f.field_id === field_id);
    return row ? row.value : null;
  };
};

export const useUpdateSettings = () => {
  const queryClient = useQueryClient();
  const mutation = useMutation({
    mutationFn: updateSettingsReq,
    onSuccess: (data: Settings) => {
      queryClient.setQueryData(["settings"], data);
    },
  });
  return mutation;
};
```

- [ ] **Step 2: 驗證 typecheck**

Run: `cd sales-order-frontend && pnpm exec tsc --noEmit`
Expected: 本檔 0 錯誤。

- [ ] **Step 3: Commit**

```bash
cd sales-order-frontend && git add src/lib/setting/index.ts && git commit -m "feat(setting): field-value hook helpers"
```

---

### Task 3: 設定頁 data-driven 改版 + superadmin email

**Files:**
- Modify: `sales-order-frontend/src/pages/admin/setting/Setting.tsx`

**Interfaces:**
- Consumes: `useSettings`/`useSettingsValue`/`useUpdateSettings`（Task 2）、`FIELD_GROUPS`（Task 1）、`useAuth`（email）。
- Produces: 頁面渲染自 API rows（label/type/desc）；secret 依 `field_type === "secret"` 判定；superadmin 依 email；保存送 `{fields}`。

- [ ] **Step 1: 頁面改版**

`Setting.tsx` 改為（保留數值 raw-string + save 轉換 + trim 慣例）：

```tsx
import { Component, For, createMemo, createSignal } from "solid-js";
import { toast } from "solid-sonner";
import { Button } from "~/components/ui/button";
import { Label } from "~/components/ui/label";
import { TextField, TextFieldInput } from "~/components/ui/text-field";
import { FIELD_GROUPS, SettingsField } from "~/models";
import { useSettings, useUpdateSettings } from "~/lib/setting";
import { useAuth } from "~/pages/auth/context";

const SUPERADMIN_EMAIL = "ssd@sowinsoft.com";
const isNumericType = (t: string) => t === "int64" || t === "bool";

const Setting: Component<{}> = () => {
  const fields = useSettings();
  const updateSettings = useUpdateSettings();
  const [authState] = useAuth(); // [AuthState, AuthFuncs] tuple

  // superadmin 依 email（與 backend 403 一致）
  const isSuper = createMemo(() => authState.info?.user?.email === SUPERADMIN_EMAIL);

  // form state：field_id → raw string（null = 未輸入/不變）
  const [form, setForm] = createSignal<Record<string, string | null>>({});
  const raw = (fieldId: string) => form()[fieldId] ?? fields().find((f) => f.field_id === fieldId)?.value ?? "";

  const setRaw = (fieldId: string, v: string | null) =>
    setForm((f) => ({ ...f, [fieldId]: v }));

  const save = async () => {
    const payload: SettingsField[] = [];
    for (const f of fields()) {
      const r = form()[f.field_id];
      const def = { ...f, value: null as string | null };
      if (f.field_type === "secret") {
        def.value = r == null || r === "" ? null : r; // null = 不變
      } else if (isNumericType(f.field_type)) {
        const trimmed = (r ?? "").trim();
        if (trimmed === "") {
          def.value = f.value; // 清空 = 沿用目前值
        } else {
          const n = Number(trimmed);
          if (!Number.isInteger(n)) {
            toast.error(`「${f.name}」需為整數`);
            return;
          }
          def.value = f.field_type === "bool" ? (trimmed === "true" ? "true" : "false") : String(n);
        }
      } else {
        def.value = r ?? f.value ?? "";
      }
      payload.push(def);
    }
    try {
      await updateSettings.mutateAsync({ fields: payload });
      toast.success("設定已儲存");
    } catch (e: any) {
      toast.error(e?.message ?? "儲存失敗");
    }
  };

  return (
    <div class="p-6 space-y-6">
      <h1 class="text-xl font-bold">系統設定</h1>
      <For each={FIELD_GROUPS}>
        {(group) => (
          <section class="rounded-lg border p-4 space-y-3">
            <h2 class="font-semibold text-lg">{group.title}</h2>
            <For each={group.fieldIds}>
              {(fieldId) => {
                const f = fields().find((x) => x.field_id === fieldId);
                if (!f) return <></>; // 未知 field_id 不渲染
                const secret = f.field_type === "secret";
                return (
                  <div class="grid grid-cols-[220px_1fr] items-center gap-2">
                    <Label>
                      {f.name}
                      <span class="block text-xs text-muted-foreground">{f.desc}</span>
                    </Label>
                    {secret ? (
                      <TextField>
                        <TextFieldInput
                          type="password"
                          disabled={!isSuper()}
                          placeholder={f.value ? f.value : "（未設定）"}
                          value={raw(fieldId) === f.value ? "" : (raw(fieldId) ?? "")}
                          onInput={(e) => setRaw(fieldId, (e.target as HTMLInputElement).value || null)}
                        />
                      </TextField>
                    ) : (
                      <TextField>
                        <TextFieldInput
                          type={isNumericType(f.field_type) ? "text" : "text"}
                          inputmode={isNumericType(f.field_type) ? "numeric" : undefined}
                          value={raw(fieldId) ?? ""}
                          onInput={(e) => setRaw(fieldId, (e.target as HTMLInputElement).value)}
                        />
                      </TextField>
                    )}
                  </div>
                );
              }}
            </For>
            <Show when={group.fieldIds.some((id) => fields().find((x) => x.field_id === id)?.field_type === "secret") && !isSuper()}>
              <p class="text-sm text-muted-foreground">僅 superadmin（ssd@sowinsoft.com）可編輯憑證欄位。</p>
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

（保留 secret 輸入框邏輯：`value` 綁定避開遮罩回寫 — 顯示遮罩於 placeholder、輸入框本身以 raw 值驅動；`Show` 需自 solid-js import。）

- [ ] **Step 2: 驗證 typecheck**

Run: `cd sales-order-frontend && pnpm exec tsc --noEmit`
Expected: 本檔 0 錯誤（舊使用點錯誤仍存在，F4 修）。

- [ ] **Step 3: Commit**

```bash
cd sales-order-frontend && git add src/pages/admin/setting/Setting.tsx && git commit -m "feat(setting): data-driven page with email superadmin gate"
```

---

### Task 4: 11 個使用點遷移（settingsValue）

**Files:**（與前版相同 11 檔）
- `src/components/datatable/DepartmentFilter.tsx`
- `src/pages/admin/customer/widgets/qrcode-form.tsx`
- `src/pages/admin/dispatch/widgets/{boards,card,column,setting-context,sub-board}.tsx`
- `src/pages/admin/sales-order/widgets/{create-vendorreturn-form,form/main-from,form/vendor-main-from}.tsx`
- `src/pages/auth/context.tsx`

**Interfaces:**
- Consumes: `useSettingsValue`（Task 2）。
- Produces: 上述檔案改用 `settingsValue("field_id")`（回傳 `string | null` → 使用點自行 `Number()` 或 `?? fallback`）。

- [ ] **Step 1: 逐檔替換**

每檔：import `useSettingsValue`（取代 `useSettings`），元件內 `const settingsValue = useSettingsValue();`；替換（**`settingsValue` 回傳 `string | null` — 數值處必須 `Number(settingsValue(id) ?? "<既有 fallback 常數值>")`，避免 `Number(null) === 0`**）：
- `appSettings().system_department_id` → `Number(settingsValue("system_department_id") ?? "-16888")`
- `appSettings().temp_car_number` → `Number(settingsValue("temp_car_number") ?? "1")`
- `appSettings().default_department_id` → `Number(settingsValue("default_department_id") ?? "6")`
- `appSettings().default_vendor_id` → `Number(settingsValue("default_vendor_id") ?? "807")`
- `appSettings().default_vendor_name` → `settingsValue("default_vendor_name") ?? "樹森開發股份有限公司"`
- `appSettings().system_salesrep_id` → `Number(settingsValue("system_salesrep_id") ?? "-16888")`
- `appSettings().system_test_salesrep_id` → `Number(settingsValue("system_test_salesrep_id") ?? "-17888")`
- `appSettings().frontend_url` → `settingsValue("frontend_url") ?? ""`（deeplink 組裝不變）

（fallback 字串值與各檔先前使用之常數一致；若該檔原本無 fallback（值必存在），仍以 `?? "<同值>"` 保護。）

- [ ] **Step 2: 驗證 + 全量**

Run: `cd sales-order-frontend && pnpm exec tsc --noEmit && pnpm run build && pnpm run test`
Expected: 全綠（tsc 0 errors、build 過、16 tests 過 — Setting.test.tsx 需依 Task 5 更新）。

- [ ] **Step 3: Commit**

```bash
cd sales-order-frontend && git add src && git commit -m "refactor: read settings via field-value helper"
```

---

### Task 5: 測試更新 + 文件

**Files:**
- Modify: `sales-order-frontend/src/pages/admin/setting/Setting.test.tsx`
- Modify: `docs/AGENTS/frontend.md`（superproject root 提交）

**Interfaces:**
- Produces: 測試改用 `{fields: [...]}` mock（FALLBACK_SETTINGS）；文件更新。

- [ ] **Step 1: 測試改版**

`Setting.test.tsx`：mock `~/lib/setting` 的 `useSettings: () => () => FALLBACK_SETTINGS.fields`、`useUpdateSettings` 同前；mock `~/pages/auth/context` 的 `useAuth: () => [{ info: { user: { email: "ssd@sowinsoft.com" } } }, {}]`（super 案例）與 email 非 ssd 案例（secret disabled + 提示文案「僅 superadmin（ssd@sowinsoft.com）」）。斷言：5 群組標題、masked placeholder（fallback secret null → 未設定）、save payload `{fields}` 且 secret null。

Run: `cd sales-order-frontend && pnpm run test`
Expected: 全 PASS。

- [ ] **Step 2: 文件**

`docs/AGENTS/frontend.md`：設定頁說明改為 data-driven（field rows）、superadmin email 判定、`useSettingsValue` helper。

- [ ] **Step 3: Commit**

```bash
cd sales-order-frontend && git add src/pages/admin/setting/Setting.test.tsx && git commit -m "test(setting): field-row page tests"
cd .. && git add docs/AGENTS/frontend.md && git commit -m "docs(frontend): field-based settings page guide"
```
