# Swift Package Manager 迁移：从 CocoaPods 到 SwiftPM

**Flutter iOS/macOS 原生依赖管理从 CocoaPods 迁移到 Swift Package Manager**

> 作者：Loong (SSLoong) | 日期：2026-11-28

本文对齐 Flutter 3.44+ 引入的 Swift Package Manager 默认启用机制,以及官方文档 [Swift Package Manager for app developers](https://docs.flutter.dev/packages-and-plugins/swift-package-manager/for-app-developers)、[Swift Package Manager for plugin authors](https://docs.flutter.dev/packages-and-plugins/swift-package-manager/for-plugin-authors) 和博客文章 [What's new in Flutter 3.50](https://flutter.dev/blog/whats-new-in-flutter-3-50)。

**版本要求**：
- Flutter 3.44+（已默认启用 SwiftPM）
- Flutter 3.50+ 推荐（关闭 SwiftPM 会在每个 CLI 命令报警）

**核心认知**：CocoaPods registry 于 **2026-12-02** 永久变为只读,Flutter 3.50 起禁用 SwiftPM 会在每个命令报警,未来版本会移除 opt-out 并最终去掉 CocoaPods fallback。这是技术负责人现在就必须做的迁移决策。

---

## 📋 背景

CocoaPods 在 Flutter iOS/macOS 插件生态中服务多年,但随着 Apple 生态演进与 Swift Package Manager 成熟,官方依赖管理路径正在切换：

1. **Flutter 3.44（2026-08）**：SwiftPM 默认启用,升级后运行应用会自动添加 SwiftPM 集成
2. **2026-12-02**：CocoaPods trunk registry 永久变为只读,不再接受新版本提交
3. **Flutter 3.50（2026-11）**：关闭 SwiftPM 会在每个 Flutter CLI 命令报警
4. **未来版本**：移除禁用 SwiftPM 的选项,最终去掉 CocoaPods fallback

Flutter 仍会对尚不支持 SwiftPM 的插件自动 fallback 到 CocoaPods,但这只是过渡期权宜之计。**必须先确认全部插件支持 SwiftPM,才能安全移除 CocoaPods。**

AI 在此流程中适合：
- 审计项目依赖,列出哪些插件支持 SwiftPM、哪些仍 fallback 到 CocoaPods
- 解读迁移失败报错,建议替代插件或等待更新
- 为插件起草 `Package.swift`、目录结构变更（需人工审查）
- 生成移除 CocoaPods 的命令清单
- 验证 Xcode 配置（pre-action、target dependency）
- 协助排查 Flavor / 多 scheme 项目的迁移问题

AI **不适合**：
- 在未确认全部插件支持前删除 `Podfile`
- 盲目修改 `ios/Runner.xcodeproj/project.pbxproj` 或 `.xcscheme`（官方无手工步骤）
- 跳过真机/模拟器构建验证
- 把生产签名或证书交给自动化工具改动

---

## 🎯 目标

- ✅ 判断项目是否已启用 SwiftPM,是否需要重新启用
- ✅ 审计依赖列表,确认哪些插件仍 fallback 到 CocoaPods
- ✅ 在全部插件支持 SwiftPM 后,安全移除 CocoaPods
- ✅ 插件作者：为现有插件补 SwiftPM 支持,同时保留 CocoaPods（过渡期）
- ✅ 验证迁移后应用在 iOS/macOS 真机与模拟器上构建与运行正常

---

## 🔧 适用场景

| 场景 | 适合度 | 说明 |
|------|--------|------|
| Flutter 3.44+ 新项目 | ⭐⭐⭐⭐⭐ | 已默认启用 SwiftPM,不需要额外操作 |
| 升级到 3.44+ 的现有项目 | ⭐⭐⭐⭐⭐ | 运行应用自动添加 SwiftPM 集成 |
| 曾经禁用 SwiftPM 的项目 | ⭐⭐⭐⭐⭐ | 重新启用后可移除 CocoaPods |
| 全部插件已支持 SwiftPM | ⭐⭐⭐⭐⭐ | 可安全移除 CocoaPods |
| 部分插件尚不支持 SwiftPM | ⭐⭐⭐ | Flutter 会 fallback,等插件更新 |
| 手工改过 Podfile 的项目 | ⭐⭐⭐ | 需人工处理非 Flutter 插件依赖 |
| Flavor / 多 scheme 项目 | ⭐⭐⭐⭐ | 自动迁移可能失败,按官方指引报 issue |
| add-to-app 项目 | ⭐⭐⭐⭐ | 参考官方 add-to-app 文档单独处理 |

**怎么选：**

- **应用开发者（本文主路径）**：审计插件支持情况 → 在全部就绪后移除 CocoaPods
- **插件作者**：给现有插件补 SwiftPM 支持,同时保留 CocoaPods（直到另行通知）
- **add-to-app / 自定义 Xcode target**：参考官方文档单独处理,不在本文范围

---

## 🚀 完整流程

### 应用开发者路径

#### Step 1: 检查项目是否已启用 SwiftPM

**人工操作**：

```bash
cd /path/to/your_flutter_app
flutter --version
# 确认 Flutter 3.44+ 或更高
```

如果 Flutter ≥ 3.44,升级后第一次 `flutter run` 会自动添加 SwiftPM 集成。

**如何确认已启用**：

1. **命令行**：项目运行后 Flutter 会下载 Swift packages（插件依赖）
2. **Xcode**：
   - 打开 `ios/Runner.xcworkspace` 或 `macos/Runner.xcworkspace`
   - 运行应用
   - 确认 `Run Prepare Flutter Framework Script` 作为 pre-action 执行
   - 确认 `FlutterGeneratedPluginSwiftPackage` 是 target dependency

如果曾经禁用过 SwiftPM,需要重新启用：

```bash
flutter config --enable-swift-package-manager
```

**AI 提示词**（可复制）：

```markdown
我想确认项目是否已启用 Swift Package Manager。请：

1. 读取 `pubspec.yaml`,查看 `flutter.config.enable-swift-package-manager` 字段（如果有）
2. 我的 Flutter 版本是 `[粘贴 flutter --version 输出]`
3. 如果 Flutter ≥ 3.44 且没有禁用,已默认启用；如果曾经禁用,生成重新启用命令
4. 生成验证命令：在 Xcode 中检查 pre-action 和 target dependency
```

**检查点**：
- [ ] Flutter ≥ 3.44
- [ ] `flutter config --enable-swift-package-manager` 已执行（如果曾禁用）
- [ ] `flutter run` 成功运行

---

#### Step 2: 审计依赖：哪些插件仍 fallback 到 CocoaPods

Flutter 仍会对不支持 SwiftPM 的插件回退到 CocoaPods。**必须先确认全部插件支持 SwiftPM,才能移除 CocoaPods。**

**人工操作**：

```bash
cd ios
# 如果 Pods 目录存在,说明有插件仍走 CocoaPods fallback
ls -la Pods/

# 查看 Podfile.lock,列出当前 CocoaPods 管理的插件
cat Podfile.lock
```

**AI 提示词**（可复制）：见 [prompts/spm/audit-spm-and-cocoapods-fallback.md](../prompts/spm/audit-spm-and-cocoapods-fallback.md)。

**审查要点**：

1. **全部插件已支持 SwiftPM**：`ios/Pods` 不存在或为空,可以移除 CocoaPods
2. **部分插件仍 fallback**：等待插件更新或寻找替代插件
3. **手工添加的 CocoaPods 依赖**：`Podfile` 中有非 Flutter 插件的 `pod` 行,需人工处理

---

#### Step 3: 在全部插件支持后,移除 CocoaPods

**仅当 Step 2 确认全部插件支持 SwiftPM 时执行。**

**iOS 路径**：

```bash
cd ios
pod deintegrate
cd ..

# 删除 CocoaPods 文件
rm -rf ios/Podfile ios/Podfile.lock ios/Pods ios/.symlinks

# 检查并清理 xcconfig 中对 Pods 的引用
# 打开 ios/Flutter/Debug.xcconfig 和 ios/Flutter/Release.xcconfig
# 删除类似 #include "Pods/Target Support Files/..." 的行（如果有）

# 清理并验证
flutter clean
flutter pub get
flutter run
```

**macOS 路径**：

```bash
cd macos
pod deintegrate
cd ..

# 删除 CocoaPods 文件
rm -rf macos/Podfile macos/Podfile.lock macos/Pods macos/.symlinks

# 检查并清理 xcconfig 中对 Pods 的引用
# 打开 macos/Flutter/Flutter-Debug.xcconfig 和 macos/Flutter/Flutter-Release.xcconfig
# 删除类似 #include "Pods/Target Support Files/..." 的行（如果有）

# 清理并验证
flutter clean
flutter pub get
flutter run
```

**AI 提示词**（可复制）：见 [prompts/spm/migrate-app-or-plugin-to-spm.md](../prompts/spm/migrate-app-or-plugin-to-spm.md)。

**检查点**：
- [ ] `pod deintegrate` 成功
- [ ] CocoaPods 文件已删除
- [ ] `ios/Flutter/*.xcconfig` 和 `macos/Flutter/*.xcconfig` 中无 Pods 引用
- [ ] `flutter clean && flutter pub get` 成功
- [ ] `flutter run` 在真机/模拟器上运行正常

---

#### Step 4: 手工处理 Podfile 中的非 Flutter 依赖

如果 `Podfile` 中有手工添加的非 Flutter 插件依赖,不能只靠自动清理。

**人工决策**：

1. **Firebase、Google Analytics 等 SDK**：检查是否有 Swift Package 版本,在 Xcode 中用 SPM 重新添加
2. **自定义原生库**：联系维护者或自行迁移到 SwiftPM
3. **仅用于特定 Flavor 的依赖**：按 Flavor 逐个处理

**AI 提示词**（可复制）：

```markdown
我的 `ios/Podfile` 中有以下手工添加的依赖：

[粘贴 Podfile 中非 Flutter 插件的 pod 行]

请：
1. 在 pub.dev 或 GitHub 上查这些库是否有 SwiftPM 支持
2. 如果有,生成在 Xcode 中用 SPM 添加的步骤
3. 如果没有,建议替代方案或等待迁移建议
```

---

#### Step 5: Flavor / 多 scheme 项目迁移

Flavor 或多 scheme 项目的自动迁移可能改 `ios/Runner.xcodeproj/project.pbxproj` 和对应 `.xcscheme` 文件。

**自动迁移失败时**：

官方文档要求：收集以下文件报 issue,**不要自行改 `pbxproj` 或 `.xcscheme`**：

- `ios/Runner.xcodeproj/project.pbxproj`
- `ios/Runner.xcodeproj/xcshareddata/xcschemes/Runner.xcscheme`（或 Flavor 对应的 scheme）

**AI 提示词**（可复制）：

```markdown
我的项目有多个 Flavor / scheme,SwiftPM 自动迁移失败,报错：

[粘贴完整报错]

请：
1. 确认报错是否与 Flavor / scheme 相关
2. 列出需要收集的文件（project.pbxproj 和 .xcscheme）
3. 生成报 issue 的模板（包含报错、文件列表）
4. 不要尝试手工改 pbxproj,等官方指引
```

---

#### Step 6: 验证迁移（必须）

**人工操作**：

```bash
# 1. 清理构建缓存
flutter clean
flutter pub get

# 2. 命令行构建（iOS）
flutter build ios --debug --no-codesign
# 或 macOS
flutter build macos --debug

# 3. Xcode 构建
# 打开 ios/Runner.xcworkspace 或 macos/Runner.xcworkspace
# 确认 Xcode 左侧 Project Navigator 显示 Package Dependencies
# 运行应用,确认无报错
```

**验证清单**：

在 Xcode 中确认：

1. **Package Dependencies**：`FlutterGeneratedPluginSwiftPackage` 显示在项目左侧
2. **Pre-action**：Product > Scheme > Edit Scheme > Build > Pre-actions,确认 `Run Prepare Flutter Framework Script` 存在并执行
3. **Target dependency**：`Runner` target 的依赖列表中有 `FlutterGeneratedPluginSwiftPackage`
4. **真机/模拟器运行**：在 iOS/macOS 真机与模拟器上运行正常

**AI 提示词**（可复制）：

```markdown
SwiftPM 迁移后,`[flutter build ios / flutter run / Xcode 构建]` 报以下错误：

[粘贴完整报错]

请：
1. 判断是插件不兼容、Xcode 配置缺失还是版本约束问题
2. 给出修复方案（更新插件、手工添加 SwiftPM 集成、或回退）
3. 如果是已知问题,引用官方文档或 issue
```

**检查点**：
- [ ] `flutter clean && flutter pub get` 成功
- [ ] `flutter build ios --debug --no-codesign` 或 `flutter build macos --debug` 通过
- [ ] Xcode 中 Package Dependencies 正常显示
- [ ] Pre-action 和 target dependency 正确配置
- [ ] 真机/模拟器运行无报错

---

### 插件作者路径

#### 插件作者核心原则

**直到另行通知前,插件应同时支持 SwiftPM 与 CocoaPods。** 不要让插件只剩 SwiftPM,会导致未迁移应用无法使用。

#### Step 1: 给现有插件补 SwiftPM 支持

官方文档提供了详细步骤：[Swift Package Manager for plugin authors](https://docs.flutter.dev/packages-and-plugins/swift-package-manager/for-plugin-authors)

**核心步骤摘要**（以官方为准）：

1. 在 `ios/`、`macos/` 或 `darwin/` 目录下创建 `plugin_name/` 目录
2. 创建 `Package.swift`、`Sources/plugin_name/` 目录结构
3. 移动代码：
   - `ios/Classes/` → `ios/plugin_name/Sources/plugin_name/`
   - `ios/Assets/` → `ios/plugin_name/Sources/plugin_name/`（如有）
4. 更新 `Package.swift`：
   - 设置支持的平台（iOS、macOS）
   - 添加 `FlutterFramework` 依赖
   - 配置资源文件（如 `PrivacyInfo.xcprivacy`）
5. 更新 `pubspec.yaml`：设置最低版本 `flutter: ">=3.41.0"`
6. 更新 `.podspec`：指向新路径
7. 验证：
   - 关闭 SwiftPM,验证 CocoaPods 仍可用
   - 启用 SwiftPM,验证插件在 example app 中运行正常

**AI 提示词**（可复制）：见 [prompts/spm/migrate-app-or-plugin-to-spm.md](../prompts/spm/migrate-app-or-plugin-to-spm.md)（插件作者部分）。

**AI 适合做**：
- 生成 `Package.swift` 模板（需人工审查）
- 列出需要移动的文件
- 更新 `.podspec` 路径
- 生成验证命令

**AI 不适合做**：
- 直接改 `.podspec` 而不保留 CocoaPods 支持
- 在未验证前发布插件新版本

---

#### Step 2: 验证插件同时支持两种方式

**人工操作**：

```bash
cd path/to/plugin

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

**检查点**：
- [ ] CocoaPods 模式下 example app 运行正常
- [ ] SwiftPM 模式下 example app 运行正常
- [ ] `pod lib lint` 通过
- [ ] 插件测试通过

---

## ✅ 验证清单

### 应用开发者

完成迁移后确认：

- [ ] Flutter ≥ 3.44
- [ ] `flutter config --enable-swift-package-manager` 已执行（如果曾禁用）
- [ ] 全部插件支持 SwiftPM（`ios/Pods` 不存在或为空）
- [ ] `pod deintegrate` 成功（iOS 和 macOS）
- [ ] CocoaPods 文件已删除（Podfile、Podfile.lock、Pods、.symlinks）
- [ ] xcconfig 中无 Pods 引用
- [ ] `flutter clean && flutter pub get` 成功
- [ ] `flutter build ios --debug --no-codesign` 或 `flutter build macos --debug` 通过
- [ ] Xcode 中 Package Dependencies 显示 `FlutterGeneratedPluginSwiftPackage`
- [ ] Pre-action `Run Prepare Flutter Framework Script` 存在
- [ ] `Runner` target 依赖列表中有 `FlutterGeneratedPluginSwiftPackage`
- [ ] iOS/macOS 真机与模拟器运行正常

### 插件作者

发布前确认：

- [ ] 插件同时支持 SwiftPM 和 CocoaPods
- [ ] `Package.swift` 配置正确（平台、依赖、资源）
- [ ] 代码已移至 `plugin_name/Sources/plugin_name/`
- [ ] `.podspec` 路径已更新
- [ ] `pubspec.yaml` 设置 `flutter: ">=3.41.0"`
- [ ] CocoaPods 模式下 example app 运行正常
- [ ] SwiftPM 模式下 example app 运行正常
- [ ] `pod lib lint` 通过
- [ ] 测试通过
- [ ] `CHANGELOG.md` 注明已支持 SwiftPM

---

## 🤔 FAQ

### Q1: Flutter 3.44 是不是会自动移除 CocoaPods？

**A**: **不会**。Flutter 3.44+ 默认启用 SwiftPM,但仍会对不支持 SwiftPM 的插件 fallback 到 CocoaPods。**必须先确认全部插件支持 SwiftPM,才能手工移除 CocoaPods。**

### Q2: 我的项目曾经禁用过 SwiftPM,如何重新启用？

**A**: 运行 `flutter config --enable-swift-package-manager`,然后 `flutter clean && flutter pub get && flutter run`。

### Q3: 如何判断哪些插件仍 fallback 到 CocoaPods？

**A**: 检查 `ios/Pods` 目录是否存在且非空,或查看 `ios/Podfile.lock` 列出的插件。如果某插件在 `Podfile.lock` 中,说明它仍走 CocoaPods。

### Q4: 移除 CocoaPods 后还能回退吗？

**A**: 能。只要插件仍支持 CocoaPods,可以：
1. 关闭 SwiftPM：`flutter config --no-enable-swift-package-manager`
2. 恢复 `Podfile`（如果未提交删除）
3. `flutter clean && flutter pub get && flutter run`

但不推荐长期关闭 SwiftPM,因为 CocoaPods registry 已只读,未来版本会移除 opt-out。

### Q5: Flavor / 多 scheme 项目迁移失败怎么办？

**A**: 官方文档要求：收集 `ios/Runner.xcodeproj/project.pbxproj` 和对应 `.xcscheme` 文件,报 issue 给 Flutter 团队。**不要自行手工改 `pbxproj` 或 `.xcscheme`**,除非官方文档给出可执行步骤。

### Q6: Flutter 3.50 起禁用 SwiftPM 会怎样？

**A**: 每个 Flutter CLI 命令（`flutter run`、`flutter build`、`flutter test` 等）都会显示警告,提醒 CocoaPods fallback 即将移除。未来版本会移除 `--no-enable-swift-package-manager` 选项。

### Q7: 我的 Podfile 里有手工添加的非 Flutter 依赖,如何处理？

**A**: 检查这些依赖是否有 SwiftPM 版本,在 Xcode 中用 SPM 重新添加。如果没有,联系维护者或寻找替代库。不能只靠 `pod deintegrate` 自动清理。

### Q8: 插件作者是否应该移除 CocoaPods 支持？

**A**: **不应该,直到另行通知。** 插件应同时支持 SwiftPM 和 CocoaPods,确保未迁移应用仍可使用。

### Q9: add-to-app 项目如何处理？

**A**: 参考官方文档：
- [Integrate a Flutter app into your iOS project](https://docs.flutter.dev/add-to-app/ios/project-setup)
- [Integrate a Flutter app into your macOS project](https://docs.flutter.dev/add-to-app/macos/project-setup)

本文不覆盖 add-to-app 完整流程。

### Q10: 验证时 Xcode 报 "Package product requires minimum platform version 14.0"

**A**: 某插件要求更高的 iOS/macOS 版本。处理：
1. 在 Xcode 中打开 `Runner.xcworkspace`
2. 提高 `Runner` target 的 Minimum Deployments（如 iOS 14.0）
3. iOS：运行 `flutter build ios --config-only`
4. macOS：运行 `flutter build macos --config-only`

---

## 💡 最佳实践

1. **先审计,再移除**
   - 不要在未确认全部插件支持前删除 `Podfile`
   - 用 AI 生成插件支持清单,人工确认

2. **分阶段迁移**
   - 先在 dev 分支验证全部插件支持
   - 通过后再移除 CocoaPods 并合入主分支

3. **保留 CocoaPods 配置备份**
   - 移除前提交一次,保留 `Podfile` 和 `Podfile.lock`
   - 如果迁移失败,可快速回退

4. **Flavor 项目谨慎操作**
   - 自动迁移可能失败,按官方指引报 issue
   - 不要盲目手工改 `pbxproj`

5. **插件作者：同时支持两种方式**
   - 在过渡期保留 CocoaPods 支持
   - 验证两种模式都能正常运行

6. **CI 配置同步更新**
   - 如果 CI 显式安装 CocoaPods,移除后需更新 CI 脚本
   - 确保 CI 使用 Flutter 3.44+

---

## 🔗 相关资源

- 审计 SwiftPM 与 CocoaPods fallback：[prompts/spm/audit-spm-and-cocoapods-fallback.md](../prompts/spm/audit-spm-and-cocoapods-fallback.md)
- 迁移应用或插件到 SwiftPM：[prompts/spm/migrate-app-or-plugin-to-spm.md](../prompts/spm/migrate-app-or-plugin-to-spm.md)
- 官方应用开发者文档：[https://docs.flutter.dev/packages-and-plugins/swift-package-manager/for-app-developers](https://docs.flutter.dev/packages-and-plugins/swift-package-manager/for-app-developers)
- 官方插件作者文档：[https://docs.flutter.dev/packages-and-plugins/swift-package-manager/for-plugin-authors](https://docs.flutter.dev/packages-and-plugins/swift-package-manager/for-plugin-authors)
- Flutter 3.50 公告：[https://flutter.dev/blog/whats-new-in-flutter-3-50](https://flutter.dev/blog/whats-new-in-flutter-3-50)

---

**先审计插件支持情况,在全部就绪后再移除 CocoaPods。**
