# 提示词：应用并验证 dart migrate

**用途**：在 dry-run 审查通过、工作区干净的前提下，安全执行 `dart migrate --apply`，处理失败，并用分析/测试/构建验证

---

## 📋 适用场景

- dry-run 预览已完成，确认变更可接受
- 准备正式应用迁移并提交
- 应用后遇到编译、测试、或构建错误

---

## 📥 输入要求

在 Cursor 中准备：

1. **dry-run 结果**：已执行 `dart migrate --dry-run`，确认无依赖阻塞
2. **工作区状态**：`git status` 显示 `working tree clean`，或已 commit/stash
3. **验证策略**：是否要跑 `dart analyze`、`flutter test`、`flutter build`
4. **步骤选择**（可选）：
   - 默认：`prepare + bump`
   - 全部：`prepare + bump + cleanup`
   - 分步：先 `prepare`，审查后再 `bump`

---

## 💬 提示词模板

```markdown
dry-run 预览已完成，现在请执行 `dart migrate --apply` 并验证。

**当前状态**：
- 项目路径：[当前目录或包路径列表]
- 是否多包项目：[是/否]，包路径：[列出所有包路径]
- 目标 SDK 版本：[如 3.13.0 / 默认提升一个 minor]
- 是否应用 cleanup：[否（默认只 prepare + bump）/ 是（全部应用）]
- 工作区是否干净：[粘贴 `git status` 输出]

**请执行**：

1. **检查工作区**：
   - 执行 `git status`
   - 如果有未提交改动，先生成 commit 或 stash 命令，等我确认后继续

2. **应用迁移**：
   - 生成对应的 `dart migrate --apply` 命令（多包项目传所有路径）
   - 如果我要求只应用 prepare，用 `--step=prepare`
   - 如果我要求应用 cleanup，用 `--step=all` 或 `--step=prepare,bump,cleanup`
   - 执行命令，粘贴完整输出

3. **处理失败**（如果有）：
   - 如果工具报错，分析失败原因（依赖阻塞 / 源码错误 / 其他）
   - 给出修复方案，修复后重新生成 `--apply` 命令

4. **刷新依赖**：
   - 执行 `dart pub get`
   - 如果失败，分析依赖冲突并给出解决方案

5. **验证**：
   - 执行 `dart analyze`，如果有新错误，给出修复方案
   - 执行 `flutter test`，如果失败，分析原因（行为变化 / 依赖不兼容 / mocking 包）
   - 执行 `flutter build apk --debug`（如果是 Flutter 项目）
   - 执行 `flutter build ios --no-codesign --debug`（如果是 Flutter 项目）
   - 汇总所有验证结果

6. **生成提交信息**：
   - 格式：「迁移：Dart SDK 语言版本从 X 升至 Y」
   - 包含：应用了哪些 lint 修复（prepare）、是否应用 cleanup、验证清单
   - 生成 `git add -A` 和 `git commit` 命令

**约束**：
- 任一验证失败时停下来，给出修复方案，不要继续下一步
- 如果工具报「保持文件不变」，说明迁移失败，重点分析失败原因
- 多包项目每个包都要传给命令
```

---

## ✅ 验证步骤

应用后确认：

```bash
# 1. 应用迁移
dart migrate --apply
# 或
dart migrate --apply --step=all
# 或（多包）
dart migrate --apply my_app packages/pkg_a

# 2. 刷新依赖
dart pub get

# 3. 验证
dart analyze
flutter test
flutter build apk --debug
flutter build ios --no-codesign --debug

# 4. 提交
git add -A
git commit -m "迁移：Dart SDK 语言版本从 3.12 升至 3.13

- 应用 avoid_final_parameters 预备修复
- 提升 pubspec.yaml 的 environment.sdk 至 ^3.13.0
- 验证通过：dart analyze、flutter test、flutter build"
```

**检查清单**：

- [ ] `dart migrate --apply` 退出码为 0
- [ ] `pubspec.yaml` 的 `environment.sdk` 已更新
- [ ] `dart pub get` 成功
- [ ] `dart analyze` 通过
- [ ] `flutter test` 全部通过
- [ ] `flutter build apk --debug` 通过
- [ ] `flutter build ios --no-codesign --debug` 通过
- [ ] 提交信息清晰描述变更

---

## 🛠️ 常见问题

1. **dart migrate --apply 报「dependencies are incompatible」**
   - 工具会保持文件不变。按 AI 给出的依赖更新方案处理，重新 `--apply`。

2. **apply 成功，但 dart analyze 报新错误**
   - 新语言版本规则更严格。AI 应给出修复方案，不要回退约束。

3. **flutter test 失败，提示某些测试用例行为改变**
   - 检查是否是语言版本引入的 breaking change。AI 应引用官方 changelog 并给出修复方案。

4. **flutter build 失败，报 Gradle 或 CocoaPods 错误**
   - 可能是原生侧依赖不兼容新 Flutter 版本。AI 应给出更新 Gradle/CocoaPods 依赖的命令。

5. **cleanup 修复太多，团队不接受**
   - 只应用 prepare + bump：`dart migrate --apply --step=prepare,bump`
   - cleanup 单独 PR：先把 prepare + bump 合入，稍后再 `--step=cleanup`

6. **AI 生成的提交信息没有包含 lint 名称**
   - 追问：「请在提交信息里列出应用了哪些 lint 修复（如 avoid_final_parameters）。」

---

## 🔗 相关

- 工作流：[workflows/dart-migrate-sdk-upgrade.md](../../workflows/dart-migrate-sdk-upgrade.md)
- 预览变更：[preview-dart-migrate.md](preview-dart-migrate.md)
- 官方文档：[https://dart.dev/tools/dart-migrate](https://dart.dev/tools/dart-migrate)
