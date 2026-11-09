# 提示词：生成 Patrol 端到端测试

**用途**：为 Flutter 应用生成完整的 Patrol UI 自动化测试用例

---

## 📋 适用场景

- 需要测试跨页面用户流程
- 涉及原生交互（权限、通知、相机）
- 验证关键业务路径（注册、支付、订单）
- 替代手动回归测试

---

## 📥 输入要求

在 Cursor 中准备：

1. **应用入口**：`lib/main.dart`（或测试专用入口）
2. **关键页面/Widget**：涉及的页面源码（如登录页、商品列表）
3. **已有测试示例**（可选）：项目中现有的 Patrol 或集成测试作为风格参考
4. **明确测试场景**：用户操作步骤清单

---

## 💬 提示词模板

```markdown
我需要为以下 Flutter 应用流程生成 Patrol 端到端测试。

**应用场景**：[简述应用类型，如电商应用、社交应用]

**测试流程**：
1. [第一步操作，如"打开应用"]
2. [第二步操作，如"点击搜索框"]
3. [第三步操作，如"输入关键词并搜索"]
4. [验证点，如"验证搜索结果显示"]
5. [可能的原生交互，如"授予位置权限"]

**涉及的页面/Widget**：
- [列出关键页面，如 HomePage, SearchPage, ProductDetailPage]

**原生交互**（如有）：
- [ ] 相机权限
- [ ] 位置权限
- [ ] 通知权限
- [ ] 深链接跳转
- [ ] 其他：[说明]

**要求**：

1. **测试文件路径**：`integration_test/[场景名称]_test.dart`（如 `integration_test/product_search_test.dart`）

2. **代码规范**：
   - 使用 `patrolTest()` 定义测试用例
   - 使用 `$(Finder)` 扩展方法定位元素（Patrol 推荐语法）
   - 用 `Key` 或 `Semantics label` 定位 Widget
   - 使用 `$.native.grantPermission*()` 处理权限弹窗
   - 使用 `$.native.tap(Selector(...))` 点击原生 UI 元素
   - 用 `waitUntilVisible()` 等待异步加载
   - 用 `expect()` 验证结果

3. **覆盖场景**：
   - ✅ 正常路径（happy path）
   - ✅ 至少一个边界场景（如网络异常、权限拒绝、空数据）

4. **完整性**：
   - 包含所有必要的 import
   - 包含 `app.main()` 启动应用
   - 可直接运行 `patrol test -t integration_test/[文件名]_test.dart`

**示例 Widget Key 标记**（如需要）：
```dart
// 在源码中为关键 Widget 添加 Key
TextField(key: Key('searchField'), ...)
ElevatedButton(key: Key('searchButton'), ...)
```

生成完整测试代码，不要省略任何部分。
```

---

## ✅ 验证步骤

生成后执行：

```bash
# 运行 Patrol 测试（iOS 模拟器）
patrol test -t integration_test/[文件名]_test.dart

# 或指定 Android
patrol test -t integration_test/[文件名]_test.dart --flavor dev
```

**检查清单**：
- [ ] 测试在模拟器/真机上通过（0 failed）
- [ ] 页面跳转流畅，无卡顿
- [ ] 原生交互正确触发（如权限弹窗）
- [ ] 断言覆盖核心验证点
- [ ] 边界场景已测试

---

## 🛠️ 常见调整

生成后可能需要手动微调：

### 1. 添加 Key 到源码

AI 可能假设 Widget 已有 Key，但实际需要手动添加：

```dart
// lib/pages/search_page.dart
class SearchPage extends StatelessWidget {
  @override
  Widget build(BuildContext context) {
    return Scaffold(
      body: Column(
        children: [
          TextField(
            key: Key('searchField'),  // 添加 Key
            decoration: InputDecoration(hintText: '搜索商品'),
          ),
          ElevatedButton(
            key: Key('searchButton'),  // 添加 Key
            onPressed: _onSearch,
            child: Text('搜索'),
          ),
        ],
      ),
    );
  }
}
```

### 2. 等待异步加载

```dart
// ❌ 可能失败：元素还在加载中
await $(#productList).tap();

// ✅ 稳定：等待元素出现
await $(#productList).waitUntilVisible();
await $(#productList).tap();
```

### 3. 处理原生权限弹窗

```dart
// iOS 权限授予
await $.native.grantPermissionWhenInUse();  // 位置权限 "使用时允许"
await $.native.grantPermissionOnlyThisTime();  // 一次性权限

// Android 权限授予
await $.native.tap(Selector(text: 'Allow'));

// 拒绝权限（测试拒绝场景）
await $.native.tap(Selector(text: 'Deny'));
```

### 4. 处理滚动查找元素

```dart
// 滚动直到元素可见
await $(#submitButton).scrollTo();
await $(#submitButton).tap();

// 或在特定容器内滚动
await $(ListView).containing($(#targetItem)).scrollTo();
```

---

## 💡 提示

1. **先运行一遍手动流程**：在模拟器上手动操作一遍，确认流程可行
2. **逐步验证**：先让 AI 生成主流程，验证通过后再加边界场景
3. **使用语义化 Key**：如 `Key('loginButton')` 比 `Key('btn1')` 更易维护
4. **CI 环境注意事项**：
   - 使用 `--dart-define=ENV=test` 切换到测试环境
   - Mock 网络请求或使用独立测试数据库
   - 设置合理的 timeout

---

## 🔗 相关

- 工作流：[workflows/patrol-ui-automation.md](../../workflows/patrol-ui-automation.md)
- 修复不稳定测试：[fix-patrol-flaky.md](fix-patrol-flaky.md)
- Widget 测试生成：[generate-widget-test.md](generate-widget-test.md)
