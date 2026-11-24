# 提示词：生成 Widget Preview 注解

**用途**：为 Flutter Widget 生成 @Preview 注解，在 Widget Previewer 中实时预览组件渲染效果

---

## 📋 适用场景

- 纯展示型 Widget 需要快速验证设计效果
- 多主题/暗色模式适配验证
- 响应式布局（手机/平板/桌面）验证
- 文本缩放（无障碍）验证
- 组件设计迭代与原型演示

---

## 📥 输入要求

在 Cursor 中准备：

1. **目标 Widget 源码**（如 `lib/widgets/info_card.dart`）
2. **数据模型定义**（若 Widget 依赖外部数据）
3. **依赖说明**（如 Theme、Provider、InheritedWidget）
4. **（可选）设计需求**（如需要验证的尺寸、主题、状态变体）

---

## 💬 提示词模板

```markdown
我需要为以下 Flutter Widget 添加 @Preview 注解，在 Flutter Widget Previewer 中实时预览。

**目标 Widget**：`lib/widgets/[WIDGET_FILE_NAME].dart`

**要求**：

1. **基础结构**（官方 API，核对日期 2026-11-24）：
   - import `package:flutter/widget_previews.dart`
   - 创建返回目标 Widget 的顶层函数或静态方法
   - 用 @Preview 注解标注（可叠多个）

2. **必须覆盖的参数**：
   - `name`: 预览描述名称（如 '[Widget名称] - Light (Phone)'）
   - `group`: 分组名称（如 '[Widget名称]'），便于过滤
   - `size`: 尺寸约束 `Size(宽, 高)`（如 `Size(375, 200)`，避免无约束警告）
   - `textScaleFactor`: 文本缩放因子（如 `1.5` 模拟无障碍大字号，可选）
   - `wrapper`: 返回 Widget 的回调，包裹 MaterialApp / Theme / Provider（必须 public + const）
   - `theme`: 返回 `PreviewThemeData` 的回调，自定义主题（可选）
   - `brightness`: `Brightness.light` 或 `Brightness.dark`
   - `localizations`: 本地化配置回调（可选）

3. **设计变体覆盖**（至少）：
   - ✅ 亮色模式（Brightness.light）+ 手机尺寸（如 375x200）
   - ✅ 暗色模式（Brightness.dark）+ 手机尺寸
   - 【可选】平板尺寸（如 768x300）
   - 【可选】大文本缩放（textScaleFactor: 1.5）
   - 【可选】其他状态变体：加载中、错误、空数据

4. **处理依赖**：
   - 若组件依赖 Theme，用 `wrapper` 包裹 MaterialApp
   - 若依赖 Provider / InheritedWidget / Riverpod，在 wrapper 中注入 mock 数据
   - 若依赖外部数据，创建 const mock 实例（避免动态值如 `DateTime.now()`、`Random()`）

5. **处理预览器限制**（官方文档 Restrictions，核对日期 2026-11-24）：
   - **回调约束**：传给注解的回调（wrapper / theme / localizations）必须 public 且 constant
   - **原生 API**：不支持 dart:io / dart:ffi / 原生插件（预览器基于 Flutter Web）
     - 若必须调用，提示用条件导入，在 Web 平台返回占位 Widget
   - **资源路径**：必须用 package 路径（如 `packages/my_package_name/assets/my_image.png`），不要用本地 `assets/` 相对路径
   - **无约束 Widget**：会被自动限制到约预览器宽高的一半；建议用 `size` 参数显式约束

6. **代码组织**：
   - 可叠多个 @Preview 注解在同一函数上，用于不同配置
   - 或扩展 `MultiPreview` 类批量生成多个预览（如亮暗色、多尺寸）
   - 用 `group` 参数给相关预览分组
   - wrapper 函数命名清晰（如 `_materialWrapper`），签名为 `Widget fn(BuildContext, Widget)`，标记为顶层或 static

7. **完整性**：
   - 包含所有必要的 import（含 `package:flutter/widget_previews.dart`）
   - Mock 数据使用 const 定义
   - 添加简要注释说明每个预览的用途

生成完整代码，不要省略任何部分。
```

---

## ✅ 验证步骤

生成后执行：

```bash
# 1. 保存生成的代码
# 2. 启动 Widget Previewer（若尚未启动）

# 方案 A：IDE 自动启动（Android Studio / IntelliJ / VS Code）
# 打开侧栏 "Flutter Widget Preview" 标签页

# 方案 B：命令行启动（适用所有编辑器，含 Cursor）
cd /path/to/your/flutter/project
flutter widget-preview start

# 3. 在预览器中查看
# - 用搜索栏按 "Preview name" / "Group name" 过滤
# - 检查所有预览是否正确渲染
# - 用预览器控件测试：缩放、切换亮暗色、单预览热重启
```

**检查清单**：
- [ ] 预览器中能看到所有生成的预览
- [ ] 预览按 `group` 正确分组
- [ ] 亮暗色切换正常
- [ ] 尺寸约束符合预期，无布局溢出警告
- [ ] wrapper 正确注入了必要依赖（如 MaterialApp / Provider）
- [ ] Mock 数据稳定（无动态值）

---

## 🛠️ 常见调整

生成后可能需要手动微调：

### 1. 补充更多状态变体

```markdown
请补充以下状态的预览：
- 加载中状态（`isLoading: true`）
- 错误状态（`error: 'Network error'`）
- 空数据状态（`data: null`）

每个状态生成一个 @Preview，命名清晰，使用相同的 `group`。
```

### 2. 添加 Riverpod Provider wrapper

```dart
// 示例：注入 ProviderScope 和 mock Provider
Widget _riverpodWrapper(BuildContext context, Widget child) {
  return ProviderScope(
    overrides: [
      userProvider.overrideWithValue(mockUser),
      themeProvider.overrideWithValue(mockTheme),
    ],
    child: MaterialApp(
      home: Scaffold(
        body: Center(child: child),
      ),
    ),
  );
}

@Preview(
  name: 'My Widget - With Provider',
  group: 'My Widget',
  size: Size(375, 200),
  wrapper: _riverpodWrapper,
)
Widget _myWidgetWithProvider() {
  return MyWidget();
}
```

### 3. 自定义 MultiPreview 批量生成亮暗色

```dart
/// 自动生成亮暗色两个预览
final class _BrightnessMultiPreview extends MultiPreview {
  const _BrightnessMultiPreview({
    required this.name,
    this.size = const Size(375, 200),
  });

  final String name;
  final Size size;

  @override
  List<Preview> get previews => [
        Preview(
          name: '$name - Light',
          group: name,
          size: size,
          brightness: Brightness.light,
          wrapper: _materialWrapper,
        ),
        Preview(
          name: '$name - Dark',
          group: name,
          size: size,
          brightness: Brightness.dark,
          wrapper: _materialWrapper,
        ),
      ];
}

@_BrightnessMultiPreview(name: 'My Widget')
Widget _myWidgetBrightness() {
  return MyWidget();
}
```

### 4. 自定义 PreviewThemeData

```dart
// 使用项目品牌色
final class _MyPreviewThemeData extends PreviewThemeData {
  const _MyPreviewThemeData();

  @override
  Widget apply(BuildContext context, Widget child) {
    return Theme(
      data: ThemeData.from(
        colorScheme: ColorScheme.fromSeed(
          seedColor: Colors.blue,  // 项目品牌色
          brightness: MediaQuery.platformBrightnessOf(context),
        ),
      ),
      child: child,
    );
  }
}

@Preview(
  name: 'My Widget - Custom Theme',
  group: 'My Widget',
  size: Size(375, 200),
  theme: _MyPreviewThemeData.new,
)
Widget _myWidgetCustomTheme() {
  return MyWidget();
}
```

### 5. 处理条件导入（原生 API）

```dart
// lib/widgets/file_widget.dart
import 'file_widget_stub.dart'
    if (dart.library.io) 'file_widget_io.dart'
    if (dart.library.html) 'file_widget_web.dart';

// file_widget_stub.dart
Widget buildFileWidget() => throw UnimplementedError();

// file_widget_io.dart（原生平台）
import 'dart:io';
Widget buildFileWidget() {
  // 真实实现，可用 File、dart:io API
  return /* ... */;
}

// file_widget_web.dart（Web / 预览器）
Widget buildFileWidget() {
  // 占位实现，不调用 dart:io
  return const Placeholder(
    fallbackWidth: 200,
    fallbackHeight: 100,
    child: Center(child: Text('File widget (Web placeholder)')),
  );
}
```

---

## 🚨 注意事项

### 1. Mock 数据必须稳定

- ❌ 错误：`Text('${DateTime.now()}')`、`Random().nextInt(100)`、`Image.network('https://...')`
- ✅ 正确：`const _mockDate = '2026-11-24'`、`const _mockValue = 42`、`Image.asset('packages/.../mock.png')`

### 2. wrapper 必须 public + const

- 顶层函数或 static 方法
- 签名：`Widget fn(BuildContext, Widget)`
- 不要在 wrapper 内部创建动态数据

```dart
// ✅ 正确
Widget _materialWrapper(BuildContext context, Widget child) {
  return MaterialApp(home: Scaffold(body: child));
}

// ❌ 错误：私有类方法
class MyWidget {
  Widget _wrapper(BuildContext context, Widget child) { /* ... */ }
}
```

### 3. 显式约束尺寸

- 总是用 `size` 参数
- 避免无约束警告：`Preview automatically constrained unconstrained widget...`

```dart
// ✅ 正确
@Preview(size: Size(375, 200))

// ❌ 不推荐（会触发自动约束）
@Preview()  // 无 size 参数
```

### 4. 资源路径使用 package 格式

```dart
// ✅ 正确
Image.asset('packages/my_app/assets/images/logo.png')

// ❌ 错误（预览器中加载失败）
Image.asset('assets/images/logo.png')
```

### 5. group 参数用于组织

- 相关预览用相同 `group` 名称
- 在预览器中可按 group 过滤

```dart
@Preview(name: 'Card - Light', group: 'Product Card')
@Preview(name: 'Card - Dark', group: 'Product Card')
@Preview(name: 'Card - Tablet', group: 'Product Card')
```

---

## 🔗 相关

- 工作流：[workflows/flutter-widget-previewer.md](../../workflows/flutter-widget-previewer.md)
- 定制与排障：[customize-preview.md](customize-preview.md)
- Widget 测试生成：[../testing/generate-widget-test.md](../testing/generate-widget-test.md)
- Golden 测试生成：[../testing/generate-golden-test.md](../testing/generate-golden-test.md)
