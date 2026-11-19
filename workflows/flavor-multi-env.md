# Flavor 多环境配置工作流

**用 AI 为 Flutter 项目构建 dev/staging/prod 环境隔离方案**

> 作者：Loong (SSLoong) | 日期：2026-11-10

---

## 📋 背景

现代 App 开发需要在不同环境间切换（开发、预发布、生产），手动管理配置易错且不安全。Flutter 提供 `--flavor` 和 `--dart-define` 机制，但涉及 Android Gradle、iOS Scheme、入口文件等多处配置。AI 能帮你快速生成框架代码，但**必须验证编译和运行**后才能合入。

---

## 🎯 目标

- ✅ 配置多环境入口（dev/staging/prod）和独立 main 文件
- ✅ 使用 `--flavor` + `--dart-define-from-file` 隔离配置（API baseUrl、功能开关）
- ✅ 适配 Android productFlavors 和 iOS Schemes/xcconfig
- ✅ 保证密钥不进仓库，通过环境变量或本地文件注入
- ✅ 提供 CI/CD 友好的验证命令

---

## 🔧 适用场景

| 场景 | 适合度 | 说明 |
|------|--------|------|
| 单环境项目首次引入 Flavor | ⭐⭐⭐⭐⭐ | 从 0 到 1 完整改造 |
| 已有部分环境变量，需规范化 | ⭐⭐⭐⭐ | 迁移到 Flavor 体系 |
| 排查 Flavor 构建失败 | ⭐⭐⭐⭐⭐ | 定位 Android/iOS 配置问题 |
| CI 集成多环境自动打包 | ⭐⭐⭐⭐ | 配合 GitHub Actions/Fastlane |

---

## 🚀 完整流程

### Step 1: 确认需求与架构

**人工操作**：在 Cursor 中打开项目根目录，准备好以下信息：

1. **环境列表**：dev、staging、prod（或自定义）
2. **环境变量清单**：API baseUrl、应用名称、Bundle ID、功能开关等
3. **平台范围**：仅 Android？仅 iOS？还是双平台？

**AI 提示词**（可复制）：

```markdown
我需要为 Flutter 项目配置 Flavor 多环境支持。

## 需求

**目标环境**：
- dev（开发）
- staging（预发布）
- prod（生产）

**环境变量**：
- API baseUrl（如 dev: https://dev-api.example.com）
- 应用显示名称（如 dev: MyApp Dev, prod: MyApp）
- Android applicationIdSuffix（如 dev: .dev, staging: .staging, prod: 无后缀）
- iOS Bundle ID（如 dev: com.example.myapp.dev, prod: com.example.myapp）
- 功能开关（如 enableDebugTools: dev=true, prod=false）

**平台**：Android + iOS

请给出技术方案：
1. 使用 `--flavor` 还是 `--dart-define`？
2. 入口文件如何组织（单 main.dart vs 多 main_xxx.dart）？
3. Android 和 iOS 需要修改哪些配置文件？
4. 如何避免密钥进入 Git？
```

**检查点**：AI 应明确：
- `--flavor` 用于原生层（Android/iOS）区分构建产物
- `--dart-define-from-file` 用于 Dart 层注入配置
- 推荐多入口文件（`main_dev.dart`、`main_staging.dart`、`main_prod.dart`）+ 共享 `app.dart`

---

### Step 2: 生成 Dart 层配置代码

**AI 提示词**（可复制）：

```markdown
基于上述方案，生成以下文件：

## 1. 环境配置类 `lib/config/env_config.dart`

要求：
- 使用 `const String.fromEnvironment()` 读取 dart-define 变量
- 提供类型安全的访问接口（apiBaseUrl、appName、bundleId、featureFlags）
- 支持默认值（避免未传参数时崩溃）

## 2. 入口文件 `lib/main_dev.dart`、`lib/main_staging.dart`、`lib/main_prod.dart`

要求：
- 每个文件调用 `EnvConfig.initialize(Environment.dev)` 等
- 然后运行 `runApp(MyApp())`
- 主逻辑复用 `lib/app.dart`

## 3. 共享 App 文件 `lib/app.dart`

要求：
- 使用 `EnvConfig.instance` 获取当前环境配置
- 在 MaterialApp 的 title 中显示环境名称（方便调试）

生成完整可运行代码，包含所有 import。
```

**输出示例**（参考）：

```dart
// lib/config/env_config.dart
enum Environment { dev, staging, prod }

class EnvConfig {
  final Environment environment;
  final String apiBaseUrl;
  final String appName;
  final bool enableDebugTools;

  const EnvConfig({
    required this.environment,
    required this.apiBaseUrl,
    required this.appName,
    required this.enableDebugTools,
  });

  static late EnvConfig instance;

  static void initialize(Environment env) {
    instance = EnvConfig(
      environment: env,
      apiBaseUrl: const String.fromEnvironment(
        'API_BASE_URL',
        defaultValue: 'https://api.example.com',
      ),
      appName: const String.fromEnvironment(
        'APP_NAME',
        defaultValue: 'MyApp',
      ),
      enableDebugTools: const bool.fromEnvironment(
        'ENABLE_DEBUG_TOOLS',
        defaultValue: false,
      ),
    );
  }

  bool get isProduction => environment == Environment.prod;
}

// lib/main_dev.dart
import 'package:flutter/material.dart';
import 'config/env_config.dart';
import 'app.dart';

void main() {
  EnvConfig.initialize(Environment.dev);
  runApp(const MyApp());
}

// lib/main_staging.dart
import 'package:flutter/material.dart';
import 'config/env_config.dart';
import 'app.dart';

void main() {
  EnvConfig.initialize(Environment.staging);
  runApp(const MyApp());
}

// lib/main_prod.dart
import 'package:flutter/material.dart';
import 'config/env_config.dart';
import 'app.dart';

void main() {
  EnvConfig.initialize(Environment.prod);
  runApp(const MyApp());
}

// lib/app.dart
import 'package:flutter/material.dart';
import 'config/env_config.dart';

class MyApp extends StatelessWidget {
  const MyApp({super.key});

  @override
  Widget build(BuildContext context) {
    final config = EnvConfig.instance;
    return MaterialApp(
      title: '${config.appName} (${config.environment.name})',
      home: Scaffold(
        appBar: AppBar(
          title: Text(config.appName),
        ),
        body: Center(
          child: Column(
            mainAxisAlignment: MainAxisAlignment.center,
            children: [
              Text('Environment: ${config.environment.name}'),
              Text('API: ${config.apiBaseUrl}'),
              if (config.enableDebugTools)
                const Text('Debug Tools Enabled', style: TextStyle(color: Colors.red)),
            ],
          ),
        ),
      ),
    );
  }
}
```

---

### Step 3: 配置 Android Flavor

**AI 提示词**（可复制）：

```markdown
修改 `android/app/build.gradle`，添加 productFlavors 配置：

**要求**：
1. 在 `android {}` 块中添加 `flavorDimensions "environment"`
2. 定义三个 flavor：dev、staging、prod
3. 每个 flavor 设置：
   - `applicationIdSuffix`（dev: ".dev", staging: ".staging", prod: ""）
   - `resValue "string", "app_name"`（dev: "MyApp Dev", prod: "MyApp"）
4. 保持现有配置（compileSdk、defaultConfig 等）不变

生成完整的 `android {}` 块。
```

**输出示例**：

```gradle
android {
    namespace "com.example.myapp"
    compileSdk flutter.compileSdkVersion
    ndkVersion flutter.ndkVersion

    // ... defaultConfig 等

    flavorDimensions "environment"
    
    productFlavors {
        dev {
            dimension "environment"
            applicationIdSuffix ".dev"
            resValue "string", "app_name", "MyApp Dev"
        }
        staging {
            dimension "environment"
            applicationIdSuffix ".staging"
            resValue "string", "app_name", "MyApp Staging"
        }
        prod {
            dimension "environment"
            resValue "string", "app_name", "MyApp"
        }
    }
}
```

**验证步骤**：

```bash
# 测试 Android dev flavor 构建
cd android
./gradlew assembleDev
cd ..

# 查看生成的 APK
ls -lh build/app/outputs/flutter-apk/app-dev-release.apk
```

---

### Step 4: 配置 iOS Scheme 和 xcconfig

**AI 提示词**（可复制）：

```markdown
为 iOS 添加 Flavor 支持，需要：

## 1. 创建 xcconfig 文件

在 `ios/Flutter/` 下创建：
- `Dev.xcconfig`
- `Staging.xcconfig`
- `Prod.xcconfig`

每个文件内容：
```xcconfig
#include "Release.xcconfig"  // 或 Debug.xcconfig

PRODUCT_BUNDLE_IDENTIFIER = com.example.myapp.dev  // 按环境调整
PRODUCT_NAME = MyApp Dev  // 按环境调整
ASSETCATALOG_COMPILER_APPICON_NAME = AppIcon-Dev  // 可选：不同环境用不同图标
```

## 2. Xcode Scheme 配置说明

由于 AI 无法直接修改 Xcode 项目文件（.pbxproj），请人工在 Xcode 中操作：

1. 打开 `ios/Runner.xcworkspace`
2. Product → Scheme → Manage Schemes
3. 复制 Runner 为 dev、staging、prod
4. 对每个 Scheme：
   - Edit Scheme → Build Configuration 选择 Release（或 Debug）
   - Run → Arguments → Environment Variables 添加 `PRODUCT_BUNDLE_IDENTIFIER`
5. 在 Build Settings 中关联对应 xcconfig

生成三个 xcconfig 文件的完整内容。
```

**输出示例**：

```xcconfig
// ios/Flutter/Dev.xcconfig
#include "Release.xcconfig"

PRODUCT_BUNDLE_IDENTIFIER = com.example.myapp.dev
PRODUCT_NAME = MyApp Dev
```

```xcconfig
// ios/Flutter/Staging.xcconfig
#include "Release.xcconfig"

PRODUCT_BUNDLE_IDENTIFIER = com.example.myapp.staging
PRODUCT_NAME = MyApp Staging
```

```xcconfig
// ios/Flutter/Prod.xcconfig
#include "Release.xcconfig"

PRODUCT_BUNDLE_IDENTIFIER = com.example.myapp
PRODUCT_NAME = MyApp
```

**人工操作**：
1. 创建上述 xcconfig 文件
2. 在 Xcode 中配置 Scheme（见上述步骤）
3. 验证 Info.plist 中 `CFBundleDisplayName` 是否使用 `${PRODUCT_NAME}`

---

### Step 5: 创建 dart-define 配置文件

**AI 提示词**（可复制）：

```markdown
创建 `config/` 目录（在项目根下，**不纳入 Git**），生成三个 JSON 文件：

- `config/dev.json`
- `config/staging.json`
- `config/prod.json`

每个文件包含：
```json
{
  "API_BASE_URL": "https://dev-api.example.com",
  "APP_NAME": "MyApp Dev",
  "ENABLE_DEBUG_TOOLS": "true"
}
```

按环境调整值。

同时生成 `config/.gitignore` 忽略敏感文件（如包含密钥的 local.json）。
```

**输出示例**：

```json
// config/dev.json
{
  "API_BASE_URL": "https://dev-api.example.com",
  "APP_NAME": "MyApp Dev",
  "ENABLE_DEBUG_TOOLS": "true"
}
```

```json
// config/staging.json
{
  "API_BASE_URL": "https://staging-api.example.com",
  "APP_NAME": "MyApp Staging",
  "ENABLE_DEBUG_TOOLS": "false"
}
```

```json
// config/prod.json
{
  "API_BASE_URL": "https://api.example.com",
  "APP_NAME": "MyApp",
  "ENABLE_DEBUG_TOOLS": "false"
}
```

```gitignore
# config/.gitignore
local.json
*.secret.json
```

**人工操作**：将 `config/` 加入根目录 `.gitignore`（可选择性提交示例配置）。

---

### Step 6: 本地验证（必须！）

**人工操作**：

```bash
# Android Dev
flutter run --flavor dev --dart-define-from-file=config/dev.json -t lib/main_dev.dart

# Android Staging
flutter run --flavor staging --dart-define-from-file=config/staging.json -t lib/main_staging.dart

# Android Prod
flutter run --flavor prod --dart-define-from-file=config/prod.json -t lib/main_prod.dart

# iOS Dev（需先在 Xcode 中配置好 Scheme）
flutter run --flavor dev --dart-define-from-file=config/dev.json -t lib/main_dev.dart

# 构建 APK
flutter build apk --flavor dev --dart-define-from-file=config/dev.json -t lib/main_dev.dart

# 构建 iOS（需 macOS）
flutter build ios --flavor prod --dart-define-from-file=config/prod.json -t lib/main_prod.dart
```

**检查清单**：
- [ ] 每个 flavor 都能成功运行（Android/iOS）
- [ ] App 显示名称与环境匹配
- [ ] API baseUrl 正确注入（在 UI 中显示验证）
- [ ] Android applicationId 带正确后缀（用 `adb shell pm list packages` 查看）
- [ ] iOS Bundle ID 正确（在 Xcode 或 `flutter run` 输出中确认）
- [ ] 构建产物可正常安装并运行

**常见问题**：
- ❌ `Flavor dev is not defined`：Android 未配置 productFlavors
- ❌ iOS 构建失败 `Scheme not found`：未在 Xcode 中创建对应 Scheme
- ❌ `String.fromEnvironment` 返回空：未传 `--dart-define-from-file`
- ❌ Android 和 iOS flavor 名不一致：必须完全匹配（大小写）

---

### Step 7: 排查问题与迭代

**若验证失败**，用以下提示词让 AI 诊断：

```markdown
运行 `flutter run --flavor dev` 报错：

[粘贴完整错误信息]

当前配置：
- Android build.gradle 中已添加 productFlavors
- iOS Scheme 已创建
- dart-define 配置文件：config/dev.json

请分析问题并给出修复方案。
```

**常见修复场景**：
1. **Android flavor 名大小写不匹配**：确保 `build.gradle` 中 `dev {}` 与命令行 `--flavor dev` 一致
2. **iOS Scheme 未关联 xcconfig**：在 Xcode Build Settings 中手动关联
3. **dart-define 变量未生效**：检查 JSON 文件路径和键名拼写

---

## 🎨 进阶：CI/CD 集成

**适合 GitHub Actions、GitLab CI 等场景**。

**AI 提示词**：

```markdown
生成 GitHub Actions workflow，实现：

1. PR 时自动构建所有 flavor（dev/staging/prod）
2. 分别运行 `flutter build apk --flavor xxx --dart-define-from-file=config/xxx.json -t lib/main_xxx.dart`
3. 上传构建产物为 Artifact
4. 失败时报告具体哪个 flavor 失败

生成 `.github/workflows/build-flavors.yml`。
```

**输出示例**：

```yaml
name: Build All Flavors

on:
  pull_request:
    branches: [main]

jobs:
  build-android:
    runs-on: ubuntu-latest
    strategy:
      matrix:
        flavor: [dev, staging, prod]
    steps:
      - uses: actions/checkout@v3
      - uses: subosito/flutter-action@v2
        with:
          flutter-version: '3.24.0'
      
      - name: Build APK - ${{ matrix.flavor }}
        run: |
          flutter build apk \
            --flavor ${{ matrix.flavor }} \
            --dart-define-from-file=config/${{ matrix.flavor }}.json \
            -t lib/main_${{ matrix.flavor }}.dart
      
      - name: Upload APK
        uses: actions/upload-artifact@v3
        with:
          name: app-${{ matrix.flavor }}-release
          path: build/app/outputs/flutter-apk/app-${{ matrix.flavor }}-release.apk
```

---

## ✅ 验证清单

完成后确认：

- [ ] 所有 flavor 在本地 Android 可运行
- [ ] 所有 flavor 在本地 iOS 可运行（如支持）
- [ ] config/ 目录已加入 .gitignore（或仅提交示例文件）
- [ ] Android build.gradle 中 flavor 名与命令行一致
- [ ] iOS Schemes 已在 Xcode 中配置
- [ ] dart-define 变量在 Dart 代码中可正常读取
- [ ] 构建产物（APK/IPA）Bundle ID/Application ID 正确

---

## 🤔 FAQ

### Q1: `--flavor` 和 `--dart-define` 的区别？
**A**:
- **`--flavor`**: 原生层（Android/iOS）区分构建产物，影响 Bundle ID、应用名称、签名配置等
- **`--dart-define`**: Dart 层注入编译时常量，用于 API URL、功能开关等业务配置
- **结合使用**: `flutter run --flavor dev --dart-define-from-file=config/dev.json`

### Q2: 必须使用多个 main 文件吗？
**A**:
- **推荐多文件**（`main_dev.dart` 等）：清晰、便于 CI 脚本化、避免运行时判断
- **单文件可行**：在 `main()` 中读取 `String.fromEnvironment('ENV')`，但需确保每次运行都传参

### Q3: 如何管理敏感配置（如 API Key）？
**A**:
- **不要**提交到 Git：config/ 加入 .gitignore
- **本地开发**：手动创建 `config/local.json`
- **CI/CD**：用 GitHub Secrets 注入，构建时动态生成 JSON 文件
  ```yaml
  - name: Create config
    run: echo '${{ secrets.DEV_CONFIG }}' > config/dev.json
  ```

### Q4: Android 和 iOS flavor 名称必须一致吗？
**A**:
- **必须一致**：Flutter CLI 的 `--flavor` 参数同时传给两个平台
- **大小写敏感**：Android `dev {}` ≠ `Dev {}`，iOS Scheme 名称也要完全匹配

### Q5: 现有项目如何迁移？
**A**:
1. 先在新分支测试，避免破坏主分支
2. 保留原 `main.dart`，逐步添加 `main_dev.dart` 等
3. 配置 Android 和 iOS 后，用 `flutter run --flavor dev` 验证
4. 迁移完成后可删除旧的单环境配置

### Q6: CI 上 iOS 构建失败怎么办？
**A**:
- **检查 Scheme 是否共享**：在 Xcode 中勾选 "Shared"，提交 `.xcscheme` 文件
- **Xcode 版本**：确保 CI 环境 Xcode 版本与本地一致
- **手动触发测试**：在 CI 中先运行 `xcodebuild -list` 查看可用 Scheme

---

## 💡 最佳实践

1. **环境变量分层**
   - **编译时**：用 `--dart-define-from-file` 注入（API URL、App 名称）
   - **运行时**：用 .env 文件或远程配置（功能开关、A/B 测试）

2. **Flavor 命名规范**
   - 使用小写：dev、staging、prod（避免大小写混淆）
   - Android 和 iOS 完全一致

3. **Bundle ID 后缀策略**
   - dev/staging 加后缀（`.dev`、`.staging`），可与生产版本共存
   - prod 不加后缀，使用正式 Bundle ID

4. **验证标准前置**
   - 告诉 AI："生成配置后必须通过 `flutter run --flavor dev` 验证"
   - 在 PR 中加入 CI 检查，自动构建所有 flavor

5. **记录常见坑**
   - 在项目 Wiki 或 `.cursor/rules` 中记录 Android/iOS 配置差异
   - 团队新成员快速上手

6. **多环境图标区分**（可选）
   - dev 环境使用带角标的图标（如 "DEV" 水印）
   - 避免误操作生产环境数据

---

## 🔗 相关资源

- 提示词模板：
  - [prompts/flavor/generate-flavor-config.md](../prompts/flavor/generate-flavor-config.md) — 从零生成 Flavor 配置
  - [prompts/flavor/debug-flavor-build-error.md](../prompts/flavor/debug-flavor-build-error.md) — 排查构建错误
- Flutter 官方文档：[https://docs.flutter.dev/deployment/flavors](https://docs.flutter.dev/deployment/flavors)
- Android Product Flavors：[https://developer.android.com/studio/build/build-variants](https://developer.android.com/studio/build/build-variants)
- iOS Multiple Environments：[https://docs.flutter.dev/deployment/ios#create-a-build-archive-with-xcode](https://docs.flutter.dev/deployment/ios#create-a-build-archive-with-xcode)

---

**打造稳定的多环境体系 🚀**
