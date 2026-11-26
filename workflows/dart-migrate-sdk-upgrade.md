# 用 dart migrate 做 Dart/Flutter SDK 语言版本升级

**自动迁移项目的 Dart 语言版本约束并应用必要的代码修复**

> 作者：Loong (SSLoong) | 日期：2026-11-26

本文对齐 Dart 3.14（2026-11-11 发布）引入的 `dart migrate` 命令，以及官方文档 [dart migrate](https://dart.dev/tools/dart-migrate) 和博客文章 [Announcing Dart 3.14](https://dart.dev/blog/announcing-dart-3-14)。

**版本要求**：
- 本机必须先安装 Dart 3.14 或更高版本（Flutter 用户需 Flutter 3.50+）
- 项目的最低 SDK 约束必须已经是 Dart 3.0 或更高（`pubspec.yaml` 中 `environment.sdk` ≥ `^3.0.0`）

**核心认知**：`dart migrate` 只更新**项目的语言版本约束和源码**，**不会**升级本机已安装的 Dart 或 Flutter SDK。本机 SDK 升级仍需走 [Get the Dart SDK](https://dart.dev/get-dart) 或 `flutter upgrade` 命令。

---

## 📋 背景

每次 Dart 新版本发布时，可能会引入新的语言特性、废弃旧语法、或要求代码进行预备修复（例如 Dart 3.13 将 `final` 和 `var` 保留给主构造函数的声明参数，要求移除非声明参数上的 `final`）。手工找出需要改哪些 lint、应用哪些 fix、何时提高 `pubspec.yaml` 的 SDK 约束，往往需要查阅 changelog 和多次试错。

`dart migrate` 自动化了这一流程：

1. 检查依赖兼容性
2. 应用该版本**必须的预备修复**（prepare）
3. 提升 `pubspec.yaml` 中的 `environment.sdk` 下限（bump）
4. 可选地应用**推荐的清理修复**（cleanup），以采用新版本的惯用写法

工具在任一步失败时会**保持文件不变**，并输出不兼容的依赖或错误文件清单，便于人工处理后重跑。

AI 在此流程中适合：
- 解读 `--dry-run` 摘要，评估影响范围
- 识别依赖阻塞并建议升级策略
- 判断 cleanup 修复是否应立刻应用
- 协助排查失败摘要和编译错误
- 起草提交说明

AI **不适合**：
- 在未 dry-run、未确认工作区干净时直接 `--apply`
- 把 `dart migrate` 当成本机 SDK 安装器（它不会改变 `dart --version` 的输出）

---

## 🎯 目标

- ✅ 判断该不该升级项目语言版本，以及何时升
- ✅ 先升级本机 SDK，再迁移项目语言版本
- ✅ 用 `--dry-run` 预览变更，审查 prepare/bump/cleanup 步骤
- ✅ 处理依赖阻塞（某些包尚不支持目标 SDK 版本）
- ✅ 安全执行 `--apply`，并用分析、测试、构建验证
- ✅ 多包项目与 Pub workspace 的迁移路径

---

## 🔧 适用场景

| 场景 | 适合度 | 说明 |
|------|--------|------|
| 新 Dart 版本发布后，升级项目语言版本 | ⭐⭐⭐⭐⭐ | 官方自动化路径 |
| 多个包需要统一升级到同一 SDK 版本 | ⭐⭐⭐⭐⭐ | 可一次传入多个目录 |
| 跨多个 minor 版本升级（如 3.12 → 3.14） | ⭐⭐⭐⭐⭐ | `--target-sdk` 会逐个 minor 递增迁移 |
| 大型代码库，希望分步审查修复 | ⭐⭐⭐⭐⭐ | 用 `--step` 单独跑 prepare、bump、cleanup |
| 本机 SDK 还未升级 | ⭐ | 必须先 `flutter upgrade` 或安装新 Dart SDK |
| 项目最低约束还在 Dart 2.x | ⭐ | 工具不支持；需先手工迁移到 Dart 3.0+ |
| 只想修 lint 或包 API，不改 SDK 约束 | ⭐ | 用 `dart fix`，不用 `dart migrate` |

**怎么选：**

- **dart migrate（本文）**：要提升项目的 `environment.sdk` 下限，自动应用该版本的必选/推荐修复。**不取决于** `analysis_options.yaml` 是否启用对应 lint。
- **dart fix**：修复当前语言版本下的诊断、lint 警告、包 API 的 data-driven 修复。**不改** SDK 约束，lint 修复只对 `analysis_options.yaml` 已启用的规则生效。
- **手工升级**：项目还在 Dart 2.x，或依赖链无法升级，只能人工处理。

---

## 🚀 完整流程

下面用「从 Dart 3.12 迁移到 3.13」示例。包名 `my_app` 换成你的 `pubspec.yaml` 的 `name`。

### Step 1: 判断该不该升级，以及何时升

**人工决策**：

1. **当前约束**：打开 `pubspec.yaml`，查看 `environment.sdk`。如果还是 `^2.19.0` 或更低，`dart migrate` 不支持，必须先手工迁移到 `^3.0.0`。
2. **本机 SDK 版本**：运行 `dart --version` 或 `flutter --version`。如果本机还在 Dart 3.12 或更低，必须先升级到 3.14+（Flutter 3.50+）。
3. **目标版本**：如果只想提升一个 minor（如 3.12 → 3.13），工具默认即可；如果想直接到 3.14，需传 `--target-sdk=3.14.0`，工具会逐个 minor 递增。
4. **依赖兼容性**：关键依赖（如 `flutter_riverpod`、`dio`）是否已支持目标 SDK？pub.dev 上查看包的 `pubspec.yaml`，或先 dry-run 让工具检查。
5. **团队节奏**：CI、同事的本机环境是否已升级？如果 CI 还在 Dart 3.12，不要先提升项目约束。

**AI 提示词**（可复制）：

```markdown
我想升级项目的 Dart 语言版本。请先确认，不要执行命令：

1. 当前 `pubspec.yaml` 的 `environment.sdk` 是多少？是否 ≥ 3.0.0？
2. 本机 `dart --version` 或 `flutter --version` 输出是什么？是否 ≥ 3.14 / Flutter 3.50？
3. 我想升级到哪个 SDK 版本（如 3.13、3.14）？
4. 关键依赖（列出 `pubspec.yaml` 的 dependencies）是否支持目标版本？
5. 是否有 CI 或团队成员的本机 SDK 还未升级？

如果本机 SDK 低于 3.14，先告诉我如何升级本机 SDK；如果项目约束低于 3.0，先告诉我手工升级路径。确认都满足后，再生成 dry-run 命令。
```

**检查点**：
- 本机 SDK 版本必须 ≥ 目标版本
- 项目当前约束必须 ≥ 3.0.0
- 关键依赖已有版本支持目标 SDK

---

### Step 2: 升级本机 Dart/Flutter SDK

**人工操作**（`dart migrate` 执行前必须完成）：

#### Flutter 用户

```bash
flutter upgrade
flutter --version
# 确认输出 "Dart 3.14.x" 或更高
```

#### 纯 Dart 用户

参考 [Get the Dart SDK](https://dart.dev/get-dart)，更新 Homebrew、Chocolatey、或下载最新安装包。

**AI 不能代替这一步。** `dart migrate` 不会改变 `dart --version` 的输出。

---

### Step 3: 确认工作区干净

`dart migrate --apply` **不会**检查 Git 未提交改动。如果有改动，工具会直接修改文件，造成难以回滚的混乱。

**人工操作**：

```bash
cd /path/to/my_app
git status
# 如果有未提交改动，先 commit 或 stash
git add -A
git commit -m "工作进度：迁移前备份"
```

**检查点**：`git status` 输出 `working tree clean`。

---

### Step 4: 预览变更（dry-run）

**人工操作**：

```bash
# 默认只提升一个 minor（如 3.12 → 3.13）
dart migrate --dry-run

# 或指定目标版本（如 3.12 → 3.14，会逐个 minor 递增）
dart migrate --dry-run --target-sdk=3.14.0
```

输出示例（官方文档示例，从 3.12 → 3.13）：

```
my_app:
  3.12.0 -> 3.13.0:
    Preparatory changes:
      2 changes would be made in 1 file.

      my_app/lib/greeter.dart
        avoid_final_parameters • 2 changes

    SDK constraint:
      Would bump ^3.12.0 -> ^3.13.0

Optional cleanup fixes can modernize your code further:

  my_app: 1 change in 1 file
    empty_container_bodies • 1 change

To preview them, apply this migration, run 'dart pub get', and then run 'dart migrate --dry-run --step=cleanup' on the same packages.

To apply the proposed changes, run:
  dart migrate --apply
```

**AI 提示词**（可复制）：见 [prompts/sdk/preview-dart-migrate.md](../prompts/sdk/preview-dart-migrate.md)。

**审查要点**：

1. **Preparatory changes**：这些是**必须**在提升约束前应用的修复，否则新语言版本下会编译失败或行为改变。
2. **SDK constraint**：确认目标版本是否符合预期。
3. **Optional cleanup fixes**：这些是**可选**的惯用写法修复，不影响编译。是否立刻应用取决于团队风格和代码审查带宽。
4. **依赖阻塞**：如果输出「dependencies are incompatible」，工具会列出需要先更新的包。处理后重新 dry-run。

---

### Step 5: 处理依赖阻塞（如果有）

如果 dry-run 输出：

```
The following dependencies do not support SDK ^3.13.0:
  - package_a ^1.0.0 (requires SDK ^3.0.0 <3.13.0)
  - package_b ^2.5.0 (requires SDK >=3.0.0 <3.13.0)
```

**人工操作**：

1. 打开 pub.dev，搜索这些包，查看是否有新版本支持 3.13。
2. 更新 `pubspec.yaml` 的版本约束，或 `dart pub upgrade <package_name>`。
3. 如果包尚无支持版本，联系维护者或临时 fork，或等待。
4. 更新依赖后，重新 `dart migrate --dry-run`。

**AI 提示词**（可复制）：

```markdown
`dart migrate --dry-run` 报告以下依赖不兼容：

[粘贴完整报错]

请：
1. 在 pub.dev 上查每个包的最新版本，列出其 SDK 约束。
2. 建议 `pubspec.yaml` 应改成哪个版本范围。
3. 如果某包确实不支持，给出替代方案或等待建议。
```

**检查点**：依赖都兼容后，dry-run 不再报阻塞错误。

---

### Step 6: 应用迁移（apply）

**人工操作**：

```bash
# 应用 prepare + bump（默认）
dart migrate --apply

# 或同时应用 cleanup
dart migrate --apply --step=all

# 或只应用 prepare，稍后再 bump
dart migrate --apply --step=prepare
```

输出示例（成功）：

```
my_app:
  3.12.0 -> 3.13.0:
    Preparatory changes:
      2 changes made in 1 file.

      my_app/lib/greeter.dart
        avoid_final_parameters • 2 changes

    SDK constraint:
      Bumped ^3.12.0 -> ^3.13.0

Optional cleanup fixes can modernize your code further:

  my_app: 1 change in 1 file
    empty_container_bodies • 1 change

To preview them, run 'dart pub get' and then 'dart migrate --dry-run --step=cleanup' on the same packages.
```

**失败时**：工具会保持文件不变，并输出错误文件。修复后重跑 `--apply`。

**AI 提示词**（可复制）：见 [prompts/sdk/apply-and-verify-dart-migrate.md](../prompts/sdk/apply-and-verify-dart-migrate.md)。

**检查点**：
- `pubspec.yaml` 的 `environment.sdk` 已更新
- 源码中必要的修复已应用
- 工具退出码为 0

---

### Step 7: 刷新依赖

SDK 约束变化后，依赖解析可能更新。

```bash
dart pub get
```

如果某些依赖在新约束下不兼容，`pub get` 会报错。按提示升级或等待。

---

### Step 8: 验证（必须）

**人工操作**：

```bash
# 1. 分析（新语言版本下是否有错误）
dart analyze

# 2. 单元测试
flutter test

# 3. 编译（Android / iOS）
flutter build apk --debug
flutter build ios --no-codesign --debug

# 4. 如果有集成测试或 Patrol 测试，也跑一遍
flutter test integration_test/
```

**常见失败**：

| 现象 | 原因 | 处理 |
|------|------|------|
| `dart analyze` 报新错误 | 新语言版本的规则更严格 | 修复源码，不要回退约束 |
| `flutter test` 失败 | 行为变化或依赖不兼容 | 修复测试，检查 mocking 包版本 |
| `flutter build` 失败 | 原生侧依赖或配置不兼容 | 更新 Gradle、CocoaPods 依赖 |
| cleanup 修复太激进 | 团队不接受惯用写法 | 只应用 prepare + bump，cleanup 单独 PR |

**AI 提示词**（可复制）：

```markdown
`dart migrate --apply` 执行后，`[dart analyze / flutter test / flutter build]` 报以下错误：

[粘贴完整报错]

请：
1. 判断是新语言版本引入的规则，还是依赖不兼容。
2. 给出修复方案（修改源码、更新依赖、或调整配置）。
3. 如果是已知 breaking change，引用官方 changelog。
```

**检查点**：
- [ ] `dart analyze` 通过
- [ ] `flutter test` 全部通过
- [ ] `flutter build apk --debug` 通过
- [ ] `flutter build ios --no-codesign --debug` 通过

---

### Step 9: 多包与 workspace

#### 多包项目

工具**不会**自动展开 Pub workspace。必须显式传入每个包路径：

```bash
dart migrate --dry-run my_app packages/my_pkg packages/another_pkg
dart migrate --apply my_app packages/my_pkg packages/another_pkg
```

#### Pub workspace

假设 `my_workspace/pubspec.yaml` 定义了 workspace，包含 `packages/pkg_a`、`packages/pkg_b`：

```bash
cd my_workspace
dart migrate --apply packages/pkg_a packages/pkg_b
```

工具会逐个迁移，依赖检查会考虑 workspace 解析。

**AI 提示词**（可复制）：

```markdown
我的项目是 Pub workspace，结构如下：

```
my_workspace/
├── pubspec.yaml  # workspace: ["packages/*"]
├── packages/
│   ├── pkg_a/
│   │   └── pubspec.yaml
│   └── pkg_b/
│       └── pubspec.yaml
```

请生成迁移命令，确保每个包都传给 `dart migrate`。
```

---

### Step 10: 可选的 cleanup 修复

如果 dry-run 或 apply 后提示有 cleanup 修复：

```bash
# 预览 cleanup
dart migrate --dry-run --step=cleanup

# 应用 cleanup
dart migrate --apply --step=cleanup
```

cleanup 修复通常包括：
- 采用新版本的惯用写法（如 `empty_container_bodies`：空类体从 `{}` 改成 `;`）
- 简化表达式（如 `unnecessary_parenthesis`）

是否立刻应用取决于：
1. 团队是否接受自动化风格改动
2. 代码审查带宽（cleanup 可能改动很多行）
3. 是否与已有 PR 冲突

**最佳实践**：prepare + bump 合入主分支后，cleanup 单独开 PR，便于审查和回退。

---

### Step 11: 提交与推送

**人工操作**：

```bash
git add -A
git commit -m "迁移：Dart SDK 语言版本从 3.12 升至 3.13

- 应用 avoid_final_parameters 预备修复
- 提升 pubspec.yaml 的 environment.sdk 至 ^3.13.0
- 验证通过：dart analyze、flutter test、flutter build"

git push origin <branch-name>
```

提交信息建议包含：
- 旧版本 → 新版本
- 应用了哪些 prepare/cleanup 修复（lint 名称）
- 验证命令清单

---

## ✅ 验证清单

完成后确认：

- [ ] 本机 SDK 版本 ≥ 目标版本（`dart --version` 或 `flutter --version`）
- [ ] 项目当前约束 ≥ 3.0.0
- [ ] 执行前 `git status` 显示 `working tree clean`
- [ ] `dart migrate --dry-run` 预览无阻塞依赖
- [ ] `dart migrate --apply` 退出码为 0
- [ ] `pubspec.yaml` 的 `environment.sdk` 已更新
- [ ] `dart pub get` 成功
- [ ] `dart analyze` 通过
- [ ] `flutter test` 全部通过
- [ ] `flutter build apk --debug` 与 `flutter build ios --no-codesign --debug` 通过
- [ ] 多包项目每个包都传给 `dart migrate`
- [ ] cleanup 修复按团队约定应用或单独 PR

---

## 🤔 FAQ

### Q1: dart migrate 会升级我的本机 SDK 吗？

**A**: **不会**。它只更新项目的 `pubspec.yaml` 和源码。本机 SDK 升级必须走 `flutter upgrade` 或 [Get the Dart SDK](https://dart.dev/get-dart)。

### Q2: 我的项目还在 Dart 2.19，能用 dart migrate 吗？

**A**: **不能**。工具只支持最低约束 ≥ 3.0.0 的项目。先手工迁移到 Dart 3.0：
1. 升级本机 SDK 到 3.0+
2. 手工改 `pubspec.yaml` 的 `environment.sdk: ^3.0.0`
3. 运行 `dart fix --apply` 修复 Dart 3.0 breaking changes
4. 通过 `dart analyze` 和 `flutter test` 后，再用 `dart migrate` 升级到更高版本

### Q3: dry-run 说依赖不兼容，怎么办？

**A**: 工具会列出不兼容的包和它们的 SDK 约束。处理方式：
1. 在 pub.dev 上查新版本，更新 `pubspec.yaml`
2. 如果包无新版本，检查是否有替代包
3. 联系包维护者或临时 fork
4. 更新依赖后重新 dry-run

### Q4: 应该一次跨多个 minor 升级（如 3.12 → 3.14），还是逐个升级？

**A**: 工具支持 `--target-sdk`，会**自动逐个 minor 递增**。例如 3.12 → 3.14 时，工具会先 3.12 → 3.13，再 3.13 → 3.14。
- **推荐**：让工具自动递增。每个 minor 的修复都会应用。
- **不推荐**：手工跳版本。可能遗漏中间版本的必要修复。

### Q5: prepare、bump、cleanup 是什么？

**A**: 迁移分为四个阶段：

1. **依赖检查**：验证 `pubspec.yaml` 的依赖是否支持目标 SDK。不兼容时停止并报告。
2. **prepare**：应用**必须**在提升约束前完成的修复（如移除非声明参数的 `final`）。不完成 prepare，新语言版本下会编译失败。
3. **bump**：提升 `pubspec.yaml` 的 `environment.sdk` 下限，并按新语言版本重新分析。
4. **cleanup**：可选的惯用写法修复（如空类体从 `{}` 改成 `;`）。默认只在摘要里列出，不自动应用。

用 `--step` 单独跑某些步骤：
```bash
# 只跑 prepare，不 bump（大型代码库先审查修复）
dart migrate --apply --step=prepare

# prepare 审查通过后，再 bump
dart migrate --apply --step=bump

# 最后单独 PR 应用 cleanup
dart migrate --apply --step=cleanup
```

默认 `dart migrate --apply` 等价于 `--step=prepare,bump`。

### Q6: 执行 --apply 后失败，文件会被改坏吗？

**A**: **不会**。任一步失败时，工具**保持源码和 pubspec.yaml 不变**，只输出错误摘要。修复后重跑 `--apply`。

### Q7: dart migrate 和 dart fix 有什么区别？

**A**:

| 工具 | 改不改 SDK 约束 | 修什么 | 是否取决于 analysis_options.yaml |
|------|----------------|--------|----------------------------------|
| `dart migrate` | ✅ 改（提升 `environment.sdk`） | 该版本的**必选/推荐**迁移修复 | ❌ 不取决（工具内置规则） |
| `dart fix` | ❌ 不改 | 当前语言版本下的诊断、**已启用的** lint、包 API 修复 | ✅ lint 修复只对已启用规则生效 |

**场景**：
- 要升级项目语言版本 → `dart migrate`
- 只想修当前版本的 lint 警告 → `dart fix`

### Q8: Flutter 项目的 pubspec.yaml 里是不是应该写 Flutter 版本号？

**A**: **不是**。`pubspec.yaml` 的 `environment.sdk` 是 **Dart 语言版本约束**，不是 Flutter 版本号。

```yaml
# ✅ 正确：Dart SDK 约束
environment:
  sdk: ^3.13.0

# ❌ 错误：没有 environment.flutter 字段
environment:
  flutter: ^3.50.0  # 这样写无效
```

Flutter 版本与 Dart 版本的对应关系（2026-11-11）：
- Flutter 3.50 搭配 Dart 3.14
- Flutter 3.44 搭配 Dart 3.12

升级 Flutter 版本用 `flutter upgrade`，升级项目 Dart 语言版本用 `dart migrate`。

### Q9: 多 Engine 或 add-to-app 项目能用 dart migrate 吗？

**A**: **能**。`dart migrate` 只改 Dart 侧代码，不涉及 Engine 配置。但要注意：
1. 所有 Engine 的 Flutter 版本必须一致，否则 Dart 编译产物不兼容。
2. 宿主原生侧（Android、iOS）的 Gradle / CocoaPods 依赖也要兼容新 Flutter 版本。

### Q10: CI 构建失败了，说 SDK 约束不满足？

**A**: 迁移后 `pubspec.yaml` 要求 `^3.13.0`，但 CI 的 Dart/Flutter 还在 3.12。处理：
1. 更新 CI 配置，升级 Docker 镜像或 `actions/setup-flutter` 版本
2. 或暂时不合入迁移 PR，等 CI 升级后再合入

---

## 💡 最佳实践

1. **先升本机，再迁项目**
   - `dart migrate` 目标版本不能新于本机 SDK。先 `flutter upgrade`，再 `dart migrate`。

2. **每次升级前 dry-run**
   - 预览变更范围，提前发现依赖阻塞。

3. **工作区必须干净**
   - 工具不检查未提交改动。先 commit 或 stash，避免混乱。

4. **大型代码库分步走**
   - 用 `--step=prepare` 先审查修复，通过后再 `--step=bump`。cleanup 单独 PR。

5. **多包项目显式传路径**
   - 工具不自动展开 workspace。每个包路径都要传给命令。

6. **验证全覆盖**
   - 分析、测试、构建都跑一遍。不要只跑 `dart analyze`。

7. **cleanup 按团队节奏**
   - 自动化风格改动可能影响代码审查。prepare + bump 先合入，cleanup 单独 PR 或延后。

8. **CI 与团队同步升级**
   - 不要先提升项目约束，导致 CI 或同事编不过。协调好顺序。

---

## 🔗 相关资源

- 预览迁移摘要：[prompts/sdk/preview-dart-migrate.md](../prompts/sdk/preview-dart-migrate.md)
- 应用并验证迁移：[prompts/sdk/apply-and-verify-dart-migrate.md](../prompts/sdk/apply-and-verify-dart-migrate.md)
- 官方文档：[https://dart.dev/tools/dart-migrate](https://dart.dev/tools/dart-migrate)
- Dart 3.14 公告：[https://dart.dev/blog/announcing-dart-3-14](https://dart.dev/blog/announcing-dart-3-14)
- Dart CLI 工具：[https://dart.dev/tools/dart-tool](https://dart.dev/tools/dart-tool)
- 获取 Dart SDK：[https://dart.dev/get-dart](https://dart.dev/get-dart)
- Flutter 3.50 公告：[https://flutter.dev/blog/whats-new-in-flutter-3-50](https://flutter.dev/blog/whats-new-in-flutter-3-50)

---

**先升本机，再 dry-run，再 apply，最后验证。**
