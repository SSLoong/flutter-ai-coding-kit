# 提示词：配置 gen-l10n 工具链

**用途**：为 Flutter 项目落地官方 gen-l10n 国际化工具链（依赖、配置、ARB 文件、接入 MaterialApp）

---

## 📋 适用场景

- 新项目首次配置国际化
- 从旧工具链（intl_translation）迁移到 gen-l10n
- 修复 gen-l10n 配置错误

---

## 📥 输入要求

在 Cursor 中准备：

1. **项目根目录已打开**（可访问 `pubspec.yaml`）
2. **（可选）已有字符串清单**（来自 [extract-hardcoded-strings.md](extract-hardcoded-strings.md)）
3. **确认支持的语言列表**（如英文 `en`、简体中文 `zh`）

---

## 💬 提示词模板

```markdown
为我的 Flutter 项目配置官方 gen-l10n 国际化工具链。

---

## Step 1: 添加依赖

修改 `pubspec.yaml`，添加以下依赖：

```yaml
dependencies:
  flutter:
    sdk: flutter
  flutter_localizations:
    sdk: flutter
  intl: any

flutter:
  generate: true  # 启用代码生成
  uses-material-design: true
```

不要删除已有的其他依赖。

---

## Step 2: 创建 l10n.yaml

在项目根目录创建 `l10n.yaml` 文件，内容如下：

```yaml
arb-dir: lib/l10n
template-arb-file: app_en.arb
output-localization-file: app_localizations.dart
nullable-getter: false
```

**字段说明**：
- `arb-dir`：ARB 文件存放目录（相对于项目根）
- `template-arb-file`：模板 ARB 文件名（通常为主语言，如英文）
- `output-localization-file`：生成的 Dart 文件名
- `nullable-getter`：生成的 getter 是否可空（设为 `false` 避免空检查）

---

## Step 3: 创建 ARB 文件

创建 `lib/l10n/` 目录，并生成以下文件：

### `lib/l10n/app_en.arb`（英文模板）

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
  }
}
```

### `lib/l10n/app_zh.arb`（简体中文翻译）

```json
{
  "@@locale": "zh",
  "appTitle": "我的应用",
  "homeWelcome": "欢迎",
  "loginButton": "登录"
}
```

**如果我已有字符串清单**：
[在此粘贴清单或说明需要迁移的字符串列表]

基于清单生成完整的 ARB 文件，包含所有 key，并正确处理以下场景：

### 带占位符（参数传递）

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

### 复数形式（plural）

```json
{
  "itemCount": "{count, plural, =0{No items} =1{1 item} other{{count} items}}",
  "@itemCount": {
    "description": "Number of items with plural forms",
    "placeholders": {
      "count": {
        "type": "int",
        "example": "5"
      }
    }
  }
}
```

### 选择形式（select）

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

---

## Step 4: 生成本地化代码

运行以下命令生成 Dart 代码：

```bash
flutter pub get
flutter gen-l10n
```

验证生成结果：

```bash
ls .dart_tool/flutter_gen/gen_l10n/
```

应看到 `app_localizations.dart` 等文件。

---

## Step 5: 接入 MaterialApp

修改 `lib/main.dart`，接入本地化：

```dart
import 'package:flutter/material.dart';
import 'package:flutter_localizations/flutter_localizations.dart';
import 'package:flutter_gen/gen_l10n/app_localizations.dart';

void main() {
  runApp(const MyApp());
}

class MyApp extends StatelessWidget {
  const MyApp({super.key});

  @override
  Widget build(BuildContext context) {
    return MaterialApp(
      title: 'My App',
      localizationsDelegates: const [
        AppLocalizations.delegate,
        GlobalMaterialLocalizations.delegate,
        GlobalWidgetsLocalizations.delegate,
        GlobalCupertinoLocalizations.delegate,
      ],
      supportedLocales: AppLocalizations.supportedLocales,
      // 可选：强制使用指定语言
      // locale: const Locale('zh'),
      theme: ThemeData(primarySwatch: Colors.blue),
      home: const HomePage(),
    );
  }
}
```

**如果项目使用 CupertinoApp**：

```dart
CupertinoApp(
  localizationsDelegates: const [
    AppLocalizations.delegate,
    GlobalMaterialLocalizations.delegate,
    GlobalWidgetsLocalizations.delegate,
    GlobalCupertinoLocalizations.delegate,
  ],
  supportedLocales: AppLocalizations.supportedLocales,
  // ...
)
```

---

## Step 6: 使用本地化字符串

在任意 Widget 中访问本地化字符串：

```dart
import 'package:flutter_gen/gen_l10n/app_localizations.dart';

class HomePage extends StatelessWidget {
  const HomePage({super.key});

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
          ElevatedButton(
            onPressed: () {},
            child: Text(l10n.loginButton),
          ),
          // 带占位符
          Text(l10n.welcomeUser('Alice')),
          // 复数形式
          Text(l10n.itemCount(5)),
        ],
      ),
    );
  }
}
```

---

## Step 7: iOS 配置（必须）

编辑 `ios/Runner/Info.plist`，添加支持的语言列表：

```xml
<key>CFBundleLocalizations</key>
<array>
  <string>en</string>
  <string>zh</string>
</array>
```

将上述 XML 插入到 `<dict>` 标签内（通常在 `CFBundleName` 之后）。

---

## 验证清单

完成后运行以下命令验证：

```bash
# 生成本地化代码
flutter gen-l10n

# 静态分析
flutter analyze

# 运行应用
flutter run
```

检查：
- [ ] 应用启动无报错
- [ ] 所有文本正确显示
- [ ] 切换系统语言后应用语言跟随变化

---

## 输出要求

生成以下文件的完整内容（可直接复制使用）：

1. 修改后的 `pubspec.yaml`（仅 dependencies 和 flutter 部分）
2. `l10n.yaml`
3. `lib/l10n/app_en.arb`
4. `lib/l10n/app_zh.arb`（或其他语言）
5. 修改后的 `lib/main.dart`（完整可运行）
6. 修改后的 `ios/Runner/Info.plist`（仅 CFBundleLocalizations 部分）

不要省略任何必要的 import 或配置项。
```

---

## ✅ 验证步骤

收到 AI 生成的配置后：

1. **应用配置**：
   - 复制 `pubspec.yaml` / `l10n.yaml` / ARB 文件到项目
   - 运行 `flutter pub get`

2. **验证生成**：
   ```bash
   flutter gen-l10n
   ```
   - [ ] 无错误输出
   - [ ] `.dart_tool/flutter_gen/gen_l10n/` 目录下有生成文件

3. **静态检查**：
   ```bash
   flutter analyze
   ```
   - [ ] 无错误或警告

4. **运行应用**：
   ```bash
   flutter run
   ```
   - [ ] 应用启动成功
   - [ ] 文本正确显示（无「找不到 key」错误）

5. **测试语言切换**：
   - 切换设备系统语言
   - 重启应用
   - 验证应用语言跟随系统变化

---

## 🛠️ 常见调整

### 问题 1：`AppLocalizations` 未定义

**原因**：未运行 `flutter gen-l10n` 或 import 路径错误。

**解决**：
```bash
flutter pub get
flutter gen-l10n
```

确认 import 路径为：
```dart
import 'package:flutter_gen/gen_l10n/app_localizations.dart';
```

### 问题 2：ARB 文件 JSON 格式错误

**症状**：`flutter gen-l10n` 报错 `FormatException`。

**解决**：
- 检查 JSON 是否有尾随逗号
- 验证引号配对
- 使用在线工具验证 JSON 合法性（如 jsonlint.com）

### 问题 3：占位符不生效

**症状**：生成的方法签名中没有参数。

**原因**：元数据 `placeholders` 拼写错误或位置不对。

**修正**：
```json
{
  "welcomeUser": "Hello, {username}!",
  "@welcomeUser": {
    "description": "...",
    "placeholders": {  // 必须在 @welcomeUser 元数据内
      "username": {
        "type": "String"
      }
    }
  }
}
```

### 问题 4：复数形式显示异常

**症状**：所有数字都显示同一个文本。

**原因**：ICU MessageFormat 语法错误。

**检查**：
- 格式必须为 `{变量名, plural, =0{...} =1{...} other{...}}`
- `plural` 前后无多余空格
- `other` 分支必须存在

### 问题 5：iOS 不显示语言切换选项

**原因**：未配置 `Info.plist` 的 `CFBundleLocalizations`。

**解决**：
编辑 `ios/Runner/Info.plist`，添加：
```xml
<key>CFBundleLocalizations</key>
<array>
  <string>en</string>
  <string>zh</string>
</array>
```

---

## 💡 高级配置

### 自定义生成类名

在 `l10n.yaml` 中添加：
```yaml
output-class: MyAppLocalizations
```

### 生成缺失翻译报告

在 `l10n.yaml` 中添加：
```yaml
untranslated-messages-file: lib/l10n/untranslated.json
```

运行 `flutter gen-l10n` 后检查报告文件。

### 支持更多语言

1. 添加新 ARB 文件（如 `app_ja.arb` 日语）
2. 在 `ios/Runner/Info.plist` 的 `CFBundleLocalizations` 中添加 `<string>ja</string>`
3. 重新运行 `flutter gen-l10n`

---

## 🔗 相关

- 工作流：[workflows/l10n-internationalization.md](../../workflows/l10n-internationalization.md)
- 审计字符串：[extract-hardcoded-strings.md](extract-hardcoded-strings.md)
- Flutter 官方文档：[Internationalizing Flutter apps](https://docs.flutter.dev/ui/accessibility-and-internationalization/internationalization)
