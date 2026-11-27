# 提示词：安装官方 Flutter AI 插件

**用途**：在指定编辑器（默认 Cursor）中安装官方 Dart and Flutter 插件、配置 MCP、下载官方 rules，并说明与现有 AGENTS.md / .cursor/rules 如何并存

---

## 📋 适用场景

- 新 Flutter 项目，团队首次引入 AI 编程工具
- 现有项目只有 AGENTS.md，想增强 AI 能力（实时 analyzer、package skills）
- 需要验证插件安装是否成功

---

## 📥 输入要求

在 Cursor 中准备：

1. **项目路径**：Flutter 项目根目录（包含 `pubspec.yaml`）
2. **Flutter 版本**：≥ 3.50（`flutter --version` 确认）
3. **编辑器**：默认 Cursor；如果是其他编辑器（Claude Code、Antigravity、GitHub Copilot），在提示词中说明
4. **现有配置**：如果项目已有 `AGENTS.md` 或 `.cursor/rules/flutter-core.mdc`，附上内容或路径

---

## 💬 提示词模板

```markdown
为当前 Flutter 项目安装官方 Dart and Flutter AI 插件。请按以下步骤指导：

**编辑器**：[Cursor / Claude Code / Antigravity / GitHub Copilot / 其他]

**任务**：

1. **安装插件**
   - 给出在 [编辑器] 中安装官方 Dart and Flutter 插件的具体步骤（Cursor: Customize → 搜索 Dart and Flutter → Add to Cursor；其他编辑器参考官方文档）
   - 说明插件会自动配置什么（MCP server、skills）

2. **配置 Agent Rules（仅 Cursor）**
   - 说明 Cursor 插件不会自动加载全部 rules，需要手动从官方仓库下载
   - 给出从 [flutter/agent-plugins rules 目录](https://github.com/flutter/agent-plugins/tree/main/rules) 下载推荐 rules 的步骤（如 `flutter-hot-reload.mdc`、`flutter-widget-testing.mdc`）
   - 说明下载到哪里：项目的 `.cursor/rules/` 目录

3. **与现有配置并存**
   - 如果项目已有 `AGENTS.md` 或 `.cursor/rules/flutter-core.mdc`，说明：
     - 官方插件与项目模板是**互补**，不是互斥
     - 官方插件负责：SDK 能力（analyzer、测试、运行时、热重载、package skills）
     - 项目模板负责：业务规范（项目结构、状态管理、业务禁区）
     - 两者应并存在 `.cursor/rules/` 下（多个 `.mdc` 文件都会被加载）
   - 给出并存后的目录结构示例：
     ```
     .cursor/rules/
     ├── flutter-core.mdc         # 项目模板（已有）
     └── flutter-hot-reload.mdc   # 官方 rule（新增）
     ```

4. **验证安装**
   - 在 [编辑器] Agent panel / Chat 中询问：`总结当前项目可用的 Dart 和 Flutter tools 与 skills。`
   - 期望回答包含：
     - MCP server `dart` 已连接
     - Skills 列表（来自 `flutter/agent-plugins` 和 `dart-lang/skills`）
     - Rules 列表（包括项目和官方的 rules）
   - 额外验证：询问 `用 MCP 工具列出当前项目的所有 analyzer 错误与警告。`，确认 MCP 工具可用

5. **总结下一步**
   - 安装 package skills：`dart run skills@ get`（从依赖获取 skills）
   - 阅读工作流文档：[workflows/flutter-ai-agent-plugins.md](../../workflows/flutter-ai-agent-plugins.md)

**约束**：

- 以官方文档为准：[docs.flutter.dev/ai/get-started](https://docs.flutter.dev/ai/get-started)
- 不编造命令或路径；不确定时提示查阅官网
- 不删除现有 AGENTS.md 或 .cursor/rules；说明并存的价值
- 核对 Flutter ≥ 3.50（`flutter --version`）
```

---

## ✅ 验证步骤

执行后确认：

1. **插件已安装**
   - Cursor: Customize → 插件列表中看到 Dart and Flutter
   - 其他编辑器：用对应命令（如 `claude plugin list`）确认

2. **MCP 已连接**
   - 询问 AI `用 MCP 列出 analyzer 错误`，能返回结果（即使项目没有错误，也应返回空列表，而不是报 "MCP 工具不可用"）

3. **Skills 已安装**
   - 检查 `.agents/skills/` 目录，包含官方 skills（如 `flutter/agent-plugins/`、`dart-lang/skills/`）

4. **Rules 已配置（仅 Cursor）**
   - 检查 `.cursor/rules/` 目录，包含官方 rules（如 `flutter-hot-reload.mdc`）和项目 rules（如 `flutter-core.mdc`，如果有）

5. **并存验证**
   - 询问 AI：`本项目用什么状态管理？编辑 widget 后应该做什么？`
   - 期望回答：状态管理来自项目 AGENTS.md；编辑 widget 后提示热重载来自官方 rule

---

## 🛠️ 常见问题

1. **Cursor 插件安装后，AI 不会主动提示热重载？**
   - 检查 `.cursor/rules/` 是否包含 `flutter-hot-reload.mdc`
   - 官方插件不自动加载全部 rules，需要手动下载

2. **MCP 工具报错 "dart mcp-server not found"？**
   - 确认 Dart ≥ 3.14（`dart --version`）
   - 确认 `dart` 在 PATH 中（`which dart` / `where dart`）
   - 重启编辑器

3. **插件安装后，`.agents/skills/` 目录不存在？**
   - 插件可能还在后台下载 skills
   - 重启编辑器或手动运行 `npx skills add flutter/agent-plugins --skill '*' --agent universal --yes`

4. **项目已有 AGENTS.md，安装插件后 AI 不遵守项目规范？**
   - 确认 AGENTS.md 或 `.cursor/rules/flutter-core.mdc` 未被删除
   - 在 Cursor chat 中明确 @ AGENTS.md 或 @.cursor/rules/flutter-core.mdc
   - 官方插件不会覆盖项目规范；如果 AI 忽略，是提示词问题，不是插件问题

---

## 🔗 相关

- 工作流：[workflows/flutter-ai-agent-plugins.md](../../workflows/flutter-ai-agent-plugins.md)
- 官方文档：[docs.flutter.dev/ai/get-started](https://docs.flutter.dev/ai/get-started)
- 官方 rules 仓库：[flutter/agent-plugins rules](https://github.com/flutter/agent-plugins/tree/main/rules)
- 项目模板：[templates/AGENTS.md](../../templates/AGENTS.md)
- Cursor rules 模板：[templates/cursor/rules/flutter-core.mdc](../../templates/cursor/rules/flutter-core.mdc)
