# Store 截圖 Maestro Flow 全面改用 Semantic ID

## 背景

`sales-order-app` 的商店截圖由 Maestro 自動化產生。現有 `screenshots_store/Flow.yaml` 與 `screenshots_store_ios/Flow.yaml` 混用多種 selector：

- `tapOn: "客戶列表"` — 依賴 tab 文字
- `tapOn: "新增客戶"` — 依賴按鈕 tooltip
- `tapOn: "Tab 2 of 4"` — 依賴自動產生的 tab 索引文字
- `point: "80%,8%"` / `point: "93%,16%"` — 依賴螢幕相對座標
- `(?s).*首頁` / `(?s).*全部部門` — 依賴正規表示式比對文字

這些 selector 容易因為：語言切換、UI 文案調整、裝置尺寸差異、Flutter 元件更新而失效。最近一次 iOS store 截圖就在「新增客戶」後的返回步驟失敗，因為舊 flow 假設右上角有 X 鈕，但實際上現在是獨立全螢幕 stepper。

## 目標

把兩條 store 截圖 flow 中所有**靜態互動元件**都改用 `id:` selector，並在 Flutter 端用統一方式提供對應的 semantic id。動態資料產生的項目（客戶卡片、下拉選項）暫時維持文字/regex 比對。

## 範圍

- **Flutter**：`sales-order-app/lib/layer_presentation/` 中與 store 截圖 flow 相關的畫面
- **Maestro**：
  - `sales-order-app/integration_test/.maestro/screenshots_store/Flow.yaml`
  - `sales-order-app/integration_test/.maestro/screenshots_store_ios/Flow.yaml`
- **不處理**：
  - 其他 Maestro flow（例如 `screenshots/`、`salesrep_login/`）
  - 純顯示文字/圖片
  - 動態產生的列表項目與下拉選項

## 架構

```
Flutter widget
  └─ Testable(id: 'xxx_yyy_zzz', child: IconButton(...))
        └─ Semantics(identifier: 'xxx_yyy_zzz', child: IconButton(...))

Maestro
  └─ tapOn:
       id: 'xxx_yyy_zzz'
```

新增一個 `Testable` widget，把所有需要給 Maestro 定位的靜態互動元件包起來。`Testable` 唯一職責就是注入 `Semantics(identifier: id)`，讓 Maestro 能用 `id:` selector 穩定命中。

## 命名規範

格式：`<screen>_<element>_<action>`

- 全小寫 `snake_case`
- `screen`：route / screen 語意名稱，例如 `customer_layout`、`salesorder_item_layout`、`order_form`
- `element`：元件語意，例如 `back_button`、`search_icon`、`add_button`
- `action`：動作或元件類型，例如 `button`、`field`、`icon`、`tab`
- 同畫面多個同類型時，用動作前綴區分，例如 `customer_signin_submit_button`、`customer_signin_cancel_button`

### 既有 ID 不更動

以下已經存在的 id 保持原樣：

- `select_signin_screen_salesrep_button`
- `auth_form_email_field`
- `auth_form_password_field`
- `customer_signin_screen_submit_button`
- `customer_signin_screen_cancel_button`
- `select_signin_screen_customer_button`
- `customer_create_back_button`
- `customer_layout_back_button`
- `salesorder_item_layout_back_button`

## 實作細節

### `Testable` widget

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

### 使用範例

```dart
// AppBar 返回鈕
AppBar(
  leading: Testable(
    id: 'customer_layout_back_button',
    child: leading,
  ),
)

// 無文字圖示按鈕
Testable(
  id: 'customer_layout_search_icon',
  child: IconButton(
    icon: const Icon(Icons.person_search),
    onPressed: _openSearch,
  ),
)

// FloatingActionButton
Testable(
  id: 'customer_layout_add_button',
  child: FloatingActionButton(
    onPressed: _createCustomer,
    child: const Icon(Icons.add),
  ),
)
```

### Fallback

若目標 Flutter 版本不支援 `Semantics.identifier`（需要較新版本），則改用 `key: ValueKey(id)` 包裝 child，並在文件中註明。本專案目前使用的 Flutter 版本經實測已支援 `Semantics.identifier`，因此優先使用它。

## Maestro 更新項目

兩條 flow 中預計改為 `id:` selector 的靜態元件：

| 畫面 | 原 selector | 新 id |
|---|---|---|
| 功能頁 | `tapOn: "客戶列表"`（選單項目，非底部 tab） | `profile_customer_list_item` |
| 客戶列表 | `point: "80%,8%"`（右上角搜尋圖示） | `customer_layout_search_icon` |
| 客戶列表 | `tapOn: "新增客戶"`（FAB） | `customer_layout_add_button` |
| 新增客戶 | `point: "95%,14%"`（舊 X 鈕） | `customer_create_back_button`（已完成） |
| 客戶列表 | `back` / `tapOn: "Back"` | `customer_layout_back_button`（已完成） |
| 訂單表單 | `point: "93%,16%"`（放大鏡搜尋） | `order_form_customer_search_icon` |
| 訂單項目 | `back` / `tapOn: "Back"` | `salesorder_item_layout_back_button`（已完成） |
| 登入選擇 | `id: "select_signin_screen_*"` 等 | 維持既有 id |

`extendedWaitUntil` 中等待頁面標題（例如 `(?s).*首頁`、`(?s).*全部部門`）暫時維持 regex，因為它們是頁面載入完成判斷，不是互動元件。

## 測試與驗證

1. **本地執行兩條 store 截圖 flow**：
   - `task fastlane:screenshots:ios`
   - `task fastlane:screenshots`
2. **成功標準**：
   - 兩條 flow 都能跑完並產出截圖
   - 沒有使用 `point:` 座標的靜態按鈕
   - 所有靜態 tab / 按鈕 / 圖示都用 `id:` selector
   - 不再出現 `Element not found: text matching regex: ...`
3. **驗收方式**：跑完 flow 後，檢視 `test_output_directory/` 中的截圖與 `commands.json` 確認沒有失敗步驟。

## 風險與對策

| 風險 | 對策 |
|---|---|
| `Semantics.identifier` 在某些 Widget 上無效 | 改用 `ValueKey` 或雙重包裝 |
| 同一畫面多個相似元件難以命名 | 用動作/位置前綴區分，例如 `order_form_customer_search_icon` |
| Maestro 對某些 `Semantics` 層級不敏感 | 用 Maestro hierarchy 檢查確認 id 出現在無障礙樹中 |

## 後續建議

- 日後新增互動元件時，若會被 Maestro 操作，優先用 `Testable` 包裝並給予 id。
- 動態資料項目可進一步改為「容器 id + 資料 key」，例如 `customer_list_item_C000026`，但本次不處理。
