# Patrol UI 自动化工作流

**用 AI 辅助编写 Flutter 端到端 UI 自动化测试**

> 作者：Loong (SSLoong) | 日期：2026-11-09

---

## 📋 背景

Widget 测试聚焦单个组件，Golden 测试验证视觉快照，但都无法覆盖**完整用户旅程**和**原生平台交互**（权限、通知、深链接）。Patrol 是一个强大的端到端测试框架，支持跨页面流程验证和原生操作模拟，AI 可以帮助生成测试骨架和复杂场景用例。

---

## 🎯 目标

- ✅ 自动化完整用户流程（登录 → 浏览 → 操作 → 验证）
- ✅ 测试原生交互（权限请求、推送通知、相机调用）
- ✅ 跨页面导航与状态验证
- ✅ 在真实/模拟器上运行，验证实际体验

---

## 🔧 适用场景

| 场景 | 适合度 | 说明 |
|------|--------|------|
| 关键业务流程 | ⭐⭐⭐⭐⭐ | 如注册、支付、订单流程 |
| 原生权限交互 | ⭐⭐⭐⭐⭐ | 相机、位置、通知等 |
| 多页面导航 | ⭐⭐⭐⭐ | 验证路由跳转正确性 |
| 复杂手势操作 | ⭐⭐⭐⭐ | 长按、滑动、多点触控 |
| 纯 UI 单组件 | ⭐⭐ | Widget 测试更合适 |
| 视觉回归 | ⭐⭐ | Golden 测试更合适 |

---

## 🚀 完整流程

### Step 0: 环境准备

**人工操作**（首次使用需配置）：

```bash
# 安装 Patrol CLI
dart pub global activate patrol_cli

# 在项目中添加依赖
flutter pub add --dev patrol
flutter pub add --dev patrol_finders

# 初始化 Patrol（生成配置文件）
patrol bootstrap

# 验证安装
patrol doctor
```

**检查点**：
- `patrol_cli` 可执行（`patrol --version`）
- `pubspec.yaml` 包含 `patrol` 和 `patrol_finders`
- `integration_test/` 目录已创建

---

### Step 1: 定义测试场景

**人工操作**：明确要测试的用户旅程，列出关键步骤。

**示例场景**（电商应用）：
1. 用户打开应用
2. 点击搜索框，输入"笔记本"
3. 点击搜索结果第一个商品
4. 验证商品详情页显示正确
5. 点击"加入购物车"
6. 授予通知权限（原生弹窗）
7. 验证购物车角标数字更新

**AI 提示词**（场景分析）：

```markdown
我需要为以下用户流程设计 Patrol 端到端测试：

**应用类型**：[电商/社交/工具类]

**流程描述**：
[逐步描述用户操作路径]

**涉及原生交互**：
- [ ] 权限请求（相机/位置/通知）
- [ ] 深链接跳转
- [ ] 分享/打开第三方应用
- [ ] 系统设置修改

请列出测试需要覆盖的关键验证点和可能的边界场景（网络异常、权限拒绝等）。
```

**输出示例**：AI 会列出需要的 `patrol.native.tap()` 操作、状态断言等。

---

### Step 2: 生成测试代码

**AI 提示词**（可复制，见 [prompts/testing/generate-patrol-test.md](../prompts/testing/generate-patrol-test.md)）：

```markdown
基于上述场景，生成 Patrol 端到端测试代码。

**文件路径**：`integration_test/app_test.dart`

**要求**：

1. 使用 `patrolTest()` 定义测试用例
2. 使用 `patrol.native` 处理原生交互（权限弹窗）
3. 使用 `$(Finder)` 扩展方法定位元素（Patrol 语法糖）
4. 用 `waitUntilVisible()` 等待异步加载
5. 覆盖正常路径 + 至少一个边界场景（如网络失败）

生成完整可运行代码，包含所有 import。
```

**输出示例**（简化版）：

```dart
import 'package:flutter_test/flutter_test.dart';
import 'package:patrol/patrol.dart';
import 'package:my_app/main.dart' as app;

void main() {
  patrolTest(
    '用户搜索商品并加入购物车',
    ($) async {
      // 启动应用
      app.main();
      await $.pumpAndSettle();

      // 点击搜索框
      await $(#searchField).tap();
      
      // 输入搜索关键词
      await $(#searchField).enterText('笔记本');
      await $.native.pressHome(); // 收起键盘
      
      // 点击搜索按钮
      await $(#searchButton).tap();
      
      // 等待搜索结果加载
      await $(#productList).waitUntilVisible();
      
      // 点击第一个商品
      await $(#productCard).at(0).tap();
      
      // 验证详情页显示
      expect($(#productDetailTitle).visible, true);
      
      // 加入购物车
      await $(#addToCartButton).tap();
      
      // 处理通知权限弹窗（原生交互）
      await $.native.grantPermissionWhenInUse();
      
      // 验证购物车角标更新
      expect($(#cartBadge).text, '1');
    },
  );

  patrolTest(
    '搜索无结果时显示空状态',
    ($) async {
      app.main();
      await $.pumpAndSettle();

      await $(#searchField).tap();
      await $(#searchField).enterText('不存在的商品xyz123');
      await $(#searchButton).tap();

      // 验证空状态提示
      await $(#emptyStateMessage).waitUntilVisible();
      expect($(#emptyStateMessage).text, contains('未找到相关商品'));
    },
  );
}
```

---

### Step 3: 本地验证（必须！）

**人工操作**：

```bash
# 在 iOS 模拟器或 Android 模拟器上运行
patrol test -t integration_test/app_test.dart

# 或指定平台
patrol test --target integration_test/app_test.dart --flavor dev --dart-define=ENV=staging
```

**检查清单**：
- [ ] 测试在模拟器/真机上通过（0 failed）
- [ ] 原生交互正确触发（权限弹窗、通知）
- [ ] 页面跳转流畅，无卡顿或崩溃
- [ ] 边界场景覆盖（网络异常、权限拒绝）

**常见问题**：
- ❌ `Finder 找不到元素`：检查 Widget 是否设置了 `Key` 或 `Semantics label`
- ❌ `原生权限弹窗未出现`：确保模拟器/真机首次运行，权限未被缓存
- ❌ `测试超时`：增加 `timeout` 或检查异步等待逻辑
- ❌ `patrol_cli 未找到`：运行 `dart pub global activate patrol_cli`

---

### Step 4: 处理不稳定性（Flaky Tests）

**若测试间歇性失败**，用以下提示词让 AI 诊断：

```markdown
Patrol 测试间歇性失败，错误信息：

[粘贴错误日志]

测试代码片段：
[粘贴相关代码]

可能原因：
- 异步等待时间不足？
- 动画未完成？
- 网络请求时序问题？

请分析并提供修复方案（增加 waitUntilVisible、使用 retry、mock 网络请求等）。
```

**修复示例**（见 [prompts/testing/fix-patrol-flaky.md](../prompts/testing/fix-patrol-flaky.md)）：

```dart
// ❌ 不稳定：立即点击可能元素还在动画中
await $(#submitButton).tap();

// ✅ 稳定：等待元素可见且可交互
await $(#submitButton).waitUntilVisible();
await $(#submitButton).tap();

// ✅ 或使用重试机制
await $.tester.runAsync(() async {
  await Future.delayed(Duration(milliseconds: 500));
  await $(#submitButton).tap();
});
```

---

### Step 5: CI/CD 集成（可选）

**GitHub Actions 示例**：

```yaml
name: Patrol E2E Tests

on: [pull_request]

jobs:
  patrol_test:
    runs-on: macos-latest
    steps:
      - uses: actions/checkout@v3
      - uses: subosito/flutter-action@v2
      - run: flutter pub get
      - run: dart pub global activate patrol_cli
      
      # iOS 测试
      - run: patrol test --target integration_test/app_test.dart

      # Android 测试（需配置模拟器）
      # - uses: reactivecircus/android-emulator-runner@v2
      #   with:
      #     api-level: 29
      #     script: patrol test
```

---

## ✅ 验证清单

完成后确认：

- [ ] 测试文件位于 `integration_test/` 目录
- [ ] 所有测试通过 `patrol test`
- [ ] 覆盖关键用户流程（至少 1 个完整路径）
- [ ] 原生交互（如权限）已验证
- [ ] 边界场景（网络异常、权限拒绝）已测试
- [ ] 测试运行稳定（连续运行 3 次不失败）

---

## 🚦 与其他测试类型的边界

| 测试类型 | 职责 | 何时用 |
|----------|------|--------|
| **Widget 测试** | 单个组件逻辑和交互 | 组件开发阶段 |
| **Golden 测试** | UI 视觉回归 | 防止样式意外变化 |
| **Patrol 测试** | 端到端流程 + 原生交互 | 关键业务路径验证 |
| **单元测试** | 纯业务逻辑（ViewModel/Service） | 算法、状态管理 |

**原则**：
- 不要用 Patrol 测试单个 Widget 内部逻辑（用 Widget 测试）
- 不要用 Patrol 验证像素级 UI 一致性（用 Golden 测试）
- Patrol 聚焦"用户能否完成核心任务"

---

## 🤔 FAQ

### Q1: Patrol 和 Integration Test 有什么区别？
**A**: 
- `flutter drive` (旧集成测试)：需要单独运行驱动程序，配置复杂
- **Patrol**：基于 `integration_test` 包，语法更简洁，支持原生操作 (`$.native`)，推荐新项目使用

### Q2: 如何处理登录态？
**A**: 
```dart
patrolTest('需要登录的流程', ($) async {
  // 方法 1: 每次测试都登录
  await loginAsUser($, email: 'test@example.com');
  
  // 方法 2: 使用 setUp 注入 mock token
  // (需配合 Dependency Injection)
});
```

### Q3: 测试数据如何管理？
**A**: 
- 开发环境：使用独立测试数据库 / mock API
- 生产环境：绝不运行自动化测试
- 建议：在 `main.dart` 中根据 `--dart-define=ENV=test` 切换 API 端点

### Q4: Patrol 能测试哪些原生操作？
**A**: 
- ✅ 权限请求 (`$.native.grantPermission*`)
- ✅ 原生弹窗点击 (`$.native.tap(Selector(text: 'Allow'))`)
- ✅ 打开系统设置 (`$.native.openApp(iosSelector: 'Settings')`)
- ✅ 深链接跳转 (`$.native.openUrl('myapp://product/123')`)
- ❌ 复杂原生 UI 定制（需编写原生测试）

---

## 💡 最佳实践

1. **先手写一个 Patrol 测试作为示例**
   - 让 AI 学习你的项目风格（Key 命名、页面结构）
   
2. **用 Key 标记关键 Widget**
   ```dart
   ElevatedButton(
     key: Key('submitButton'),  // 或使用 ValueKey(#submitButton)
     onPressed: _handleSubmit,
     child: Text('提交'),
   )
   ```

3. **分层测试金字塔**
   - 单元测试：大量（快速、便宜）
   - Widget 测试：适量（覆盖组件逻辑）
   - Patrol 测试：少量（聚焦核心流程，运行慢）

4. **CI 中选择性运行**
   - PR 阶段：只跑关键流程（如登录、支付）
   - 发版前：全量运行所有 Patrol 测试

5. **记录 AI 生成的错误模式**
   - 把常见修复（如增加 `waitUntilVisible`）加入 `.cursor/rules`

---

## 🔗 相关资源

- 提示词模板：
  - [prompts/testing/generate-patrol-test.md](../prompts/testing/generate-patrol-test.md)
  - [prompts/testing/fix-patrol-flaky.md](../prompts/testing/fix-patrol-flaky.md)
- 姊妹仓库 XCUITest 工作流：[ios-ai-coding-kit](https://github.com/SSLoong/ios-ai-coding-kit)
- Patrol 官方文档：[https://patrol.leancode.co](https://patrol.leancode.co)
- Flutter 集成测试：[https://docs.flutter.dev/testing/integration-tests](https://docs.flutter.dev/testing/integration-tests)

---

**让 AI 帮你测试完整用户旅程 🚁**
