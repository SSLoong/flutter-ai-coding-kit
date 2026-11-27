# 提示词：审计代码变更是否可补丁

**用途**：根据 git diff 或变更清单判断能否用 Shorebird patch，还是必须发新 release

---

## 📋 适用场景

- 准备发 Shorebird patch 前，预先确认变更范围
- CI 中自动检查 PR 是否可补丁
- Code Review 时标记哪些变更需要走商店审核

---

## 📥 输入要求

在 Cursor 中准备：

1. **Git diff 输出**：`git diff <base-commit> HEAD` 或 `git diff`
2. **变更文件列表**：`git status --short` 或 `git diff --name-only`
3. **变更说明**：每个文件做了什么（如「修复空指针」「新增图片资源」）
4. **当前 release 版本**：如 `1.0.0+1`（用于后续 patch 命令）

---

## 💬 提示词模板

```markdown
根据下面的代码变更，判断是否可以用 Shorebird patch，还是必须发新 release 到商店。

## Git Diff

\`\`\`diff
[粘贴 git diff 输出，或 git diff <file1> <file2> ...]
\`\`\`

## 变更文件列表

\`\`\`
[粘贴 git status --short 或 git diff --name-only 输出]
\`\`\`

## 变更说明

- `[文件路径]`：[做了什么，如「修复空指针」「新增图片」「更新依赖版本」]
- `[文件路径]`：...

## 判断标准（Shorebird 官方规则）

**可补丁的**：
- ✅ Dart 代码变更（`lib/` 下的 `.dart` 文件）
- ✅ 生成的 Dart 代码（如 `*.g.dart`、`*.freezed.dart`、`app_localizations.dart`）
- ✅ 纯 Dart 依赖更新（`pubspec.yaml` 中不包含原生代码的包）

**不可补丁的**：
- ❌ 原生代码变更（`android/`、`ios/`、`macos/`、`windows/`、`linux/` 下的 Kotlin/Swift/Java/ObjC/C++/build 配置）
- ❌ 资产文件变更（`pubspec.yaml` 的 `assets` 部分，或 `assets/`、`images/`、`fonts/` 等目录）
- ❌ Flutter 版本变更（`pubspec.yaml` 的 `flutter` SDK 版本）
- ❌ 包含原生代码的插件更新（如 `image_picker`、`camera`、`url_launcher` 等）

## 请回答

1. **结论**：`可补丁 ✅` / `必须新 release ❌` / `需要人工二次确认 ⚠️`
2. **理由**：列出关键文件和变更类型
3. **推荐命令**：
   - 可补丁 → 给出 `shorebird patch` 命令（含 `--release-version`、`--track staging`）
   - 必须 release → 给出 `shorebird release` 命令，并提示「需提交商店」
   - 需确认 → 列出可疑点，让人工判断
4. **风险提示**（如果有）：
   - 例如：「虽然技术上可补丁,但改动涉及支付流程,建议走商店审核以避免合规风险」

## 示例输出格式

### 可补丁示例

> **可补丁 ✅**
>
> **理由**：
> - 变更仅涉及 `lib/screens/profile_screen.dart`（Dart 代码）
> - 未修改 `android/`、`ios/`、`pubspec.yaml` 的 assets
> - 无原生依赖更新
>
> **推荐命令**：
> \`\`\`bash
> shorebird patch android \\
>   --release-version 1.0.0+1 \\
>   --track staging \\
>   --flavor prod \\
>   --target lib/main_prod.dart
> \`\`\`
>
> **无风险**

### 不可补丁示例

> **必须新 release ❌**
>
> **理由**：
> - `pubspec.yaml` 新增 `image_picker: ^1.0.0`，该插件包含原生代码
> - `android/app/src/main/AndroidManifest.xml` 新增 `CAMERA` 权限
> - `ios/Runner/Info.plist` 新增 `NSCameraUsageDescription`
>
> **推荐命令**：
> \`\`\`bash
> # 1. 创建新 release
> shorebird release android --flavor prod --target lib/main_prod.dart
>
> # 2. 提交到 Google Play Console
> # build/app/outputs/bundle/release/app-prod-release.aab
>
> # 3. iOS 同理（需 macOS）
> shorebird release ios --flavor prod --target lib/main_prod.dart
> \`\`\`
>
> **原因**：原生代码和权限变更无法通过 OTA 推送，必须走商店审核。

### 需人工确认示例

> **需要人工二次确认 ⚠️**
>
> **理由**：
> - `lib/services/payment_service.dart` 修改了支付跳转逻辑（Dart 代码，技术上可补丁）
> - 但涉及支付流程，可能被 App Store 视为「绕过内购审核」
>
> **推荐**：
> 1. 如果只是修复支付 bug（如空指针、逻辑错误）→ 可补丁
> 2. 如果改变支付方式或新增支付入口 → 走商店审核
>
> **请人工判断**：变更是否属于「核心功能变更」或「合规敏感操作」。
```

---

## ✅ 验证步骤

AI 给出判断后：

```bash
# 1. 如果判断为「可补丁」
shorebird patch android --release-version <version> --track staging --dry-run

# 2. 观察输出
# - 若出现 "Native code changes detected" → 不可补丁（AI 可能判断错误）
# - 若出现 "Asset changes detected" → 不可补丁
# - 若 --dry-run 通过 → 确认可补丁
```

**检查清单**：
- [ ] AI 列出的文件与 `git diff --name-only` 一致
- [ ] AI 判断的「原生/资产/Dart」分类正确
- [ ] `--dry-run` 与 AI 判断结果一致
- [ ] 如有合规风险，已标注「需人工确认」

---

## 🛠️ 常见问题

1. **AI 说可补丁，但 `shorebird patch` 报错**
   - 检查 `git diff` 是否包含隐藏的原生文件变更（如 `.gradle`、`.pbxproj`）
   - 用 `git diff android/ ios/` 单独确认原生目录
   - AI 可能未看到 submodule 或 symlink 变更

2. **`pubspec.yaml` 依赖更新是否可补丁？**
   - **纯 Dart 包**（如 `intl`、`provider`、`freezed`）：✅ 可补丁
   - **带原生代码的插件**（如 `camera`、`path_provider`）：❌ 不可补丁
   - 判断方法：打开包的 pub.dev 页面，看是否有 Android/iOS/macOS 等平台标签

3. **生成代码（`*.g.dart`）是否可补丁？**
   - ✅ `build_runner` 生成的 Dart 代码（`*.g.dart`、`*.freezed.dart`）可补丁
   - ✅ `flutter gen-l10n` 生成的 l10n 代码（`app_localizations.dart`）可补丁
   - ❌ Pigeon 生成的原生代码（`.kt`、`.swift`）不可补丁（但 Pigeon 生成的 Dart 侧可补丁）

4. **AI 建议「需人工确认」时怎么办？**
   - 咨询团队法务或合规负责人
   - 查阅 App Store Review Guidelines（[https://developer.apple.com/app-store/review/guidelines/](https://developer.apple.com/app-store/review/guidelines/)）
   - 查阅 Google Play 政策（[https://play.google.com/about/developer-content-policy/](https://play.google.com/about/developer-content-policy/)）
   - 保守起见：走商店审核

5. **能否只给 AI 看文件列表，不给 diff 内容？**
   - 可以，但判断会不准确
   - AI 无法识别「原生文件里只改了注释」vs「改了关键逻辑」
   - 推荐至少提供 `git diff --stat` 和部分关键文件的 diff

---

## 🔗 相关

- 工作流：[workflows/shorebird-code-push.md](../../workflows/shorebird-code-push.md)
- 操作提示词：[setup-release-and-patch.md](setup-release-and-patch.md)
- Shorebird 官方规则：[https://docs.shorebird.dev/code-push#is-your-change-patchable](https://docs.shorebird.dev/code-push#is-your-change-patchable)
- 商店合规：[https://docs.shorebird.dev/faq#store-compliance](https://docs.shorebird.dev/faq#store-compliance)
