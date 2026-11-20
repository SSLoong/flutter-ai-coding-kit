# 提示词：落地 Material 3 主题体系

**用途**：为 Flutter 项目建立完整的 Material 3 主题系统，支持明暗色模式与动态切换

---

## 📋 适用场景

- 新项目需要从零搭建主题体系
- 已审计硬编码问题，准备落地统一主题
- 需要支持明暗色主题或动态切换
- 希望使用 Material 3 的最佳实践

---

## 📥 输入要求

在 Cursor 中准备：

1. **品牌色信息**：主色调的十六进制值（如 `#6750A4`）或 Flutter Color 值
2. **（可选）设计规范**：如有设计稿，提供关键颜色、字体、间距定义
3. **（可选）审计报告**：从 [audit-hardcoded-styles.md](audit-hardcoded-styles.md) 生成的报告
4. **项目结构**：了解现有目录结构（如是否有 `lib/core/` 或 `lib/config/`）

---

## 💬 提示词模板

```markdown
为这个 Flutter 项目建立完整的 Material 3 主题系统。

---

## 一、主题配置文件

### 要求

1. **文件位置**：`lib/core/theme/app_theme.dart`
   - 如果项目已有其他主题文件，告知我路径，我会遵循现有结构

2. **使用 Material 3 最佳实践**：
   - 基于 `ColorScheme.fromSeed` 生成配色
   - 种子色（seed color）：[指定颜色，如 `Color(0xFF6750A4)` 或 `Colors.deepPurple`]
   - 同时生成浅色和深色两套主题

3. **配置内容**：
   ```dart
   class AppTheme {
     static const Color _seedColor = [种子色];

     // 浅色主题
     static ThemeData get lightTheme {
       final colorScheme = ColorScheme.fromSeed(
         seedColor: _seedColor,
         brightness: Brightness.light,
       );
       
       return ThemeData(
         colorScheme: colorScheme,
         useMaterial3: true,
         
         // 配置常用组件主题
         appBarTheme: AppBarTheme(...),
         cardTheme: CardTheme(...),
         elevatedButtonTheme: ElevatedButtonThemeData(...),
         textButtonTheme: TextButtonThemeData(...),
         outlinedButtonTheme: OutlinedButtonThemeData(...),
         inputDecorationTheme: InputDecorationTheme(...),
         // 根据项目常用组件补充
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
         // 与 lightTheme 相同的组件配置
       );
     }
   }
   ```

4. **添加注释**：
   - 在文件顶部说明设计规范来源（如 Figma 链接或品牌指南）
   - 为关键配置添加中文注释

---

## 二、自定义颜色扩展（ThemeExtension）

### 要求

1. **文件位置**：`lib/core/theme/app_colors.dart`

2. **定义自定义颜色**：
   - 仅在 `ColorScheme` 无法满足时使用（如品牌特定色、成功/警告/信息语义色）
   - 必须实现 `copyWith` 和 `lerp` 方法

3. **示例结构**：
   ```dart
   class AppColors extends ThemeExtension<AppColors> {
     final Color success;   // 成功状态（如绿色）
     final Color warning;   // 警告状态（如橙色）
     final Color info;      // 信息提示（如蓝色）

     const AppColors({
       required this.success,
       required this.warning,
       required this.info,
     });

     @override
     AppColors copyWith({...}) { ... }

     @override
     AppColors lerp(ThemeExtension<AppColors>? other, double t) { ... }
     
     // 浅色模式实例
     static const light = AppColors(
       success: Color(0xFF4CAF50),
       warning: Color(0xFFFFA726),
       info: Color(0xFF29B6F6),
     );
     
     // 深色模式实例（颜色需更柔和）
     static const dark = AppColors(
       success: Color(0xFF81C784),
       warning: Color(0xFFFFB74D),
       info: Color(0xFF4FC3F7),
     );
   }
   ```

4. **注册到主题**：
   在 `AppTheme.lightTheme` 和 `darkTheme` 中添加：
   ```dart
   extensions: [AppColors.light],  // lightTheme 中
   extensions: [AppColors.dark],   // darkTheme 中
   ```

---

## 三、应用主题到 MaterialApp

### 要求

1. **修改主入口文件**（通常是 `lib/main.dart`）：
   ```dart
   import 'package:flutter/material.dart';
   import 'core/theme/app_theme.dart';

   void main() {
     runApp(const MyApp());
   }

   class MyApp extends StatelessWidget {
     const MyApp({super.key});

     @override
     Widget build(BuildContext context) {
       return MaterialApp(
         title: '[应用名称]',
         
         // 应用主题
         theme: AppTheme.lightTheme,
         darkTheme: AppTheme.darkTheme,
         themeMode: ThemeMode.system, // 跟随系统设置
         
         home: const HomeScreen(),
       );
     }
   }
   ```

2. **确保完整性**：
   - 所有必要的 import
   - 移除旧的主题配置代码（如之前的 `theme: ThemeData(primarySwatch: ...)` 等）

---

## 四、（可选）主题切换功能

如果需要让用户手动切换主题（浅色/深色/跟随系统），添加：

1. **状态管理**：
   - 使用 `ChangeNotifier` + `Provider`，或
   - 项目现有方案（如 Riverpod、Bloc）

2. **示例实现**（基于 Provider）：
   ```dart
   // lib/core/theme/theme_provider.dart
   class ThemeProvider extends ChangeNotifier {
     ThemeMode _themeMode = ThemeMode.system;

     ThemeMode get themeMode => _themeMode;

     void setThemeMode(ThemeMode mode) {
       _themeMode = mode;
       notifyListeners();
       // 可选：保存到 SharedPreferences
     }

     Future<void> loadThemeMode() async {
       // 从 SharedPreferences 加载
     }
   }
   ```

3. **UI 控件**（设置页）：
   ```dart
   RadioListTile<ThemeMode>(
     title: const Text('浅色模式'),
     value: ThemeMode.light,
     groupValue: themeProvider.themeMode,
     onChanged: (mode) => themeProvider.setThemeMode(mode!),
   ),
   RadioListTile<ThemeMode>(
     title: const Text('深色模式'),
     value: ThemeMode.dark,
     groupValue: themeProvider.themeMode,
     onChanged: (mode) => themeProvider.setThemeMode(mode!),
   ),
   RadioListTile<ThemeMode>(
     title: const Text('跟随系统'),
     value: ThemeMode.system,
     groupValue: themeProvider.themeMode,
     onChanged: (mode) => themeProvider.setThemeMode(mode!),
   ),
   ```

---

## 五、（可选）间距系统

定义统一的间距常量：

```dart
// lib/core/theme/app_spacing.dart
class AppSpacing {
  static const double xs = 4.0;
  static const double sm = 8.0;
  static const double md = 16.0;
  static const double lg = 24.0;
  static const double xl = 32.0;
  static const double xxl = 48.0;
}
```

**使用方式**：
```dart
padding: EdgeInsets.all(AppSpacing.md),  // 替代 16.0
SizedBox(height: AppSpacing.lg),         // 替代 24.0
```

---

## 输出要求

生成以下完整文件：

1. ✅ `lib/core/theme/app_theme.dart` — 主题定义
2. ✅ `lib/core/theme/app_colors.dart` — 自定义颜色扩展（如需要）
3. ✅ `lib/core/theme/app_spacing.dart` — 间距常量（如需要）
4. ✅ 修改后的 `lib/main.dart` 或主入口文件
5. ✅ `lib/core/theme/theme_provider.dart` — 主题切换状态管理（如需要）

---

## 代码规范

- 所有文件包含必要的 import
- 使用 `const` 构造函数（性能优化）
- 添加中文注释说明关键配置
- 遵循 Dart 命名规范（类名大驼峰、变量名小驼峰）
- 确保代码可直接运行 `flutter run`

---

## 验证检查点

生成代码后，我会验证：

- [ ] 编译无错误（`flutter analyze`）
- [ ] 浅色模式下视觉正常
- [ ] 深色模式下视觉正常
- [ ] `Theme.of(context).colorScheme.primary` 可正确访问
- [ ] 自定义颜色 `Theme.of(context).extension<AppColors>()!.success` 可访问（如有）
```

---

## ✅ 验证步骤

代码生成后执行：

```bash
# 1. 静态分析
flutter analyze

# 2. 运行应用
flutter run

# 3. 切换系统明暗模式验证
# iOS: 设置 → 显示与亮度 → 外观
# Android: 设置 → 显示 → 深色主题

# 4. 检查关键颜色
# 在任意 Widget 中测试：
final primary = Theme.of(context).colorScheme.primary;
final onPrimary = Theme.of(context).colorScheme.onPrimary;
final appColors = Theme.of(context).extension<AppColors>();
print('Primary: $primary, Success: ${appColors?.success}');
```

**检查清单**：
- [ ] 应用启动无报错
- [ ] 浅色模式下所有文字清晰可读（对比度足够）
- [ ] 深色模式下所有文字清晰可读
- [ ] 主题切换时无闪烁或异常
- [ ] 按钮、卡片、AppBar 等组件样式统一
- [ ] 自定义颜色（如有）在两种模式下表现合理

---

## 🛠️ 常见调整

### 1. 覆盖 ColorScheme 中的特定颜色

如果 `ColorScheme.fromSeed` 生成的某个颜色不符合品牌要求：

```dart
final colorScheme = ColorScheme.fromSeed(
  seedColor: _seedColor,
  brightness: Brightness.light,
).copyWith(
  primary: const Color(0xFF6200EE),      // 覆盖主色
  secondary: const Color(0xFF03DAC6),    // 覆盖次要色
);
```

### 2. 配置组件主题示例

**AppBar**（无阴影、浅色背景）：
```dart
appBarTheme: AppBarTheme(
  backgroundColor: colorScheme.surface,
  foregroundColor: colorScheme.onSurface,
  elevation: 0,
  centerTitle: true,
),
```

**Card**（圆角、无阴影、使用 surfaceContainer）：
```dart
cardTheme: CardTheme(
  color: colorScheme.surfaceContainerLow,
  elevation: 0,
  shape: RoundedRectangleBorder(
    borderRadius: BorderRadius.circular(12),
  ),
),
```

**ElevatedButton**（使用主色）：
```dart
elevatedButtonTheme: ElevatedButtonThemeData(
  style: ElevatedButton.styleFrom(
    backgroundColor: colorScheme.primary,
    foregroundColor: colorScheme.onPrimary,
    shape: RoundedRectangleBorder(
      borderRadius: BorderRadius.circular(8),
    ),
  ),
),
```

**InputDecoration**（边框样式、标签色）：
```dart
inputDecorationTheme: InputDecorationTheme(
  border: OutlineInputBorder(
    borderRadius: BorderRadius.circular(8),
  ),
  focusedBorder: OutlineInputBorder(
    borderRadius: BorderRadius.circular(8),
    borderSide: BorderSide(color: colorScheme.primary, width: 2),
  ),
  labelStyle: TextStyle(color: colorScheme.onSurfaceVariant),
),
```

### 3. 使用项目现有状态管理

如果项目使用 **Riverpod**：

```dart
// lib/core/theme/theme_provider.dart
final themeModeProvider = StateNotifierProvider<ThemeModeNotifier, ThemeMode>(
  (ref) => ThemeModeNotifier(),
);

class ThemeModeNotifier extends StateNotifier<ThemeMode> {
  ThemeModeNotifier() : super(ThemeMode.system);

  void setThemeMode(ThemeMode mode) {
    state = mode;
    // 可选：保存到 SharedPreferences
  }
}

// main.dart
class MyApp extends ConsumerWidget {
  const MyApp({super.key});

  @override
  Widget build(BuildContext context, WidgetRef ref) {
    final themeMode = ref.watch(themeModeProvider);
    
    return MaterialApp(
      theme: AppTheme.lightTheme,
      darkTheme: AppTheme.darkTheme,
      themeMode: themeMode,
      home: const HomeScreen(),
    );
  }
}
```

如果项目使用 **Bloc**：

```dart
// lib/core/theme/theme_bloc.dart
class ThemeBloc extends Bloc<ThemeEvent, ThemeState> {
  ThemeBloc() : super(ThemeState(ThemeMode.system)) {
    on<ThemeChanged>((event, emit) {
      emit(ThemeState(event.themeMode));
    });
  }
}

// main.dart
BlocBuilder<ThemeBloc, ThemeState>(
  builder: (context, state) {
    return MaterialApp(
      theme: AppTheme.lightTheme,
      darkTheme: AppTheme.darkTheme,
      themeMode: state.themeMode,
      home: const HomeScreen(),
    );
  },
)
```

### 4. 持久化主题选择

使用 `shared_preferences` 保存用户选择：

```dart
import 'package:shared_preferences/shared_preferences.dart';

class ThemeProvider extends ChangeNotifier {
  static const String _key = 'theme_mode';
  ThemeMode _themeMode = ThemeMode.system;

  ThemeMode get themeMode => _themeMode;

  Future<void> loadThemeMode() async {
    final prefs = await SharedPreferences.getInstance();
    final index = prefs.getInt(_key) ?? 0;
    _themeMode = ThemeMode.values[index];
    notifyListeners();
  }

  Future<void> setThemeMode(ThemeMode mode) async {
    _themeMode = mode;
    notifyListeners();
    
    final prefs = await SharedPreferences.getInstance();
    await prefs.setInt(_key, mode.index);
  }
}

// main.dart 中初始化
void main() async {
  WidgetsFlutterBinding.ensureInitialized();
  final themeProvider = ThemeProvider();
  await themeProvider.loadThemeMode();
  
  runApp(
    ChangeNotifierProvider.value(
      value: themeProvider,
      child: const MyApp(),
    ),
  );
}
```

---

## 🎨 高级技巧

### 1. 根据平台自动调整样式

```dart
static ThemeData get lightTheme {
  final colorScheme = ColorScheme.fromSeed(...);
  
  return ThemeData(
    colorScheme: colorScheme,
    useMaterial3: true,
    platform: TargetPlatform.iOS, // 或根据 Platform.isIOS 判断
    // iOS 风格滚动行为
    scrollBehavior: const CupertinoScrollBehavior(),
  );
}
```

### 2. 自定义文本主题

```dart
textTheme: TextTheme(
  displayLarge: TextStyle(
    fontSize: 57,
    fontWeight: FontWeight.bold,
    color: colorScheme.onSurface,
  ),
  titleLarge: TextStyle(
    fontSize: 22,
    fontWeight: FontWeight.w500,
    color: colorScheme.onSurface,
  ),
  bodyMedium: TextStyle(
    fontSize: 14,
    color: colorScheme.onSurfaceVariant,
  ),
  // 其他样式...
),
```

### 3. 支持自定义字体

```dart
// pubspec.yaml 中配置字体
fonts:
  - family: CustomFont
    fonts:
      - asset: assets/fonts/CustomFont-Regular.ttf
      - asset: assets/fonts/CustomFont-Bold.ttf
        weight: 700

// app_theme.dart 中应用
fontFamily: 'CustomFont',
```

---

## 🔗 相关

- 工作流：[workflows/theme-consistency.md](../../workflows/theme-consistency.md)
- 审计硬编码：[audit-hardcoded-styles.md](audit-hardcoded-styles.md)
- Flutter 官方文档：
  - [ColorScheme.fromSeed](https://api.flutter.dev/flutter/material/ColorScheme/ColorScheme.fromSeed.html)
  - [ThemeData](https://api.flutter.dev/flutter/material/ThemeData-class.html)
  - [ThemeExtension](https://api.flutter.dev/flutter/material/ThemeExtension-class.html)
  - [Material 3 迁移指南](https://docs.flutter.dev/release/breaking-changes/material-3-migration)
