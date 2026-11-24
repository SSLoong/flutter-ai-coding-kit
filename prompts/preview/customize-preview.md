# 提示词：定制与排障 Widget Preview

**用途**：自定义 Widget Previewer 配置（wrapper / theme / 自定义注解），以及排查预览器加载失败

---

## 📋 适用场景

- 预览器加载失败，提示 `dart:io` / `dart:ffi` / 原生插件异常
- 资源加载失败（图片/字体）
- 预览未出现在列表
- wrapper 报错或依赖注入问题
- 布局溢出或裁切
- 需要自定义 Preview 注解或 MultiPreview 批量生成
- 回调不是 public + const 导致的错误

---

## 📥 输入要求

准备：

1. **错误日志或截图**（若预览失败）
2. **目标 Widget 源码**
3. **当前 @Preview 注解代码**
4. **依赖说明**（如 Theme、Provider、原生插件）

---

## 💬 提示词模板

### 场景 1：预览加载失败排障

```markdown
我的 Widget Previewer 加载失败，错误信息：

[粘贴错误日志、堆栈跟踪或截图]

**Widget 源码**：`lib/widgets/[WIDGET_FILE_NAME].dart`

**当前 @Preview 代码**：
[粘贴现有 @Preview 注解代码]

请分析可能原因并给出修复代码。考虑以下常见问题（官方文档 Restrictions，核对日期 2026-11-24）：

1. **原生 API 调用**：
   - 是否调用了 `dart:io` / `dart:ffi` / 原生插件？
   - 预览器基于 Flutter Web，不支持这些 API
   - 解决方案：用条件导入，在 Web 平台返回占位 Widget

2. **资源路径错误**：
   - 是否使用了本地 `assets/` 相对路径？
   - 预览器要求 package 路径：`packages/my_package_name/assets/...`
   - 解决方案：修改资源路径为 package 格式

3. **wrapper 约束**：
   - wrapper 函数是否 public 且 const？
   - 签名是否为 `Widget fn(BuildContext, Widget)`？
   - 解决方案：确保 wrapper 是顶层函数或 static 方法

4. **无约束布局**：
   - Widget 是否无约束（如裸 `Text` / `Icon`）？
   - 预览器会自动限制到约预览器宽高的一半
   - 解决方案：用 `size` 参数显式约束，或在 wrapper 中用 `SizedBox`

5. **预览位置**：
   - 注解是否在 `lib/` 目录外？
   - 预览器只扫描 `lib/` 内的注解
   - 解决方案：移动预览函数到 `lib/` 目录下

给出修复后的完整 @Preview 代码，并说明修复理由。
```

---

### 场景 2：添加条件导入（原生 API）

```markdown
我的 Widget 调用了 `dart:io` / 原生插件，导致预览器加载失败。

**Widget 源码**：`lib/widgets/[WIDGET_FILE_NAME].dart`

请生成条件导入方案：
1. 创建 stub / io / web 三个文件
2. 在 Web 平台（预览器）返回占位 Widget
3. 在原生平台（iOS/Android）使用真实实现
4. 更新 Widget 代码以使用条件导入

提供完整文件代码和目录结构说明。
```

**AI 输出示例**（参考）：

```
目录结构：
lib/widgets/
  ├── file_viewer.dart           # 主 Widget，使用条件导入
  ├── file_viewer_stub.dart      # Stub（默认）
  ├── file_viewer_io.dart        # 原生平台实现
  └── file_viewer_web.dart       # Web / 预览器实现

file_viewer.dart:
---
import 'file_viewer_stub.dart'
    if (dart.library.io) 'file_viewer_io.dart'
    if (dart.library.html) 'file_viewer_web.dart';

class FileViewer extends StatelessWidget {
  @override
  Widget build(BuildContext context) {
    return buildFileViewerImpl();  // 由条件导入提供
  }
}

file_viewer_stub.dart:
---
import 'package:material_ui/material_ui.dart';

Widget buildFileViewerImpl() {
  throw UnimplementedError('Platform not supported');
}

file_viewer_io.dart:
---
import 'dart:io';
import 'package:material_ui/material_ui.dart';

Widget buildFileViewerImpl() {
  // 真实实现，可用 File、Directory 等 dart:io API
  return /* ... */;
}

file_viewer_web.dart:
---
import 'package:material_ui/material_ui.dart';

Widget buildFileViewerImpl() {
  // 占位实现，不调用 dart:io
  return const Placeholder(
    fallbackWidth: 300,
    fallbackHeight: 200,
    child: Center(
      child: Text('File viewer (Web placeholder)'),
    ),
  );
}
```

---

### 场景 3：自定义 wrapper 注入依赖

```markdown
我的 Widget 依赖 [Provider / Riverpod / InheritedWidget / 其他]，需要在预览器中注入 mock 数据。

**Widget 源码**：`lib/widgets/[WIDGET_FILE_NAME].dart`
**依赖说明**：[描述依赖的 Provider / InheritedWidget]

请生成：
1. wrapper 函数，注入必要的 mock 数据
2. mock 数据定义（使用 const）
3. 更新后的 @Preview 注解

确保 wrapper 是 public + const，签名为 `Widget fn(BuildContext, Widget)`。
```

**AI 输出示例**（Riverpod）：

```dart
import 'package:flutter_riverpod/flutter_riverpod.dart';
import 'package:material_ui/material_ui.dart';

// Mock 数据
const _mockUser = User(id: '123', name: 'Test User');
const _mockSettings = Settings(theme: 'light', lang: 'en');

// Riverpod wrapper
Widget _riverpodWrapper(BuildContext context, Widget child) {
  return ProviderScope(
    overrides: [
      userProvider.overrideWithValue(_mockUser),
      settingsProvider.overrideWithValue(_mockSettings),
    ],
    child: MaterialApp(
      home: Scaffold(
        body: Center(child: child),
      ),
    ),
  );
}

@Preview(
  name: 'My Widget - With Riverpod',
  group: 'My Widget',
  size: Size(375, 200),
  wrapper: _riverpodWrapper,
)
Widget _myWidgetPreview() {
  return MyWidget();
}
```

---

### 场景 4：自定义 Preview 注解（减少重复配置）

```markdown
我的项目有多个组件需要相同的预览配置（wrapper / theme / size），每次都写一遍太麻烦。

**通用配置需求**：
- wrapper: MaterialApp + ProviderScope
- theme: 项目品牌色（seedColor: [颜色]）
- size: 手机尺寸 375x200

请生成自定义 Preview 注解，扩展 `Preview` 类，预设这些参数。
```

**AI 输出示例**：

```dart
import 'package:flutter/widget_previews.dart';
import 'package:material_ui/material_ui.dart';

// 自定义 PreviewThemeData
final class _MyAppPreviewThemeData extends PreviewThemeData {
  const _MyAppPreviewThemeData();

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

// 自定义 Preview 注解
final class MyAppPreview extends Preview {
  const MyAppPreview({
    super.name,
    super.group,
    super.brightness,
    super.textScaleFactor,
  }) : super(
         size: const Size(375, 200),
         wrapper: _myAppWrapper,
         theme: _MyAppPreviewThemeData.new,
       );
}

// wrapper
Widget _myAppWrapper(BuildContext context, Widget child) {
  return ProviderScope(
    overrides: [
      // 通用 mock
    ],
    child: MaterialApp(
      home: Scaffold(body: Center(child: child)),
    ),
  );
}

// 使用示例
@MyAppPreview(name: 'My Widget - Light', brightness: Brightness.light)
@MyAppPreview(name: 'My Widget - Dark', brightness: Brightness.dark)
Widget _myWidgetPreview() {
  return MyWidget();
}
```

---

### 场景 5：自定义 MultiPreview（批量生成配置）

```markdown
我需要批量生成多个预览配置，例如：
- 亮暗色 × 手机/平板尺寸（2×2 = 4 个预览）
- 或亮暗色 × 多种文本缩放（2×3 = 6 个预览）

请生成自定义 MultiPreview 注解，扩展 `MultiPreview` 类，自动生成这些配置。
```

**AI 输出示例**（亮暗色 × 尺寸矩阵）：

```dart
import 'package:flutter/widget_previews.dart';
import 'package:material_ui/material_ui.dart';

/// 生成亮暗色 × 手机/平板尺寸矩阵（4 个预览）
final class BrightnessSizeMatrixPreview extends MultiPreview {
  const BrightnessSizeMatrixPreview({required this.name});

  final String name;

  @override
  List<Preview> get previews {
    final brightnessOptions = [Brightness.light, Brightness.dark];
    final sizeOptions = [
      ('Phone', Size(375, 200)),
      ('Tablet', Size(768, 300)),
    ];

    final result = <Preview>[];
    for (final brightness in brightnessOptions) {
      for (final (sizeLabel, size) in sizeOptions) {
        result.add(Preview(
          name: '$name - ${brightness.name} ($sizeLabel)',
          group: name,
          size: size,
          brightness: brightness,
          wrapper: _materialWrapper,
        ));
      }
    }
    return result;
  }
}

Widget _materialWrapper(BuildContext context, Widget child) {
  return MaterialApp(
    home: Scaffold(body: Center(child: child)),
  );
}

// 使用：一个注解生成 4 个预览
@BrightnessSizeMatrixPreview(name: 'Product Card')
Widget _productCardMatrix() {
  return ProductCard(product: mockProduct);
}
```

---

### 场景 6：修复资源路径错误

```markdown
我的预览器加载图片/字体失败，提示资源找不到。

**当前资源路径**：
- 图片：`Image.asset('assets/images/logo.png')`
- 字体：`pubspec.yaml` 中 `assets/fonts/...`

请修复为 package 路径格式，确保预览器能加载。
```

**AI 输出示例**：

```
问题：预览器基于 Flutter Web，要求资源路径使用 package 格式。

修复：

1. 图片路径修改：
   ❌ 错误：Image.asset('assets/images/logo.png')
   ✅ 正确：Image.asset('packages/my_app/assets/images/logo.png')

2. pubspec.yaml 保持不变：
   assets:
     - assets/images/

3. 若使用 fromAsset（dart:ui）：
   ❌ 错误：ui.Image.fromAssetBundle('assets/data.bin', ...)
   ✅ 正确：ui.Image.fromAssetBundle('packages/my_app/assets/data.bin', ...)

注意：package 名称是 pubspec.yaml 中的 `name` 字段。
```

---

### 场景 7：修复无约束布局

```markdown
我的预览器显示布局溢出警告：
"Preview automatically constrained unconstrained widget..."

**Widget 源码**：`lib/widgets/[WIDGET_FILE_NAME].dart`

请修复：
1. 在 @Preview 注解中添加 `size` 参数
2. 或在 wrapper 中用 `SizedBox` 约束

选择更合适的方案并给出代码。
```

**AI 输出示例**：

```
方案 1：用 size 参数（推荐，简洁）

@Preview(
  name: 'My Widget - Constrained',
  group: 'My Widget',
  size: Size(375, 200),  // 显式约束
  wrapper: _materialWrapper,
)
Widget _myWidgetPreview() {
  return MyWidget();
}

---

方案 2：在 wrapper 中用 SizedBox（适用需要更复杂布局的情况）

Widget _sizedWrapper(BuildContext context, Widget child) {
  return MaterialApp(
    home: Scaffold(
      body: Center(
        child: SizedBox(
          width: 375,
          height: 200,
          child: child,
        ),
      ),
    ),
  );
}

@Preview(
  name: 'My Widget - Wrapper Constrained',
  group: 'My Widget',
  wrapper: _sizedWrapper,
)
Widget _myWidgetPreview() {
  return MyWidget();
}

推荐：方案 1（用 size 参数），除非 wrapper 需要其他额外布局。
```

---

## ✅ 验证步骤

修复后执行：

```bash
# 1. 保存修复后的代码

# 2. 若预览器已启动，刷新
# - IDE：点击预览器右下角的热重启按钮
# - 浏览器：刷新页面

# 3. 检查预览器
# - 预览是否出现在列表
# - 是否正确渲染，无错误/警告
# - wrapper 是否正确注入依赖
# - 资源是否正确加载

# 4. 若仍失败，查看预览器日志
# - IDE：查看 Flutter 输出面板
# - 命令行：查看终端输出
```

**检查清单**：
- [ ] 预览器中能看到修复后的预览
- [ ] 无加载错误或异常
- [ ] 布局正常，无溢出/裁切
- [ ] 资源（图片/字体）正确加载
- [ ] wrapper 正确注入了依赖

---

## 🛠️ 常见问题与修复速查表

| 问题 | 表现 | 修复方法 |
|------|------|----------|
| `dart:io` 不支持 | 预览加载失败，提示 `dart:io` 异常 | 条件导入，Web 平台返回占位 Widget |
| 原生插件调用失败 | 预览加载失败，提示 `MissingPluginException` | 条件导入或在 wrapper 中 mock 插件 |
| 资源加载失败 | 图片/字体不显示 | 改为 package 路径：`packages/my_app/assets/...` |
| 预览未出现 | 列表中看不到预览 | 确认注解在 `lib/` 目录内，函数是顶层/static |
| wrapper 报错 | 编译错误或运行时异常 | 确保 wrapper 是 public，签名为 `Widget fn(BuildContext, Widget)` |
| 布局溢出 | 渲染时显示黄黑条纹 | 用 `size` 参数显式约束，或 wrapper 中用 `SizedBox` |
| 无约束警告 | 控制台提示自动约束 | 添加 `size` 参数 |
| 依赖注入失败 | 预览加载但数据为空/null | 在 wrapper 中注入 ProviderScope / InheritedWidget |
| Mock 数据不稳定 | 每次预览不一致 | 确保 mock 数据是 const，避免 `DateTime.now()` / `Random()` |

---

## 🚨 注意事项

### 1. 条件导入目录结构

- 三个文件必须在同一目录：`_stub.dart` / `_io.dart` / `_web.dart`
- import 语句使用相对路径，不要用 package 路径

```dart
// ✅ 正确
import 'file_viewer_stub.dart'
    if (dart.library.io) 'file_viewer_io.dart'
    if (dart.library.html) 'file_viewer_web.dart';

// ❌ 错误
import 'package:my_app/widgets/file_viewer_stub.dart' ...
```

### 2. wrapper 回调约束

- 必须 public（顶层函数或 static 方法）
- 必须 const（不能内部创建动态对象）
- 签名：`Widget fn(BuildContext, Widget)`

```dart
// ✅ 正确
Widget _wrapper(BuildContext context, Widget child) { /* ... */ }

// ❌ 错误：私有类方法
class _MyClass {
  Widget _wrapper(BuildContext context, Widget child) { /* ... */ }
}

// ❌ 错误：签名不对
Widget _wrapper(Widget child) { /* ... */ }  // 缺少 BuildContext
```

### 3. package 路径格式

- package 名称：`pubspec.yaml` 中的 `name` 字段
- 路径从 `assets/` 开始：`packages/<name>/assets/...`
- 适用：`Image.asset()` / `ui.fromAsset()` / 字体等

```dart
// pubspec.yaml:
// name: my_app
// flutter:
//   assets:
//     - assets/images/

// ✅ 正确
Image.asset('packages/my_app/assets/images/logo.png')

// ❌ 错误
Image.asset('assets/images/logo.png')
```

### 4. 自定义注解命名

- 扩展 `Preview`：用于单个预览配置
- 扩展 `MultiPreview`：用于批量生成多个预览
- 命名清晰：如 `MyAppPreview` / `BrightnessMultiPreview`

---

## 🔗 相关

- 工作流：[workflows/flutter-widget-previewer.md](../../workflows/flutter-widget-previewer.md)
- 生成 @Preview：[generate-widget-preview.md](generate-widget-preview.md)
- Widget 测试生成：[../testing/generate-widget-test.md](../testing/generate-widget-test.md)
- Flutter 官方文档：[Widget Previewer - Restrictions and limitations](https://docs.flutter.dev/tools/widget-previewer#restrictions-and-limitations)
