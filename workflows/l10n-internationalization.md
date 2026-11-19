# L10n 国际化工作流

**用 AI 把 Flutter 应用走完官方 gen-l10n / ARB 国际化路径**

> 作者：Loong (SSLoong) | 日期：2026-11-14

---

## 📋 背景

手动维护多语言 ARB 文件、处理占位符和复数形式容易出错且耗时。AI 能帮你审计硬编码字符串、生成 ARB 模板、配置 gen-l10n 工具链，但**必须本地验证**通过并在真机测试语言切换。

---

## 🎯 目标

- ✅ 审计项目中所有面向用户的硬编码字符串
- ✅ 配置官方 gen-l10n 工具链（flutter_localizations + intl）
- ✅ 编写符合规范的 ARB 文件（含 placeholders、plural、select）
- ✅ 在 MaterialApp / CupertinoApp 中接入本地化
- ✅ 实现语言切换与本地验证
- ✅ 配置 iOS Info.plist 支持多语言

---

## 🔧 适用场景

| 场景 | 适合度 | 说明 |
|------|--------|------|
| 已有英文应用需国际化 | ⭐⭐⭐⭐⭐ | 从 0 到 1 |
| 新增语言支持 | ⭐⭐⭐⭐⭐ | 快速生成新 ARB |
| 重构硬编码文案 | ⭐⭐⭐⭐ | AI 辅助审计 |
| 复杂占位符场景 | ⭐⭐⭐ | 需人工校验 ARB 语法 |

---

## 🚀 完整流程

### Step 1: 审计硬编码字符串

**人工操作**：在 Cursor 中打开 `lib/` 目录，准备让 AI 扫描所有 Dart 文件。

**AI 提示词**（可复制）：

```markdown
扫描 `lib/` 目录下所有 Dart 文件，列出所有面向用户的硬编码字符串。

**需要提取**：
- Text() / RichText() 的字符串内容
- TextField / TextFormField 的 hintText、labelText、helperText
- AppBar / SnackBar / Dialog 的 title、message
- Tooltip / Semantics label
- 按钮文字、错误提示、表单验证消息

**不应提取**（明确排除）：
- 日志输出（如 debugPrint、log）
- 分析事件名（如 analytics.logEvent）
- 路由路径（如 '/home', '/settings'）
- 资产 key（如 'assets/images/logo.png'）
- 单元测试断言字符串
- API endpoint URL
- 枚举值或常量标识符

输出格式：

| 文件路径 | 行号 | 原始字符串 | 建议 ARB Key | 优先级 |
|---------|------|------------|--------------|--------|
| lib/pages/home_page.dart | 23 | "Welcome" | homeWelcome | 高 |

优先级判断：
- 高：首屏、主流程文案
- 中：设置、次要页面
- 低：错误提示、极少出现文案
```

**检查点**：
- [ ] 表格列出至少 20 条字符串（小项目可少于此）
- [ ] 没有包含日志、路由 path 等不应翻译内容
- [ ] ARB Key 遵循 camelCase 命名

**常见问题**：
- ❌ AI 混入了 `print()` 日志：明确告知「排除所有 log/print 语句」
- ❌ 提取了枚举值：补充说明「只提取显示给最终用户的文本」

---

### Step 2: 添加依赖与配置

**人工操作**：在项目根目录执行以下命令。

```bash
# 添加依赖
flutter pub add flutter_localizations --sdk=flutter
flutter pub add intl:any
```

**修改 `pubspec.yaml`**（手动或让 AI 修改）：

```yaml
flutter:
  generate: true  # 启用代码生成
  uses-material-design: true
  # ... 其他配置
```

**AI 提示词**（可复制）：

```markdown
在项目根目录创建 `l10n.yaml` 文件，配置如下：

```yaml
arb-dir: lib/l10n
template-arb-file: app_en.arb
output-localization-file: app_localizations.dart
nullable-getter: false
```

说明每个字段的作用，并告知如何调整（如自定义 output-class 名称）。
```

**检查点**：
- [ ] `pubspec.yaml` 中 `generate: true` 已添加
- [ ] 依赖成功添加（运行 `flutter pub get` 无报错）
- [ ] `l10n.yaml` 文件已创建

**l10n.yaml 可选配置说明**（FAQ 部分会详细展开）：
- `output-class`：自定义生成的类名（默认为 AppLocalizations）
- `untranslated-messages-file`：生成缺失翻译报告
- `use-escaping`：是否转义特殊字符
- `synthetic-package`：是否生成为独立包（默认 true）

---

### Step 3: 编写 ARB 文件

**人工操作**：在项目根目录创建 `lib/l10n/` 目录。

**AI 提示词**（可复制）：

```markdown
基于 Step 1 的字符串清单，生成以下 ARB 文件：

1. **`lib/l10n/app_en.arb`**（英文模板）
2. **`lib/l10n/app_zh.arb`**（中文翻译）

**要求**：

1. **基础格式**：
```json
{
  "@@locale": "en",
  "homeWelcome": "Welcome",
  "@homeWelcome": {
    "description": "Home page welcome message"
  }
}
```

2. **带占位符**（参数传递）：
```json
{
  "welcomeUser": "Hello, {username}!",
  "@welcomeUser": {
    "description": "Greeting with username",
    "placeholders": {
      "username": {
        "type": "String",
        "example": "Alice"
      }
    }
  }
}
```

3. **复数形式**（plural）：
```json
{
  "itemCount": "{count, plural, =0{No items} =1{1 item} other{{count} items}}",
  "@itemCount": {
    "description": "Number of items",
    "placeholders": {
      "count": {
        "type": "int",
        "example": "5"
      }
    }
  }
}
```

4. **选择形式**（select）：
```json
{
  "userGender": "{gender, select, male{He} female{She} other{They}}",
  "@userGender": {
    "description": "Gender pronoun",
    "placeholders": {
      "gender": {
        "type": "String"
      }
    }
  }
}
```

优先处理 Step 1 中「高优先级」的字符串，生成完整的英文和中文 ARB。
```

**输出示例**（`lib/l10n/app_en.arb`）：

```json
{
  "@@locale": "en",
  "appTitle": "My App",
  "@appTitle": {
    "description": "Application title"
  },
  "homeWelcome": "Welcome",
  "@homeWelcome": {
    "description": "Home page welcome text"
  },
  "loginButton": "Log In",
  "@loginButton": {
    "description": "Login button label"
  },
  "welcomeUser": "Hello, {username}!",
  "@welcomeUser": {
    "description": "Personalized greeting",
    "placeholders": {
      "username": {
        "type": "String",
        "example": "Alice"
      }
    }
  },
  "itemCount": "{count, plural, =0{No items} =1{1 item} other{{count} items}}",
  "@itemCount": {
    "description": "Display item count with plural forms",
    "placeholders": {
      "count": {
        "type": "int",
        "example": "5"
      }
    }
  }
}
```

**输出示例**（`lib/l10n/app_zh.arb`）：

```json
{
  "@@locale": "zh",
  "appTitle": "我的应用",
  "homeWelcome": "欢迎",
  "loginButton": "登录",
  "welcomeUser": "你好，{username}！",
  "itemCount": "{count, plural, =0{无项目} =1{1 个项目} other{{count} 个项目}}"
}
```

**检查点**：
- [ ] ARB 文件 JSON 格式正确（可用 `jsonlint` 验证）
- [ ] 模板 ARB（app_en.arb）包含 `@@locale` 和所有 `@key` 元数据
- [ ] 翻译 ARB（app_zh.arb）只包含 `@@locale` 和翻译文本
- [ ] 占位符名称在元数据和消息中一致
- [ ] 复数/选择形式语法正确（ICU MessageFormat）

**常见问题**：
- ❌ JSON 解析错误：检查是否有尾随逗号、引号配对
- ❌ 占位符不生效：确认元数据中 `placeholders` 拼写正确
- ❌ 复数形式显示异常：验证 `{count, plural, ...}` 格式无空格错误

---

### Step 4: 生成本地化代码

**人工操作**：运行代码生成命令。

```bash
# 方法 1：触发代码生成
flutter pub get

# 方法 2：显式生成（推荐首次验证）
flutter gen-l10n

# 检查生成结果
ls .dart_tool/flutter_gen/gen_l10n/
```

**检查点**：
- [ ] `.dart_tool/flutter_gen/gen_l10n/` 目录下生成了 `app_localizations.dart`
- [ ] 生成文件包含所有 ARB key 对应的 getter 方法
- [ ] 无生成错误（如 ARB 格式错误会在此步报错）

**常见问题**：
- ❌ `Error: arb-dir does not exist`：检查 `l10n.yaml` 中的路径
- ❌ 占位符类型错误：ARB 元数据中 `type` 必须是合法 Dart 类型（String, int, double, DateTime 等）

---

### Step 5: 接入 MaterialApp

**AI 提示词**（可复制）：

```markdown
修改 `lib/main.dart` 中的 MaterialApp（或 CupertinoApp），接入本地化。

**修改前**：
```dart
MaterialApp(
  title: 'My App',
  home: HomePage(),
)
```

**修改后**：
```dart
import 'package:flutter_localizations/flutter_localizations.dart';
import 'package:flutter_gen/gen_l10n/app_localizations.dart';

MaterialApp(
  localizationsDelegates: const [
    AppLocalizations.delegate,
    GlobalMaterialLocalizations.delegate,
    GlobalWidgetsLocalizations.delegate,
    GlobalCupertinoLocalizations.delegate,
  ],
  supportedLocales: AppLocalizations.supportedLocales,
  // 可选：显式设置当前语言
  // locale: const Locale('zh'),
  title: 'My App',
  home: HomePage(),
)
```

**使用示例**（在 Widget 中）：
```dart
@override
Widget build(BuildContext context) {
  final l10n = AppLocalizations.of(context)!;
  
  return Scaffold(
    appBar: AppBar(
      title: Text(l10n.appTitle),
    ),
    body: Column(
      children: [
        Text(l10n.homeWelcome),
        Text(l10n.welcomeUser('Alice')),
        Text(l10n.itemCount(5)),
      ],
    ),
  );
}
```

生成完整可运行的 `main.dart`。
```

**检查点**：
- [ ] 所有必需的 delegate 已添加（Material、Widgets、Cupertino）
- [ ] `AppLocalizations.delegate` 在首位
- [ ] `supportedLocales` 使用生成的静态属性
- [ ] 代码可编译通过（`flutter analyze` 无错误）

**常见问题**：
- ❌ `AppLocalizations.of(context)` 返回 null：确保在 MaterialApp 子树中调用，且 app 已启动
- ❌ 显示 `AppLocalizations` 未定义：检查 import 路径是否为 `flutter_gen/gen_l10n/app_localizations.dart`

---

### Step 6: 本地验证

**人工操作**：

```bash
# 重新生成（确保 ARB 变更已同步）
flutter gen-l10n

# 静态分析
flutter analyze

# 运行应用（模拟器或真机）
flutter run

# 可选：运行测试
flutter test
```

**验证清单**：
- [ ] 应用启动无报错
- [ ] 所有硬编码字符串已替换为 `l10n.xxx`
- [ ] 带占位符的文本正确显示参数
- [ ] 复数形式显示正确（测试 0、1、多个的情况）
- [ ] 切换系统语言后应用语言跟随变化

**切换语言测试**：
- **模拟器**：Settings → Language & Input → 选择语言
- **真机**：系统设置 → 语言 → 添加并切换语言

**常见问题**：
- ❌ 切换语言不生效：检查 `supportedLocales` 是否包含目标语言
- ❌ 部分文本未翻译：运行 `flutter gen-l10n` 并检查 ARB 文件是否包含该 key
- ❌ 复数形式错误：验证 ARB 中 ICU MessageFormat 语法

---

### Step 7: 实现语言切换功能（可选）

**适用场景**：应用内提供语言切换开关，而非仅依赖系统语言。

**AI 提示词**（可复制）：

```markdown
实现应用内语言切换功能，要求：

1. **状态管理**：用 ValueNotifier 或应用层状态（Riverpod/Bloc/Provider 均可）保存用户选择的 Locale
2. **MaterialApp 配置**：
```dart
MaterialApp(
  locale: _currentLocale, // 从状态管理获取
  localizationsDelegates: ...,
  supportedLocales: AppLocalizations.supportedLocales,
  ...
)
```

3. **语言选择 UI**：
   - 设置页显示当前语言
   - 提供语言列表选择器（英文、简体中文、其他）
   - 选择后立即生效（rebuild MaterialApp）

4. **持久化**（可选）：用 SharedPreferences 保存用户选择，下次启动恢复

生成完整可运行代码。
```

**示例代码片段**（参考）：

```dart
import 'package:flutter/material.dart';
import 'package:flutter_gen/gen_l10n/app_localizations.dart';

class MyApp extends StatefulWidget {
  @override
  State<MyApp> createState() => _MyAppState();
}

class _MyAppState extends State<MyApp> {
  Locale _locale = const Locale('en');

  void _changeLanguage(Locale newLocale) {
    setState(() {
      _locale = newLocale;
    });
  }

  @override
  Widget build(BuildContext context) {
    return MaterialApp(
      locale: _locale,
      localizationsDelegates: const [
        AppLocalizations.delegate,
        GlobalMaterialLocalizations.delegate,
        GlobalWidgetsLocalizations.delegate,
        GlobalCupertinoLocalizations.delegate,
      ],
      supportedLocales: AppLocalizations.supportedLocales,
      home: SettingsPage(onLanguageChanged: _changeLanguage),
    );
  }
}

class SettingsPage extends StatelessWidget {
  final Function(Locale) onLanguageChanged;
  
  const SettingsPage({required this.onLanguageChanged});

  @override
  Widget build(BuildContext context) {
    final l10n = AppLocalizations.of(context)!;
    
    return Scaffold(
      appBar: AppBar(title: Text(l10n.appTitle)),
      body: ListTile(
        title: Text('Language'),
        trailing: DropdownButton<Locale>(
          value: Localizations.localeOf(context),
          items: [
            DropdownMenuItem(value: Locale('en'), child: Text('English')),
            DropdownMenuItem(value: Locale('zh'), child: Text('简体中文')),
          ],
          onChanged: (locale) {
            if (locale != null) onLanguageChanged(locale);
          },
        ),
      ),
    );
  }
}
```

**检查点**：
- [ ] 切换语言后 UI 立即更新
- [ ] 用户选择持久化（重启后保持）
- [ ] 支持系统语言优先（首次启动时）

---

### Step 8: iOS 配置（必须！）

**人工操作**：编辑 `ios/Runner/Info.plist`，添加支持的语言列表。

**AI 提示词**（可复制）：

```markdown
在 `ios/Runner/Info.plist` 中添加 `CFBundleLocalizations` 键，列出所有支持的语言代码。

**示例**（支持英文和简体中文）：
```xml
<key>CFBundleLocalizations</key>
<array>
  <string>en</string>
  <string>zh</string>
</array>
```

如果 Info.plist 已有其他配置，插入到合适位置（通常在 `CFBundleName` 之后）。
```

**检查点**：
- [ ] Info.plist 格式正确（XML 合法）
- [ ] 列出的语言代码与 ARB 文件的 `@@locale` 一致
- [ ] 在 iOS 真机「设置 → 通用 → 语言与地区」中可看到应用支持的语言

**常见问题**：
- ❌ iOS 系统设置中看不到语言选项：检查 `CFBundleLocalizations` 是否正确添加
- ❌ 切换语言后 iOS 应用未更新：杀掉应用重新启动（iOS 语言切换需完全重启）

---

## 🎨 完整代码示例

**最小可运行项目结构**：

```
my_app/
├── lib/
│   ├── l10n/
│   │   ├── app_en.arb
│   │   └── app_zh.arb
│   ├── main.dart
│   └── pages/
│       └── home_page.dart
├── l10n.yaml
├── pubspec.yaml
└── ios/
    └── Runner/
        └── Info.plist
```

**`lib/main.dart`**：

```dart
import 'package:flutter/material.dart';
import 'package:flutter_localizations/flutter_localizations.dart';
import 'package:flutter_gen/gen_l10n/app_localizations.dart';
import 'pages/home_page.dart';

void main() {
  runApp(const MyApp());
}

class MyApp extends StatelessWidget {
  const MyApp({super.key});

  @override
  Widget build(BuildContext context) {
    return MaterialApp(
      title: 'L10n Demo',
      localizationsDelegates: const [
        AppLocalizations.delegate,
        GlobalMaterialLocalizations.delegate,
        GlobalWidgetsLocalizations.delegate,
        GlobalCupertinoLocalizations.delegate,
      ],
      supportedLocales: AppLocalizations.supportedLocales,
      // 可选：强制使用中文
      // locale: const Locale('zh'),
      theme: ThemeData(primarySwatch: Colors.blue),
      home: const HomePage(),
    );
  }
}
```

**`lib/pages/home_page.dart`**：

```dart
import 'package:flutter/material.dart';
import 'package:flutter_gen/gen_l10n/app_localizations.dart';

class HomePage extends StatefulWidget {
  const HomePage({super.key});

  @override
  State<HomePage> createState() => _HomePageState();
}

class _HomePageState extends State<HomePage> {
  int _itemCount = 0;

  @override
  Widget build(BuildContext context) {
    final l10n = AppLocalizations.of(context)!;

    return Scaffold(
      appBar: AppBar(
        title: Text(l10n.appTitle),
      ),
      body: Padding(
        padding: const EdgeInsets.all(16.0),
        child: Column(
          crossAxisAlignment: CrossAxisAlignment.start,
          children: [
            Text(
              l10n.homeWelcome,
              style: Theme.of(context).textTheme.headlineMedium,
            ),
            const SizedBox(height: 16),
            Text(l10n.welcomeUser('Alice')),
            const SizedBox(height: 16),
            Text(l10n.itemCount(_itemCount)),
            const SizedBox(height: 16),
            ElevatedButton(
              onPressed: () => setState(() => _itemCount++),
              child: Text(l10n.loginButton),
            ),
          ],
        ),
      ),
    );
  }
}
```

**`lib/l10n/app_en.arb`**：

```json
{
  "@@locale": "en",
  "appTitle": "My App",
  "@appTitle": {
    "description": "Application title shown in AppBar"
  },
  "homeWelcome": "Welcome",
  "@homeWelcome": {
    "description": "Welcome message on home page"
  },
  "loginButton": "Log In",
  "@loginButton": {
    "description": "Login button label"
  },
  "welcomeUser": "Hello, {username}!",
  "@welcomeUser": {
    "description": "Personalized greeting with username",
    "placeholders": {
      "username": {
        "type": "String",
        "example": "Alice"
      }
    }
  },
  "itemCount": "{count, plural, =0{No items} =1{1 item} other{{count} items}}",
  "@itemCount": {
    "description": "Display item count with correct plural forms",
    "placeholders": {
      "count": {
        "type": "int",
        "example": "5"
      }
    }
  }
}
```

**`lib/l10n/app_zh.arb`**：

```json
{
  "@@locale": "zh",
  "appTitle": "我的应用",
  "homeWelcome": "欢迎",
  "loginButton": "登录",
  "welcomeUser": "你好，{username}！",
  "itemCount": "{count, plural, =0{无项目} =1{1 个项目} other{{count} 个项目}}"
}
```

**`l10n.yaml`**：

```yaml
arb-dir: lib/l10n
template-arb-file: app_en.arb
output-localization-file: app_localizations.dart
nullable-getter: false
```

**`pubspec.yaml`（关键部分）**：

```yaml
dependencies:
  flutter:
    sdk: flutter
  flutter_localizations:
    sdk: flutter
  intl: any

flutter:
  generate: true
  uses-material-design: true
```

**`ios/Runner/Info.plist`（添加部分）**：

```xml
<key>CFBundleLocalizations</key>
<array>
  <string>en</string>
  <string>zh</string>
</array>
```

---

## ✅ 验证清单

完成后确认：

- [ ] 所有硬编码字符串已迁移到 ARB 文件
- [ ] ARB 文件 JSON 格式正确，元数据完整
- [ ] `flutter gen-l10n` 无错误
- [ ] `flutter analyze` 无警告
- [ ] 应用启动正常，文本显示正确
- [ ] 切换系统语言后应用语言跟随变化
- [ ] 带占位符和复数形式的文本正确渲染
- [ ] iOS Info.plist 已添加 `CFBundleLocalizations`
- [ ] 在真机上验证至少两种语言

---

## 🤔 FAQ

### Q1: gen-l10n 和 intl_translation 有什么区别？
**A**: 
- **gen-l10n**（推荐）：Flutter 官方工具，从 Flutter 1.20+ 开始推荐，配置简单，与 `flutter pub get` 集成
- **intl_translation**：旧工具链，需要手动运行 `extract_to_arb` / `generate_from_arb`，更灵活但配置复杂
- 本工作流默认使用 **gen-l10n**，适合绝大多数场景

### Q2: 如何检查未翻译的文本？
**A**: 
在 `l10n.yaml` 中添加：
```yaml
untranslated-messages-file: lib/l10n/untranslated.json
```
运行 `flutter gen-l10n` 后会生成报告文件，列出所有缺失的翻译 key。

### Q3: 支持 RTL（从右到左）语言吗？
**A**: 
支持。MaterialApp 会自动检测语言方向。对于阿拉伯语或希伯来语：
1. 在 ARB 文件中添加 `"@@locale": "ar"`
2. 确保 UI 布局使用 `Row` / `Column` 而非固定方向
3. 测试时切换到 RTL 语言验证

### Q4: 如何使用更复杂的 Locale（如 zh_CN vs zh_TW）？
**A**: 
```dart
// 使用 Locale.fromSubtags 指定 script 或 country code
const Locale('zh', 'CN'), // 简体中文（中国大陆）
const Locale('zh', 'TW'), // 繁体中文（台湾）
```
ARB 文件命名：`app_zh_CN.arb`, `app_zh_TW.arb`。

### Q5: MaterialApp 启动前如何访问 AppLocalizations？
**A**: 
不能。`AppLocalizations.of(context)` 需要 MaterialApp 已经构建。如果需要在启动前（如 splash 页）显示文本，选项：
1. 使用英文硬编码占位符（仅 splash）
2. 延迟显示需要 l10n 的内容直到 MaterialApp ready
3. 使用独立的本地化逻辑（不依赖 context）

### Q6: 如何在测试中 mock AppLocalizations？
**A**: 
```dart
await tester.pumpWidget(
  MaterialApp(
    localizationsDelegates: AppLocalizations.localizationsDelegates,
    supportedLocales: AppLocalizations.supportedLocales,
    locale: const Locale('en'),
    home: MyWidget(),
  ),
);

final l10n = AppLocalizations.of(tester.element(find.byType(MyWidget)))!;
expect(l10n.appTitle, 'My App');
```

### Q7: ARB 文件中如何转义特殊字符？
**A**: 
- 单引号在 ICU MessageFormat 中有特殊含义，需双写：`"message": "It''s working"`
- 大括号：使用单引号包裹：`"message": "Use '{' and '}'"`
- 换行：使用 `\n`（在 JSON 中为 `"\\n"`）

### Q8: l10n.yaml 的高级配置有哪些？
**A**: 

| 配置项 | 说明 | 默认值 |
|--------|------|--------|
| `output-class` | 生成的类名 | `AppLocalizations` |
| `output-dir` | 输出目录（相对于项目根） | `.dart_tool/flutter_gen/gen_l10n` |
| `preferred-supported-locales` | 优先语言列表 | 无（按 ARB 文件顺序） |
| `use-deferred-loading` | 是否延迟加载语言资源 | `false` |
| `synthetic-package` | 生成为独立包 | `true` |
| `header` | 生成文件的头部注释 | 默认版权注释 |

详见官方文档：https://docs.flutter.dev/ui/accessibility-and-internationalization/internationalization

---

## 💡 最佳实践

1. **优先迁移主流程**
   - 先完成首屏、核心功能的国际化
   - 错误提示和边缘场景可后续补充

2. **ARB Key 命名规范**
   - 使用 camelCase：`homeWelcome`, `loginButtonLabel`
   - 包含上下文：`errorNetworkTimeout`（而非仅 `timeout`）
   - 避免过长：不超过 40 字符

3. **验证标准前置**
   - 告诉 AI："生成的 ARB 必须通过 `flutter gen-l10n` 且应用无编译错误"
   - 在 CI/CD 中集成 `flutter gen-l10n` 检查

4. **渐进式迁移**
   - 不必一次性迁移所有文案
   - 先配置工具链，然后按模块逐步替换硬编码

5. **保持元数据完整**
   - 模板 ARB 的 `@key` 元数据帮助团队理解上下文
   - `description` 字段建议填写使用场景，非仅翻译原文

6. **测试多种语言长度**
   - 某些语言翻译后字符更长（如德语）
   - 测试 UI 布局是否适应长文本（使用 `FittedBox` 或 `overflow` 处理）

7. **CI 中运行 gen-l10n**
   - 在 CI 流程中加入 `flutter gen-l10n` 和 `flutter analyze`
   - 防止合入不合法的 ARB 文件

---

## 🔗 相关资源

- 提示词模板：
  - [prompts/l10n/extract-hardcoded-strings.md](../prompts/l10n/extract-hardcoded-strings.md) — 审计硬编码字符串
  - [prompts/l10n/setup-gen-l10n.md](../prompts/l10n/setup-gen-l10n.md) — 配置 gen-l10n 工具链
- Flutter 官方文档：
  - [Internationalizing Flutter apps](https://docs.flutter.dev/ui/accessibility-and-internationalization/internationalization)
  - [Using the Dart intl tools](https://docs.flutter.dev/ui/accessibility-and-internationalization/internationalization#using-the-dart-intl-tools)
- ICU MessageFormat 语法：https://unicode-org.github.io/icu/userguide/format_parse/messages/

---

**让你的 Flutter 应用走向世界！🌍**
