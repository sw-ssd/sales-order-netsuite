# Settings Field-Based Schema — App Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** App 的 SettingsModel 改為 field rows（`SettingsFieldModel[]`），SettingsService 內建 typed adapter（by field_id），9 個使用點語法不變。

**Architecture:** `SettingsModel` → rows 列表（field_id/name/field_type/desc/value）；Sembast cache 存 rows；`SettingsService` 提供 typed getters（`int? systemSalesrepId` 等，by field_id + 型別轉換），fallback 以 SystemConstants 組 adapter；登入/啟動 refresh + loadCache 不變。

**Tech Stack:** Flutter 3.35.2, freezed/json_serializable, dio, flutter_solidart, Sembast, GetIt。

## Global Constraints

- 所有程式碼置於 `sales-order-app/`（submodule，分支 `setting`；BASE = 現行實作 e76fb9a）。
- 依賴 backend 新契約：`GET /api/v1/settings` 回傳 `{ fields: [{field_id, name, field_type, desc, value}] }`（secret 遮罩；app 唯讀）。
- `SettingsFieldModel`：`fieldId/name/fieldType/value`（可空）；`SettingsModel` 改為 `fields: List<SettingsFieldModel>`。
- typed adapter getters（供 9 使用點，語法不變）：`systemSalesrepId`、`systemDepartmentId`、`systemTestSalesrepId`、`companyAdminSalesrepId`、`defaultDepartmentId`、`aboutUrl`、`supportEmail`、`defaultTimeout`、`frontendUrl` — `int?`/`String?`，by field_id。
- fallback：SystemConstants（現值）。
- 產生檔（`*.freezed.dart`、`*.g.dart`）入版控；build_runner 後提交。
- 每個 task 結束 commit（app submodule 內）。

---

### Task 1: Model 改為 field rows

**Files:**
- Modify: `sales-order-app/lib/layer_data/models/setting/setting.dart`
- Generate: `setting.freezed.dart`、`setting.g.dart`

**Interfaces:**
- Produces: `SettingsFieldModel`（`fieldId`/`name`/`fieldType`/`desc`/`value`，全可空）+ `SettingsModel`（`fields: List<SettingsFieldModel>?`）；fromJson。

- [ ] **Step 1: 改 model**

`setting.dart` 整個替換：

```dart
import 'package:freezed_annotation/freezed_annotation.dart';

part 'setting.freezed.dart';
part 'setting.g.dart';

@freezed
abstract class SettingsFieldModel with _$SettingsFieldModel {
  const factory SettingsFieldModel({
    @JsonKey(name: 'field_id') String? fieldId,
    @JsonKey(name: 'name') String? name,
    @JsonKey(name: 'field_type') String? fieldType,
    @JsonKey(name: 'desc') String? desc,
    @JsonKey(name: 'value') String? value,
  }) = _SettingsFieldModel;

  factory SettingsFieldModel.fromJson(Map<String, dynamic> json) =>
      _$SettingsFieldModelFromJson(json);
}

@freezed
abstract class SettingsModel with _$SettingsModel {
  const factory SettingsModel({
    @JsonKey(name: 'fields') List<SettingsFieldModel>? fields,
  }) = _SettingsModel;

  factory SettingsModel.fromJson(Map<String, dynamic> json) =>
      _$SettingsModelFromJson(json);
}
```

- [ ] **Step 2: 產生碼 + 驗證**

Run: `cd sales-order-app && fvm dart run build_runner build --delete-conflicting-outputs && fvm flutter analyze lib/layer_data/models/setting`
Expected: 產生檔更新、analyze 無問題。

- [ ] **Step 3: Commit**

```bash
cd sales-order-app && git add lib/layer_data/models/setting && git commit -m "refactor(models): settings as field rows"
```

---

### Task 2: SettingsService typed adapter

**Files:**
- Modify: `sales-order-app/lib/layer_business/services/settings/settings_service.dart`

**Interfaces:**
- Produces: 保留 `settingsSignal`/`current`/`loadCache`/`refresh`（型別改 `SettingsModel` = rows）；新增 typed getters（by field_id，`fieldValue(id)` + 型別轉換）；`fallback` 組 rows-from-SystemConstants。

- [ ] **Step 1: service 改版**

`settings_service.dart` 新增（保留既有 Signal/cache/refresh 邏輯，僅 `fallback` 與取值改版）：

```dart
  /// 依 field_id 取值（未命中 → null）。
  String? fieldValue(String fieldId) {
    final rows = current.fields;
    if (rows == null) return null;
    for (final r in rows) {
      if (r.fieldId == fieldId) return r.value;
    }
    return null;
  }

  int? intValue(String fieldId) => int.tryParse(fieldValue(fieldId) ?? '');

  // ---- typed adapter（9 個使用點語法不變）----
  int? get systemSalesrepId => intValue('system_salesrep_id');
  int? get systemDepartmentId => intValue('system_department_id');
  int? get systemTestSalesrepId => intValue('system_test_salesrep_id');
  int? get companyAdminSalesrepId => intValue('company_admin_salesrep_id');
  int? get defaultDepartmentId => intValue('default_department_id');
  String? get aboutUrl => fieldValue('about_url');
  String? get supportEmail => fieldValue('support_email');
  int? get defaultTimeout => intValue('default_timeout');
  String? get frontendUrl => fieldValue('frontend_url');
```

`fallback` 改為 rows 型別：

```dart
  SettingsModel get fallback => SettingsModel(fields: [
        SettingsFieldModel(fieldId: 'default_department_id', fieldType: 'int64', value: '${SystemConstants.defaultDepartment}'),
        SettingsFieldModel(fieldId: 'system_department_id', fieldType: 'int64', value: '${SystemConstants.systemDepartmentId}'),
        SettingsFieldModel(fieldId: 'system_salesrep_id', fieldType: 'int64', value: '${SystemConstants.systemSalesrepId}'),
        SettingsFieldModel(fieldId: 'system_test_salesrep_id', fieldType: 'int64', value: '${SystemConstants.defaultTestSalesrepId}'),
        SettingsFieldModel(fieldId: 'company_admin_salesrep_id', fieldType: 'int64', value: '${SystemConstants.companyAdminSalesrepId}'),
        SettingsFieldModel(fieldId: 'about_url', fieldType: 'string', value: SystemConstants.aboutUrl),
        SettingsFieldModel(fieldId: 'support_email', fieldType: 'string', value: SystemConstants.supportEmail),
        SettingsFieldModel(fieldId: 'default_timeout', fieldType: 'int64', value: '${SystemConstants.defaultTimeout}'),
        SettingsFieldModel(fieldId: 'frontend_url', fieldType: 'string', value: SystemConstants.frontendUrl),
      ]);
```

- [ ] **Step 2: 驗證 + 修使用點型別錯誤**

Run: `cd sales-order-app && fvm flutter analyze lib`
Expected: 使用點 `s.systemSalesrepId` 等仍可編譯（getter 已提供）；若舊 `SettingsModel` 欄位使用處報錯（如 `s.defaultDepartmentId` 已由 getter 提供 — 無誤；`SettingsModel(fields: ...)` 建構處為 service 內部），修正 service 內部建構。

- [ ] **Step 3: Commit**

```bash
cd sales-order-app && git add lib/layer_business/services/settings && git commit -m "feat(settings): typed adapter over field rows"
```

---

### Task 3: 使用點驗證 + 測試更新 + 文件

**Files:**
- Modify（驗證，預期語法不變）：9 個使用點（services 3 + stories 6，見前版 plan）— 僅確認 compile，不改邏輯
- Modify: `sales-order-app/test/services/settings_service_test.dart`（改 rows 型別）
- Modify: `docs/AGENTS/app.md`（superproject root 提交）

**Interfaces:**
- Produces: 測試涵蓋 adapter（rows → getters）、fallback（無 rows → SystemConstants 值）；文件更新。

- [ ] **Step 1: 測試改版**

`test/services/settings_service_test.dart`：fake store 的 JSON 改為 `{'fields': [{'field_id': 'system_salesrep_id', 'field_type': 'int64', 'value': '-16888'}, ...]}`；斷言：`svc.current.systemSalesrepId == -16888`、`svc.frontendUrl`、fallback 路徑（無快取 + API 失敗 → `svc.current.systemSalesrepId == SystemConstants.systemSalesrepId`）。

Run: `cd sales-order-app && fvm flutter test`
Expected: 全 PASS（68 + 更新）。

- [ ] **Step 2: 全量驗證**

Run: `cd sales-order-app && fvm flutter analyze lib && fvm flutter test`
Expected: 全綠；9 使用點無需改動（adapter getters 保持語法）。

- [ ] **Step 3: 文件**

`docs/AGENTS/app.md`：SettingsModel 改 rows、typed adapter、superadmin 無關（app 唯讀）。

- [ ] **Step 4: Commit**

```bash
cd sales-order-app && git add test && git commit -m "test(settings): adapter and fallback over field rows"
cd .. && git add docs/AGENTS/app.md && git commit -m "docs(app): field-based settings guide"
```
