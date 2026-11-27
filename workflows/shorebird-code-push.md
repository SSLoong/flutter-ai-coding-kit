# Shorebird Code Push / OTA 补丁工作流

**用 AI 管理 Flutter 应用的即时热修复与功能迭代，无需等待商店审核**

> 作者：Loong (SSLoong) | 日期：2026-11-22

本文对齐 **Shorebird CLI 1.6.140** 与 **Flutter 3.50.1**（2026-11-20 发布）。先在本机执行 `shorebird --version` 和 `flutter --version`。

---

## 📋 背景

应用商店审核周期（iOS 通常 1-3 天、Google Play 数小时至数天)阻碍了紧急修复与快速迭代。Shorebird Code Push 允许你**直接向用户设备推送 Dart 代码更新**，绕过商店审核流程。用户在下次启动应用时自动获取补丁，无需手动更新。

AI 在此工作流中能帮你：判断变更是否可补丁、生成 `shorebird release` / `shorebird patch` 命令、审计 git diff 中的原生/资产变更、规划分阶段放量策略。但**补丁的商店合规性与业务决策由你负责**，AI 只能提供技术判断。

---

## 🎯 目标

- ✅ 判断何时能用 OTA 补丁、何时必须发新商店包
- ✅ 配置 Shorebird（`shorebird init`、选择签名模式）
- ✅ 创建可补丁的 release 并提交商店
- ✅ 用 AI 审计代码变更是否可补丁（Dart、原生、资产）
- ✅ 在 staging / beta track 验证补丁，再分阶段放量到 stable
- ✅ 配合 Flavor 多环境，每个 flavor 独立 app_id
- ✅ 理解补丁签名（默认开启）与回滚机制

---

## 🔧 适用场景与决策表

### 场景适用度

| 场景 | 适合度 | 说明 |
|------|--------|------|
| 线上 Dart 层崩溃热修复 | ⭐⭐⭐⭐⭐ | 修复空指针、逻辑错误、状态管理问题 |
| 纯 Dart 功能小改（UI、业务逻辑） | ⭐⭐⭐⭐⭐ | 按钮文案、列表样式、API 请求逻辑 |
| 更新 Dart 依赖（未改原生） | ⭐⭐⭐⭐ | 仅更新 pubspec.yaml 中的纯 Dart 包版本 |
| 调整生成代码（如 l10n、freezed） | ⭐⭐⭐⭐ | 生成的 Dart 代码可补丁（遵循官方 l10n 方式） |
| 新增图片/字体或更改资产 | ⭐ | 资产变更不可补丁，必须发新 release |
| 原生代码变更（Kotlin/Swift/build 配置） | ⭐ | 必须发新商店包 |
| 更改 Flutter 版本 | ⭐ | Flutter 引擎版本由 release 锁定，不可补丁 |

### 该补丁还是发商店包？

```
你的变更包含：
├─ 原生代码（Kotlin/Swift/ObjC/Java）？
│  └─ 是 → 发新 release（shorebird release）→ 提交商店
│
├─ 资产文件（图片/字体/pubspec assets）？
│  └─ 是 → 发新 release → 提交商店
│
├─ Flutter 版本升级？
│  └─ 是 → 发新 release → 提交商店
│
└─ 只有 Dart 代码（含生成代码、纯 Dart 依赖）？
   └─ 是 → 可补丁（shorebird patch）
```

**商店合规提醒**：
- Shorebird 能检测原生/资产变更并在 `patch` 时警告或阻止上传，但**不能替你判断 Dart 代码变更是否符合商店政策**。
- 例如：改变应用核心功能、绕过商店内购、违反地区法规等，即使技术上可补丁，也可能违反 App Store Review Guidelines 或 Google Play 政策。
- 见官方文档 [Store Compliance](https://docs.shorebird.dev/faq#store-compliance) 和 [What can't Shorebird be used for](https://docs.shorebird.dev/faq#what-cant-shorebird-be-used-for)。

---

## 🚀 完整流程

下面用**线上崩溃修复**贯穿：生产环境（flavor `prod`）出现 Dart 层空指针崩溃，需热修。背景环境已有 dev / staging / prod 三个 flavor（见 [workflows/flavor-multi-env.md](flavor-multi-env.md)）。

### Step 0: 环境准备与版本确认

**人工操作**：

```bash
# 1. 安装或升级 Shorebird CLI（需要 1.6.139+ 支持百分比放量，1.6.140+ 修复 iOS obfuscate 崩溃）
curl -fsSL https://raw.githubusercontent.com/shorebirdtech/install/main/install.sh | bash
shorebird upgrade  # 已安装则升级到最新版

# 2. 确认 Flutter 版本（推荐 3.50.1 或 3.47.9+）
flutter --version
# 若版本低于 3.35.7，百分比放量将不可用（设备无法参与 rollout）

# 3. 检查项目结构
ls pubspec.yaml android/app/build.gradle ios/Runner.xcodeproj
```

**检查清单**：
- [ ] Shorebird CLI ≥ 1.6.140
- [ ] Flutter ≥ 3.35.7（百分比放量需要）或 ≥ 3.47.9（推荐）
- [ ] 项目已配置 Flavor（如果有多环境需求）

---

### Step 1: 初始化 Shorebird（首次）

**人工操作**（每个项目只需一次）：

```bash
cd your_flutter_project
shorebird init
```

提示时选择：
1. **登录或创建账号**（会打开浏览器）
2. **确认应用名称**（默认使用 `pubspec.yaml` 的 `name`）
3. **选择签名模式**（重要，首次 release 前决定）：
   - **Managed signing（托管签名，推荐）**：Shorebird 服务端用你的私钥对补丁签名，补丁到设备前验证。适合团队协作，无需每台 CI 机器存储私钥。
   - **Self-managed signing（自管理签名）**：你自己保管私钥，自行签名。适合严格安全要求。
   - **Unsigned（无签名，不推荐）**：补丁无签名，设备不验证。仅限测试或内部分发。

**生成文件**：
- `shorebird.yaml`（项目根目录）：包含 `app_id`，提交到 Git。

**AI 提示词**（配置检查）：

```markdown
检查我的项目是否已正确初始化 Shorebird：

1. 是否存在 `shorebird.yaml` 文件？
2. 文件中 `app_id` 是否为有效 UUID？
3. 如果有 Flavor，是否每个 flavor 需要单独的 `app_id`？（Shorebird 要求每个 applicationId 独立 app_id）

如果有多个 flavor，提示我：每个 flavor 需要在其目录下单独 `shorebird init`，或在 `shorebird.yaml` 中用注释标明「哪个 app_id 对应哪个 flavor」。
```

**Flavor 多环境注意事项**：
- **每个 flavor 独立 `app_id`**：例如 `com.example.app`（prod）、`com.example.app.dev`（dev）需要两个不同的 `app_id`。
- 在 `shorebird.yaml` 可以这样记录：
  ```yaml
  # Production
  app_id: xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx
  
  # Dev flavor: yyyyyyyy-yyyy-yyyy-yyyy-yyyyyyyyyyyy
  # Staging flavor: zzzzzzzz-zzzz-zzzz-zzzz-zzzzzzzzzzzz
  ```
- 创建 release 和 patch 时必须带 `--flavor` 和 `--target` 参数。

---

### Step 2: 创建首个 release 并提交商店

**人工操作**：

```bash
# Android（无 Flavor）
shorebird release android

# Android（有 Flavor）
shorebird release android \
  --flavor prod \
  --target lib/main_prod.dart \
  --dart-define-from-file=config/prod.json

# iOS（需 macOS）
shorebird release ios \
  --flavor prod \
  --target lib/main_prod.dart \
  --dart-define-from-file=config/prod.json
```

**`shorebird release` 做了什么**：
1. 调用 `flutter build aab` / `flutter build ipa`（替代 `flutter build`）
2. 上传编译后的 Dart artifacts 到 Shorebird 服务器（用于后续 patch diff）
3. 生成 `.aab`（Android）或 `.ipa`（iOS）文件在 `build/` 目录

**关键参数**：
- `--flavor` / `-t` / `--dart-define*`：**必须与后续 patch 保持完全一致**，否则 patch 可能链接失败或变更意外代码。
- `--obfuscate` / `--split-debug-info`：代码混淆。iOS 在 Flutter 3.41.2+ 支持；早期版本可能崩溃（见官方 [Flutter version notes](https://docs.shorebird.dev/getting-started/flutter-version/)）。
- `--flutter-version`：指定 Flutter 版本，如 `--flutter-version=3.50.0`。省略时用默认。

**签名模式锁定**：
- 首次 `shorebird release` 时，补丁签名模式被固定（managed / self-managed / unsigned）。
- 后续 patch 沿用该模式，**不能再改**。
- 如需更换签名模式，必须发新 release。

**提交商店**：
- **Android**：`build/app/outputs/bundle/release/app-prod-release.aab` 上传到 Google Play Console。
- **iOS**：`build/ios/archive/Runner.xcarchive` 在 Xcode 中归档并上传 App Store Connect。

**检查清单**：
- [ ] `shorebird release` 退出码为 0
- [ ] `.aab` / `.ipa` 文件已生成
- [ ] 已上传到商店并发布给用户（或内部测试组）

---

### Step 3: 用 AI 审计代码变更是否可补丁

**场景**：线上 `1.0.0+1` 版本在用户点击「个人资料」时崩溃（空指针）。你在本地修复后，准备发补丁。

**人工操作**：

```bash
# 查看自上次 release 以来的所有变更
git diff v1.0.0+1 HEAD

# 或查看当前未提交的变更
git status
git diff
```

**AI 提示词**（可复制）：见 [prompts/code-push/audit-patchable-change.md](../prompts/code-push/audit-patchable-change.md)。

**示例**：

```markdown
根据下面的 git diff，判断这些变更是否可以用 Shorebird patch，还是必须发新 release：

## Git Diff

\`\`\`diff
diff --git a/lib/screens/profile_screen.dart b/lib/screens/profile_screen.dart
index 1234567..abcdefg 100644
--- a/lib/screens/profile_screen.dart
+++ b/lib/screens/profile_screen.dart
@@ -42,7 +42,7 @@ class _ProfileScreenState extends State<ProfileScreen> {
   @override
   Widget build(BuildContext context) {
-    final user = context.read<AuthProvider>().currentUser;
+    final user = context.read<AuthProvider>().currentUser ?? User.guest();
     return Scaffold(
       appBar: AppBar(title: Text(user.name)),
       body: Text('Welcome ${user.email}'),
\`\`\`

## 变更清单

- 文件：`lib/screens/profile_screen.dart`
- 类型：Dart 代码（空指针修复）
- 说明：`currentUser` 可能为 null，增加 `?? User.guest()` 兜底

请回答：
1. 这次变更是否可以打补丁？
2. 是否需要人工二次确认？
3. 如果不可补丁，需要改什么？
```

**AI 应返回**（示例）：

> **可补丁 ✅**
> 
> 理由：
> - 变更仅涉及 Dart 代码（`lib/screens/profile_screen.dart`）
> - 未修改 `android/`、`ios/`、`pubspec.yaml` 的 assets
> - 未更改 Flutter 版本或原生依赖
> 
> 无需人工确认，可直接 `shorebird patch`。

**不可补丁的例子**：

```diff
diff --git a/pubspec.yaml b/pubspec.yaml
@@ -15,6 +15,7 @@ dependencies:
   flutter:
     sdk: flutter
+  image_picker: ^1.0.0  # 新增，包含原生代码

diff --git a/android/app/src/main/AndroidManifest.xml b/android/app/src/main/AndroidManifest.xml
+    <uses-permission android:name="android.permission.CAMERA"/>
```

> **不可补丁 ❌**
> 
> 理由：
> - `image_picker` 插件包含原生代码（Kotlin/Swift）
> - `AndroidManifest.xml` 变更属于原生层
> 
> 必须发新 release：`shorebird release android`，再提交 Google Play。

---

### Step 4: 创建并发布 patch

确认可补丁后：

**人工操作**：

```bash
# Android patch（先发 staging track 验证）
shorebird patch android \
  --release-version 1.0.0+1 \
  --flavor prod \
  --target lib/main_prod.dart \
  --dart-define-from-file=config/prod.json \
  --track staging

# iOS patch
shorebird patch ios \
  --release-version 1.0.0+1 \
  --flavor prod \
  --target lib/main_prod.dart \
  --dart-define-from-file=config/prod.json \
  --track staging
```

**关键参数**：
- `--release-version`：**必填**（CI 中）或交互式选择。指定要补丁的 release 版本。
- `--track`：部署通道，默认 `stable`。推荐先用 `staging` 或 `beta` 验证。
- `--rollout`：百分比放量，如 `--rollout 10`（10% 用户先看到）。需 Shorebird 1.6.139+，设备需 Flutter 3.35.7+。
- `--dry-run`（`-n`）：只构建和检查，不上传。适合 CI 预检。

**`shorebird patch` 做了什么**：
1. 下载对应 release 的 artifacts
2. 用 release 时的 Flutter 版本构建当前代码
3. 生成 diff（补丁）
4. 上传到 Shorebird 后端
5. 发布到指定 track

**Shorebird 自动检查**（Shorebird 1.6.138+）：
- **原生代码变更**：对比 `android/`、`ios/` 与 release 时的快照，发现变更则警告或阻止（可用 `--allow-native-diffs` 强制跳过，**不推荐**）。
- **资产变更**：对比 `pubspec.yaml` 的 assets，发现变更则警告或阻止（可用 `--allow-asset-diffs` 强制跳过，**不推荐**）。
- **构建标志不一致**：对比 `--dart-define*` / `--target`，不一致则警告。可用 `--allow-build-flag-diffs` 强制，但可能导致非预期代码变更。

**输出示例**：

```
✓ Fetching release artifacts (0.3s)
Building patch with Flutter 3.50.1 (...)

Running Gradle task 'bundleRelease'...                             38.4s
✓ Built build/app/outputs/bundle/release/app-release.aab (20.1MB)
✓ Verifying patch can be applied to release (0.4s)
✓ Creating patch artifacts (2.0s)

🚀 Ready to publish a new patch!

📱 App: MyApp (app-id-here)
📦 Release Version: 1.0.0+1
🕹️  Platform: Android [arm64 (158.32 KB), arm32 (152.10 KB), x86_64 (159.45 KB)]
🟢 Track: staging

✓ Creating patch (85ms)
✓ Uploading artifacts (1.4s)
✓ Promoting patch to staging (72ms)

✅ Published Patch 1!
```

---

### Step 5: 本地与预发验证 patch

**验证策略**：
1. **本地 preview**：用 `shorebird preview` 下载 release 并运行，手动触发更新。
2. **预发 track**（staging / beta）：先发 `--track staging`，内部测试组验证。
3. **分阶段放量**：验证无问题后，逐步放量到 stable（10% → 50% → 100%）。

**本地验证**（可选）：

```bash
# 下载 release 1.0.0+1 并运行在设备上
shorebird preview --release-version 1.0.0+1

# 在应用内手动检查更新（需集成 package:shorebird_code_push）
# 或等待下次启动时自动拉取
```

**集成手动更新检查**（可选，需要 `package:shorebird_code_push ^2.0.0`）：

```yaml
# pubspec.yaml
dependencies:
  shorebird_code_push: ^2.0.0
```

```dart
// lib/main.dart
import 'package:shorebird_code_push/shorebird_code_push.dart';

final _updater = ShorebirdUpdater();

Future<void> checkForUpdates() async {
  final status = await _updater.checkForUpdate();
  if (status == UpdateStatus.outdated) {
    await _updater.update();  // 下载补丁，下次启动生效
  }
}
```

**检查清单**（staging track）：
- [ ] 应用启动无崩溃
- [ ] 崩溃已修复（检查日志或复现原场景）
- [ ] 无非预期 UI 变化
- [ ] 性能无明显下降（iOS 补丁的已变更代码运行在解释器，可能略慢；Android/桌面无影响）

---

### Step 6: 分阶段放量到 stable

验证通过后，逐步放量：

**AI 提示词**（规划放量策略）：

```markdown
我要把 patch 1 从 staging 提升到 stable，并分阶段放量。给我推荐一个放量计划：

## 背景

- Release 版本：1.0.0+1
- Patch 内容：修复空指针崩溃
- 风险评估：低风险（仅修改一行代码）
- 用户规模：Android 10 万日活，iOS 5 万日活

## 要求

1. 给出放量步骤（百分比和间隔时间）
2. 每个阶段的监控指标（崩溃率、性能）
3. 何时回滚的判断标准
```

**AI 应返回**（示例）：

> **推荐放量计划**
> 
> | 阶段 | 百分比 | 持续时间 | 监控指标 |
> |------|--------|----------|----------|
> | 1    | 1%     | 4 小时   | 崩溃率 < 0.1% |
> | 2    | 10%    | 12 小时  | 崩溃率 < 0.1%，用户反馈无异常 |
> | 3    | 50%    | 24 小时  | 崩溃率 < 0.1%，性能指标正常 |
> | 4    | 100%   | -        | 全量覆盖 |
> 
> **回滚条件**：
> - 崩溃率 > 0.2%（高于基线 0.05% 的 4 倍）
> - 出现新类型崩溃
> - 用户反馈负面情绪集中

**人工操作**（手动放量）：

```bash
# 1. 提升到 stable，初始 1%
shorebird patches promote \
  --release-version 1.0.0+1 \
  --patch-number 1 \
  --rollout 1

# 2. 等待 4 小时后，提升到 10%
shorebird patches rollout \
  --release-version 1.0.0+1 \
  --patch-number 1 \
  --percentage 10

# 3. 继续提升到 50%
shorebird patches rollout \
  --release-version 1.0.0+1 \
  --patch-number 1 \
  --percentage 50

# 4. 最终 100%
shorebird patches rollout \
  --release-version 1.0.0+1 \
  --patch-number 1 \
  --percentage 100
```

**自动定时放量**（Enterprise 计划，Shorebird 1.6.143+）：

```bash
# 一条命令定义完整时间表
shorebird patch android \
  --release-version 1.0.0+1 \
  --track stable \
  --rollout-steps 1:4h,10:12h,50:24h,100 \
  --pause-weekends \
  --timezone America/New_York
```

**查看放量进度**：

```bash
# 查看采纳率（设备占比）
shorebird metrics adoption --release-version 1.0.0+1

# 列出所有 patch 及其状态
shorebird patches list --release-version 1.0.0+1
```

---

### Step 7: 回滚与迭代

**回滚 patch**（发现问题时）：

```bash
shorebird patches rollback \
  --release-version 1.0.0+1 \
  --patch-number 1
```

- 设备在下次检查更新时**自动卸载**该 patch，回到上一个 patch（或 release 基线）。
- 回滚后可用 `rollforward` 恢复：
  ```bash
  shorebird patches rollforward \
    --release-version 1.0.0+1 \
    --patch-number 1
  ```

**迭代新 patch**：

```bash
# 修复问题后，创建 patch 2
shorebird patch android \
  --release-version 1.0.0+1 \
  --track staging  # 先在 staging 验证
```

- 每个 release 可以有无限个 patch（官方 FAQ 确认无数量限制）。
- 设备始终拉取其 track 上编号最大的 patch（除非被回滚）。

---

## 🎨 进阶场景

### Flavor 多环境与 Shorebird

**背景**：项目有 dev / staging / prod 三个 flavor（见 [workflows/flavor-multi-env.md](flavor-multi-env.md)），每个 flavor 独立 app_id。

**配置**：

```yaml
# shorebird.yaml（项目根目录）
# Production
app_id: xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx

# Dev flavor: 在 CLI 中用 SHOREBIRD_APP_ID 环境变量覆盖
# Staging flavor: 同上
```

**发布不同 flavor 的 release**：

```bash
# Prod
shorebird release android --flavor prod --target lib/main_prod.dart

# Dev（用环境变量切换 app_id）
SHOREBIRD_APP_ID=yyyyyyyy-yyyy-yyyy-yyyy-yyyyyyyyyyyy \
  shorebird release android --flavor dev --target lib/main_dev.dart
```

**或**在 CI 中动态替换 `shorebird.yaml` 的 `app_id`（不推荐，易出错）。

**检查清单**：
- [ ] 每个 flavor 的 `app_id` 不重复
- [ ] Release 时 `--flavor` 与 `--target` 对应 flavor 入口文件
- [ ] Patch 时参数与 release 完全一致

**交叉引用**：见 [workflows/flavor-multi-env.md](flavor-multi-env.md) 了解如何配置 Android `productFlavors` 和 iOS Schemes。

---

### add-to-app 与混合工程

**平台支持**：
- **Android add-to-app**：✅ 支持（Flutter module 嵌入原生应用）
- **iOS add-to-app**：✅ 支持
- **macOS / Windows / Linux add-to-app**：❌ 不支持

**注意事项**：
- Flutter module 的代码变更可补丁，**但宿主应用（Host app）的原生变更仍然不可补丁**。
- 如果原生宿主修改了与 Flutter 的通信接口（如 MethodChannel / Pigeon），必须重新发布宿主应用。

**交叉引用**：见 [workflows/pigeon-platform-channels.md](pigeon-platform-channels.md) 了解如何用 Pigeon 管理 Flutter ↔ 原生通信。原生侧变更走商店包，Dart 侧可补丁。

---

### 补丁签名的三种模式

**Shorebird 1.6.135+** 默认在新 release 时启用补丁签名。旧版本需手动开启。

| 模式 | 说明 | 适用场景 | 安全性 |
|------|------|----------|--------|
| **Managed** | Shorebird 服务端托管私钥，自动签名 | 团队协作、CI/CD | ⭐⭐⭐⭐ |
| **Self-managed** | 你自己保管私钥，每次 patch 手动签名 | 严格安全要求、合规审计 | ⭐⭐⭐⭐⭐ |
| **Unsigned** | 无签名，设备不验证 | 仅测试或内部分发 | ⭐ |

**模式选择时机**：首次 `shorebird init` 或首次 `shorebird release`。一旦 release，该 release 的所有 patch 沿用该模式，**不能更改**。

**验证级别**（Flutter 3.47.9+ 默认 `install_only`）：
- **`strict`**：每次启动验证签名（桌面应用推荐，防磁盘篡改）
- **`install_only`**（默认）：仅下载时验证一次（移动端推荐，减少启动延迟）

在 `shorebird.yaml` 可配置：

```yaml
app_id: xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx
patch_verification: install_only  # 或 strict
```

**已有未签名 release 如何升级**：
- 发新 release，启用签名。
- 旧 release 的 patch 仍为 unsigned，无法补救。

**官方文档**：[Patch Signing Guide](https://docs.shorebird.dev/code-push/guides/patch-signing/)

---

### 生成代码的补丁能力

**可补丁的生成代码**：
- ✅ **l10n**（`flutter gen-l10n` / `intl` 官方方式，生成 `app_localizations.dart`）
- ✅ **freezed**（数据类、JSON 序列化）
- ✅ **json_serializable**
- ✅ **Drift** Schema（纯 Dart 生成的表结构，**不包括原生 DB 文件本身**）
- ✅ **Riverpod** 生成的 Provider（`riverpod_generator`）

**不可补丁的生成代码**：
- ❌ **Pigeon**（跨 Dart 和原生，原生侧变更不可补丁，但 Dart 侧可补丁。见 [workflows/pigeon-platform-channels.md](pigeon-platform-channels.md)）
- ❌ **FFIgen / JNIgen**（原生绑定，涉及原生代码）

**检查点**：
- 生成代码本身是否只在 `lib/` 下生成 `.dart` 文件？ → 可补丁
- 生成代码是否修改 `android/` 或 `ios/`？ → 不可补丁，必须 release

**交叉引用**：
- [workflows/l10n-internationalization.md](l10n-internationalization.md)：官方 l10n 方式，生成代码可补丁。
- [workflows/freezed-codegen.md](freezed-codegen.md)：freezed 生成的 Dart 代码可补丁。

---

### iOS 补丁性能与 link percentage

**iOS 补丁原理**：
- **未变更代码**：继续在 CPU 上原生执行（AOT）
- **已变更代码**：运行在 Dart 解释器中（稍慢）

**link percentage**：衡量变更代码中有多少仍能以原生方式运行（而非解释器）。
- Shorebird 1.6.135+ 默认 `--min-link-percentage 20`（至少 20% 原生运行）
- 低于此阈值时，patch 被拒绝（可用 `--min-link-percentage 0` 强制接受）

**性能影响**：
- 大部分应用感知不到（UI、业务逻辑通常不是瓶颈）
- 性能敏感代码（如图像处理、大数据集计算）若被补丁，可能变慢
- **建议**：先在 staging track 测试，监控帧率和响应时间

**Android/桌面平台**：补丁直接替换编译后的 Dart 代码，**无性能影响**。

---

### 商店合规与决策责任

**Shorebird 只能检查**：
- ✅ 原生代码变更（阻止你发 patch）
- ✅ 资产文件变更（阻止你发 patch）

**Shorebird 不能检查**：
- ❌ Dart 代码变更是否符合 App Store / Google Play 政策
- ❌ 是否改变应用核心功能、绕过内购、违反地区法规

**你的责任**：
- 判断 Dart 代码变更是否属于「核心功能变更」（App Store 禁止）
- 确保不绕过商店的内购、订阅、广告审核
- 遵守 GDPR、COPPA 等地区法规

**灰色地带示例**：
- ❓ 修改支付流程跳转逻辑（可能被视为绕过内购）
- ❓ 更改 App 主要用途（如「工具」变「游戏」）
- ❓ 新增未申报的数据收集

**建议**：对于敏感变更，优先走商店审核；小修小补（UI、文案、轻微逻辑）可放心用 patch。

**官方 FAQ**：[Store Compliance](https://docs.shorebird.dev/faq#store-compliance)

---

## ✅ 验证清单

完成工作流后确认：

- [ ] 本地已安装 Shorebird CLI 1.6.139+，Flutter 3.35.7+
- [ ] 项目根目录存在 `shorebird.yaml`，包含有效 `app_id`
- [ ] 已创建至少一个 release（`shorebird release`）并提交商店
- [ ] 已成功发布至少一个 patch（`shorebird patch`）到 staging track
- [ ] Patch 参数（`--flavor` / `--target` / `--dart-define*`）与 release 完全一致
- [ ] 本地或预发环境验证 patch 无崩溃、UI 正常
- [ ] 理解何时可补丁、何时必须发商店包（原生/资产变更走商店）
- [ ] 如有 Flavor，每个 flavor 独立 `app_id` 且无混淆
- [ ] 理解补丁签名默认启用（Shorebird 1.6.135+）
- [ ] 知道如何回滚 patch（`shorebird patches rollback`）

---

## 🤔 FAQ

### Q1: `shorebird patch` 报错 `Native code or asset changes detected`？

**A**:
- Shorebird 检测到 `android/`、`ios/` 或 `pubspec.yaml` assets 变更。
- 补丁**不能修改原生代码或资产**，必须发新 release。
- 如果确认变更无关紧要（如注释、格式），可用 `--allow-native-diffs` / `--allow-asset-diffs` 强制，但**不推荐**。

### Q2: 用户何时看到补丁？

**A**:
- **默认**：启动时后台检查更新，下载完成后**下次启动**生效（即用户第二次启动才看到）。
- **立即更新**：集成 `package:shorebird_code_push`，在启动时调用 `checkForUpdate()` + `update()`，重启 App 后立即生效。
- 见官方文档 [How devices get patches](https://docs.shorebird.dev/code-push#when-do-users-see-the-update)。

### Q3: Patch 参数必须与 release 完全一致吗？

**A**: 是。
- `--dart-define` / `--dart-define-from-file` / `--target` / `--flavor` 必须与 release 时一致。
- Shorebird 1.6.138+ 会检查这些参数的 SHA-256 哈希，不一致会警告或阻止。
- Flutter 版本由 release 锁定，patch 自动使用相同版本。

### Q4: 能改变 Flutter 版本吗？

**A**: 不能。
- Patch 使用 release 时的 Flutter 版本（包括引擎）。
- 升级 Flutter 版本需要发新 release。

### Q5: 百分比放量对所有设备生效吗？

**A**: 仅 Flutter 3.35.7+ 的 release。
- 旧版本（< 3.35.7）的设备**不参与** rollout，只有放量到 100% 时才能获取 patch。
- 见官方文档 [How devices are chosen](https://docs.shorebird.dev/code-push/guides/percentage-based-rollouts/#how-devices-are-chosen)。

### Q6: 我有 10 个 flavor，需要 10 个 `app_id` 吗？

**A**: 是。
- Shorebird 根据 `applicationId`（Android）/ `bundleId`（iOS）区分应用。
- 每个不同的 `applicationId` 需要独立 `app_id`。
- 在 `shorebird.yaml` 可以用注释标记每个 flavor 的 `app_id`，或用环境变量 `SHOREBIRD_APP_ID` 切换。

### Q7: Patch 签名何时默认开启？

**A**:
- **Shorebird 1.6.135+** 创建的 release 默认启用 managed 签名。
- 旧版本需要手动开启（在 `shorebird init` 或 release 时选择）。
- 一旦 release，签名模式不可更改；要换模式需发新 release。

### Q8: iOS 用 `--obfuscate` 会崩溃吗?

**A**:
- **Flutter 3.47.9 at `6f0092351b`（Shorebird 1.6.140+）及更新版本**：修复了 iOS obfuscate 崩溃问题。
- **Flutter 3.44.9 及更早版本，或 3.47.9 早期构建**：iOS release 用 `--obfuscate` 或 `--split-debug-info` 打包后，patch 安装可能导致启动崩溃。
- 见官方 [Flutter version notes](https://docs.shorebird.dev/getting-started/flutter-version/)。

### Q9: 补丁有数量限制吗？

**A**: 无限制。
- 官方 FAQ 确认「没有 patch 数量上限」。
- 设备始终获取编号最大的未回滚 patch。

### Q10: 能否降低放量百分比（如从 50% 降到 10%）？

**A**: CLI 不支持。
- `shorebird patches rollout` 只能提升百分比。
- 要停止放量，可回滚 patch（`rollback`），所有已安装设备会卸载它。
- 或自行在应用内用 `package:shorebird_code_push` 的 track 机制控制（高级用法）。

---

## 💡 最佳实践

1. **先在 staging / beta track 验证，再放量到 stable**
   - 避免直接对全量用户发 patch。
   - 内部测试组或 1% 灰度用户先体验。

2. **Patch 参数与 release 保持一致**
   - 用脚本或 CI 变量存储 `--flavor` / `--dart-define-from-file` / `--target`，避免人工输错。

3. **监控崩溃率与性能指标**
   - 对接 Firebase Crashlytics、Sentry 等崩溃监控平台。
   - 放量每个阶段观察 4-24 小时，确认无异常再继续。

4. **小步快跑，快速回滚**
   - 补丁出问题时，立即 `shorebird patches rollback`。
   - 修复后重新发 patch（新的 patch number）。

5. **重要功能走商店审核，小修小补用 patch**
   - 核心功能变更、新功能上线：商店审核
   - Bug 修复、文案调整、样式微调：patch

6. **记录每个 patch 的变更内容**
   - 在 commit message 或 PR 中标注「[Shorebird Patch]」。
   - CI 中用 `shorebird api` 或 Console 查看 patch 历史。

7. **Flavor 与 Shorebird 配合的 CI 自动化**
   - 每个 flavor 独立 CI job，用环境变量切换 `SHOREBIRD_APP_ID`。
   - 例如：
     ```yaml
     # .github/workflows/patch-prod.yml
     env:
       SHOREBIRD_APP_ID: ${{ secrets.SHOREBIRD_APP_ID_PROD }}
     run: |
       shorebird patch android \
         --release-version ${{ github.event.inputs.version }} \
         --flavor prod \
         --target lib/main_prod.dart \
         --dry-run  # CI 先检查，通过后人工确认发布
     ```

8. **补丁签名用 managed 模式（除非有特殊安全要求）**
   - Managed 签名简化 CI 配置，团队成员无需管理私钥。
   - 企业若有合规要求（如「密钥必须本地保管」），选 self-managed。

9. **iOS 性能敏感代码避免频繁补丁**
   - 图像处理、音视频编解码、大数据集计算等，尽量在 release 时优化完毕。
   - Patch 后这些代码可能运行在解释器中（变慢）。

10. **定期清理旧 release 与 patch（可选）**
    - Shorebird Console 可以删除不再使用的 release（永久操作，谨慎）。
    - 保留至少最近 2-3 个 release，以便回滚或查看历史。

---

## 🔗 相关资源

### 本仓库资源

- 提示词模板：
  - [prompts/code-push/audit-patchable-change.md](../prompts/code-push/audit-patchable-change.md) — 审计变更是否可补丁
  - [prompts/code-push/setup-release-and-patch.md](../prompts/code-push/setup-release-and-patch.md) — init、release、patch 操作提示词
- 交叉工作流：
  - [workflows/flavor-multi-env.md](flavor-multi-env.md) — Flavor 多环境配置（与 Shorebird 配合）
  - [workflows/pigeon-platform-channels.md](pigeon-platform-channels.md) — 原生通信接口变更走商店包
  - [workflows/l10n-internationalization.md](l10n-internationalization.md) — l10n 生成代码可补丁

### Shorebird 官方文档

- **Code Push 首页**：[https://docs.shorebird.dev/code-push/](https://docs.shorebird.dev/code-push/)
- **Patch 命令详解**：[https://docs.shorebird.dev/code-push/patch/](https://docs.shorebird.dev/code-push/patch/)
- **Release 命令详解**：[https://docs.shorebird.dev/code-push/release/](https://docs.shorebird.dev/code-push/release/)
- **补丁签名指南**：[https://docs.shorebird.dev/code-push/guides/patch-signing/](https://docs.shorebird.dev/code-push/guides/patch-signing/)
- **百分比放量指南**：[https://docs.shorebird.dev/code-push/guides/percentage-based-rollouts/](https://docs.shorebird.dev/code-push/guides/percentage-based-rollouts/)
- **Track（部署通道）**：[https://docs.shorebird.dev/code-push/tracks/](https://docs.shorebird.dev/code-push/tracks/)
- **Flavor 支持**：[https://docs.shorebird.dev/guides/flavors/](https://docs.shorebird.dev/guides/flavors/)
- **add-to-app 支持**：[https://docs.shorebird.dev/guides/hybrid-app/](https://docs.shorebird.dev/guides/hybrid-app/)
- **商店合规 FAQ**：[https://docs.shorebird.dev/faq#store-compliance](https://docs.shorebird.dev/faq#store-compliance)
- **Flutter 版本支持矩阵**：[https://docs.shorebird.dev/getting-started/flutter-version/](https://docs.shorebird.dev/getting-started/flutter-version/)
- **Changelog**：[https://docs.shorebird.dev/changelog/](https://docs.shorebird.dev/changelog/)
- **package:shorebird_code_push**：[https://pub.dev/packages/shorebird_code_push](https://pub.dev/packages/shorebird_code_push)

---

**让热修复和快速迭代成为常态 🚀**
