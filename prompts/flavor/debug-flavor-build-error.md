# 提示词：排查 Flavor 构建错误

**用途**：诊断和修复 Flutter Flavor 配置导致的构建或运行失败

---

## 📋 适用场景

- 运行 `flutter run --flavor xxx` 时报错
- 构建 APK/IPA 失败，提示 Flavor 相关问题
- 多环境配置后，某个环境无法正常运行
- Android 和 iOS 表现不一致

---

## 📥 输入要求

在 Cursor 中准备：

1. **完整错误信息**（终端输出或构建日志）
2. **失败的命令**（如 `flutter run --flavor dev ...`）
3. **相关配置文件**：
   - Android: `android/app/build.gradle`
   - iOS: `ios/Flutter/*.xcconfig` 或 Xcode Scheme
   - Dart: `lib/config/env_config.dart`、`lib/main_dev.dart` 等
   - dart-define 配置：`config/dev.json` 等

---

## 💬 提示词模板

```markdown
我在配置 Flutter Flavor 时遇到错误，请帮我诊断和修复。

## 错误信息

**运行命令**：
```bash
[粘贴失败的完整命令，如：
flutter run --flavor dev --dart-define-from-file=config/dev.json -t lib/main_dev.dart
]
```

**错误输出**：
```
[粘贴完整的错误信息或日志，包括堆栈跟踪]
```

## 当前配置

**平台**：[Android / iOS / 双平台]

**目标环境**：[dev / staging / prod]

**已完成的配置**：
- [✅/❌] 创建了 `lib/main_dev.dart` 等入口文件
- [✅/❌] 修改了 `android/app/build.gradle` 添加 productFlavors
- [✅/❌] 创建了 `ios/Flutter/Dev.xcconfig` 等文件
- [✅/❌] 在 Xcode 中配置了 Scheme
- [✅/❌] 创建了 `config/dev.json` 等配置文件

## 附上相关配置文件内容

### Android build.gradle 的 productFlavors 部分
```gradle
[粘贴 android {} 块中的 flavorDimensions 和 productFlavors 配置]
```

### iOS xcconfig 文件（如有）
```xcconfig
[粘贴 Dev.xcconfig 或其他 xcconfig 内容]
```

### Dart 配置文件
```dart
[粘贴 lib/config/env_config.dart 关键部分]
```

### dart-define 配置文件
```json
[粘贴 config/dev.json 或相关 JSON 内容]
```

## 请帮我

1. **定位问题**：分析错误信息，指出具体是哪个配置环节出错（Android Gradle、iOS Scheme、Dart 层、命令参数等）
2. **提供修复方案**：给出完整的修复代码或配置，可直接替换
3. **验证命令**：提供修复后的完整验证命令
4. **预防措施**：说明如何避免类似错误

如果需要更多信息（如 Xcode 日志、完整 build.gradle），请明确告知。
```

---

## 🔍 常见错误类型与诊断

### 类型 1: Flavor 未定义

**错误信息示例**：
```
Error: The Gradle project does not define a task suitable for the requested build.

The android flavor 'dev' is not defined by the Gradle build at /path/to/android.
```

**AI 诊断提示词**：
```markdown
错误提示 Android flavor 'dev' 未定义。

请检查：
1. `android/app/build.gradle` 中是否正确添加了 `flavorDimensions` 和 `productFlavors { dev {} }`
2. flavor 名称是否完全匹配（大小写敏感）
3. 是否在 `android {}` 块内（不是 `dependencies {}` 或其他块）

生成正确的 build.gradle 配置，包含 dev、staging、prod 三个 flavor。
```

---

### 类型 2: iOS Scheme 不存在

**错误信息示例**：
```
Could not find an option named "dev".

Xcode's output:
↳ xcodebuild: error: The workspace named "Runner" does not contain a scheme named "dev".
```

**AI 诊断提示词**：
```markdown
错误提示 iOS Scheme 'dev' 不存在。

这是因为 Flutter 的 `--flavor` 参数在 iOS 上对应的是 Xcode Scheme 名称，需要人工在 Xcode 中创建。

请提供详细步骤：
1. 如何在 Xcode 中创建 dev Scheme
2. 如何将 Scheme 与 Dev.xcconfig 关联
3. 如何确保 Scheme 已共享（可在 CI 中使用）

并生成可能缺失的 Dev.xcconfig 文件内容。
```

---

### 类型 3: dart-define 变量未生效

**错误信息示例**：
```
Exception: API_BASE_URL is not defined.

或者：App 运行时显示默认值，而不是配置文件中的值
```

**AI 诊断提示词**：
```markdown
错误：dart-define 变量未正确读取，代码中 `String.fromEnvironment('API_BASE_URL')` 返回空或默认值。

可能原因：
1. 未传 `--dart-define-from-file=config/dev.json` 参数
2. JSON 文件路径错误或格式错误
3. JSON 键名与代码中 `fromEnvironment()` 的键名不匹配
4. 使用了 `const` 构造，但变量在运行时才赋值

请检查：
- 我的运行命令：`[粘贴命令]`
- config/dev.json 内容：`[粘贴 JSON]`
- lib/config/env_config.dart 中读取方式：`[粘贴代码]`

给出修复方案。
```

---

### 类型 4: Android 和 iOS flavor 名不一致

**错误信息示例**：
```
Android 可以运行，但 iOS 报错 Scheme not found，或反之
```

**AI 诊断提示词**：
```markdown
问题：Android flavor 可以正常运行，但 iOS 失败（或反之）。

请检查：
1. Android `build.gradle` 中的 flavor 名称：[粘贴 productFlavors 配置]
2. iOS Xcode Scheme 名称：[列出已创建的 Scheme，如 dev、staging、prod]
3. 运行命令中的 `--flavor` 参数：[粘贴命令]

Flutter 要求 Android 和 iOS 的 flavor 名称**完全一致**（包括大小写）。

给出标准化方案，统一为小写 dev/staging/prod。
```

---

### 类型 5: 构建产物 Bundle ID/Application ID 错误

**错误信息示例**：
```
安装时提示包名冲突，或 App Store Connect 拒绝上传
```

**AI 诊断提示词**：
```markdown
问题：构建的 APK/IPA 的 Bundle ID 或 Application ID 不符合预期。

**预期**：
- dev: com.example.myapp.dev
- staging: com.example.myapp.staging
- prod: com.example.myapp

**实际**：[粘贴 `adb shell pm list packages` 或 Xcode 显示的 Bundle ID]

请检查：
- Android `build.gradle` 中的 `applicationIdSuffix` 配置
- iOS `xcconfig` 中的 `PRODUCT_BUNDLE_IDENTIFIER` 配置
- 是否在构建时正确传递了 `--flavor` 参数

给出修复方案。
```

---

### 类型 6: CI/CD 构建失败

**错误信息示例**：
```
GitHub Actions 或其他 CI 环境中构建失败，本地可以通过
```

**AI 诊断提示词**：
```markdown
问题：本地可以成功运行 `flutter run --flavor dev`，但 CI 上构建失败。

**CI 平台**：[GitHub Actions / GitLab CI / Bitrise / 其他]

**CI 日志**：
```
[粘贴 CI 构建失败的日志]
```

可能原因：
1. config/ 目录未提交，CI 环境缺少配置文件
2. iOS Scheme 未共享（.xcscheme 文件未提交）
3. CI 环境 Flutter 或 Xcode 版本不一致
4. 缺少必要的环境变量或密钥

请分析并给出修复方案：
- 如何在 CI 中动态生成 config/dev.json（从 Secrets）
- 如何确保 iOS Scheme 已共享
- 完整的 CI workflow 示例（GitHub Actions）
```

---

## ✅ 验证修复步骤

修复后执行：

```bash
# 1. 清理构建缓存
flutter clean

# 2. 重新获取依赖
flutter pub get

# 3. 验证 Android
cd android
./gradlew clean
./gradlew assembleDev  # 或其他 flavor
cd ..

# 4. 验证 iOS（需 macOS）
cd ios
pod install  # 如果使用 CocoaPods
cd ..

# 5. 完整运行测试
flutter run --flavor dev --dart-define-from-file=config/dev.json -t lib/main_dev.dart

# 6. 构建测试
flutter build apk --flavor dev --dart-define-from-file=config/dev.json -t lib/main_dev.dart
```

**检查清单**：
- [ ] 错误信息已消失
- [ ] App 可以正常启动
- [ ] 环境变量正确注入（在 UI 中显示验证）
- [ ] Android applicationId 正确（用 `adb shell pm list packages` 确认）
- [ ] iOS Bundle ID 正确（在 Xcode 或运行日志中确认）
- [ ] 所有 flavor（dev/staging/prod）均可通过验证

---

## 🛠️ 高级诊断技巧

### 1. Android Gradle 任务列表

查看可用的 Gradle 任务：
```bash
cd android
./gradlew tasks --all | grep assemble
```

应该看到：
```
assembleDevDebug
assembleDevRelease
assembleStagingDebug
...
```

如果没有对应 flavor 的任务，说明 build.gradle 配置未生效。

### 2. iOS Scheme 列表

查看可用的 Xcode Scheme：
```bash
cd ios
xcodebuild -list -workspace Runner.xcworkspace
```

应该看到：
```
Schemes:
    Runner
    dev
    staging
    prod
```

如果缺失，需要在 Xcode 中手动创建。

### 3. 验证 dart-define 注入

在 `lib/main_dev.dart` 中临时添加打印：
```dart
void main() {
  const apiUrl = String.fromEnvironment('API_BASE_URL', defaultValue: 'NOT_SET');
  print('🔍 API_BASE_URL: $apiUrl');
  
  EnvConfig.initialize(Environment.dev);
  runApp(const MyApp());
}
```

运行后检查日志，确认变量已注入。

### 4. Android applicationId 检查

构建后查看生成的 APK 信息：
```bash
aapt dump badging build/app/outputs/flutter-apk/app-dev-release.apk | grep package:
```

应显示：`package: name='com.example.myapp.dev'`

### 5. iOS Bundle ID 检查

在 Xcode 中：
1. 打开 `ios/Runner.xcworkspace`
2. 选择对应 Scheme（dev）
3. 查看 General → Identity → Bundle Identifier

或通过命令行：
```bash
/usr/libexec/PlistBuddy -c "Print :CFBundleIdentifier" ios/Runner/Info.plist
```

---

## 💡 预防错误的最佳实践

1. **统一命名规范**
   - 所有 flavor 名称使用小写（dev、staging、prod）
   - Android 和 iOS 完全一致

2. **版本控制**
   - 提交 config/ 示例文件（如 `config/dev.example.json`）
   - 提交 iOS .xcscheme 文件（勾选 Shared）
   - 不提交敏感配置（config/*.json 加入 .gitignore）

3. **本地测试全覆盖**
   - 每次修改配置后，测试所有 flavor 的 run 和 build
   - 使用脚本自动化验证：
     ```bash
     for flavor in dev staging prod; do
       flutter build apk --flavor $flavor --dart-define-from-file=config/$flavor.json -t lib/main_$flavor.dart
     done
     ```

4. **CI 集成验证**
   - PR 检查自动构建所有 flavor
   - 失败时明确报告是哪个 flavor 和平台

5. **文档化配置**
   - 在项目 README 或 Wiki 中记录 Flavor 配置步骤
   - 新成员加入时快速上手

---

## 🔗 相关

- 工作流：[workflows/flavor-multi-env.md](../../workflows/flavor-multi-env.md)
- 生成配置：[generate-flavor-config.md](generate-flavor-config.md)
- Flutter 官方 Troubleshooting：[https://docs.flutter.dev/testing/errors](https://docs.flutter.dev/testing/errors)
