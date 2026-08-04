# auto_route 與 WoltModalSheet 整合 Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** 將客戶 create/edit 表單從 WoltModalSheet modal 遷移至 auto_route 路由頁面（含 stepper），使表單狀態可被 401 還原、導航統一。

**Architecture:** 新增 `CustomerFormDraftProvider`（disco，持有表單草稿 + 目前步驟），新增 `/customer/create`、`/customer/edit/:id` 路由（AuthGuard）與 `CustomerCreateScreen`/`CustomerEditScreen`（內部共用 `CustomerFormStepper` 三步：主要/地址/聯絡人），遷移 `modal_service.dart` 的 create/edit 邏輯，移除舊 modal 呼叫。訂單明細與 QRCode modal 保留。

**Tech Stack:** Flutter 3.35.2 / auto_route 11 / disco / flutter_solidart / reactive_forms / WoltModalSheet 0.11（保留）

## Global Constraints

- Flutter 3.35.2（`.fvmrc`）；`fvm flutter analyze` 0 errors、`fvm flutter test` 全過
- 產生檔（`routes.gr.dart`）修改來源後必須重新 `fvm dart run build_runner build --delete-conflicting-outputs`
- `RoutePath` 是 `@MappableEnum()`（dart_mappable）— 新增值後需重新 build_runner
- 繁體中文 UI 字串
- **不動**：訂單明細 modal、QRCode modal、`wolt_modal_sheet` 套件、alertable_mixin/AwesomeDialog、deep link 設定
- 表單狀態一律經 `CustomerFormDraftProvider`，不直接跨頁面傳參
- 401 還原時草稿優先（同 id）；僅無草稿時重新載入
- 每個 task 完成後可獨立測試；commit message 用 conventional format

---

## 階段 1：草稿 Provider

### Task 1: CustomerFormDraftProvider + 單元測試

**Files:**
- Create: `lib/layer_business/services/customer/customer_form_draft_provider.dart`
- Test: `test/unit/services/customer_form_draft_provider_test.dart`

**Interfaces:**
- Consumes: `CustomerModel`（既有 model）、disco `Provider`、flutter_solidart `Signal`
- Produces: `customerFormDraftProvider`（disco Provider）、`CustomerFormDraftProvider` class（`startCreate`/`startEdit`/`setDraft`/`setStep`/`clearDraft` + signals `draftSignal`/`currentStepSignal`/`editingCustomerIdSignal`）— 供 Task 2 路由頁面使用

- [ ] **Step 1: 寫 failing test**

```dart
// test/unit/services/customer_form_draft_provider_test.dart
import 'package:flutter_test/flutter_test.dart';
import 'package:hexagon_food_app/layer_business/services/customer/customer_form_draft_provider.dart';
import 'package:hexagon_food_app/layer_data/models/customer/customer.dart';

void main() {
  group('CustomerFormDraftProvider', () {
    late CustomerFormDraftProvider provider;

    setUp(() => provider = CustomerFormDraftProvider());
    tearDown(() => provider.dispose());

    test('initial state is empty', () {
      expect(provider.draft, isNull);
      expect(provider.currentStep, 0);
      expect(provider.editingCustomerId, isNull);
    });

    test('startCreate clears draft and step', () {
      provider.setDraft(CustomerModel(), step: 2);
      provider.startCreate();
      expect(provider.draft, isNull);
      expect(provider.currentStep, 0);
      expect(provider.editingCustomerId, isNull);
    });

    test('startEdit records target id and resets step', () {
      provider.setDraft(CustomerModel(), step: 1);
      provider.startEdit(42);
      expect(provider.editingCustomerId, 42);
      expect(provider.currentStep, 0);
      // startEdit 不清草稿 — 401 還原時需保留
    });

    test('setDraft stores draft and optional step', () {
      final customer = CustomerModel(companyName: '測試公司');
      provider.setDraft(customer, step: 2);
      expect(provider.draft?.companyName, '測試公司');
      expect(provider.currentStep, 2);
    });

    test('setStep updates current step', () {
      provider.setStep(1);
      expect(provider.currentStep, 1);
    });

    test('clearDraft resets everything', () {
      provider.setDraft(CustomerModel(), step: 2);
      provider.startEdit(42);
      provider.clearDraft();
      expect(provider.draft, isNull);
      expect(provider.currentStep, 0);
      expect(provider.editingCustomerId, isNull);
    });

    test('signals emit on value change', () {
      int draftChanges = 0;
      int stepChanges = 0;
      // Signal 是 ValueNotifier — 用 addListener
      provider.draftSignal.addListener((_) => draftChanges++);
      provider.currentStepSignal.addListener((_) => stepChanges++);

      provider.setDraft(CustomerModel());
      provider.setStep(1);

      expect(draftChanges, 1);
      expect(stepChanges, 1);
    });
  });
}
```

> **注意**：`Signal` 是 `ValueNotifier`（`addListener`/`removeListener`，無 `subscribe`）。測試用 `addListener` 監聽 signal 變更。

- [ ] **Step 2: 執行測試確認 fail**

```bash
cd sales-order-app
fvm flutter test test/unit/services/customer_form_draft_provider_test.dart
```

預期：compile error（`CustomerFormDraftProvider` undefined）。

- [ ] **Step 3: 實作 Provider**

```dart
// lib/layer_business/services/customer/customer_form_draft_provider.dart
import 'package:disco/disco.dart';
import 'package:flutter_solidart/flutter_solidart.dart';
import 'package:hexagon_food_app/layer_data/models/customer/customer.dart';

final customerFormDraftProvider = Provider(
  (context) => CustomerFormDraftProvider(),
  dispose: (p) => p.dispose(),
);

class CustomerFormDraftProvider {
  final _draft = Signal<CustomerModel?>(null);
  final _currentStep = Signal<int>(0);
  final _editingCustomerId = Signal<int?>(null);

  CustomerModel? get draft => _draft.value;
  int get currentStep => _currentStep.value;
  int? get editingCustomerId => _editingCustomerId.value;

  Signal<CustomerModel?> get draftSignal => _draft;
  Signal<int> get currentStepSignal => _currentStep;
  Signal<int?> get editingCustomerIdSignal => _editingCustomerId;

  /// 開啟 create 流程：清空草稿，步驟歸零。
  void startCreate() {
    _draft.value = null;
    _currentStep.value = 0;
    _editingCustomerId.value = null;
  }

  /// 開啟 edit 流程：記錄目標 id；不清草稿（401 還原時草稿優先）。
  void startEdit(int customerId) {
    _editingCustomerId.value = customerId;
    _currentStep.value = 0;
  }

  /// 草稿優先：401 還原時若已有同 id 草稿則沿用。
  void setDraft(CustomerModel draft, {int? step}) {
    _draft.value = draft;
    if (step != null) _currentStep.value = step;
  }

  void setStep(int step) => _currentStep.value = step;

  /// 提交成功或取消時清除。
  void clearDraft() {
    _draft.value = null;
    _currentStep.value = 0;
    _editingCustomerId.value = null;
  }

  void dispose() {
    _draft.dispose();
    _currentStep.dispose();
    _editingCustomerId.dispose();
  }
}
```

- [ ] **Step 4: 執行測試確認 pass**

```bash
cd sales-order-app
fvm flutter test test/unit/services/customer_form_draft_provider_test.dart
```

預期：全部 PASS。

- [ ] **Step 5: analyze + 全測試**

```bash
cd sales-order-app
fvm flutter analyze
fvm flutter test
```

預期：0 errors；既有測試全過。

- [ ] **Step 6: Commit**

```bash
git add lib/layer_business/services/customer/customer_form_draft_provider.dart test/unit/services/customer_form_draft_provider_test.dart
git commit -m "feat: add CustomerFormDraftProvider for router-based customer form state"
```

---

## 階段 2：路由與頁面骨架

### Task 2: 路由定義 + 頁面骨架

**Files:**
- Modify: `lib/layer_business/router/paths.dart`（新增 `customerCreate`、`customerEdit`）
- Modify: `lib/layer_business/router/routes.dart`（customerLayout children 新增兩路由）
- Create: `lib/layer_presentation/stories/admin/customer/customer_create_screen.dart`
- Create: `lib/layer_presentation/stories/admin/customer/customer_edit_screen.dart`
- Create: `lib/layer_presentation/stories/admin/customer/widgets/customer_form_stepper.dart`（骨架）

**Interfaces:**
- Consumes: `customerFormDraftProvider`（Task 1）、`CustomerCreateRoute`/`CustomerEditRoute`（本 task 產生）
- Produces: `CustomerCreateScreen`（`@RoutePage()`）、`CustomerEditScreen`（`@RoutePage()`，`@PathParam('id') int id`）、`CustomerFormStepper` 骨架（供 Task 3 填入內容）

- [ ] **Step 1: RoutePath 新增值**

`lib/layer_business/router/paths.dart` 的 enum 新增（與既有 `customer` 同風格）：

```dart
  customerCreate,
  customerEdit,
```

- [ ] **Step 2: routes.dart 新增兩路由**

`lib/layer_business/router/routes.dart` 的 customerLayout children：

```dart
    AutoRoute(
      path: rootSplash(customerLayout),
      page: CustomerLayoutRoute.page,
      guards: [AuthGuard()],
      children: [
        RedirectRoute(path: "", redirectTo: customer),
        AutoRoute(path: customer, page: CustomerRoute.page),
        AutoRoute(path: customerCreate, page: CustomerCreateRoute.page),
        AutoRoute(path: '$customerEdit/:id', page: CustomerEditRoute.page),
      ],
    ),
```

並在建構子加 `final customerCreate = RoutePath.customerCreate.toValue();`、`final customerEdit = RoutePath.customerEdit.toValue();`（與既有 `final customer = ...` 同位置）。

- [ ] **Step 3: 建 create screen 骨架**

```dart
// lib/layer_presentation/stories/admin/customer/customer_create_screen.dart
import 'package:auto_route/auto_route.dart';
import 'package:hexagon_food_app/layer_business/services/customer/customer_form_draft_provider.dart';
import 'package:hexagon_food_app/layer_presentation/general/abstract/base_widget.dart';
import 'package:hexagon_food_app/layer_presentation/stories/admin/customer/widgets/customer_form_stepper.dart';

@RoutePage()
class CustomerCreateScreen extends BaseStatelessWidget {
  const CustomerCreateScreen({super.key});

  @override
  Widget build(BuildContext context) {
    customerFormDraftProvider.of(context).startCreate();
    return const Scaffold(
      appBar: null,
      body: CustomerFormStepper(mode: CustomerFormMode.create),
    );
  }
}
```

- [ ] **Step 4: 建 edit screen 骨架**

```dart
// lib/layer_presentation/stories/admin/customer/customer_edit_screen.dart
import 'package:auto_route/auto_route.dart';
import 'package:hexagon_food_app/layer_business/services/customer/customer_form_draft_provider.dart';
import 'package:hexagon_food_app/layer_presentation/general/abstract/base_widget.dart';
import 'package:hexagon_food_app/layer_presentation/stories/admin/customer/widgets/customer_form_stepper.dart';

@RoutePage()
class CustomerEditScreen extends BaseStatelessWidget {
  const CustomerEditScreen({super.key, @PathParam('id') required this.id});

  final int id;

  @override
  Widget build(BuildContext context) {
    customerFormDraftProvider.of(context).startEdit(id);
    return Scaffold(
      appBar: null,
      body: CustomerFormStepper(mode: CustomerFormMode.edit, customerId: id),
    );
  }
}
```

- [ ] **Step 5: 建 stepper 骨架**

```dart
// lib/layer_presentation/stories/admin/customer/widgets/customer_form_stepper.dart
import 'package:flutter/material.dart';
import 'package:hexagon_food_app/layer_business/services/customer/customer_form_draft_provider.dart';

enum CustomerFormMode { create, edit }

/// 客戶表單三步 stepper（主要資料 / 地址 / 聯絡人）。
/// 內容在 Task 3 填入；此骨架先建立步驟容器與導航。
class CustomerFormStepper extends StatefulWidget {
  const CustomerFormStepper({
    super.key,
    required this.mode,
    this.customerId,
  });

  final CustomerFormMode mode;
  final int? customerId;

  @override
  State<CustomerFormStepper> createState() => _CustomerFormStepperState();
}

class _CustomerFormStepperState extends State<CustomerFormStepper> {
  static const _stepTitles = ['主要資料', '地址', '聯絡人'];

  @override
  Widget build(BuildContext context) {
    final draftProvider = customerFormDraftProvider.of(context);
    return SignalBuilder(
      signal: draftProvider.currentStepSignal,
      builder: (context, currentStep) {
        return Column(
          children: [
            // 步驟指示器（實作時依現有元件風格）
            Text('步驟 ${currentStep + 1} / 3：${_stepTitles[currentStep]}'),
            // 步驟內容（Task 3 填入）
            Expanded(child: _buildStepContent(context, currentStep)),
            // 導航列（前一步 / 下一步 / 提交 — Task 3 接線）
            _buildNavBar(context, currentStep),
          ],
        );
      },
    );
  }

  Widget _buildStepContent(BuildContext context, int step) {
    // TODO: Task 3 填入 — 主要資料 / 地址 / 聯絡人表單
    return const SizedBox.shrink();
  }

  Widget _buildNavBar(BuildContext context, int step) {
    // TODO: Task 3 接線 — 前一步 / 下一步 / 提交
    return const SizedBox.shrink();
  }
}
```

> **注意**：此骨架的 `SignalBuilder` 用法需確認 flutter_solidart 實際 API（`SignalBuilder(signal: ..., builder: ...)` 或 `SignalBuilder(builder: (context, _) => ...)`）。實作時依現有程式碼（如 `history_screen.dart` 的 SignalBuilder 用法）調整。

- [ ] **Step 6: 重新產生 routes.gr.dart + analyze**

```bash
cd sales-order-app
fvm dart run build_runner build --delete-conflicting-outputs
fvm flutter analyze
```

預期：`CustomerCreateRoute`/`CustomerEditRoute` 生成；0 errors（骨架的 TODO 是容器，不影響 compile）。

- [ ] **Step 7: 手動 smoke — 路由可導航**

（可選，若有模擬器）`fvm flutter run --flavor dev` 登入 → 客戶列表 → 點新增 → 應進 `CustomerCreateScreen`（顯示「步驟 1 / 3」）。無模擬器則跳過，由後續 task 測試覆蓋。

- [ ] **Step 8: Commit**

```bash
git add lib/layer_business/router/paths.dart lib/layer_business/router/routes.dart lib/layer_business/router/routes.gr.dart lib/layer_presentation/stories/admin/customer/customer_create_screen.dart lib/layer_presentation/stories/admin/customer/customer_edit_screen.dart lib/layer_presentation/stories/admin/customer/widgets/customer_form_stepper.dart
git commit -m "feat: add customer create/edit routes and form stepper skeleton"
```

---

## 階段 3：表單遷移

### Task 3: 遷移表單內容至 stepper

**Files:**
- Modify: `lib/layer_presentation/stories/admin/customer/widgets/customer_form_stepper.dart`（填入內容）
- Modify: `lib/layer_presentation/stories/admin/customer/customer_create_screen.dart`（建 `CustomerModelFormBuilder` + 提交）
- Modify: `lib/layer_presentation/stories/admin/customer/customer_edit_screen.dart`（載入 id + 草稿優先 + 提交）
- Consume: `lib/layer_business/services/customer/modal_service.dart` 的 `_mainContent`/`_addressContent`/`_contactsContent` 表單 widget（`main_form.dart`/`address_form.dart`/`contact_form.dart`、`*_edit_form.dart`）

**Interfaces:**
- Consumes: `CustomerFormStepper`（Task 2）、`customerProvider`、`metadictProvider`、`customerApi`（edit 載入）、`customerCreateButton`/`customerEditButton`（`customer_button.dart`）
- Produces: 完整 stepper（三步表單 + 導航 + 提交）；create/edit screen 完整

- [ ] **Step 1: 讀取 modal_service 的表單 widget 用法**

```bash
cd sales-order-app
# 確認 create/edit 表單 widget 的 import 與用法
grep -n "mainForm\|mainEditForm\|addressForm\|addressEditForm\|contactForm\|contactEditForm" lib/layer_business/services/customer/modal_service.dart | head -10
```

**遷移對應表**（modal_service → stepper）：

| modal_service | stepper |
|---------------|---------|
| `_createPageList` 的 `mainForm` build callback | step 0（create 模式）|
| `_editPageList` 的 `mainEditForm` build callback | step 0（edit 模式）|
| `_addressContent` 的 `addressForm`/`addressEditForm` | step 1 |
| `_contactsContent` 的 `contactForm`/`contactEditForm` | step 2 |
| `leadingNavBarWidget` 前後箭頭 | `_buildNavBar` 的 back/next |
| `stickyActionBar` + `_checkValid` + `tapCallback` | `_buildNavBar` 最後一步提交 |
| `modalDecorator` 的 ProviderScope + SignalBuilder（metadict）| create/edit screen 的 builder |
| `_initState`（salesrep/currency/address 預設）| create screen 的 `initState` |

`_createPageList` 用 `mainForm`/`addressForm`/`contactForm`；`_editPageList` 用 `mainEditForm`/`addressEditForm`/`contactEditForm` — 這些是 `CustomerModelForm` 的 reactive_forms 定義（在 `customer_form.dart` 或 form model）。stepper 需在頁面層建 `CustomerModelFormBuilder`（沿用 modal 的 `modalCreateShow`/`modalEditShow` 結構），把 `_mainContent`/`_addressContent`/`_contactsContent` 的 `child: build(context)` 內容移入 stepper 步驟。

- [ ] **Step 2: stepper 步驟內容 — 主要資料**

在 `_buildStepContent` 的 step 0 填入（內容來自 `_mainContent` 的 `child: build(context)`，即 `mainForm`/`mainEditForm` widget）：

```dart
  Widget _buildStepContent(BuildContext context, int step) {
    switch (step) {
      case 0:
        // 主要資料表單（modal _mainContent 的 child）
        // create: mainForm(context) / edit: mainEditForm(context)
        return widget.mode == CustomerFormMode.create
            ? mainForm(context)
            : mainEditForm(context);
      case 1:
        // 地址表單（modal _addressContent 的 child）
        return widget.mode == CustomerFormMode.create
            ? addressForm(context)
            : addressEditForm(context);
      case 2:
        // 聯絡人表單（modal _contactsContent 的 child）
        return widget.mode == CustomerFormMode.create
            ? contactForm(context)
            : contactEditForm(context);
      default:
        return const SizedBox.shrink();
    }
  }
```

> **注意**：`mainForm(context)` 等是 `modal_service.dart` 中傳給 `_mainContent` 的 build callback（`Widget Function(BuildContext)`）。實作時需從 modal_service 找出這些 callback 的實際定義（可能在同檔或 `customer/form/` 目錄），將它們的實作移入 stepper 或作為 stepper 的參數傳入。若 callback 定義在 modal_service 內部，改為 stepper 的建構參數或直接在 stepper 內建表單。

- [ ] **Step 3: stepper 導航列**

在 `_buildNavBar` 填入（對應 modal 的 `leadingNavBarWidget` 前後箭頭 + `stickyActionBar` 提交）：

```dart
  Widget _buildNavBar(BuildContext context, int step) {
    final draftProvider = customerFormDraftProvider.of(context);
    final isLast = step == 2;

    return Container(
      padding: const EdgeInsets.all(8),
      child: Row(
        children: [
          IconButton(
            icon: const Icon(Icons.arrow_back_rounded),
            onPressed: step > 0 ? () => draftProvider.setStep(step - 1) : null,
          ),
          const Spacer(),
          if (!isLast)
            IconButton(
              icon: const Icon(Icons.arrow_forward_rounded),
              // 前進：驗證目前步驟 → 寫回草稿 → setStep(step+1)
              onPressed: () {
                _advance(context, draftProvider, step);
              },
            )
          else
            // 最後一步：提交按鈕（沿用 customerCreateButton/customerEditButton）
            _buildSubmitButton(context, draftProvider),
        ],
      ),
    );
  }
```

> **注意**：提交按鈕與步驟驗證的實際邏輯（`_checkValid`、`ReactiveCustomerModelFormConsumer`、`_addAddressItem`/`_addContactItem`）需從 modal_service 搬入。`_checkValid` 檢查 `formModel.currentForm.valid && formModel.addressbookForm.currentForm.valid` — 需要 formModel 的 access，stepper 需包在 `CustomerModelFormBuilder` 內或接收 formModel。

- [ ] **Step 4: create screen 完整化**

`customer_create_screen.dart` 改為持有 `CustomerModelFormBuilder`（沿用 `modalCreateShow` 結構）：

```dart
@RoutePage()
class CustomerCreateScreen extends StatelessWidget {
  const CustomerCreateScreen({super.key});

  @override
  Widget build(BuildContext context) {
    final draftProvider = customerFormDraftProvider.of(context);
    draftProvider.startCreate();

    return Scaffold(
      body: CustomerModelFormBuilder(
        model: CustomerModel(),
        initState: (context, formModel) {
          // 沿用 modal_service._initState：salesrep/currency/address 預設
          formModel.salesrepControl.value =
              customerProvider.of(context).salesrepInfo?.id == SystemConstants.systemSalesrepId
                  ? -5
                  : customerProvider.of(context).salesrepInfo?.id ?? 0;
          formModel.currencyControl.value = 1;
          formModel.addressbookForm.addItemsItem(
            AddressItem(
              label: '主要地址',
              defaultShipping: true,
              defaultBilling: true,
              isResidential: false,
              addressBookEntityAddress: AddressEntity(addr1: '', addr2: '', country: 'TW', state: '', city: '', zip: ''),
            ),
          );
        },
        builder: (context, formModel, child) {
          // 表單值變更即寫回草稿
          formModel.formGroup.valueChanges?.listen((_) {
            draftProvider.setDraft(formModel.rawModel, step: draftProvider.currentStep);
          });
          return CustomerFormStepper(
            mode: CustomerFormMode.create,
            formModel: formModel,
            onSubmit: (form) async {
              await customerProvider.of(context).createCustomer(form);
              draftProvider.clearDraft();
              if (context.mounted) context.router.maybePop();
            },
          );
        },
      ),
    );
  }
}
```

> **注意**：`CustomerModelFormBuilder`、`formModel.formGroup`、`rawModel` 等 API 需對照 reactive_forms_generator 產生的實際型別（`customer.gform.dart`）。`valueChanges` 的 listen 需在 dispose 時取消或確認 builder 生命週期。實作時以實際 code 為準。

- [ ] **Step 5: edit screen 完整化**

`customer_edit_screen.dart` 加入載入 + 草稿優先：

```dart
@RoutePage()
class CustomerEditScreen extends StatefulWidget {
  const CustomerEditScreen({super.key, @PathParam('id') required this.id});
  final int id;
  @override
  State<CustomerEditScreen> createState() => _CustomerEditScreenState();
}

class _CustomerEditScreenState extends State<CustomerEditScreen> {
  CustomerModel? _initial; // 草稿或載入的客戶
  bool _loading = true;
  String? _error;

  @override
  void initState() {
    super.initState();
    final draftProvider = customerFormDraftProvider.of(context);
    draftProvider.startEdit(widget.id);

    // 草稿優先：401 殘留草稿（同 id）直接用
    final draft = draftProvider.draft;
    if (draft != null && draft.id == widget.id) {
      _initial = draft;
      _loading = false;
      return;
    }
    _loadCustomer();
  }

  Future<void> _loadCustomer() async {
    setState(() { _loading = true; _error = null; });
    try {
      final customer = await customerProvider.of(context).getCustomerById(widget.id);
      if (mounted) {
        customerFormDraftProvider.of(context).setDraft(customer!, step: 0);
        setState(() { _initial = customer; _loading = false; });
      }
    } catch (e) {
      if (mounted) setState(() { _error = e.toString(); _loading = false; });
    }
  }

  @override
  Widget build(BuildContext context) {
    if (_loading) return const Scaffold(body: Center(child: CircularProgressIndicator()));
    if (_error != null) {
      return Scaffold(
        body: Center(
          child: Column(mainAxisSize: MainAxisSize.min, children: [
            Text('加載失敗: $_error'),
            TextButton(onPressed: _loadCustomer, child: const Text('重試')),
          ]),
        ),
      );
    }
    return Scaffold(
      body: CustomerModelFormBuilder(
        model: _initial!,
        builder: (context, formModel, child) {
          final draftProvider = customerFormDraftProvider.of(context);
          formModel.formGroup.valueChanges?.listen((_) {
            draftProvider.setDraft(formModel.rawModel, step: draftProvider.currentStep);
          });
          return CustomerFormStepper(
            mode: CustomerFormMode.edit,
            formModel: formModel,
            onSubmit: (form) async {
              await customerProvider.of(context).updateCustomer(widget.id, form);
              draftProvider.clearDraft();
              if (context.mounted) context.router.maybePop();
            },
          );
        },
      ),
    );
  }
}
```

> **注意**：`getCustomerById` 回傳 `Future<CustomerModel?>`（customer_service.dart）— 需處理 null。草稿優先邏輯（`draft.id == widget.id`）是 spec 的關鍵。實作時確認 `CustomerModel` 有 `id` 欄位（有，`customer.dart`）。

- [ ] **Step 6: 更新 CustomerFormStepper 接收 formModel/onSubmit**

Task 2 的骨架 `CustomerFormStepper` 需加 `formModel` 與 `onSubmit` 參數（供 create/edit screen 傳入）：

```dart
class CustomerFormStepper extends StatefulWidget {
  const CustomerFormStepper({
    super.key,
    required this.mode,
    required this.formModel,   // CustomerModelForm（reactive_forms）
    required this.onSubmit,    // Future<void> Function(CustomerModel)
    this.customerId,
  });

  final CustomerFormMode mode;
  final CustomerModelForm formModel;
  final Future<void> Function(CustomerModel) onSubmit;
  final int? customerId;
  // ...
}
```

- [ ] **Step 7: analyze + 全測試**

```bash
cd sales-order-app
fvm flutter analyze
fvm flutter test
```

預期：0 errors；既有測試全過（modal 相關測試若引用 `modalCreateShow` 需一併檢查 — 目前無此類測試）。

- [ ] **Step 8: Commit**

```bash
git add lib/layer_presentation/stories/admin/customer/
git commit -m "feat: migrate customer create/edit form from modal to router stepper"
```

---

## 階段 4：移除舊 modal 呼叫

### Task 4: 更新呼叫點 + 清理 modal_service

**Files:**
- Modify: `lib/layer_presentation/stories/admin/customer/customer_layout_screen.dart`（`modalCreateShow` → push）
- Modify: `lib/layer_presentation/stories/admin/customer/widgets/customer_list.dart`（`modalEditShow` → push）
- Modify: `lib/layer_presentation/stories/admin/tabs/profile/widgets/profile_list.dart`（`modalEditShow` → push）
- Modify: `lib/layer_business/services/customer/modal_service.dart`（移除 create/edit 邏輯，保留 QRCode）

**Interfaces:**
- Consumes: `CustomerCreateRoute`/`CustomerEditRoute`（Task 2）、`customerModalProvider`（保留 — QRCode 用）
- Produces: create/edit 只走路由；modal_service 只剩 QRCode

- [ ] **Step 1: customer_layout_screen.dart 呼叫點**

`lib/layer_presentation/stories/admin/customer/customer_layout_screen.dart:39`：

```dart
// 舊：await cmp.modalCreateShow(customerCreateButton);
// 新：
context.router.push(CustomerCreateRoute());
```

（需 import `routes.gr.dart` — 若已有則不加。）

- [ ] **Step 2: customer_list.dart 呼叫點**

`lib/layer_presentation/stories/admin/customer/widgets/customer_list.dart:89`：

```dart
// 舊：await cmp.modalEditShow(customer, customerEditButton);
// 新：
context.router.push(CustomerEditRoute(id: customer.id!));
```

（`customer.id` 可能 null — 用 `!` 或 guard，依既有 code 風格。`customerEditButton` 本身已有 `if (form.id == null) return null` guard，路由呼叫可提前 guard。）

- [ ] **Step 3: profile_list.dart 呼叫點**

`lib/layer_presentation/stories/admin/tabs/profile/widgets/profile_list.dart:34`：

```dart
// 舊：await cmp.modalEditShow(cm, customerEditButton);
// 新：
context.router.push(CustomerEditRoute(id: cm.id!));
```

- [ ] **Step 4: modal_service.dart 清理**

移除 `modalCreateShow`、`modalEditShow`、`_createPageList`、`_editPageList`、`_mainContent`、`_addressContent`、`_contactsContent`、`_initState`（若僅 create/edit 用）。保留 `modalQRCodeShow`、`_qrcodeContent`、`_closeSet`、`_checkValid`（若 QRCode 用）、`_addAddressItem`/`_addContactItem`（若 QRCode 不用則可移除 — 檢查）。移除不再使用的 import（`customer_button.dart`、`main_form.dart` 等 create/edit 專用）。

> **注意**：`_checkValid` 若只被 create/edit 用則移除；`_addAddressItem`/`_addContactItem` 若只被 create/edit 用則移除。保留與 QRCode 相關的（`pretty_qr_code`、`share_plus`、`Assets.images.logo`）。

- [ ] **Step 5: grep 確認無殘留**

```bash
cd sales-order-app
grep -rn "modalCreateShow\|modalEditShow" lib/ --include="*.dart"
```

預期：零 matches（modal_service.dart 內定義處已移除）。

- [ ] **Step 6: analyze + 全測試 + build_runner 確認**

```bash
cd sales-order-app
fvm dart run build_runner build --delete-conflicting-outputs
fvm flutter analyze
fvm flutter test
```

預期：0 errors；全測試過。

- [ ] **Step 7: Commit**

```bash
git add lib/layer_presentation/stories/admin/ lib/layer_business/services/customer/modal_service.dart
git commit -m "refactor: route customer create/edit through auto_route, keep QRCode modal"
```

---

## 階段 5：測試補強

### Task 5: Stepper + 401 還原 widget 測試

**Files:**
- Create: `test/widget/customer_form_stepper_test.dart`
- Create: `test/widget/customer_edit_screen_test.dart`

**Interfaces:**
- Consumes: `CustomerFormStepper`（Task 3）、`CustomerEditScreen`（Task 3）、`CustomerFormDraftProvider`（Task 1）、測試 helpers（`test/helpers/`）

- [ ] **Step 1: stepper widget 測試**

```dart
// test/widget/customer_form_stepper_test.dart
import 'package:flutter_test/flutter_test.dart';
// ...（pump CustomerFormStepper，驗證步驟切換）

void main() {
  // 1. 初始顯示步驟 1 / 3（主要資料）
  // 2. 點下一步 → currentStep 變 1（地址）
  // 3. 點下一步 → currentStep 變 2（聯絡人）
  // 4. 最後一步顯示提交按鈕
  // 5. 點前一步 → 回步驟 2
}
```

> **注意**：完整 pump 需要 `CustomerModelFormBuilder` + ProviderScope（disco）+ 可能的 API fake（metadict）。實作時用 `test/helpers/pump_app.dart` 或現有 pattern。若 stepper 依賴 API 載入（metadict dropdown），測試需 fake API 或 mock。

- [ ] **Step 2: edit screen 草稿優先測試**

```dart
// test/widget/customer_edit_screen_test.dart
void main() {
  // 1. Provider 有同 id 草稿 → 頁面直接用草稿（不呼叫載入 API）
  // 2. Provider 無草稿 → 依 id 載入 → setDraft
  // 3. 401 情境模擬：Provider 草稿存在 → 重建頁面 → 草稿恢復 + currentStep 恢復
}
```

> **注意**：需要 fake `customerApi`（`test/helpers/fake_api.dart` 或新 fake）。`customerProvider.getCustomerById` 依賴 `customerApi.of(context)` — 測試需 ProviderScope 注入 fake。

- [ ] **Step 3: analyze + 全測試**

```bash
cd sales-order-app
fvm flutter analyze
fvm flutter test
```

預期：0 errors；新測試 + 既有全過。

- [ ] **Step 4: Commit**

```bash
git add test/widget/
git commit -m "test: add customer form stepper and edit screen draft-priority tests"
```

---

## 執行順序

```
Phase 1 (Task 1): CustomerFormDraftProvider + unit tests
Phase 2 (Task 2): routes + page skeletons + stepper skeleton
Phase 3 (Task 3): migrate form content into stepper
Phase 4 (Task 4): update call sites + clean modal_service
Phase 5 (Task 5): widget tests (stepper + 401 draft priority)
```

每個 phase 依序；Task 2 依賴 Task 1，Task 3 依賴 Task 2，Task 4 依賴 Task 3，Task 5 依賴 Task 3。
