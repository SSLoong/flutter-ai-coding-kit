# 提示词：预览 dart migrate 变更

**用途**：生成并解读 `dart migrate --dry-run` 命令，审查 prepare/bump/cleanup 步骤，输出变更摘要与建议

---

## 📋 适用场景

- 新 Dart 版本发布后，想升级项目语言版本但不确定影响范围
- 多个包需要统一升级，先预览每个包的变更
- 大型代码库，希望分步审查 prepare/bump/cleanup

---

## 📥 输入要求

在 Cursor 中准备：

1. **当前项目结构**：打开 `pubspec.yaml`，确认 `environment.sdk` 的当前值
2. **本机 SDK 版本**：运行 `dart --version` 或 `flutter --version`，确认 ≥ 3.14 / Flutter 3.50
3. **目标版本**（可选）：如果不指定，工具默认提升一个 minor；如果要跨多个 minor（如 3.12 → 3.14），需传 `--target-sdk`
4. **多包项目**：列出需要迁移的包路径（如 `packages/pkg_a`、`packages/pkg_b`）
5. **Pub workspace**：如果是 workspace，确认每个包都要传给命令

---

## 💬 提示词模板

```markdown
我想预览 Dart SDK 语言版本升级的影响。请生成 `dart migrate --dry-run` 命令并解读输出。

**当前状态**：
- 本机 SDK 版本：[粘贴 `dart --version` 或 `flutter --version` 输出]
- 项目 `pubspec.yaml` 的 `environment.sdk`：[当前值，如 ^3.12.0]
- 是否多包项目：[是/否]，包路径：[列出所有包路径，如 my_app, packages/pkg_a]
- 是否 Pub workspace：[是/否]

**目标**：
- 目标 SDK 版本：[默认提升一个 minor / 或指定如 3.14.0]
- 是否只预览 cleanup：[否（默认预览 prepare + bump）/ 是（只预览 cleanup）]

**请执行**：

1. 生成对应的 `dart migrate --dry-run` 命令（多包项目要传所有路径）
2. 执行命令，粘贴完整输出
3. 解读输出，分类说明：
   - **Preparatory changes**：哪些 lint 修复？影响哪些文件？是否会改变行为？
   - **SDK constraint**：从哪个版本提升到哪个版本？
   - **Optional cleanup fixes**：有哪些可选修复？是否建议立刻应用？
4. 判断是否有依赖阻塞（incompatible dependencies）：
   - 如果有，列出每个不兼容包的当前版本和 SDK 约束
   - 在 pub.dev 上查新版本（执行搜索），给出更新建议
5. 建议下一步：
   - 如果无阻塞，是否可以直接 `--apply`
   - 如果有阻塞，先更新哪些依赖
   - 如果 cleanup 修复很多，是否应分成两个 PR

**约束**：
- 只预览，不要执行 `--apply`
- 如果本机 SDK < 3.14，先告诉我如何升级本机 SDK
- 如果项目约束 < 3.0.0，先告诉我手工升级路径
```

---

## ✅ 验证步骤

预览后确认：

```bash
# AI 生成的命令应类似：
dart migrate --dry-run
# 或
dart migrate --dry-run --target-sdk=3.14.0
# 或（多包）
dart migrate --dry-run my_app packages/pkg_a packages/pkg_b
```

**检查清单**：

- [ ] 命令已执行，输出完整粘贴给 AI
- [ ] AI 已分类解读 prepare/bump/cleanup
- [ ] 如果有依赖阻塞，AI 已给出更新建议
- [ ] AI 已明确下一步（直接 apply / 先更新依赖 / 分步走）
- [ ] 多包项目每个包路径都传给命令

---

## 🛠️ 常见问题

1. **AI 只给了命令，没有解读输出**
   - 在 Cursor 里执行命令，粘贴完整输出回复给 AI，要求解读。

2. **输出说「dependencies are incompatible」**
   - AI 应给出每个包的 pub.dev 链接和更新建议。如果没给，追问：「请在 pub.dev 上查这些包的最新版本，列出 SDK 约束。」

3. **AI 建议直接 --apply，但我还没确认工作区干净**
   - 追问：「执行前我需要确认 `git status` 是 clean，请先生成检查命令。」

4. **cleanup 修复很多，AI 没有给分步建议**
   - 追问：「cleanup 修复涉及多少文件？是否建议 prepare + bump 先合入，cleanup 单独 PR？」

5. **多包项目 AI 只给了一个包的命令**
   - 追问：「我有多个包需要迁移，请给出包含所有路径的命令。」

---

## 🔗 相关

- 工作流：[workflows/dart-migrate-sdk-upgrade.md](../../workflows/dart-migrate-sdk-upgrade.md)
- 应用与验证：[apply-and-verify-dart-migrate.md](apply-and-verify-dart-migrate.md)
- 官方文档：[https://dart.dev/tools/dart-migrate](https://dart.dev/tools/dart-migrate)
