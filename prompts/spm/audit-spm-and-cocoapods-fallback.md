# 提示词：审计 SwiftPM 与 CocoaPods fallback

**用途**：审计 Flutter 项目是否启用 Swift Package Manager,哪些插件仍 fallback 到 CocoaPods,能否移除 CocoaPods

---

## 📋 适用场景

- Flutter 3.44+ 项目,想确认是否已启用 SwiftPM
- 准备移除 CocoaPods,需要先确认全部插件支持
- 升级 Flutter 后,不确定 SwiftPM 迁移状态
- 曾经禁用过 SwiftPM,想重新启用并移除 CocoaPods

---

## 📥 输入要求

在 Cursor 中准备：

1. **Flutter 版本**：运行 `flutter --version`,确认 ≥ 3.44
2. **项目配置**：打开 `pubspec.yaml`,查看依赖列表
3. **iOS/macOS 目录**：检查 `ios/` 和 `macos/`（如果有）目录结构
4. **CocoaPods 文件**：查看 `ios/Podfile`、`ios/Podfile.lock`、`ios/Pods/`（如果存在）

---

## 💬 提示词模板

```markdown
我想审计项目的 Swift Package Manager 与 CocoaPods 状态。请：

**当前状态**：
- Flutter 版本：[粘贴 `flutter --version` 输出]
- 项目依赖：[粘贴 `pubspec.yaml` 的 dependencies 部分]
- iOS 目录：[是否存在 `ios/` 目录]
- macOS 目录：[是否存在 `macos/` 目录]

**请执行**：

1. **检查 SwiftPM 启用状态**：
   - 读取 `pubspec.yaml` 的 `flutter.config.enable-swift-package-manager` 字段（如果有）
   - 如果 Flutter ≥ 3.44 且没有禁用,已默认启用
   - 如果曾经禁用,生成重新启用命令：`flutter config --enable-swift-package-manager`

2. **审计 CocoaPods fallback**：
   - 检查 `ios/Pods/` 和 `macos/Pods/` 目录是否存在
   - 如果存在,读取 `ios/Podfile.lock` 或 `macos/Podfile.lock`,列出当前走 CocoaPods 的插件
   - 对每个插件,在 pub.dev 上查其最新版本,判断是否支持 SwiftPM（查 GitHub 仓库的 `ios/plugin_name/Package.swift` 或 `darwin/plugin_name/Package.swift`）

3. **生成插件支持清单**：
   - 列表格：插件名、当前版本、是否支持 SwiftPM、建议操作
   - 示例：
     | 插件名 | 当前版本 | SwiftPM 支持 | 建议 |
     |--------|----------|-------------|------|
     | url_launcher | ^6.2.0 | ✅ | 已支持 |
     | some_old_plugin | ^1.0.0 | ❌ | 等待更新或寻找替代 |

4. **判断能否移除 CocoaPods**：
   - 如果全部插件支持 SwiftPM,生成移除 CocoaPods 命令（见下）
   - 如果部分插件不支持,列出需要更新或替代的插件
   - 如果 `Podfile` 中有手工添加的非 Flutter 依赖,提醒需人工处理

5. **手工 Podfile 检查**：
   - 读取 `ios/Podfile` 或 `macos/Podfile`
   - 列出所有非 Flutter 生成的 `pod` 行（如 Firebase、Analytics 等）
   - 建议如何用 SwiftPM 替代这些依赖

**如果可以移除 CocoaPods,生成以下命令清单**（iOS 示例,macOS 类似）：

```bash
# iOS
cd ios
pod deintegrate
cd ..
rm -rf ios/Podfile ios/Podfile.lock ios/Pods ios/.symlinks

# 检查 xcconfig 文件
# 打开 ios/Flutter/Debug.xcconfig 和 ios/Flutter/Release.xcconfig
# 删除类似 #include "Pods/Target Support Files/..." 的行（如果有）

# 清理并验证
flutter clean
flutter pub get
flutter run
```

**约束**：
- 只审计和建议,不要自动删除文件
- 插件信息以 pub.dev 和 GitHub 仓库实际情况为准
- 如果无法确认插件是否支持 SwiftPM,标记为「需人工确认」
```

---

## ✅ 验证步骤

审计后确认：

```bash
# AI 应输出：
# 1. SwiftPM 启用状态（已启用 / 已禁用 / 需重新启用）
# 2. 插件支持清单（哪些支持、哪些不支持）
# 3. 能否移除 CocoaPods（是 / 否,原因）
# 4. 移除命令清单（如果可以移除）
```

**检查清单**：

- [ ] AI 已确认 SwiftPM 启用状态
- [ ] AI 已列出全部依赖插件及其 SwiftPM 支持情况
- [ ] AI 已明确能否移除 CocoaPods
- [ ] 如果不能移除,AI 已列出需要更新或替代的插件
- [ ] 如果 `Podfile` 有手工依赖,AI 已提醒需人工处理

---

## 🛠️ 常见问题

1. **AI 说已启用 SwiftPM,但 `ios/Pods` 仍存在**
   - 正常。Flutter 会对不支持 SwiftPM 的插件 fallback 到 CocoaPods。等全部插件支持后才能移除。

2. **AI 无法确认某插件是否支持 SwiftPM**
   - 在 pub.dev 上找到插件,点进 GitHub 仓库,查看 `ios/plugin_name/Package.swift` 或 `darwin/plugin_name/Package.swift` 是否存在。

3. **AI 建议移除 CocoaPods,但我不确定是否安全**
   - 先在 dev 分支验证：移除后运行 `flutter clean && flutter pub get && flutter run`,确认无报错。

4. **`Podfile` 中有手工添加的 Firebase 依赖**
   - 查 Firebase 官方文档,是否有 SwiftPM 版本。如果有,在 Xcode 中用 SPM 重新添加。

5. **插件最新版本支持 SwiftPM,但我用的是旧版本**
   - 更新 `pubspec.yaml` 中该插件的版本,运行 `flutter pub upgrade <plugin_name>`,再重新审计。

---

## 🔗 相关

- 工作流：[workflows/swift-package-manager-migration.md](../../workflows/swift-package-manager-migration.md)
- 迁移应用或插件：[migrate-app-or-plugin-to-spm.md](migrate-app-or-plugin-to-spm.md)
- 官方文档：[https://docs.flutter.dev/packages-and-plugins/swift-package-manager/for-app-developers](https://docs.flutter.dev/packages-and-plugins/swift-package-manager/for-app-developers)
