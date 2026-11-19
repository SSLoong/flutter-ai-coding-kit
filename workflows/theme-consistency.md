# Material 3 主题一致性工作流

**用 AI 将 Flutter 应用迁移到统一的 Material 3 主题体系**

> 作者：Loong (SSLoong) | 日期：2026-11-13

---

## 📋 背景

Flutter 应用中常见的主题问题：
- 代码中直接硬编码颜色值（如 `Color(0xFF1E88E5)` 或 `Colors.blue`）
- 字体大小和间距散落在各处，缺乏统一规范
- 明暗色主题不一致或不支持动态切换
- 新增自定义颜色时没有遵循 Material 3 设计规范

Material 3（自 Flutter 3.16 起默认启用）提供了完整的主题系统，通过 `ColorScheme.fromSeed` 可从单一种子色生成和谐的配色方案，确保无障碍对比度要求，并原生支持明暗色模式。AI 可以帮助快速审计代码、重构硬编码样式、落地统一主题体系。

---

## 🎯 目标

- ✅ 审计并消除硬编码的颜色、字体、间距
- ✅ 建立基于 `ColorScheme`、`TextTheme`、`ThemeData` 的统一主题
- ✅ 用 `ThemeExtension` 管理自定义品牌色或语义化颜色
- ✅ 正确支持明暗色主题与动态切换
- ✅ 本地验证主题在不同模式下的表现

---

## 🔧 适用场景

| 场景 | 适合度 | 说明 |
|------|--------|------|
| 存在大量硬编码颜色 | ⭐⭐⭐⭐⭐ | AI 快速找出并替换 |
| 需要支持明暗色主题 | ⭐⭐⭐⭐⭐ | Material 3 原生支持 |
| 设计规范不统一 | ⭐⭐⭐⭐ | 统一到 Theme 系统 |
| 新项目建立主题体系 | ⭐⭐⭐⭐⭐ | 从零搭建最佳实践 |
| 已有复杂自定义主题 | ⭐⭐⭐ | 需分步迁移 |

---

## 🚀 完整流程

### Step 1: 审计硬编码样式

**人工操作**：在 Cursor 中打开项目根目录，确保 `lib/` 下所有 Widget 文件可见。

**AI 提示词**（可复制）：

```markdown
请审计这个 Flutter 项目中的硬编码主题问题，包括：

1. **硬编码颜色**：
   - 查找所有直接使用 `Color(0x...)`, `Colors.blue`, `Color.fromRGBO(...)` 等的地方
   - 列出文件路径、行号、具体代码片段

2. **硬编码字体样式**：
   - 查找 `TextStyle(fontSize: ...)`, `fontWeight: ...` 等未使用 Theme 的情况
   
3. **硬编码间距/尺寸**：
   - 查找 `padding: EdgeInsets.all(16)`, `SizedBox(height: 24)` 等魔法数字

生成一份审计报告，按类别分组，标注优先级（高/中/低）。

不要现在就修改代码，只列出问题。
```

**检查点**：
- [ ] 报告是否覆盖了主要的 UI 文件？
- [ ] 有没有遗漏的 `ThemeData` 或主题相关配置文件？
- [ ] 优先级划分是否合理（如全局组件 > 局部组件）？

---

### Step 2: 建立 Material 3 主题基础

**人工操作**：决定品牌主色（seed color），如 `Colors.indigo` 或自定义 `Color(0xFF6750A4)`。

**AI 提示词**（可复制）：

```markdown
基于 Material 3，为项目创建统一的主题配置。

**要求**：

1. **主题文件位置**：`lib/core/theme/app_theme.dart`（如已有其他位置，沿用现有结构）

2. **使用 ColorScheme.fromSeed**：
   - 种子色：[指定颜色，如 Color(0xFF6750A4)]
   - 同时生成明暗色主题（`Brightness.light` 和 `Brightness.dark`）
   - 参考：https://api.flutter.dev/flutter/material/ColorScheme/ColorScheme.fromSeed.html

3. **ThemeData 配置**：
   - 使用 `useMaterial3: true`（Flutter 3.16+ 默认，可省略）
   - 配置 `colorScheme`、`textTheme`
   - 统一设置 `appBarTheme`、`cardTheme`、`elevatedButtonTheme` 等常用组件主题

4. **在 MaterialApp 中应用**：
   ```dart
   MaterialApp(
     theme: AppTheme.lightTheme,
     darkTheme: AppTheme.darkTheme,
     themeMode: ThemeMode.system, // 跟随系统
     // ...
   )
   ```

5. **完整性**：
   - 包含所有必要 import
   - 代码可直接运行
   - 添加中文注释说明各角色用途

生成完整的 `app_theme.dart` 和修改后的 `main.dart`（或 MaterialApp 所在文件）。
```

**输出示例**（参考，实际以 AI 生成为准）：

```dart
// lib/core/theme/app_theme.dart
import 'package:flutter/material.dart';

/// Material 3 主题配置
class AppTheme {
  // 品牌种子色
  static const Color _seedColor = Color(0xFF6750A4);

  // 浅色主题
  static ThemeData get lightTheme {
    final colorScheme = ColorScheme.fromSeed(
      seedColor: _seedColor,
      brightness: Brightness.light,
    );

    return ThemeData(
      colorScheme: colorScheme,
      useMaterial3: true,
      appBarTheme: AppBarTheme(
        backgroundColor: colorScheme.surface,
        foregroundColor: colorScheme.onSurface,
        elevation: 0,
      ),
      cardTheme: CardTheme(
        color: colorScheme.surfaceContainerLow,
        elevation: 0,
        shape: RoundedRectangleBorder(
          borderRadius: BorderRadius.circular(12),
        ),
      ),
      elevatedButtonTheme: ElevatedButtonThemeData(
        style: ElevatedButton.styleFrom(
          backgroundColor: colorScheme.primary,
          foregroundColor: colorScheme.onPrimary,
        ),
      ),
    );
  }

  // 深色主题
  static ThemeData get darkTheme {
    final colorScheme = ColorScheme.fromSeed(
      seedColor: _seedColor,
      brightness: Brightness.dark,
    );

    return ThemeData(
      colorScheme: colorScheme,
      useMaterial3: true,
      appBarTheme: AppBarTheme(
        backgroundColor: colorScheme.surface,
        foregroundColor: colorScheme.onSurface,
        elevation: 0,
      ),
      cardTheme: CardTheme(
        color: colorScheme.surfaceContainerLow,
        elevation: 0,
        shape: RoundedRectangleBorder(
          borderRadius: BorderRadius.circular(12),
        ),
      ),
      elevatedButtonTheme: ElevatedButtonThemeData(
        style: ElevatedButton.styleFrom(
          backgroundColor: colorScheme.primary,
          foregroundColor: colorScheme.onPrimary,
        ),
      ),
    );
  }
}
```

---

### Step 3: 重构硬编码颜色与字体

**基于 Step 1 的审计报告**，逐个文件重构。

**AI 提示词**（可复制）：

```markdown
请重构 `[目标文件路径]` 中的硬编码样式，改用主题系统。

**重构规则**：

1. **颜色替换**：
   - 主要操作按钮、强调内容 → `Theme.of(context).colorScheme.primary`
   - 按钮/卡片背景 → `colorScheme.surface` 或 `surfaceContainer`
   - 正文文字 → `colorScheme.onSurface`
   - 次要文字/说明 → `colorScheme.onSurfaceVariant`
   - 错误提示 → `colorScheme.error`
   - 参考角色说明：https://api.flutter.dev/flutter/material/ColorScheme-class.html

2. **字体替换**：
   - 大标题 → `Theme.of(context).textTheme.headlineLarge`
   - 中标题 → `textTheme.titleMedium`
   - 正文 → `textTheme.bodyLarge` 或 `bodyMedium`
   - 标签/小字 → `textTheme.labelSmall`

3. **保持语义清晰**：
   - 如果某个颜色有特殊业务含义（如"成功绿色"），先标注出来，稍后用 ThemeExtension 处理
   - 不要把所有颜色都改成 `primary`，要根据用途选择对应角色

4. **完整性**：
   - 生成修改后的完整文件
   - 确保 import `'package:flutter/material.dart'`
   - 保留原有逻辑和注释

逐文件执行，不要一次性改动太多。
```

**检查点**：
- [ ] 修改后的代码编译通过（`flutter analyze`）
- [ ] 视觉效果与预期一致（热重载查看）
- [ ] 没有遗留明显的硬编码颜色

---

### Step 4: 添加自定义颜色（ThemeExtension）

**适用于**：品牌色、成功/警告/信息语义色等 `ColorScheme` 不直接提供的颜色。

**AI 提示词**（可复制）：

```markdown
为项目添加自定义主题扩展，管理业务特有颜色。

**要求**：

1. **扩展定义**：在 `lib/core/theme/app_colors.dart` 创建 `ThemeExtension` 子类
   - 参考：https://api.flutter.dev/flutter/material/ThemeExtension-class.html
   - 必须实现 `copyWith` 和 `lerp` 方法（保证主题切换平滑过渡）
   
2. **自定义颜色示例**（根据项目实际需求调整）：
   ```dart
   class AppColors extends ThemeExtension<AppColors> {
     final Color success;
     final Color warning;
     final Color info;

     const AppColors({
       required this.success,
       required this.warning,
       required this.info,
     });

     @override
     AppColors copyWith({Color? success, Color? warning, Color? info}) {
       return AppColors(
         success: success ?? this.success,
         warning: warning ?? this.warning,
         info: info ?? this.info,
       );
     }

     @override
     AppColors lerp(ThemeExtension<AppColors>? other, double t) {
       if (other is! AppColors) return this;
       return AppColors(
         success: Color.lerp(success, other.success, t)!,
         warning: Color.lerp(warning, other.warning, t)!,
         info: Color.lerp(info, other.info, t)!,
       );
     }
   }
   ```

3. **注册到 ThemeData**：
   ```dart
   ThemeData(
     colorScheme: colorScheme,
     extensions: [
       AppColors(
         success: const Color(0xFF4CAF50),
         warning: const Color(0xFFFFA726),
         info: const Color(0xFF29B6F6),
       ),
     ],
   )
   ```

4. **使用方式**：
   ```dart
   final appColors = Theme.of(context).extension<AppColors>()!;
   Container(color: appColors.success);
   ```

5. **明暗色适配**：
   - 浅色主题用明亮版本
   - 深色主题用柔和版本（如 `Color(0xFF81C784)` 替代 `0xFF4CAF50`）

生成完整的 `app_colors.dart` 和更新后的 `app_theme.dart`。
```

---

### Step 5: 本地验证（必须！）

**人工操作**：

```bash
# 1. 静态分析
flutter analyze

# 2. 运行应用
flutter run

# 3. 在模拟器/真机上切换明暗色模式
# iOS: 设置 → 显示与亮度 → 外观（浅色/深色）
# Android: 设置 → 显示 → 深色主题

# 4. 可选：热重载测试动态切换
# 在代码中临时修改 themeMode: ThemeMode.light / ThemeMode.dark
```

**验证清单**：
- [ ] 应用启动无报错
- [ ] 浅色模式下所有文字、图标清晰可读
- [ ] 深色模式下所有文字、图标清晰可读
- [ ] 主题切换时无闪烁或颜色异常
- [ ] 自定义颜色（如有）在两种模式下都合适
- [ ] 无硬编码颜色残留（目视检查关键页面）

**常见问题**：
- ❌ 深色模式下文字看不清：检查是否用了 `onSurface` 而非固定黑色/白色
- ❌ 按钮颜色不对：确认是否正确使用了 `colorScheme.primary` 和 `onPrimary` 的配对
- ❌ `extension<AppColors>()` 返回 null：检查是否在 `ThemeData.extensions` 中注册
- ❌ 某些第三方组件不遵循主题：查阅组件文档，可能需要单独设置

---

### Step 6: 建立主题切换功能（可选）

**适用于**：需要让用户手动选择浅色/深色/跟随系统。

**AI 提示词**（可复制）：

```markdown
为应用添加主题切换功能。

**要求**：

1. **状态管理**：使用 `ChangeNotifier` 或项目现有状态管理方案（如 Riverpod、Bloc）

2. **示例实现**（基于 Provider）：
   ```dart
   // lib/core/theme/theme_provider.dart
   class ThemeProvider extends ChangeNotifier {
     ThemeMode _themeMode = ThemeMode.system;

     ThemeMode get themeMode => _themeMode;

     void setThemeMode(ThemeMode mode) {
       _themeMode = mode;
       notifyListeners();
     }
   }
   
   // main.dart
   ChangeNotifierProvider(
     create: (_) => ThemeProvider(),
     child: Consumer<ThemeProvider>(
       builder: (context, themeProvider, _) {
         return MaterialApp(
           theme: AppTheme.lightTheme,
           darkTheme: AppTheme.darkTheme,
           themeMode: themeProvider.themeMode,
           // ...
         );
       },
     ),
   )
   ```

3. **设置页 UI**：
   - 提供"浅色"、"深色"、"跟随系统"三个选项
   - 用 `RadioListTile` 或 `SegmentedButton` 实现

4. **持久化**（可选）：用 `shared_preferences` 保存用户选择

生成完整代码，包括状态管理、UI 组件、持久化逻辑。
```

---

## 🎨 进阶：响应式间距与字体缩放

**适用于**需要适配不同屏幕尺寸或用户字体设置的应用。

**AI 提示词**：

```markdown
建立响应式间距系统。

**要求**：

1. **定义间距常量**：
   ```dart
   // lib/core/theme/app_spacing.dart
   class AppSpacing {
     static const double xs = 4.0;
     static const double sm = 8.0;
     static const double md = 16.0;
     static const double lg = 24.0;
     static const double xl = 32.0;
   }
   ```

2. **替换魔法数字**：
   - `padding: EdgeInsets.all(16)` → `EdgeInsets.all(AppSpacing.md)`
   - `SizedBox(height: 24)` → `SizedBox(height: AppSpacing.lg)`

3. **字体缩放适配**：
   - 用 `MediaQuery.textScalerOf(context)` 处理系统字体缩放
   - 或在 `MaterialApp` 中设置 `builder`:
     ```dart
     builder: (context, child) {
       return MediaQuery(
         data: MediaQuery.of(context).copyWith(
           textScaler: TextScaler.linear(1.0), // 禁用缩放
         ),
         child: child!,
       );
     }
     ```

逐步改造现有硬编码间距。
```

---

## ✅ 验证清单

完成后确认：

- [ ] 所有硬编码颜色已改用 `Theme.of(context).colorScheme.*`
- [ ] 字体样式统一使用 `textTheme.*`
- [ ] 自定义颜色通过 `ThemeExtension` 管理
- [ ] 应用在浅色和深色模式下均表现正常
- [ ] 主题配置文件清晰、可维护
- [ ] `flutter analyze` 无警告
- [ ] 关键页面已人工验证视觉效果

---

## 🤔 FAQ

### Q1: ColorScheme.fromSeed 生成的颜色不符合品牌要求？
**A**: 
- 使用 `ColorScheme.fromSeed(...).copyWith()` 覆盖特定角色颜色
- 或直接手动构造 `ColorScheme(primary: ..., secondary: ..., ...)`
- 参考设计系统时，可用设计工具（如 Figma）导出的色板

### Q2: 第三方包的组件不遵循主题？
**A**: 
- 查阅包文档，部分组件有 `theme` 或 `style` 参数
- 考虑封装一层自定义 Widget，统一注入主题色
- 实在无法改造，可局部使用硬编码但添加注释说明

### Q3: 需要支持多套品牌主题（如白标应用）？
**A**: 
- 定义多个 `ThemeData`（如 `brandATheme`, `brandBTheme`）
- 用配置文件或环境变量控制
- 可结合 Flavor 或远程配置实现

### Q4: Material 3 和 Material 2 有何区别？
**A**: 
- Material 3 默认圆角更大、配色更和谐、支持动态取色
- `useMaterial3: false` 可回退到 Material 2，但不建议长期使用
- 参考官方迁移指南：https://docs.flutter.dev/release/breaking-changes/material-3-migration

### Q5: 如何处理渐变色、阴影等复杂样式？
**A**: 
- 渐变色可定义在 `ThemeExtension` 的 `Gradient` 属性中
- 阴影用 `Theme.of(context).shadowColor` 或 `colorScheme.shadow`
- 复杂装饰可封装为独立的 `Decoration` 常量类

---

## 💡 最佳实践

1. **小步快跑，分模块重构**
   - 不要一次性改动整个项目，先从某个模块或页面开始
   - 重构一个模块后立即验证，确保无视觉回归

2. **优先使用 ColorScheme 的语义角色**
   - `primary` 用于主要操作、`secondary` 用于次要强调
   - `surface` / `surfaceContainer` 用于背景层次
   - `onXxx` 确保文字在 `xxx` 背景上可读

3. **自定义颜色要克制**
   - 只在 ColorScheme 无法满足时才用 ThemeExtension
   - 过多自定义色会破坏主题一致性

4. **明暗色主题同步维护**
   - 每次修改浅色主题，记得同步调整深色主题
   - 使用相同的 `seedColor` 保证色系一致

5. **文档化主题决策**
   - 在 `app_theme.dart` 顶部注释说明品牌色、设计规范链接
   - 记录自定义颜色的使用场景

6. **利用 AI 加速但人工把关**
   - AI 可快速生成重构代码，但要人工检查视觉效果
   - 特别注意对比度、可读性等无障碍要求

---

## 🔗 相关资源

- 提示词模板：
  - [prompts/theming/audit-hardcoded-styles.md](../prompts/theming/audit-hardcoded-styles.md) — 审计硬编码样式
  - [prompts/theming/implement-theme-system.md](../prompts/theming/implement-theme-system.md) — 落地主题体系与明暗色支持
- Flutter 官方文档：
  - [Material Design for Flutter](https://docs.flutter.dev/ui/design/material)
  - [ColorScheme class](https://api.flutter.dev/flutter/material/ColorScheme-class.html)
  - [ThemeData class](https://api.flutter.dev/flutter/material/ThemeData-class.html)
  - [ThemeExtension class](https://api.flutter.dev/flutter/material/ThemeExtension-class.html)
  - [Migrate to Material 3](https://docs.flutter.dev/release/breaking-changes/material-3-migration)
- Material Design 3 规范：
  - [Color system](https://m3.material.io/styles/color/system/overview)
  - [Typography](https://m3.material.io/styles/typography/overview)

---

**让主题统一，让 UI 更美 🎨**
