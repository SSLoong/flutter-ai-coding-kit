# 提示词：生成 Flavor 多环境配置

**用途**：为现有单环境 Flutter 项目生成完整的 Flavor（dev/staging/prod）配置

---

## 📋 适用场景

- 项目从未配置过 Flavor，需要从零开始
- 已有部分环境变量，需规范化为 Flavor 体系
- 需要同时配置 Android 和 iOS 双平台

---

## 📥 输入要求

在 Cursor 中准备：

1. **项目根目录**（确保包含 `android/` 和 `ios/` 目录）
2. **现有 `lib/main.dart`**（作为共享逻辑参考）
3. **环境变量清单**（如 API baseUrl、应用名称、Bundle ID 等）

---

## 💬 提示词模板

```markdown
我需要为 Flutter 项目配置 Flavor 多环境支持，从单环境项目改造为 dev/staging/prod 三环境。

## 目标环境

1. **dev**（开发环境）
   - API baseUrl: `https://dev-api.example.com`
   - 应用显示名称: `MyApp Dev`
   - Android applicationId: `com.example.myapp.dev`
   - iOS Bundle ID: `com.example.myapp.dev`
   - 启用调试工具: `true`

2. **staging**（预发布环境）
   - API baseUrl: `https://staging-api.example.com`
   - 应用显示名称: `MyApp Staging`
   - Android applicationId: `com.example.myapp.staging`
   - iOS Bundle ID: `com.example.myapp.staging`
   - 启用调试工具: `false`

3. **prod**（生产环境）
   - API baseUrl: `https://api.example.com`
   - 应用显示名称: `MyApp`
   - Android applicationId: `com.example.myapp`
   - iOS Bundle ID: `com.example.myapp`
   - 启用调试工具: `false`

## 任务清单

请生成以下完整配置：

### 1. Dart 层配置

- **`lib/config/env_config.dart`**：环境配置类
  - 定义 `Environment` 枚举（dev/staging/prod）
  - 使用 `String.fromEnvironment()` 读取 dart-define 变量
  - 提供单例访问接口 `EnvConfig.instance`
  - 包含字段：apiBaseUrl、appName、enableDebugTools

- **入口文件**：`lib/main_dev.dart`、`lib/main_staging.dart`、`lib/main_prod.dart`
  - 每个文件初始化对应环境
  - 调用共享的 `runApp(MyApp())`

- **共享 App**：`lib/app.dart`
  - 从 `EnvConfig.instance` 读取配置
  - 在 MaterialApp title 中显示环境名称

### 2. Android 配置

修改 `android/app/build.gradle`：
- 添加 `flavorDimensions "environment"`
- 定义三个 productFlavors：dev、staging、prod
- 每个 flavor 设置：
  - `applicationIdSuffix`（dev: ".dev", staging: ".staging", prod: ""）
  - `resValue "string", "app_name"`（对应环境显示名称）

### 3. iOS 配置

生成三个 xcconfig 文件：
- `ios/Flutter/Dev.xcconfig`
- `ios/Flutter/Staging.xcconfig`
- `ios/Flutter/Prod.xcconfig`

每个文件设置：
- `PRODUCT_BUNDLE_IDENTIFIER`（对应环境 Bundle ID）
- `PRODUCT_NAME`（对应环境显示名称）
- `#include "Release.xcconfig"`（继承默认配置）

### 4. dart-define 配置文件

在项目根目录创建 `config/` 目录（不提交到 Git），生成三个 JSON 文件：
- `config/dev.json`
- `config/staging.json`
- `config/prod.json`

每个文件包含：
```json
{
  "API_BASE_URL": "对应环境 URL",
  "APP_NAME": "对应环境名称",
  "ENABLE_DEBUG_TOOLS": "true 或 false"
}
```

同时生成 `config/.gitignore` 忽略敏感配置文件。

### 5. 验证命令

提供可直接运行的命令：
```bash
# Android Dev
flutter run --flavor dev --dart-define-from-file=config/dev.json -t lib/main_dev.dart

# Android Prod
flutter build apk --flavor prod --dart-define-from-file=config/prod.json -t lib/main_prod.dart

# iOS Dev（需 macOS）
flutter run --flavor dev --dart-define-from-file=config/dev.json -t lib/main_dev.dart
```

## 要求

1. **完整性**：生成所有文件的完整内容，包含所有必要的 import
2. **可运行**：生成的代码必须可以直接使用，不要有占位符或伪代码
3. **注释说明**：在关键配置处添加注释，说明作用和注意事项
4. **错误处理**：提供默认值，避免未传 dart-define 参数时崩溃
5. **平台一致性**：确保 Android 和 iOS 的 flavor 名称完全一致（小写）

## 不要做

- ❌ 不要在代码中硬编码 API Key 或敏感信息
- ❌ 不要省略 import 语句
- ❌ 不要使用 `// ... 其他代码` 等占位符
- ❌ 不要混用大小写 flavor 名称（统一小写）

请按顺序生成所有文件内容。
```

---

## ✅ 验证步骤

生成后执行：

```bash
# 1. 创建 config 目录和配置文件
mkdir -p config
# （复制 AI 生成的 JSON 内容到 config/dev.json 等）

# 2. 修改 Android 配置
# （复制 AI 生成的 gradle 配置到 android/app/build.gradle）

# 3. 创建 iOS xcconfig 文件
# （复制 AI 生成的内容到 ios/Flutter/Dev.xcconfig 等）

# 4. 运行验证
flutter run --flavor dev --dart-define-from-file=config/dev.json -t lib/main_dev.dart

# 5. 构建验证
flutter build apk --flavor dev --dart-define-from-file=config/dev.json -t lib/main_dev.dart
```

**检查清单**：
- [ ] 所有入口文件（main_dev.dart 等）可正常运行
- [ ] App 显示名称与环境匹配
- [ ] API baseUrl 在 UI 中正确显示
- [ ] Android 构建成功，applicationId 正确
- [ ] iOS 构建成功（需 macOS），Bundle ID 正确
- [ ] config/ 目录已加入 .gitignore

---

## 🛠️ 常见调整

生成后可能需要手动微调：

### 1. iOS Scheme 配置

AI 生成了 xcconfig 文件，但 Xcode Scheme 需要人工创建：

1. 打开 `ios/Runner.xcworkspace`
2. Product → Scheme → Manage Schemes
3. 复制 Runner 为 dev、staging、prod
4. 对每个 Scheme 编辑：
   - Build Configuration 选择 Release
   - Run → Info → Build Configuration 选择 Release
5. 在 Project → Info → Configurations 中关联对应 xcconfig

### 2. 环境变量扩展

如需添加更多配置项（如 enableAnalytics、maxCacheSize），在以下位置同步修改：
- `lib/config/env_config.dart` 添加字段
- `config/dev.json` 等文件添加键值
- 使用处读取 `EnvConfig.instance.xxx`

### 3. 原生代码读取配置

如果 Android/iOS 原生代码需要读取配置：

**Android** (Kotlin/Java):
```kotlin
val apiUrl = BuildConfig.API_BASE_URL  // 需在 gradle 中配置 buildConfigField
```

**iOS** (Swift):
```swift
let apiUrl = Bundle.main.infoDictionary?["API_BASE_URL"] as? String
```

### 4. 图标区分

为 dev/staging 环境使用不同图标：
- Android: 在 `android/app/src/dev/res/` 放置独立 `mipmap-*/ic_launcher.png`
- iOS: 创建独立 Asset Catalog 或在 xcconfig 中设置 `ASSETCATALOG_COMPILER_APPICON_NAME`

---

## 🔗 相关

- 工作流：[workflows/flavor-multi-env.md](../../workflows/flavor-multi-env.md)
- 排查错误：[debug-flavor-build-error.md](debug-flavor-build-error.md)
- Flutter 官方文档：[Flavors](https://docs.flutter.dev/deployment/flavors)
