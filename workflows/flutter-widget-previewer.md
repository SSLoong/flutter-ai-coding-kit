# Flutter Widget Previewer 工作流

**用 AI 生成 @Preview 注解，在预览器中实时查看组件渲染效果，快速迭代设计变体**

> 作者：Loong (SSLoong) | 日期：2026-11-24

---

## 📋 背景

Flutter Widget Previewer 是官方工具链内置的组件级实时预览功能（Flutter 3.47 起标为 stable）。通过 `@Preview` 注解，你可以在 IDE 侧栏或浏览器中即时渲染单个 Widget，独立于完整应用运行。这让设计迭代更快：无需重启应用、无需导航到特定页面，就能查看亮暗色、不同尺寸、各种配置下的组件效果。

Flutter 3.50（2026-11-11）针对 Widget Previewer 带来一批实用改进：
- **更安全的热重载处理**：预览器会拦截不安全的热重载并自动触发热重启
- **无约束 Widget 的更清晰报错**：当 Widget 未约束时给出明确提示
- **预览组 PageStorage 隔离**：每个预览独立保存状态，不互相干扰
- **窄面板布局优化**：在较窄的 IDE 面板中也能正常显示控件

**核对来源与日期**：
- [Flutter Widget Previewer 官方文档](https://docs.flutter.dev/tools/widget-previewer) — 核对日期：2026-11-24
- [Flutter 3.50 Release Notes](https://docs.flutter.dev/release/release-notes/release-notes-3.50.0) — 核对日期：2026-11-24
- [What's new in Flutter 3.50](https://flutter.dev/blog/whats-new-in-flutter-3-50) — 核对日期：2026-11-24

**与测试的关系**：
Widget Previewer **不是测试替代品**。它解决的是**设计迭代与视觉验证**，而非行为保证。团队仍需 Widget 测试验证交互逻辑、Golden 测试捕捉视觉回归、端到端测试覆盖完整流程。Previewer 的价值在于让开发者在写测试前就能快速确认「这个组件长什么样」。

AI 可以帮助为指定 Widget 生成 @Preview 注解（含多配置、分组、wrapper），开发者在预览器里实时看效果，决定是否调整或补测试。

---

## 🎯 目标

- ✅ 为业务展示组件添加 `@Preview` 注解，在预览器中实时渲染
- ✅ 覆盖设计关键变体：亮暗色、文本缩放、尺寸约束
- ✅ 用 `group` 组织相关预览，用自定义注解批量生成配置
- ✅ 处理预览器限制（dart:io / dart:ffi / 资源路径 / 回调约束）导致的加载失败
- ✅ 决策何时用预览器、何时补 Widget/Golden 测试

---

## 🔧 适用场景

| 场景 | 适合度 | 说明 |
|------|--------|------|
| 纯展示组件（Card、Banner、Info 面板） | ⭐⭐⭐⭐⭐ | 无复杂交互，实时预览设计变体 |
| 亮暗色主题适配验证 | ⭐⭐⭐⭐⭐ | 快速对比 Brightness.light/dark |
| 多尺寸响应式布局（手机/平板） | ⭐⭐⭐⭐⭐ | 用 `size` 参数约束不同屏幕尺寸 |
| 文本缩放（无障碍）验证 | ⭐⭐⭐⭐⭐ | 用 `textScaleFactor` 检查大字号显示 |
| 需要 InheritedWidget / Provider 的组件 | ⭐⭐⭐⭐ | 用 `wrapper` 注入依赖 |
| 调用原生插件（相机、定位、dart:io）的组件 | ⭐⭐ | 预览器基于 Flutter Web，无法调用原生 API；需条件导入 |
| 复杂交互与状态逻辑验证 | ⭐⭐ | 优先用 Widget 测试；预览器主要看外观 |

**选择建议**：
- **优先 Previewer**：设计迭代、主题对比、尺寸矩阵、原型演示
- **优先 Widget 测试**：交互逻辑、状态变化、业务断言（链到 [workflows/widget-test-generation.md](widget-test-generation.md)）
- **优先 Golden 测试**：视觉回归基线、像素级对比（链到 [workflows/golden-test-practice.md](golden-test-practice.md)）
- **优先跑整应用**：完整导航流程、平台插件、真实数据

---

## 🚀 完整流程

### Step 1: 确认环境与启动预览器

**人工操作**：确认 Flutter SDK 版本

```bash
flutter --version
# 确保 Flutter ≥ 3.47（Previewer stable）；建议 ≥ 3.50（含最新修复）
```

**启动预览器**：

**方案一：IDE 自动启动**（Android Studio / IntelliJ / Visual Studio Code）
- Android Studio / IntelliJ：打开侧栏 "Flutter Widget Preview" 标签页
- Visual Studio Code：打开侧栏 "Flutter Widget Preview" 标签页

**方案二：命令行启动**（适用所有编辑器，含 Cursor）

```bash
cd /path/to/your/flutter/project  # 进入项目根目录
flutter widget-preview start
```

这会启动本地服务并在浏览器打开实时预览环境。预览器会将构建缓存存放在项目根目录下的 `.widget_preview/` 文件夹。

**关于 Cursor**：
官方文档明确支持 Android Studio、IntelliJ、Visual Studio Code。Cursor 是 VS Code 兼容编辑器，可尝试 Flutter 扩展是否出现同名面板，但不保证。稳妥方案是用 `flutter widget-preview start` 命令行在浏览器预览。

**检查清单**：
- [ ] Flutter 版本 ≥ 3.47
- [ ] 预览器已打开（IDE 侧栏或浏览器）
- [ ] 项目根目录下出现 `.widget_preview/` 缓存目录

---

### Step 2: 准备上下文，让 AI 分析 Widget

**人工操作**：在 Cursor 中打开目标 Widget 文件（如 `lib/widgets/product_card.dart`），附上相关依赖（数据模型、主题配置、Provider 等）。

**AI 提示词**（可复制）：

```markdown
我需要为这个 Widget 添加 @Preview 注解，在 Flutter Widget Previewer 中实时预览。请先分析：

1. 这是纯展示组件还是包含复杂交互？
2. 依赖哪些外部数据（需 mock）？是否依赖 InheritedWidget / Provider？
3. 是否调用原生插件、dart:io、dart:ffi（预览器基于 Flutter Web，无法调用）？
4. 需要验证哪些设计变体：亮暗色、尺寸、文本缩放、空/错误状态？

列出建议的预览场景，说明是用多个 @Preview 注解，还是用自定义 MultiPreview 批量生成。
```

**检查点**：确认 AI 识别出必要的 wrapper（如 MaterialApp / Theme）、尺寸约束、以及可能的预览器限制。

---

### Step 3: 生成 @Preview 注解

**AI 提示词**（可复制）：

```markdown
基于上述分析,为这个 Widget 生成 @Preview 注解代码，要求：

1. **基础结构**：
   - import `package:flutter/widget_previews.dart`
   - 创建返回目标 Widget 的顶层函数或静态方法
   - 用 @Preview 注解标注（可叠多个）

2. **必须覆盖的参数**（官方 API，核对日期 2026-11-24）：
   - `name`: 预览描述名称（如 'Product Card - Light'）
   - `group`: 分组名称（如 'Product Card'），便于在预览器中过滤
   - `size`: 尺寸约束，用 `Size(宽, 高)` 显式约束（如 `Size(375, 200)` 模拟手机宽度）
   - `textScaleFactor`: 文本缩放因子（如 `1.5` 模拟无障碍大字号）
   - `wrapper`: 返回 Widget 的回调，用于包裹 MaterialApp / Theme / Provider（回调必须 public 且 const）
   - `theme`: 返回 `PreviewThemeData` 的回调，用于自定义主题
   - `brightness`: `Brightness.light` 或 `Brightness.dark`，设置初始亮暗色
   - `localizations`: 本地化配置回调

3. **设计变体覆盖**（至少）：
   - 亮色模式（Brightness.light）+ 手机尺寸（如 375x667）
   - 暗色模式（Brightness.dark）+ 手机尺寸
   - 平板尺寸（如 768x1024）或桌面尺寸（可选）
   - 大文本缩放（textScaleFactor: 1.5）（可选，建议有）

4. **处理依赖**：
   - 若组件依赖 Theme，用 `wrapper` 包裹 MaterialApp
   - 若依赖 Provider / InheritedWidget，在 wrapper 中注入 mock 数据
   - 若依赖外部数据，创建 mock 实例（避免动态值如 DateTime.now()）

5. **处理预览器限制**（官方文档 Restrictions）：
   - 传给注解的回调必须 public 且 constant（如 wrapper / theme / localizations）
   - 不支持 dart:io / dart:ffi / 原生插件；若必须调用，提示用条件导入并在 Web 平台返回占位
   - 资源路径必须用 package 路径（如 `packages/my_package_name/assets/my_image.png`），不要用本地 `assets/` 相对路径
   - 无约束 Widget 会被自动限制到约预览器宽高的一半；建议用 `size` 参数显式约束

6. **代码组织**：
   - 可叠多个 @Preview 注解在同一函数上，或用自定义 MultiPreview（扩展 MultiPreview 类）批量生成
   - 用 `group` 参数给相关预览分组
   - wrapper 函数命名清晰（如 `_materialWrapper`），并标记为 const

生成完整代码，包含必要的 import 和注释。
```

**输出示例**（参考，实际以 AI 生成为准）：

```dart
import 'package:flutter/widget_previews.dart';
import 'package:material_ui/material_ui.dart';
import 'package:my_app/widgets/product_card.dart';
import 'package:my_app/models/product.dart';

// Mock 数据
const _mockProduct = Product(
  id: '123',
  name: 'Sample Product',
  price: 99.99,
  imageUrl: 'packages/my_app/assets/product_sample.png',
  description: 'A wonderful product for testing',
);

// Material wrapper（必须 public + const）
Widget _materialWrapper(BuildContext context, Widget child) {
  return MaterialApp(
    home: Scaffold(
      body: Center(child: child),
    ),
  );
}

// 亮色 + 手机尺寸
@Preview(
  name: 'Product Card - Light (Phone)',
  group: 'Product Card',
  size: Size(375, 200),
  brightness: Brightness.light,
  wrapper: _materialWrapper,
)
Widget _productCardLightPhone() {
  return ProductCard(product: _mockProduct);
}

// 暗色 + 手机尺寸
@Preview(
  name: 'Product Card - Dark (Phone)',
  group: 'Product Card',
  size: Size(375, 200),
  brightness: Brightness.dark,
  wrapper: _materialWrapper,
)
Widget _productCardDarkPhone() {
  return ProductCard(product: _mockProduct);
}

// 平板尺寸 + 大文本
@Preview(
  name: 'Product Card - Tablet + Large Text',
  group: 'Product Card',
  size: Size(768, 300),
  textScaleFactor: 1.5,
  brightness: Brightness.light,
  wrapper: _materialWrapper,
)
Widget _productCardTabletLargeText() {
  return ProductCard(product: _mockProduct);
}

// 或用自定义 MultiPreview 批量生成亮暗色
final class _ProductCardBrightnessPreview extends MultiPreview {
  const _ProductCardBrightnessPreview();

  @override
  List<Preview> get previews => const [
        Preview(
          name: 'Product Card - Multi Light',
          group: 'Product Card Multi',
          size: Size(375, 200),
          brightness: Brightness.light,
          wrapper: _materialWrapper,
        ),
        Preview(
          name: 'Product Card - Multi Dark',
          group: 'Product Card Multi',
          size: Size(375, 200),
          brightness: Brightness.dark,
          wrapper: _materialWrapper,
        ),
      ];
}

@_ProductCardBrightnessPreview()
Widget _productCardMulti() {
  return ProductCard(product: _mockProduct);
}
```

**检查点**：确认生成的代码包含：
- [ ] 正确的 import（`package:flutter/widget_previews.dart`）
- [ ] Mock 数据（固定值，无 DateTime.now() 等动态值）
- [ ] public + const wrapper 函数（若需要）
- [ ] 至少 2 个亮暗色预览
- [ ] 显式 `size` 约束（避免无约束警告）

---

### Step 4: 在预览器中查看并迭代

**人工操作**：
1. 保存生成的 @Preview 代码
2. 在预览器中（IDE 侧栏或浏览器）找到新增的预览
   - 用搜索栏按 `Preview name` / `Group name` 过滤
   - 点击 Filter 下拉选择过滤条件
3. 查看各预览的渲染效果
   - 用预览器控件：缩放（Zoom in/out/reset）、切换亮暗色、单预览热重启
   - 若全局状态变化（如 static 初始化改变），用预览器右下角按钮热重启整个环境

**常见调整**：

**调整 1：补充更多变体**
```markdown
我想补充以下变体：
- 加载中状态（`isLoading: true`）
- 错误状态（`error: 'Network error'`）
- 桌面尺寸（1024x768）

请生成对应的 @Preview 注解。
```

**调整 2：修改 wrapper**
```markdown
这个组件依赖 Riverpod Provider。请修改 wrapper，在其中注入 ProviderScope 和 mock Provider。
```

**调整 3：自定义主题**
```markdown
请创建自定义 PreviewThemeData，使用项目的品牌色（seedColor: Colors.blue）。
```

**检查清单**：
- [ ] 预览器中能看到所有注解的预览
- [ ] 亮暗色切换正常
- [ ] 尺寸约束符合预期（无布局溢出警告）
- [ ] wrapper 正确注入了必要依赖

---

### Step 5: 处理预览器限制导致的失败

**常见失败与修复**：

| 失败现象 | 原因 | 修复方法 |
|---------|------|----------|
| 预览加载失败，提示 `dart:io` / `dart:ffi` 异常 | 组件或依赖调用了原生 API | 用条件导入，在 Web 平台返回占位 Widget |
| 资源加载失败（图片/字体） | 路径使用本地 `assets/` | 改为 package 路径：`packages/my_package/assets/...` |
| 预览未出现在列表 | 注解在 `lib/` 外 | 预览器只扫描 `lib/` 目录内的注解 |
| wrapper 报错 | wrapper 不是 public 或 const | 确保 wrapper 函数是顶层/静态、且签名为 `Widget fn(BuildContext, Widget)` |
| 布局溢出或裁切 | Widget 无约束 | 用 `size` 参数显式约束；或在 wrapper 中用 SizedBox |

**AI 排障提示词**（可复制）：

```markdown
我的预览器加载失败，错误信息：

[粘贴错误日志或截图]

请分析可能原因并给出修复代码。考虑：
1. 是否调用了 dart:io / dart:ffi / 原生插件（需条件导入）
2. 资源路径是否正确（需 package 路径）
3. wrapper 是否 public + const
4. 是否有无约束布局（需 size 参数或 SizedBox）
```

**条件导入示例**（当组件依赖 dart:io）：

```dart
// lib/widgets/file_widget.dart
import 'file_widget_stub.dart'
    if (dart.library.io) 'file_widget_io.dart'
    if (dart.library.html) 'file_widget_web.dart';

// file_widget_io.dart（原生平台）
import 'dart:io';
Widget buildFileWidget() => /* 真实实现，可用 File */;

// file_widget_web.dart（Web / 预览器）
Widget buildFileWidget() => const Placeholder(); // 占位
```

---

### Step 6: 决策是否补测试

**预览器验证完成后**，根据组件类型决策：

| 组件类型 | 是否补 Widget 测试 | 是否补 Golden 测试 | 说明 |
|---------|------------------|-------------------|------|
| 纯展示组件（无交互） | 可选（简单 smoke test） | ✅ 推荐 | Previewer 已验证外观，Golden 捕捉回归 |
| 有交互逻辑的组件 | ✅ 必须 | ✅ 推荐 | Widget 测试验证行为，Golden 捕捉视觉 |
| 表单/状态变化组件 | ✅ 必须 | 可选 | 重点是逻辑断言，视觉次要 |
| 复杂业务组件 | ✅ 必须 | ✅ 必须 | 双重保障 |

**AI 提示词（生成测试）**：

```markdown
预览器验证通过。现在请为这个组件生成：
1. Widget 测试：验证交互逻辑（如点击、状态变化）
2. Golden 测试：覆盖亮暗色 + 手机/平板尺寸（参考 workflows/golden-test-practice.md）

生成完整测试代码。
```

**检查清单**：
- [ ] 确认组件是否需要行为测试（有交互 → 必须）
- [ ] 确认组件是否需要视觉回归测试（核心展示 → 推荐）
- [ ] 生成的测试文件与源文件路径对应（`lib/widgets/X.dart` → `test/widgets/X_test.dart`）

---

## ✅ 验证清单

完成后确认：

- [ ] Flutter 版本 ≥ 3.47（建议 ≥ 3.50）
- [ ] 预览器已启动（IDE 或浏览器）
- [ ] 目标 Widget 添加了 `@Preview` 注解
- [ ] 预览器中能看到所有预览，且按 `group` 正确分组
- [ ] 覆盖至少亮暗色两种 Brightness
- [ ] 用 `size` 参数显式约束尺寸（避免无约束警告）
- [ ] wrapper / theme / localizations 回调（若有）是 public + const
- [ ] 若依赖原生 API，已用条件导入处理
- [ ] 资源路径使用 package 路径（若有）
- [ ] `.widget_preview/` 已加入 `.gitignore`（若尚未）
- [ ] 根据组件类型决策是否补 Widget / Golden 测试

---

## 🎨 Widget Previewer vs Widget 测试 vs Golden 测试

| 维度 | Widget Previewer | Widget 测试 | Golden 测试 |
|------|-----------------|------------|------------|
| **主要用途** | 设计迭代、实时预览 | 行为断言、交互验证 | 视觉回归、像素对比 |
| **运行环境** | IDE / 浏览器，基于 Flutter Web | 测试环境，模拟用户操作 | 测试环境，生成快照 |
| **验证内容** | 外观、主题、尺寸、布局 | 逻辑、状态、交互、业务规则 | 像素级视觉一致性 |
| **是否能替代测试** | ❌ 不能 | — | — |
| **失败信息** | 渲染错误、布局溢出 | 断言失败、具体行为错误 | 快照对比差异 |
| **维护成本** | 低（仅注解） | 中（断言需维护） | 中高（UI 变化需更新基线） |
| **原生 API 支持** | ❌ 不支持（基于 Web） | ✅ 支持（真实平台） | ✅ 支持（真实平台） |

**推荐组合策略**：
- **设计阶段**：用 Previewer 快速迭代亮暗色、尺寸、主题
- **开发阶段**：写 Widget 测试验证交互逻辑，补 Golden 测试建立视觉基线
- **维护阶段**：Previewer 辅助验证设计调整，Widget/Golden 测试捕捉回归
- **不要**：把 Previewer 当成唯一验证手段；不写测试的组件仍有风险

---

## 🤔 FAQ

### Q1: Widget Previewer 能替代 Widget 测试吗？
**A**: **不能**。理由：
- ❌ Previewer 不验证交互逻辑（如点击、滚动、状态变化）
- ❌ Previewer 不运行断言（无法检查数据正确性）
- ❌ Previewer 基于 Flutter Web，无法测试原生插件
- ✅ Previewer 的价值是**设计迭代与视觉检查**，测试的价值是**行为保证**

**正确做法**：Previewer 验证外观 → Widget 测试验证逻辑 → Golden 测试建立视觉基线。

### Q2: 在 Cursor 中能用 Widget Previewer 吗？
**A**: 官方文档仅明确支持 Android Studio、IntelliJ、Visual Studio Code。Cursor 是 VS Code 兼容编辑器，可尝试 Flutter 扩展是否出现 "Flutter Widget Preview" 面板，但不保证。

**稳妥方案**：用命令行 `flutter widget-preview start` 在浏览器中预览。这对所有编辑器都有效。

**不要**：把「Cursor 中可能能用」写成「官方已支持 Cursor」。

### Q3: 预览器报错「dart:io 不支持」怎么办？
**A**: Widget Previewer 基于 Flutter Web，不支持 `dart:io` / `dart:ffi` / 原生插件。解决方案：
1. **条件导入**：在 Web 平台返回占位 Widget（见 Step 5）
2. **Mock 外部依赖**：用 wrapper 注入 mock，避免真实调用原生 API
3. **接受限制**：若组件强依赖原生能力（如相机、蓝牙），预览器无法完整展示；改为跑整应用或平台测试

### Q4: 资源加载失败（图片/字体）怎么办？
**A**: 预览器加载资源时必须用 package 路径，不要用本地 `assets/` 相对路径。

**错误示例**：
```dart
Image.asset('assets/images/logo.png')  // ❌ 预览器中失败
```

**正确示例**：
```dart
Image.asset('packages/my_app/assets/images/logo.png')  // ✅
```

**背景**：预览器在 Web 环境中运行，资源必须通过 package 路径索引。

### Q5: 预览器中布局溢出或裁切怎么办？
**A**: 无约束 Widget 会被预览器自动限制到约预览器宽高的一半，可能导致意外裁切。

**解决方案**：
1. 用 `size` 参数显式约束：`@Preview(size: Size(375, 200))`
2. 在 wrapper 中用 `SizedBox` 包裹：
   ```dart
   Widget _sizedWrapper(BuildContext context, Widget child) {
     return MaterialApp(
       home: Scaffold(
         body: SizedBox(
           width: 375,
           height: 200,
           child: child,
         ),
       ),
     );
   }
   ```

### Q6: 多个预览共享配置，每次都写一遍太麻烦？
**A**: 用以下两种方式简化：
1. **扩展 `Preview` 注解**：创建自定义注解，预设 wrapper / theme 等参数
   ```dart
   final class MyCustomPreview extends Preview {
     const MyCustomPreview({super.name, super.group})
         : super(size: const Size(375, 200), wrapper: _materialWrapper);
   }
   ```

2. **扩展 `MultiPreview` 注解**：批量生成多个配置（如亮暗色）
   ```dart
   final class BrightnessMultiPreview extends MultiPreview {
     const BrightnessMultiPreview({required this.name});
     final String name;

     @override
     List<Preview> get previews => [
           Preview(name: '$name - Light', brightness: Brightness.light),
           Preview(name: '$name - Dark', brightness: Brightness.dark),
         ];
   }
   ```

### Q7: `.widget_preview/` 应该提交到 Git 吗？
**A**: **不应该**。这是预览器的本地构建缓存，每次启动可能重新生成。

**做法**：在 `.gitignore` 中添加：
```
.widget_preview/
```

本工作流 Step 1 验证清单已包含此项。

### Q8: 预览器能用于 Widgetbook 吗？
**A**: Widget Previewer 和 Widgetbook 是不同工具：
- **Widget Previewer**：官方工具链，自 Flutter 3.47 stable，基于 `@Preview` 注解
- **Widgetbook**：第三方包，提供更丰富的组件目录、交互控件、插件

两者可共存，根据团队需求选择：
- **官方工具链**：优先 Widget Previewer（无需额外依赖）
- **需要更多定制（如 Knobs、插件、Storybook 风格）**：考虑 Widgetbook

### Q9: 预览器中如何处理依赖 Provider / Riverpod 的组件？
**A**: 在 `wrapper` 中注入 mock Provider / ProviderScope。

**示例**（Riverpod）：
```dart
Widget _riverpodWrapper(BuildContext context, Widget child) {
  return ProviderScope(
    overrides: [
      userProvider.overrideWithValue(mockUser),
    ],
    child: MaterialApp(
      home: Scaffold(body: child),
    ),
  );
}

@Preview(wrapper: _riverpodWrapper)
Widget _myComponentPreview() => MyComponent();
```

### Q10: Flutter 3.50 对 Widget Previewer 有哪些改进？
**A**: 根据官方 Release Notes（核对日期 2026-11-24），Flutter 3.50 包含以下 Previewer 相关修复：
- 更安全的热重载处理（拦截不安全热重载，自动触发热重启）
- 无约束 Widget 的更清晰报错
- 预览组 PageStorage 隔离（每个预览独立状态）
- 窄面板布局优化

**注意**：Widget Previewer 在 Flutter 3.50 博客正文中着墨不多，不是头条特性，但底层稳定性有提升。

---

## 💡 最佳实践

1. **Mock 数据必须稳定**
   - ❌ 错误：`Text('${DateTime.now()}')`、`Random().nextInt(100)`
   - ✅ 正确：`const _mockDate = '2026-11-24'`、`const _mockValue = 42`

2. **显式约束尺寸**
   - 总是用 `size` 参数或在 wrapper 中用 `SizedBox`
   - 避免无约束警告和意外裁切

3. **用 `group` 组织预览**
   - 相关预览用相同 `group` 名称（如 'Product Card'）
   - 便于在预览器中过滤和管理

4. **wrapper / theme / localizations 必须 public + const**
   - 顶层函数或 static 方法
   - 签名：`Widget fn(BuildContext, Widget)` 或 `PreviewThemeData fn()`

5. **处理原生依赖**
   - 条件导入：Web 平台返回占位
   - 或在 wrapper 中 mock 外部服务

6. **资源路径用 package 格式**
   - `packages/my_package/assets/...`
   - 不要用 `assets/...`

7. **预览器不是唯一验证**
   - 设计迭代用 Previewer
   - 交互逻辑用 Widget 测试
   - 视觉回归用 Golden 测试

8. **贯穿示例：商品信息卡**
   - 场景：电商应用的商品卡片，展示图片、名称、价格、库存状态
   - 依赖：`Product` 数据模型、主题、可能的 Riverpod Provider
   - 预览配置：
     - 亮暗色 × 手机/平板尺寸（4 个）
     - 大文本缩放（1 个）
     - 库存不足状态（1 个）
   - 分组：`group: 'Product Card'`
   - wrapper：MaterialApp + ProviderScope（若需要）
   - 验证：在预览器中检查颜色、间距、响应式布局；再补 Widget 测试验证点击事件、Golden 测试建立基线

---

## 🔗 相关资源

- 提示词模板：
  - [prompts/preview/generate-widget-preview.md](../prompts/preview/generate-widget-preview.md) — 生成 @Preview 注解
  - [prompts/preview/customize-preview.md](../prompts/preview/customize-preview.md) — 定制 wrapper / theme，排查预览器限制
- 相关工作流：
  - [workflows/widget-test-generation.md](widget-test-generation.md) — Widget 测试生成
  - [workflows/golden-test-practice.md](golden-test-practice.md) — Golden 测试实践
- Flutter 官方文档（核对日期 2026-11-24）：
  - [Flutter Widget Previewer](https://docs.flutter.dev/tools/widget-previewer)
  - [Flutter 3.50 Release Notes](https://docs.flutter.dev/release/release-notes/release-notes-3.50.0)
  - [What's new in Flutter 3.50](https://flutter.dev/blog/whats-new-in-flutter-3-50)

---

**让组件设计迭代更快速！✨**
