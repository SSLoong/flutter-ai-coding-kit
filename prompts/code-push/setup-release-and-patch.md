# 提示词：Shorebird Release 与 Patch 操作

**用途**：从 `shorebird init`、签名选择、`release`、`patch`、track 管理到放量与回滚的完整操作提示词

---

## 📋 适用场景

- 首次为项目配置 Shorebird（init）
- 创建可补丁的 release 并提交商店
- 发布 patch 到 staging / beta / stable track
- 分阶段放量、回滚、查看指标

---

## 📥 输入要求

在 Cursor 中准备：

1. **项目信息**：
   - `pubspec.yaml` 的 `name` 和 `version`（如 `my_app`、`1.0.0+1`）
   - 是否有 Flavor（dev / staging / prod）？
   - Flavor 对应的 `--target` 文件（如 `lib/main_prod.dart`）
   - Flavor 对应的 `--dart-define-from-file` 配置（如 `config/prod.json`）

2. **操作场景**：
   - 首次初始化？
   - 创建 release？
   - 发布 patch？
   - 放量 / 回滚？

---

## 💬 提示词模板

### 场景 1：首次初始化 Shorebird

```markdown
为我的 Flutter 项目初始化 Shorebird Code Push，并给出后续 release 命令。

## 项目信息

- **App 名称**：[如 MyApp]
- **pubspec.yaml name**：`[my_app]`
- **是否有 Flavor**：[是 / 否]
  - 如果有，列出：dev、staging、prod（分别对应 `lib/main_dev.dart` 等）
- **Android applicationId**：
  - prod: `com.example.myapp`
  - dev: `com.example.myapp.dev`
  - staging: `com.example.myapp.staging`
- **iOS Bundle ID**：同上

## 请输出

1. **初始化命令**：`shorebird init`（我会手动执行）
2. **签名模式选择建议**：
   - Managed（托管）/ Self-managed（自管理）/ Unsigned（无签名）
   - 给出每种模式的适用场景
3. **Flavor 处理**：
   - 如果有多个 Flavor，如何为每个 Flavor 配置独立 `app_id`？
   - 是否需要多次 `shorebird init`，或在 `shorebird.yaml` 中标注？
4. **首次 release 命令**（每个 Flavor 一条）：
   - Android：`shorebird release android --flavor ... --target ...`
   - iOS：`shorebird release ios --flavor ... --target ...`
5. **检查清单**：init 后应确认哪些文件和配置
```

---

### 场景 2：创建 Release

```markdown
为我的 Flutter 项目创建 Shorebird release，准备提交商店。

## 项目信息

- **Release 版本**：`[1.0.0+1]`（`pubspec.yaml` 的 `version`）
- **Flavor**：`[prod]`（如果有）
- **Target 文件**：`[lib/main_prod.dart]`（如果有 Flavor）
- **dart-define 配置**：`[config/prod.json]`（如果有）
- **是否需要混淆**：`--obfuscate`（是 / 否）
- **平台**：Android / iOS / 两者都要

## 请输出

1. **完整的 release 命令**：
   - Android：
     \`\`\`bash
     shorebird release android \\
       --flavor prod \\
       --target lib/main_prod.dart \\
       --dart-define-from-file=config/prod.json \\
       [--obfuscate --split-debug-info=build/app/outputs/symbols]
     \`\`\`
   - iOS：同理
2. **签名模式确认**：
   - 如果是首次 release，提示「此次 release 将锁定签名模式（managed / self-managed / unsigned），后续 patch 沿用，无法更改」
3. **构建产物位置**：
   - Android：`build/app/outputs/bundle/release/app-[flavor]-release.aab`
   - iOS：`build/ios/archive/Runner.xcarchive`
4. **提交商店步骤**：
   - Android：上传 `.aab` 到 Google Play Console
   - iOS：在 Xcode 中归档并上传 App Store Connect
5. **参数记录**：提醒我「保存这些参数（`--flavor` / `--target` / `--dart-define-from-file`），后续 patch 必须完全一致」

## 检查清单

- [ ] `shorebird release` 退出码为 0
- [ ] `.aab` / `.xcarchive` 已生成
- [ ] `shorebird.yaml` 中 `app_id` 对应此 Flavor（如有多个）
- [ ] 已记录 release 参数（用于后续 patch）
```

---

### 场景 3：发布 Patch

```markdown
为已发布的 release 创建 Shorebird patch，修复线上问题。

## 背景

- **Release 版本**：`[1.0.0+1]`
- **Patch 内容**：[如「修复空指针崩溃」「更新支付回调逻辑」]
- **变更文件**：[如 `lib/screens/profile_screen.dart`]
- **Flavor**：`[prod]`（如果有）
- **Release 时的参数**：
  - `--flavor prod`
  - `--target lib/main_prod.dart`
  - `--dart-define-from-file=config/prod.json`
- **目标 track**：staging（先验证）/ stable（直接全量）
- **是否需要放量**：是（建议 10% → 50% → 100%）/ 否（直接 100%）

## 请输出

1. **完整的 patch 命令**：
   - Android：
     \`\`\`bash
     shorebird patch android \\
       --release-version 1.0.0+1 \\
       --flavor prod \\
       --target lib/main_prod.dart \\
       --dart-define-from-file=config/prod.json \\
       --track staging \\
       --dry-run  # 先检查，通过后去掉此参数
     \`\`\`
   - iOS：同理
2. **放量计划**（如果选择放量）：
   - 阶段 1：`--rollout 1`（持续 4 小时，监控崩溃率）
   - 阶段 2：`--rollout 10`（持续 12 小时）
   - 阶段 3：`--rollout 50`（持续 24 小时）
   - 阶段 4：`--rollout 100`（全量）
   - 每阶段的放量命令：
     \`\`\`bash
     shorebird patches rollout \\
       --release-version 1.0.0+1 \\
       --patch-number 1 \\
       --percentage <10/50/100>
     \`\`\`
3. **验证步骤**：
   - 在 staging track 测试：`shorebird preview --release-version 1.0.0+1`
   - 或发给内部测试组
4. **提升到 stable**：
   \`\`\`bash
   shorebird patches promote \\
     --release-version 1.0.0+1 \\
     --patch-number 1 \\
     --rollout 1  # 初始 1%
   \`\`\`
5. **监控指标**：
   - 崩溃率：`< 0.1%`（基线 0.05%）
   - 采纳率：
     \`\`\`bash
     shorebird metrics adoption --release-version 1.0.0+1
     \`\`\`
6. **回滚命令**（如发现问题）：
   \`\`\`bash
   shorebird patches rollback \\
     --release-version 1.0.0+1 \\
     --patch-number 1
   \`\`\`

## 检查清单

- [ ] `--dry-run` 通过（无原生/资产变更警告）
- [ ] 参数与 release 时完全一致
- [ ] 已在 staging track 验证，无崩溃
- [ ] 放量每阶段观察至少 4 小时
```

---

### 场景 4：管理 Track 与放量

```markdown
管理已发布的 patch 的部署 track 和放量百分比。

## 背景

- **Release 版本**：`[1.0.0+1]`
- **Patch 编号**：`[1]`
- **当前 track**：`[staging]`
- **当前放量百分比**：`[不适用 / 10% / 100%]`
- **操作**：
  - 提升到 stable
  - 提高放量百分比（10% → 50%）
  - 回滚 patch
  - 查看采纳率

## 请输出

### 1. 提升到 stable（初始 1% 放量）

\`\`\`bash
shorebird patches promote \\
  --release-version 1.0.0+1 \\
  --patch-number 1 \\
  --rollout 1
\`\`\`

### 2. 提高放量百分比

\`\`\`bash
# 提升到 10%
shorebird patches rollout \\
  --release-version 1.0.0+1 \\
  --patch-number 1 \\
  --percentage 10

# 提升到 50%
shorebird patches rollout \\
  --release-version 1.0.0+1 \\
  --patch-number 1 \\
  --percentage 50

# 提升到 100%（全量）
shorebird patches rollout \\
  --release-version 1.0.0+1 \\
  --patch-number 1 \\
  --percentage 100
\`\`\`

### 3. 回滚 patch

\`\`\`bash
shorebird patches rollback \\
  --release-version 1.0.0+1 \\
  --patch-number 1
\`\`\`

- 设备会在下次检查更新时自动卸载此 patch
- 可用 `rollforward` 恢复：
  \`\`\`bash
  shorebird patches rollforward \\
    --release-version 1.0.0+1 \\
    --patch-number 1
  \`\`\`

### 4. 查看采纳率与 patch 列表

\`\`\`bash
# 查看设备采纳率
shorebird metrics adoption --release-version 1.0.0+1

# 列出所有 patch
shorebird patches list --release-version 1.0.0+1

# 查看 patch 详情
shorebird patches info --release-version 1.0.0+1 --patch-number 1
\`\`\`

## 注意事项

- 放量只能提升，不能降低（降低需回滚整个 patch）
- 放量需要 Shorebird 1.6.139+ 和 Flutter 3.35.7+ 的 release
- 旧版本设备（< 3.35.7）只有在放量到 100% 时才能获取 patch
```

---

### 场景 5：CI/CD 集成

```markdown
为 CI/CD 管道生成 Shorebird release 和 patch 命令。

## CI 环境

- **平台**：GitHub Actions / GitLab CI / Jenkins
- **触发条件**：
  - Release：手动触发或打 tag（如 `v1.0.0+1`）
  - Patch：PR 合并到 main 分支
- **Flavor**：`[prod]`
- **参数**：
  - `--flavor prod`
  - `--target lib/main_prod.dart`
  - `--dart-define-from-file=config/prod.json`
- **Secrets**：
  - `SHOREBIRD_TOKEN`（CLI 认证 token）
  - `SHOREBIRD_APP_ID`（如有多个 Flavor）

## 请输出

### 1. Release 工作流（GitHub Actions 示例）

\`\`\`yaml
name: Shorebird Release

on:
  workflow_dispatch:  # 手动触发
    inputs:
      version:
        description: 'Release version (e.g. 1.0.0+1)'
        required: true

jobs:
  release:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      
      - uses: subosito/flutter-action@v2
        with:
          flutter-version: '3.50.1'
      
      - name: Install Shorebird
        run: |
          curl -fsSL https://raw.githubusercontent.com/shorebirdtech/install/main/install.sh | bash
          echo "$HOME/.shorebird/bin" >> $GITHUB_PATH
      
      - name: Shorebird Login
        run: |
          echo "${{ secrets.SHOREBIRD_TOKEN }}" | shorebird login --ci
      
      - name: Create Release
        run: |
          shorebird release android \\
            --flavor prod \\
            --target lib/main_prod.dart \\
            --dart-define-from-file=config/prod.json \\
            --no-codesign  # CI 中通常用 fastlane 单独签名
      
      - name: Upload AAB
        uses: actions/upload-artifact@v3
        with:
          name: app-prod-release.aab
          path: build/app/outputs/bundle/release/app-prod-release.aab
\`\`\`

### 2. Patch 工作流（GitHub Actions 示例）

\`\`\`yaml
name: Shorebird Patch

on:
  push:
    branches: [main]

jobs:
  patch:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      
      - uses: subosito/flutter-action@v2
        with:
          flutter-version: '3.50.1'
      
      - name: Install Shorebird
        run: |
          curl -fsSL https://raw.githubusercontent.com/shorebirdtech/install/main/install.sh | bash
          echo "$HOME/.shorebird/bin" >> $GITHUB_PATH
      
      - name: Shorebird Login
        run: |
          echo "${{ secrets.SHOREBIRD_TOKEN }}" | shorebird login --ci
      
      - name: Dry Run Patch (检查)
        run: |
          shorebird patch android \\
            --release-version latest \\
            --flavor prod \\
            --target lib/main_prod.dart \\
            --dart-define-from-file=config/prod.json \\
            --track staging \\
            --dry-run
      
      - name: Create Patch (staging track)
        if: success()
        run: |
          shorebird patch android \\
            --release-version latest \\
            --flavor prod \\
            --target lib/main_prod.dart \\
            --dart-define-from-file=config/prod.json \\
            --track staging
\`\`\`

### 3. 获取 Shorebird Token

在本地执行：

\`\`\`bash
shorebird login:ci
\`\`\`

将输出的 token 保存到 CI secrets 中（名为 `SHOREBIRD_TOKEN`）。

## 注意事项

- CI 中始终用 `--release-version` 明确指定版本，避免自动推断
- `--dry-run` 在 PR 检查中运行，通过后再实际发布
- Flavor 多环境用矩阵构建或多个 job
```

---

## ✅ 验证步骤

每次操作后确认：

```bash
# Release 后
shorebird releases list  # 确认新 release 已创建

# Patch 后
shorebird patches list --release-version <version>  # 确认 patch 已发布

# 放量后
shorebird patches info --release-version <version> --patch-number <number>
# 查看 track 和 rollout 百分比
```

---

## 🛠️ 常见问题

1. **`shorebird patch` 报 `Native code changes detected`**
   - 确认 `android/`、`ios/` 无变更（用 `git diff android/ ios/`）
   - 如确认无关紧要，用 `--allow-native-diffs`（不推荐）

2. **参数与 release 不一致，报错或警告**
   - 保存 release 时的参数到文件或脚本变量
   - CI 中用环境变量统一管理

3. **Flavor 多环境时 `app_id` 混乱**
   - 每个 Flavor 在 CI 中用单独 job，用 `SHOREBIRD_APP_ID` 环境变量切换
   - 或在 `shorebird.yaml` 中注释标明每个 `app_id` 对应哪个 Flavor

4. **放量后看不到百分比变化**
   - 确认 release 是用 Flutter 3.35.7+ 构建
   - 旧设备（< 3.35.7）不参与放量，只有 100% 时才获取

5. **iOS patch 后启动崩溃**
   - 检查 Flutter 版本（3.47.9 at `6f0092351b` 或更新版本修复了 obfuscate 崩溃）
   - 见 [Flutter version notes](https://docs.shorebird.dev/getting-started/flutter-version/)

---

## 🔗 相关

- 工作流：[workflows/shorebird-code-push.md](../../workflows/shorebird-code-push.md)
- 审计提示词：[audit-patchable-change.md](audit-patchable-change.md)
- Shorebird CLI 命令参考：[https://docs.shorebird.dev/reference/commands/](https://docs.shorebird.dev/reference/commands/)
- CI 集成指南：[https://docs.shorebird.dev/guides/ci-cd/](https://docs.shorebird.dev/guides/ci-cd/)
