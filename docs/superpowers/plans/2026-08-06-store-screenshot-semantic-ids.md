# Store 截圖 Semantic ID 實作計畫

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** 在 `sales-order-app` 建立 `Testable` widget 並將兩條 store 截圖 Maestro flow 的靜態互動元件全面改用 `id:` selector，提升截圖流程穩定性。

**Architecture:** 新增一個無狀態 `Testable` widget，將需要定位的 Flutter 元件包進 `Semantics(identifier: id)`。Maestro 端以 `id:` selector 取代文字、regex、座標。既有已穩定運作的 `id:` selector 與命名維持不變。

**Tech Stack:** Flutter / Dart, Maestro, Taskfile.dev, Git

## Global Constraints

- 範圍僅限兩條 flow：`screenshots_store/Flow.yaml` 與 `screenshots_store_ios/Flow.yaml`
- 只對**靜態互動元件**加 ID；動態資料項目（客戶卡片、下拉選項文字）維持現況
- ID 命名：`<screen>_<element>_<action>`，全小寫 `snake_case`
- 既有 ID 不更動
- 所有修改都要能讓 `task fastlane:screenshots` 與 `task fastlane:screenshots:ios` 跑完並產出截圖

---

## File Structure

| File | Responsibility |
|---|---|
| `sales-order-app/lib/layer_presentation/widgets/testable.dart` | 新建 `Testable` widget，封裝 `Semantics(identifier: id)` |
| `sales-order-app/lib/layer_presentation/stories/admin/customer/customer_layout_screen.dart` | 用 `Testable` 包裝返回鈕、搜尋圖示、新增 FAB |
| `sales-order-app/lib/layer_presentation/stories/admin/salesorder/salesorder_screen.dart` | 用 `Testable` 包裝訂單表單的客戶搜尋圖示 |
| `sales-order-app/lib/layer_presentation/stories/admin/tabs/`（或首頁 tab bar 所在檔案） | 用 `Testable` 包裝「客戶列表」tab |
| `sales-order-app/integration_test/.maestro/screenshots_store/Flow.yaml` | 把靜態 selector 換成 `id:` |
| `sales-order-app/integration_test/.maestro/screenshots_store_ios/Flow.yaml` | 把靜態 selector 換成 `id:` |

---

### Task 1: 建立 `Testable` widget

**Files:**
- Create: `sales-order-app/lib/layer_presentation/widgets/testable.dart`
- Test: 於下一個 Task 間接測試（Maestro 能否命中 id）

**Interfaces:**
- Produces: `Testable` widget with constructor `Testable({required String id, required Widget child, Key? key})`

- [ ] **Step 1: 新增 `Testable` widget**

```dart
import 'package:flutter/material.dart';

class Testable extends StatelessWidget {
  const Testable({
    required this.id,
    required this.child,
    super.key,
  });

  final String id;
  final Widget child;

  @override
  Widget build(BuildContext context) {
    return Semantics(
      identifier: id,
      child: child,
    );
  }
}
```

- [ ] **Step 2: 確認檔案格式與 import**

Run: `cd /Volumes/UTM2/Developer/sales-order-netsuite/sales-order-app && dart format lib/layer_presentation/widgets/testable.dart`
Expected: 無錯誤，輸出 `Formatted 1 file (0 changed)` 或類似成功訊息。

- [ ] **Step 3: Commit**

```bash
cd /Volumes/UTM2/Developer/sales-order-netsuite/sales-order-app
git add lib/layer_presentation/widgets/testable.dart
git commit -m "feat(test): add Testable widget for Maestro semantic IDs"
```

---

### Task 2: 客戶列表頁面加 ID

**Files:**
- Modify: `sales-order-app/lib/layer_presentation/stories/admin/customer/customer_layout_screen.dart`

**Interfaces:**
- Consumes: `Testable` from Task 1
- Produces:
  - `customer_layout_back_button` on AppBar leading
  - `customer_layout_search_icon` on top-right search IconButton
  - `customer_layout_add_button` on FAB

- [ ] **Step 1: 閱讀目前 `CustomerLayoutScreen` 的 AppBar 與 actions 區塊**

Use `read` on the file to locate `AppBar(leading: leading, ...)` and the `IconButton(Icons.add)` plus any search IconButton.

- [ ] **Step 2: 用 `Testable` 包裝返回鈕**

```dart
AppBar(
  leading: Testable(
    id: 'customer_layout_back_button',
    child: leading,
  ),
  title: label(context, "客戶選擇"),
  centerTitle: true,
  actions: [
    // existing actions
  ],
)
```

- [ ] **Step 3: 用 `Testable` 包裝搜尋圖示**

Find the `Icons.person_search` or similar search IconButton in actions and wrap it:

```dart
Testable(
  id: 'customer_layout_search_icon',
  child: IconButton(
    icon: const Icon(Icons.person_search),
    onPressed: () { /* existing logic */ },
  ),
)
```

If the search icon is inside a custom widget, modify that widget to accept an optional `testId` or wrap it directly if the instance is unique.

- [ ] **Step 4: 用 `Testable` 包裝新增客戶 FAB**

```dart
Testable(
  id: 'customer_layout_add_button',
  child: FloatingActionButton(
    mini: true,
    tooltip: '新增客戶',
    onPressed: () async {
      await context.router.push(CustomerCreateRoute());
      await cp.refreshCustomerList();
    },
    child: const Icon(Icons.add),
  ),
)
```

- [ ] **Step 5: 格式化並快速 build 檢查**

Run:
```bash
cd /Volumes/UTM2/Developer/sales-order-netsuite/sales-order-app
dart format lib/layer_presentation/stories/admin/customer/customer_layout_screen.dart
flutter build ios --simulator --no-codesign 2>&1 | tail -5
```
Expected: build 成功（或至少 compile 無語法錯誤）。

- [ ] **Step 6: Commit**

```bash
cd /Volumes/UTM2/Developer/sales-order-netsuite/sales-order-app
git add lib/layer_presentation/stories/admin/customer/customer_layout_screen.dart
git commit -m "feat(test): add semantic IDs to CustomerLayoutScreen"
```

---

### Task 3: 訂單表單客戶搜尋圖示加 ID

**Files:**
- Modify: `sales-order-app/lib/layer_presentation/stories/admin/salesorder/salesorder_screen.dart`

**Interfaces:**
- Consumes: `Testable` from Task 1
- Produces: `order_form_customer_search_icon` on the customer search IconButton

- [ ] **Step 1: 閱讀 `SalesorderScreen` 並定位客戶搜尋圖示**

Use `read` on `sales-order-app/lib/layer_presentation/stories/admin/salesorder/salesorder_screen.dart` and grep for `person_search` or the `point: "93%,16%"` comment to find the corresponding IconButton.

- [ ] **Step 2: 用 `Testable` 包裝搜尋圖示**

```dart
Testable(
  id: 'order_form_customer_search_icon',
  child: IconButton(
    icon: const Icon(Icons.search), // or whatever icon is used
    onPressed: () { /* existing logic */ },
  ),
)
```

- [ ] **Step 3: 格式化並 build 檢查**

Run:
```bash
cd /Volumes/UTM2/Developer/sales-order-netsuite/sales-order-app
dart format lib/layer_presentation/stories/admin/salesorder/salesorder_screen.dart
flutter build ios --simulator --no-codesign 2>&1 | tail -5
```
Expected: build 成功。

- [ ] **Step 4: Commit**

```bash
cd /Volumes/UTM2/Developer/sales-order-netsuite/sales-order-app
git add lib/layer_presentation/stories/admin/salesorder/salesorder_screen.dart
git commit -m "feat(test): add semantic ID to order customer search icon"
```

---

### Task 4: 功能頁「客戶列表」選單項目加 ID

**Files:**
- Modify: `sales-order-app/lib/layer_presentation/stories/admin/tabs/profile/widgets/profile_list.dart`

**Interfaces:**
- Consumes: `Testable` from Task 1
- Produces: `profile_customer_list_item` on the "客戶列表" menu item tile

**背景（與原始計畫的差異）：** 商店截圖 flow 中 `tapOn: "客戶列表"` 不是底部 tab，而是功能頁（profile）裡的選單項目。該項目由 `ProfileList` 的 `fp.addOntab(menus, 1, onTab, "客戶列表")` 即為 profileMenus 第一個元素（index 0），並用 `funcCustomTile` 渲染（`profile_list.dart`）。因此 ID 要加在 `ProfileList` 的 `itemBuilder` 中 index 0 的 tile 上。

- [ ] **Step 1: 在 `ProfileList.itemBuilder` 中包裝 index 0 的 tile**

在 `profile_list.dart` 的 `itemBuilder` 中，把 `funcCustomTile(...)` 的結果包上 `Testable`：

```dart
itemBuilder: (context, index) {
  final tile = funcCustomTile(
    context,
    Text(
      menus[index].title,
      style: TextStyle(
        color: cr.p700,
        fontWeight: txt.subHeading2Medium.fontWeight,
        fontSize: txt.subHeading2Medium.fontSize,
      ),
    ),
    leading: Icon(menus[index].leadingIcon, color: menus[index].leadingIconColor),
    trailing: Icon(menus[index].trailingIcon, color: menus[index].trailingIconColor),
    onTap: () => menus[index].onTap?.call(context),
  );
  return index == 0 ? Testable(id: 'profile_customer_list_item', child: tile) : tile;
},
```

注意：`addOntab` 只是就地修改 `pid == 1` 項目的 title/onTap，不插入、不排序；`profileMenus` 的第一個元素就是 `pid == 1`（客戶列表/公司資料），因此以 `index == 0` 判斷。若未來 `profileMenus` 項目順序變更需同步調整。

- [ ] **Step 2: 加入 `Testable` import**

```dart
import 'package:hexagon_food_app/layer_presentation/widgets/testable.dart';
```

- [ ] **Step 3: 格式化並 build 檢查**

Run:
```bash
cd /Volumes/UTM2/Developer/sales-order-netsuite/sales-order-app
dart format lib/layer_presentation/stories/admin/tabs/profile/widgets/profile_list.dart
flutter build ios --simulator --no-codesign --flavor dev 2>&1 | tail -5
```

- [ ] **Step 4: Commit**

```bash
cd /Volumes/UTM2/Developer/sales-order-netsuite/sales-order-app
git add lib/layer_presentation/stories/admin/tabs/profile/widgets/profile_list.dart
git commit -m "feat(test): add semantic ID to profile customer list menu item"
```

---

### Task 5: 更新 Android store 截圖 flow

**Files:**
- Modify: `sales-order-app/integration_test/.maestro/screenshots_store/Flow.yaml`

**Interfaces:**
- Consumes: IDs produced in Tasks 2, 3, and 4
- Produces: Maestro flow using only `id:` selectors for static interactive elements

- [ ] **Step 1: 開啟 `screenshots_store/Flow.yaml`**

Use `read` to view the entire file.

- [ ] **Step 2: 替換「客戶列表」tab**

```yaml
# before
- tapOn: "客戶列表"

# after
- tapOn:
    id: "profile_customer_list_item"
```

- [ ] **Step 3: 替換訂單表單右上 person_search 圖示座標**

```yaml
# before（訂單表單右上 person_search，開啟客戶選擇）
- tapOn:
    point: "80%,8%"

# after
- tapOn:
    id: "order_form_customer_search_icon"
```

- [ ] **Step 4: 替換新增客戶按鈕**

```yaml
# before
- tapOn: "新增客戶"

# after
- tapOn:
    id: "customer_layout_add_button"
```

- [ ] **Step 5: 替換客戶列表搜尋欄圖示座標**

```yaml
# before（客戶列表右上放大鏡，開啟搜尋欄）
- tapOn:
    point: "93%,16%"

# after
- tapOn:
    id: "customer_layout_search_icon"
```

- [ ] **Step 6: Commit**

```bash
cd /Volumes/UTM2/Developer/sales-order-netsuite/sales-order-app
git add integration_test/.maestro/screenshots_store/Flow.yaml
git commit -m "test(maestro): use id selectors in Android store screenshot flow"
```

---

### Task 6: 更新 iOS store 截圖 flow

**Files:**
- Modify: `sales-order-app/integration_test/.maestro/screenshots_store_ios/Flow.yaml`

**Interfaces:**
- Consumes: IDs produced in Tasks 2, 3, and 4
- Produces: Maestro flow using only `id:` selectors for static interactive elements

- [ ] **Step 1: 開啟 `screenshots_store_ios/Flow.yaml`**

Use `read` to view the entire file.

- [ ] **Step 2: 替換「客戶列表」選單項目、搜尋圖示、新增客戶按鈕、訂單客戶搜尋圖示（與 Task 5 相同）**

Apply the same selector replacements as Task 5:

```yaml
- tapOn:
    id: "profile_customer_list_item"

- tapOn:
    id: "order_form_customer_search_icon"

- tapOn:
    id: "customer_layout_add_button"

- tapOn:
    id: "customer_layout_search_icon"
```

- [ ] **Step 3: 確認已完成的返回鈕 ID**

Verify that the file already contains:

```yaml
- tapOn:
    id: "customer_create_back_button"
- tapOn:
    id: "customer_layout_back_button"
- tapOn:
    id: "salesorder_item_layout_back_button"
```

If any are missing, add them.

- [ ] **Step 4: Commit**

```bash
cd /Volumes/UTM2/Developer/sales-order-netsuite/sales-order-app
git add integration_test/.maestro/screenshots_store_ios/Flow.yaml
git commit -m "test(maestro): use id selectors in iOS store screenshot flow"
```

---

### Task 7: 跑 Android store 截圖 flow 驗證

**Files:**
- Test: `sales-order-app/integration_test/.maestro/screenshots_store/Flow.yaml`

**Interfaces:**
- Consumes: all code changes from previous tasks

- [ ] **Step 1: 執行 Android 截圖 flow**

Run:
```bash
cd /Volumes/UTM2/Developer/sales-order-netsuite/sales-order-app
task fastlane:screenshots
```

- [ ] **Step 2: 檢查結果**

Expected:
- Flow 跑完，exit code 0
- `integration_test/.maestro/test_output_directory/` 下最新目錄內有 `report.xml` 且無 failure
- 產出 `store/` 截圖

If any step fails with `Element not found` for an `id:`, go back to the corresponding Task and verify the Flutter `Semantics(identifier: ...)` is actually exposed in the accessibility tree.

- [ ] **Step 3: Commit（若有 flow 微調）**

If any small timing or selector adjustments were needed, commit them with:

```bash
git add integration_test/.maestro/screenshots_store/Flow.yaml
git commit -m "test(maestro): fix Android store screenshot flow after id migration"
```

---

### Task 8: 跑 iOS store 截圖 flow 驗證

**Files:**
- Test: `sales-order-app/integration_test/.maestro/screenshots_store_ios/Flow.yaml`

**Interfaces:**
- Consumes: all code changes from previous tasks

- [ ] **Step 1: 執行 iOS 截圖 flow**

Run:
```bash
cd /Volumes/UTM2/Developer/sales-order-netsuite/sales-order-app
task fastlane:screenshots:ios
```

- [ ] **Step 2: 檢查結果**

Expected:
- Flow 跑完，exit code 0
- `integration_test/.maestro/test_output_directory/` 下最新目錄內有 `report.xml` 且無 failure
- 產出 `store_ios/` 截圖

- [ ] **Step 3: Commit（若有 flow 微調）**

```bash
git add integration_test/.maestro/screenshots_store_ios/Flow.yaml
git commit -m "test(maestro): fix iOS store screenshot flow after id migration"
```

---

### Task 9: 更新 superproject gitlink

**Files:**
- Modify: `sales-order-app` gitlink in superproject root

**Interfaces:**
- Consumes: commits from sales-order-app submodule

- [ ] **Step 1: 在 superproject 更新 submodule pointer**

Run:
```bash
cd /Volumes/UTM2/Developer/sales-order-netsuite
git add sales-order-app
git commit -m "chore(submodule): bump sales-order-app for Maestro semantic IDs"
git push origin development
```

- [ ] **Step 2: 確認遠端提交**

Run: `git log -1 --stat sales-order-app`
Expected: shows updated gitlink.

---

## Self-Review Checklist

- [ ] Spec coverage: `Testable` widget、命名規範、兩條 flow 的靜態 selector 替換、既有 ID 不動、動態資料維持現況，都有對應 Task。
- [ ] Placeholder scan: 計畫中無 TBD/TODO，每個步驟都有具體程式碼或指令。
- [ ] Type consistency: `Testable` constructor 簽名在 Task 1 定義，後續任務用法一致。
- [ ] Scope: 僅兩條 store 截圖 flow，符合 spec 範圍。
- [ ] Task order: Flutter IDs 建立完成後，才更新 Maestro flow 與執行驗證。

## Execution Handoff

Plan complete and saved to `docs/superpowers/plans/2026-08-06-store-screenshot-semantic-ids.md`.

Two execution options:

1. **Subagent-Driven (recommended)** — Dispatch a fresh subagent per task, review between tasks, fast iteration.
2. **Inline Execution** — Execute tasks in this session using `executing-plans`, batch execution with checkpoints.

Which approach?
