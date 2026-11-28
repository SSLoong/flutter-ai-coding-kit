# 提示词：迁移应用或插件到 SwiftPM

**用途**：生成移除 CocoaPods 的命令清单（应用）或给插件补 SwiftPM 支持的步骤（插件作者）

---

## 📋 适用场景

### 应用开发者
- 全部插件已支持 SwiftPM,准备移除 CocoaPods
- 重新启用 SwiftPM 并清理 CocoaPods 文件
- 手工处理 Podfile 中的非 Flutter 依赖

### 插件作者
- 给现有插件补 SwiftPM 支持,同时保留 CocoaPods
- 迁移代码目录结构,生成 `Package.swift`
- 验证插件同时支持两种方式

---

## 📥 输入要求

### 应用开发者

在 Cursor 中准备：

1. **审计结果**：先运行 [audit-spm-and-cocoapods-fallback.md](audit-spm-and-cocoapods-fallback.md),确认全部插件支持 SwiftPM
2. **iOS/macOS 目录**：确认 `ios/` 和 `macos/`（如果有）目录存在
3. **CocoaPods 文件**：确认 `ios/Podfile`、`ios/Pods/` 等存在

### 插件作者

在 Cursor 中准备：

1. **插件目录结构**：打开插件项目,确认 `ios/Classes/`、`ios/Assets/` 等
2. **插件配置**：打开 `pubspec.yaml` 和 `ios/plugin_name.podspec`
3. **官方文档**：参考 [Swift Package Manager for plugin authors](https://docs.flutter.dev/packages-and-plugins/swift-package-manager/for-plugin-authors)

---

## 💬 提示词模板

### 应用开发者：移除 CocoaPods

```markdown
我的项目全部插件已支持 SwiftPM,想移除 CocoaPods。请生成命令清单。

**当前状态**：
- 项目路径：[项目根目录]
- iOS 目录：[是否存在 `ios/`]
- macOS 目录：[是否存在 `macos/`]
- Podfile 中有手工依赖：[是 / 否],依赖列表：[列出手工添加的 pod 行]

**请执行**：

1. **生成移除 CocoaPods 命令**（iOS 和 macOS 分别列出）：
   ```bash
   # iOS
   cd ios
   pod deintegrate
   cd ..
   rm -rf ios/Podfile ios/Podfile.lock ios/Pods ios/.symlinks
   
   # 检查 xcconfig
   # 列出需要检查的文件：ios/Flutter/Debug.xcconfig、ios/Flutter/Release.xcconfig
   # 提醒：删除类似 #include "Pods/Target Support Files/..." 的行
   
   # 清理并验证
   flutter clean
   flutter pub get
   flutter run
   ```

2. **处理手工 Podfile 依赖**（如果有）：
   - 读取 `ios/Podfile` 中非 Flutter 生成的 `pod` 行
   - 对每个依赖,查其是否有 SwiftPM 版本
   - 生成在 Xcode 中用 SPM 添加的步骤：
     1. 打开 Xcode,导航到 Package Dependencies
     2. 点击 + 按钮
     3. 搜索包名或输入 GitHub URL
     4. 添加到 `Runner` target

3. **验证清单**：
   - [ ] `flutter clean && flutter pub get` 成功
   - [ ] `flutter build ios --debug --no-codesign` 或 `flutter build macos --debug` 通过
   - [ ] Xcode 中 Package Dependencies 显示 `FlutterGeneratedPluginSwiftPackage`
   - [ ] Pre-action `Run Prepare Flutter Framework Script` 存在
   - [ ] 真机/模拟器运行正常

**约束**：
- 只生成命令,不要自动执行
- 提醒在删除前提交代码,保留回退路径
- 如果 Podfile 有手工依赖,明确标注需人工处理
```

---

### 插件作者：补 SwiftPM 支持

```markdown
我是插件作者,想给现有插件补 SwiftPM 支持,同时保留 CocoaPods。请生成步骤。

**当前状态**：
- 插件名：[plugin_name]
- 插件类型：[Swift / Objective-C / 混合]
- 目录结构：[列出 ios/Classes/ 下的文件,如有 ios/Assets/ 也列出]
- 是否有 PrivacyInfo.xcprivacy：[是 / 否]
- 是否使用 Pigeon：[是 / 否]
- 支持平台：[iOS / macOS / Darwin（通用）]

**请执行**：

1. **生成目录结构**：
   - 在 `ios/`（或 `macos/`、`darwin/`）下创建 `plugin_name/` 目录
   - 创建 `Package.swift`、`Sources/plugin_name/` 目录
   - Swift 插件：`Sources/plugin_name/`
   - Objective-C 插件：额外创建 `Sources/plugin_name/include/plugin_name/.gitkeep`

2. **生成 `Package.swift` 模板**：
   - 根据插件类型（Swift / Objective-C）选择模板
   - 设置支持的平台（iOS 13.0+、macOS 10.15+）
   - 添加 `FlutterFramework` 依赖
   - 配置资源文件（如 PrivacyInfo.xcprivacy）
   - Objective-C 插件：添加 `cSettings: [.headerSearchPath("include/plugin_name")]`

3. **文件迁移清单**：
   - 列出需要移动的文件：
     - `ios/Classes/` → `ios/plugin_name/Sources/plugin_name/`
     - `ios/Assets/` → `ios/plugin_name/Sources/plugin_name/`（如有）
     - Objective-C 插件：公开头文件 → `ios/plugin_name/Sources/plugin_name/include/plugin_name/`
   - 生成移动命令：
     ```bash
     # 示例
     mv ios/Classes/* ios/plugin_name/Sources/plugin_name/
     # （实际根据文件类型细化）
     ```

4. **更新 `.podspec`**：
   - 更新 `s.source_files` 和 `s.public_header_files`（如有）指向新路径
   - 示例：
     ```ruby
     # Swift 插件
     s.source_files = 'plugin_name/Sources/plugin_name/**/*.swift'
     
     # Objective-C 插件
     s.source_files = 'plugin_name/Sources/plugin_name/**/*.{h,m}'
     s.public_header_files = 'plugin_name/Sources/plugin_name/include/**/*.h'
     ```

5. **更新 `pubspec.yaml`**：
   - 设置最低版本：
     ```yaml
     environment:
       sdk: ^3.11.0
       flutter: ">=3.41.0"
     ```

6. **更新 Pigeon 配置**（如果使用）：
   - Swift 插件：
     ```dart
     swiftOut: 'ios/plugin_name/Sources/plugin_name/messages.g.swift',
     ```
   - Objective-C 插件：
     ```dart
     objcHeaderOut: 'ios/plugin_name/Sources/plugin_name/include/plugin_name/messages.g.h',
     objcSourceOut: 'ios/plugin_name/Sources/plugin_name/messages.g.m',
     ```

7. **验证步骤**：
   ```bash
   # 1. 验证 CocoaPods
   flutter config --no-enable-swift-package-manager
   cd example
   flutter run
   cd ..
   pod lib lint ios/plugin_name.podspec --configuration=Debug --skip-tests --use-modular-headers
   
   # 2. 验证 SwiftPM
   flutter config --enable-swift-package-manager
   cd example
   flutter clean
   flutter run
   # 在 Xcode 中打开 example app,确认 Package Dependencies 显示
   
   # 3. 运行测试
   flutter test
   ```

**约束**：
- 只生成步骤和模板,需要人工审查后执行
- 不要移除 CocoaPods 支持
- 生成的 `Package.swift` 需在 Xcode 中验证无警告/错误
- 提醒在 `CHANGELOG.md` 中注明已支持 SwiftPM
```

---

## ✅ 验证步骤

### 应用开发者

移除 CocoaPods 后确认：

```bash
# 1. 命令执行成功
pod deintegrate  # 成功
rm -rf ...       # 文件已删除

# 2. 构建验证
flutter clean
flutter pub get
flutter build ios --debug --no-codesign  # 或 flutter build macos --debug

# 3. Xcode 验证
# 打开 Runner.xcworkspace
# 确认 Package Dependencies 中有 FlutterGeneratedPluginSwiftPackage
# 确认 Pre-action "Run Prepare Flutter Framework Script" 存在
# 运行应用,无报错
```

**检查清单**：

- [ ] `pod deintegrate` 成功
- [ ] CocoaPods 文件已删除
- [ ] xcconfig 中无 Pods 引用
- [ ] `flutter clean && flutter pub get` 成功
- [ ] 构建命令通过
- [ ] Xcode 配置正确
- [ ] 真机/模拟器运行正常

---

### 插件作者

补 SwiftPM 支持后确认：

```bash
# AI 应生成：
# 1. 目录结构创建命令
# 2. Package.swift 模板（需人工审查）
# 3. 文件迁移命令
# 4. .podspec 更新建议
# 5. 验证命令清单
```

**检查清单**：

- [ ] 目录结构已创建
- [ ] `Package.swift` 在 Xcode 中无警告/错误
- [ ] 文件已移至新路径
- [ ] `.podspec` 路径已更新
- [ ] `pubspec.yaml` 版本约束已更新
- [ ] CocoaPods 模式下 example app 运行正常
- [ ] SwiftPM 模式下 example app 运行正常
- [ ] `pod lib lint` 通过
- [ ] 测试通过

---

## 🛠️ 常见问题

### 应用开发者

1. **移除 CocoaPods 后构建失败,报 "No such module"**
   - 某插件可能未完全支持 SwiftPM。重新审计插件,或回退：恢复 `Podfile`,关闭 SwiftPM。

2. **Xcode 中 Package Dependencies 不显示**
   - 运行 `flutter clean && flutter pub get && flutter run`
   - 在 Xcode 中 File > Packages > Reset Package Caches

3. **手工 Podfile 依赖找不到 SwiftPM 版本**
   - 联系库维护者或在 GitHub 搜索第三方 SwiftPM fork
   - 临时保留该依赖的 CocoaPods 配置（部分 CocoaPods + 部分 SwiftPM 可共存）

---

### 插件作者

1. **AI 生成的 `Package.swift` 在 Xcode 中报错**
   - 在 Xcode 中打开 `ios/plugin_name/` 目录
   - 查看具体错误,手工修正（如平台版本、依赖路径）

2. **CocoaPods 验证失败,报找不到文件**
   - 检查 `.podspec` 中的 `s.source_files` 和 `s.public_header_files` 路径是否正确
   - 确认文件已移至新路径

3. **SwiftPM 验证失败,example app 报 "Module not found"**
   - 在 Xcode 中 File > Packages > Reset Package Caches
   - 确认 `Package.swift` 中 `name` 和 `library` 名称正确（`_` 需改成 `-`）

4. **插件有 modulemap,如何处理？**
   - 参考官方文档 [Handling modulemap](https://docs.flutter.dev/packages-and-plugins/swift-package-manager/for-plugin-authors#handling-modulemap)
   - 可在 `Package.swift` 中用 `exclude` 排除 modulemap,仅用于 CocoaPods

---

## 🔗 相关

- 工作流：[workflows/swift-package-manager-migration.md](../../workflows/swift-package-manager-migration.md)
- 审计 SwiftPM 状态：[audit-spm-and-cocoapods-fallback.md](audit-spm-and-cocoapods-fallback.md)
- 官方应用文档：[https://docs.flutter.dev/packages-and-plugins/swift-package-manager/for-app-developers](https://docs.flutter.dev/packages-and-plugins/swift-package-manager/for-app-developers)
- 官方插件文档：[https://docs.flutter.dev/packages-and-plugins/swift-package-manager/for-plugin-authors](https://docs.flutter.dev/packages-and-plugins/swift-package-manager/for-plugin-authors)
