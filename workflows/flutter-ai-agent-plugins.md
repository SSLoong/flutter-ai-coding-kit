# 官方 Flutter AI Agent 插件与 Package Skills 工作流

**用 AI 在 Cursor 等编辑器中安装官方 Flutter/Dart 插件、配置 MCP、管理 Skills 与 Rules，并与现有项目模板协同**

> 作者:Loong (SSLoong) | 日期:2026-11-23

本文基于 **Flutter 3.50**（2026-11-11）与 **Dart 3.14**，核对官方文档 [docs.flutter.dev/ai](https://docs.flutter.dev/ai) 与 [flutter.dev/blog/whats-new-in-flutter-3-50](https://flutter.dev/blog/whats-new-in-flutter-3-50)。这是 Flutter 官方正式将 AI agent plugins、package skills、Dart/Flutter MCP、agent rules、专用 agent（如 a11y）做成一等开发者工具的里程碑版本。

---

## 📋 背景

Flutter 3.50（2026-11-11）正式推出了完整的 AI 编程工具链：

- **Agent Skills**：按需加载的程序化指南，教 AI 助手如何执行特定任务（如创建响应式布局、编写 Widget 测试）
- **Agent Rules**：持久性会话级约定（如编辑 Flutter widget 后自动触发热重载）
- **Dart and Flutter MCP Server**：通过 Model Context Protocol 连接 Dart SDK，提供实时 analyzer 诊断、符号解析、测试运行、运行时检查
- **Developer Knowledge MCP Server**：在线文档检索（官方 Flutter/Dart 文档、API 参考）
- **Package Skills**：由 pub.dev 包直接发布的 skills，让 AI 即刻掌握第三方库的用法
- **Specialized Agents**：专用 agent 人格（如 Flutter Accessibility `a11y` agent），针对特定工作流配置工具与系统提示

这套工具链**不替代**本仓库现有的 `templates/AGENTS.md`、`templates/cursor/rules/flutter-core.mdc` 以及业务提示词。它们是**增强与互补**：

- **官方插件**提供实时 SDK 诊断、官方最佳实践、热重载提示、第三方库 skills
- **项目模板**定义目录结构、状态管理、业务禁区、团队约定

两者并存，各司其职。

---

## 🎯 目标

- ✅ 判断何时安装官方 AI 插件（Cursor、Claude Code、Antigravity、GitHub Copilot、其他 MCP 客户端）
- ✅ 在 Cursor 中安装官方 Dart and Flutter 插件、配置 MCP、下载官方 rules 到 `.cursor/rules/`
- ✅ 理解 Agent Skills、Agent Rules、MCP、Package Skills、Specialized Agents、Developer Knowledge MCP 六者分工
- ✅ 从项目依赖安装 package skills（`dart run skills@ get`）
- ✅ 为即将发布的 Dart/Flutter 包编写并校验 skills（`package:skills_lint`）
- ✅ 说明官方插件与本仓库 `AGENTS.md` / `.cursor/rules/flutter-core.mdc` 的关系与共存方式
- ✅ 验证安装成功：用提问、MCP 工具列表、skills 目录检查

---

## 🔧 适用场景

| 场景 | 适合度 | 说明 |
|------|--------|------|
| 新 Flutter 项目，团队首次引入 AI 编程 | ⭐⭐⭐⭐⭐ | 官方插件 + 项目 AGENTS.md 一起配置 |
| 现有项目只有 AGENTS.md，想增强 AI 能力 | ⭐⭐⭐⭐⭐ | 安装官方插件，保留 AGENTS.md，两者并存 |
| 要用第三方包（如 Riverpod、Dio）的 AI 辅助 | ⭐⭐⭐⭐⭐ | 安装 package skills，AI 即刻掌握库用法 |
| 维护内部库，想让团队 AI 更懂这个库 | ⭐⭐⭐⭐⭐ | 为包编写 skills，配合 skills_lint 在 CI 中校验 |
| 只用 Cursor，不想配置 MCP 和 skills | ⭐⭐⭐ | 官方插件可自动配置，但跳过会失去实时诊断与第三方库 skills |
| 团队禁止 AI、或项目无网络 | ⭐ | 官方插件需联网，MCP 要本地 Dart SDK；不适用 |

**怎么选：**

- **装官方插件**：团队用 AI 辅助 Flutter 开发，想要实时 analyzer、热重载提示、官方 skills、第三方库 skills。这是增强，不会让现有 AGENTS.md 失效。
- **只用 AGENTS.md / .cursor/rules**：团队已有完善项目模板，不想引入官方工具链、或项目环境不支持 MCP。模板依然有效，AI 仍能工作，只是缺少实时 SDK 诊断与 package skills。
- **两者并存（推荐）**：官方插件提供 SDK 层面的实时能力（analyzer、测试、运行时检查）与官方最佳实践；项目模板定义业务层面的结构、规范、禁区。各取所长。

---

## 🚀 完整流程

下面以 **Cursor** 为主路径演示，同时简要说明其他编辑器（Claude Code、Antigravity、GitHub Copilot、其他 MCP 客户端）的安装入口。核对日期：2026-11-23，以官网最新说明为准。

### Step 1: 准备上下文（人工操作）

**前置条件：**

- Flutter ≥ 3.50（`flutter --version`）
- Dart ≥ 3.14
- 编辑器：Cursor / Claude Code / Antigravity / VS Code + GitHub Copilot / 其他 MCP 客户端

打开项目：

- `pubspec.yaml`（查看依赖）
- `lib/` 结构
- 现有 `AGENTS.md` / `.cursor/rules/flutter-core.mdc`（如果有）

**目标：** 确认 Flutter 版本、了解项目依赖、判断是否需要官方插件。

---

### Step 2: 在 Cursor 中安装官方插件（Cursor 路径）

**方式一（推荐）：** Cursor Marketplace

1. 在 Cursor 中打开 **Customize**（Cmd/Ctrl + ,）
2. 搜索 **Dart and Flutter**
3. 点击 **Add to Cursor**

**方式二：** Cursor Chat 命令

在 Cursor chat session 中运行：

```
/add-plugin dart-flutter
```

**方式三：** 访问 Cursor Marketplace 网页

访问 [Cursor Marketplace](https://cursor.com/marketplace)，搜索 `dart-flutter`，点击 **Add to Cursor**。

**效果：**

- Cursor 自动发现 bundled skills（来自 `flutter/agent-plugins` 和 `dart-lang/skills`）
- 自动配置 Dart and Flutter MCP server（`dart mcp-server`）
- skills 下载到项目的 `.agents/skills` 目录（progressive disclosure：AI 先读元数据，需要时再加载完整 `SKILL.md`）

---

### Step 3: 配置 Agent Rules（Cursor 专属）

**重要：** Cursor 插件**不会**自动加载全部 rules。需要手动从官方仓库下载 `.mdc` 文件到 `.cursor/rules/`。

**步骤：**

1. 访问 [flutter/agent-plugins rules 目录](https://github.com/flutter/agent-plugins/tree/main/rules)（官方 rules 仓库，核对时请用官网链接）
2. 下载你需要的 `.mdc` 文件（如 `flutter-hot-reload.mdc`、`flutter-widget-testing.mdc`）到项目的 `.cursor/rules/` 目录
3. 重启 Cursor chat session（rules 会加载到每次会话）

**与本仓库模板的关系：**

- **`templates/cursor/rules/flutter-core.mdc`**（本仓库）：定义项目结构、状态管理选择、业务禁区、团队约定。**保留**。
- **官方 rules**（如 `flutter-hot-reload.mdc`）：指导 AI 在编辑 `lib/` 后主动触发 `r` 热重载，或在改 pubspec 后提示 hot restart。**新增**。
- **共存方式：** `.cursor/rules/` 下可以有多个 `.mdc` 文件。Cursor 会加载全部。官方 rules 负责 SDK 层面的通用约定（热重载、测试、analyzer），项目 rules 负责业务层面的结构与规范。

**示例：** 假设项目已有 `.cursor/rules/flutter-core.mdc`（来自本仓库模板），现在新增官方 `flutter-hot-reload.mdc`：

```
.cursor/rules/
├── flutter-core.mdc         # 本仓库模板（项目结构、状态管理、禁区）
└── flutter-hot-reload.mdc   # 官方 rule（编辑 widget 后提示热重载）
```

---

### Step 4: 验证安装（Cursor）

在 Cursor Agent panel（Cmd/Ctrl + L）中询问：

```markdown
总结当前项目可用的 Dart 和 Flutter tools 与 skills。
```

**期望回答包含：**

- MCP server `dart` 已连接（提供 analyzer、symbol resolution、test runner、runtime inspection、package management）
- Skills 列表：来自 `flutter/agent-plugins` 和 `dart-lang/skills` 的 skills（如 `create-responsive-layout`、`write-widget-test`、`manage-state`）
- Rules 列表：`.cursor/rules/` 下的 `.mdc` 文件（包括本仓库的 `flutter-core.mdc` 和官方的 `flutter-hot-reload.mdc`）

**额外验证：** 在 Cursor 中执行一次 AI 辅助的 analyzer 诊断：

```markdown
用 MCP 工具列出当前项目的所有 analyzer 错误与警告。
```

如果 AI 能调用 MCP 工具并返回 analyzer 结果，说明 MCP server 配置成功。

---

### Step 5: 其他编辑器安装路径（简要）

#### Claude Code（terminal）

**安装插件：**

```bash
# 1. 添加 marketplace
claude plugin marketplace add flutter/agent-plugins

# 2. 安装插件
claude plugin install dart-flutter@dart-flutter

# 3. 验证
claude plugin marketplace list
```

**配置 rules：**

Claude Code 插件不自动加载 rules。从 [官方 rules 仓库](https://github.com/flutter/agent-plugins/tree/main/rules) 下载 `.md` 文件，添加到项目的 `CLAUDE.md` 文件。

---

#### Antigravity（IDE / VS Code extension / CLI）

**Antigravity IDE / extension：**

1. 打开 Settings（齿轮图标 / Cmd/Ctrl + ,）
2. Customizations → Build with Google Plugins → Customize
3. Download **Dart and Flutter integration**

验证：Agent panel（Cmd/Ctrl + L）→ 询问 `Summarize the Dart and Flutter tools and skills available in this project.`

**Antigravity CLI：**

```bash
# 1. 配置 MCP（项目 .agents/mcp_config.json 或全局 ~/.gemini/config/mcp_config.json）
{
  "mcpServers": {
    "dart": {
      "command": "dart",
      "args": ["mcp-server"]
    }
  }
}

# 2. 安装 skills
npx skills add flutter/agent-plugins --skill '*' --agent universal --yes
npx skills add dart-lang/skills --skill '*' --agent universal --yes

# 3. 启动
agy
```

---

#### Codex（terminal）

**安装插件：**

```bash
# 1. 添加 marketplace
codex plugin marketplace add flutter/agent-plugins

# 2. 安装插件
codex plugin add dart-flutter@dart-flutter

# 3. 验证
codex plugin list
```

**配置 rules：**

从 [官方 rules 仓库](https://github.com/flutter/agent-plugins/tree/main/rules) 下载 `.md` 文件到项目的 `.agent/rules/` 或追加到 `CODEX.md`。

---

#### GitHub Copilot（VS Code）

**配置 MCP：**

1. 确保 VS Code 安装了 **Model Context Protocol (MCP) extension**
2. 创建或编辑 `.vscode/mcp.json`：

```json
{
  "servers": {
    "dart": {
      "command": "dart",
      "args": ["mcp-server"]
    }
  }
}
```

**安装 skills：**

```bash
# 在项目根目录
npx skills add flutter/agent-plugins --skill '*' --agent universal --yes
npx skills add dart-lang/skills --skill '*' --agent universal --yes
```

**配置 rules：**

从 [官方 rules 仓库](https://github.com/flutter/agent-plugins/tree/main/rules) 下载 `.md` 文件，添加到项目的 `.github/copilot-instructions.md`。

---

#### 其他 MCP 客户端（Windsurf、Zed、Cline 等）

**配置 MCP：**

在客户端的 MCP 配置文件中添加：

```json
{
  "mcpServers": {
    "dart": {
      "command": "dart",
      "args": ["mcp-server"]
    }
  }
}
```

**安装 skills：**

```bash
npx skills add flutter/agent-plugins --skill '*' --agent universal --yes
npx skills add dart-lang/skills --skill '*' --agent universal --yes
```

**配置 rules：**

根据客户端支持（如 Windsurf `.windsurfrules`、Cline `.clinerules`），从 [官方 rules 仓库](https://github.com/flutter/agent-plugins/tree/main/rules) 复制或下载 rules。

---

### Step 6: 安装 Package Skills（从依赖）

**背景：** pub.dev 上的第三方包（如 Riverpod、Dio、go_router）可能发布了 package skills。安装后，AI 即刻掌握这些库的用法。

**步骤：**

1. 确保项目依赖已在 `pubspec.yaml` 中
2. 运行发现与安装命令：

```bash
dart run skills@ get
```

**效果：**

- 扫描 `pubspec.yaml` 中的依赖
- 从 pub.dev 下载带 skills 的包的 `skills/` 目录到 `.agents/skills/`
- AI 在需要时加载这些 skills

**示例：** 项目依赖 `riverpod: ^2.5.0`，且 Riverpod 已发布 package skills。运行 `dart run skills@ get` 后，`.agents/skills/riverpod/SKILL.md` 出现，AI 询问 Riverpod 用法时会参考这份 skill。

**验证：**

```bash
# 查看已安装的 skills
ls -la .agents/skills
```

在 Cursor/编辑器中询问：

```markdown
当前项目的依赖中有哪些 package skills？如何用 Riverpod 管理状态？
```

---

### Step 7: 为自己的包编写并校验 Skills（库作者）

**背景：** 你维护一个内部或开源的 Dart/Flutter 包，想让团队或社区的 AI 更懂这个库。

**步骤：**

#### 7.1 创建 skills 目录

在包的根目录（与 `pubspec.yaml` 同级）创建：

```
your_package/
├── lib/
├── pubspec.yaml
└── skills/
    └── your_package/
        └── SKILL.md
```

**注意：** `skills/` 下必须有一个以包名命名的子目录（如 `skills/your_package/`），然后在里面放 `SKILL.md`。

#### 7.2 编写 SKILL.md

**格式：** YAML frontmatter + Markdown 正文

```markdown
---
name: your_package_usage
description: How to use YourPackage for common tasks
agent: universal
---

# YourPackage Usage

## Installation

Add `your_package` to `pubspec.yaml`:

\`\`\`yaml
dependencies:
  your_package: ^1.0.0
\`\`\`

## Basic Usage

[示例代码、常见用法、最佳实践]

## Common Patterns

[典型场景、错误处理、性能优化]

## References

- [API 文档链接]
- [官方仓库]
```

**要点：**

- frontmatter 包含 `name`、`description`、`agent`（`universal` 表示所有 AI 助手）
- 正文用 Markdown，代码块用三个反引号
- 包含安装、基本用法、常见模式、参考链接

#### 7.3 校验 skills（`package:skills_lint`）

**安装 linter：**

在包的 `pubspec.yaml` 中添加：

```yaml
dev_dependencies:
  skills_lint: ^1.0.0
```

然后运行：

```bash
dart pub get
dart run skills_lint
```

**linter 检查：**

- YAML frontmatter 格式（`name`、`description`、`agent` 是否存在）
- 目录结构（`skills/<package_name>/SKILL.md`）
- 命名约定（`name` 是否与目录一致）
- 相对文件路径（SKILL.md 中引用的脚本、示例文件是否存在）

**常见错误：**

- `skills/SKILL.md`（缺少包名子目录）→ 改为 `skills/your_package/SKILL.md`
- frontmatter 缺少 `agent` 字段 → 加上 `agent: universal`
- `name` 与目录不一致 → 统一

**CI 集成：**

在 CI（GitHub Actions、GitLab CI）中加一步：

```yaml
- name: Lint skills
  run: dart run skills_lint
```

发布前必须通过 `skills_lint`，避免无效 skills 发到 pub.dev。

#### 7.4 发布到 pub.dev

```bash
dart pub publish
```

发布后，依赖你的包的项目运行 `dart run skills@ get` 时，会自动下载你的 skills。

---

### Step 8: 决策指南（官方插件 vs 项目模板 vs 两者并存）

| 场景 | 官方插件 | 项目 AGENTS.md / .cursor/rules | 两者并存（推荐） |
|------|---------|-------------------------------|-----------------|
| **职责** | SDK 层面：analyzer、测试、运行时、热重载、官方最佳实践、第三方库 skills | 业务层面：项目结构、状态管理选择、目录规范、业务禁区、团队约定 | 各司其职 |
| **实时能力** | ✅ 实时 analyzer 诊断、符号解析、测试运行、运行时检查 | ❌ 静态文本，AI 凭记忆推断 | ✅ 官方插件提供实时能力 |
| **第三方库** | ✅ 通过 package skills 获取库的 AI 指南 | ❌ 只能在模板中手写依赖列表，AI 凭训练数据推断 | ✅ package skills 自动更新 |
| **业务规范** | ❌ 官方插件不知道你的项目结构、状态管理、业务禁区 | ✅ 明确写在 AGENTS.md / .cursor/rules | ✅ 项目模板补充业务规范 |
| **团队推广** | 需要团队统一安装插件（一次性） | 随项目 Git 提交，团队自动获取 | ✅ 插件一次性安装，模板随项目走 |

**推荐方案：**

1. **安装官方插件**（Cursor / Claude Code / Antigravity / GitHub Copilot），获得实时 SDK 能力与官方 skills
2. **下载官方 rules**（如 `flutter-hot-reload.mdc`）到 `.cursor/rules/`，让 AI 主动提示热重载
3. **保留项目模板**（`templates/AGENTS.md` / `.cursor/rules/flutter-core.mdc`），定义项目结构、状态管理、业务禁区
4. **安装 package skills**（`dart run skills@ get`），让 AI 掌握第三方库用法
5. **为内部库编写 skills + skills_lint**，团队 AI 更懂自己的库

**不要做：**

- ❌ 认为装了官方插件就要删除 AGENTS.md / .cursor/rules — 它们是互补，不是互斥
- ❌ 只配了 MCP，没装 skills/rules — MCP 提供工具，skills/rules 提供指南
- ❌ 官方 rules 未放入 `.cursor/rules` — Cursor 不会自动加载全部 rules
- ❌ 在没核对官网时手写过时的 MCP 配置 — MCP server 命令是 `dart mcp-server`，不是旧版的 `dart pub global activate`
- ❌ skills_lint 没进 CI 就发布 package skills — 无效 skills 会让 AI 加载失败

---

### Step 9: 贯穿示例（中型 Flutter 应用）

**场景：** 一个中型 Flutter 应用，用 Riverpod 状态管理、go_router 路由、Dio 网络层。团队要引入官方 AI 插件，同时保留项目 AGENTS.md。

**步骤：**

1. **安装官方插件（Cursor）**
   - Customize → 搜索 Dart and Flutter → Add to Cursor
   - 验证：询问 `总结 Dart 和 Flutter tools 与 skills`

2. **下载官方 rules**
   - 从 [flutter/agent-plugins rules](https://github.com/flutter/agent-plugins/tree/main/rules) 下载 `flutter-hot-reload.mdc`、`flutter-widget-testing.mdc`
   - 放入 `.cursor/rules/`

3. **保留项目 AGENTS.md**
   - 已有 `AGENTS.md`（定义项目结构：`lib/features/`、状态管理：Riverpod、路由：go_router）
   - 保留，不删除
   - 官方 rules 负责热重载、测试；项目 AGENTS.md 负责业务规范

4. **安装 package skills**
   ```bash
   dart run skills@ get
   ```
   - 假设 Riverpod、go_router、Dio 都发布了 package skills
   - 安装后，AI 询问 Riverpod 用法时会参考 package skills

5. **验证并存**
   - 在 Cursor 中询问：`当前项目用什么状态管理？如何新增一个 Riverpod Provider？`
   - 期望：AI 参考项目 AGENTS.md（知道用 Riverpod）+ package skills（知道 Riverpod 最佳实践）+ MCP（实时 analyzer）
   - 编辑 `lib/features/home/home_page.dart` 后，AI 主动提示 `r` 热重载（来自官方 rule `flutter-hot-reload.mdc`）

**结果：** 官方插件与项目模板各司其职，AI 既懂 SDK、又懂业务。

---

## ✅ 验证清单

完成后确认：

- [ ] Flutter ≥ 3.50，Dart ≥ 3.14（`flutter --version`）
- [ ] Cursor/编辑器中安装了官方 Dart and Flutter 插件
- [ ] `.agents/skills` 目录存在，包含官方 skills（`flutter/agent-plugins`、`dart-lang/skills`）
- [ ] （仅 Cursor）`.cursor/rules/` 包含官方 rules（如 `flutter-hot-reload.mdc`）和项目 rules（如 `flutter-core.mdc`）
- [ ] MCP server `dart` 已连接（询问 AI `用 MCP 列出 analyzer 错误`，能返回结果）
- [ ] 项目依赖的 package skills 已安装（`dart run skills@ get` 成功，`.agents/skills/` 包含依赖的 skills）
- [ ] （库作者）自己的包通过 `dart run skills_lint`，skills 格式正确
- [ ] 官方插件与项目 AGENTS.md / .cursor/rules 并存，AI 既能用 MCP 诊断、又能遵守项目规范

---

## 🤔 FAQ

### Q1: 装了官方插件后，原来的 AGENTS.md 和 .cursor/rules 还有用吗？

**A**: 有用。它们是**互补**关系：

- **官方插件**：SDK 层面（analyzer、测试、运行时、热重载、官方 skills、package skills）
- **项目模板**：业务层面（项目结构、状态管理、目录规范、业务禁区）

**示例：** 官方 rule `flutter-hot-reload.mdc` 告诉 AI「编辑 widget 后提示热重载」，项目 `flutter-core.mdc` 告诉 AI「本项目用 Riverpod，状态放 `lib/features/*/domain/providers/`」。两者都需要。

---

### Q2: 只配了 MCP，没装 skills 和 rules，会怎样？

**A**: MCP 提供**工具**（analyzer、测试、运行时检查），skills 和 rules 提供**指南**（如何用这些工具、何时触发热重载）。

只配 MCP：AI 能调用工具，但不知道何时主动调用、或如何结合官方最佳实践。

**建议：** MCP、skills、rules 一起配置。

---

### Q3: Cursor 插件安装后，为什么 AI 不会自动热重载？

**A**: Cursor 插件不自动加载全部 rules。需要手动从 [flutter/agent-plugins rules](https://github.com/flutter/agent-plugins/tree/main/rules) 下载 `flutter-hot-reload.mdc` 到 `.cursor/rules/`。

---

### Q4: `dart run skills@ get` 报错 "Package 'skills' not found"？

**A**: `skills` 是一个 pub.dev 工具包，不是项目依赖。命令中的 `@` 表示临时运行。

确保：

- Dart ≥ 3.14
- 网络连接正常
- 命令是 `dart run skills@ get`（不是 `dart run skills get`）

如果仍报错，核对官网命令（可能在新版本中改为 `dart pub global run skills get` 或其他格式）。

---

### Q5: 我的包发布到 pub.dev 后，别人运行 `dart run skills@ get` 看不到我的 skills？

**A**: 检查：

- 目录结构：`skills/<package_name>/SKILL.md`（不是 `skills/SKILL.md`）
- frontmatter：包含 `name`、`description`、`agent`
- `dart run skills_lint` 通过
- pub.dev 上的包版本是否包含 `skills/` 目录（发布时不要忽略 `skills/`）

---

### Q6: 官方 MCP server 和 Developer Knowledge MCP 有什么区别？

**A**:

- **Dart and Flutter MCP server**（本地）：`dart mcp-server`，连接本地 Dart SDK，提供 analyzer、测试、运行时检查、符号解析、包管理
- **Developer Knowledge MCP server**（云端）：连接 Google 官方文档库（docs.flutter.dev、dart.dev、API 参考），提供在线文档搜索

**配置：** Dart and Flutter MCP 由官方插件自动配置；Developer Knowledge MCP 需要单独配置（官网有说明，本文未详述，核对时查看官网）。

---

### Q7: 团队成员不愿意装插件，可以只用项目 AGENTS.md 吗？

**A**: 可以。AGENTS.md 仍然有效，AI 仍能工作。

**缺少的能力：**

- 实时 analyzer 诊断（AI 凭记忆推断错误，可能不准）
- 官方 skills（AI 凭训练数据推断 Flutter 最佳实践，可能过时）
- package skills（AI 不知道第三方库的最新用法）
- 热重载提示（AI 不会主动提示 `r`）

**建议：** 至少让负责 AI 辅助开发的成员安装插件，输出的代码仍符合项目规范（由 AGENTS.md 保障）。

---

### Q8: Specialized agents（如 a11y agent）在 Cursor 中能用吗？

**A**: 目前 Flutter Accessibility agent（`a11y`）仅在 Google Antigravity 中可用（通过 agent picker 或 `@flutter_a11y_agent` 调用）。

Cursor、Claude Code、Codex、GitHub Copilot 目前不支持 specialized agents。

如果团队需要无障碍审计，可以：

- 在 Antigravity 中用 `a11y` agent 生成报告与修复建议
- 参考本仓库的 [workflows/semantics-accessibility.md](semantics-accessibility.md) 用通用 AI 助手 + 提示词完成

---

### Q9: 官方插件会自动更新吗？

**A**: Cursor / Claude Code / Antigravity 等编辑器会定期更新插件。

**手动更新：**

- Cursor：Customize → 插件列表 → Update（如果有）
- Claude Code：`claude plugin update dart-flutter@dart-flutter`
- Antigravity：Settings → Plugins → 检查更新

**skills 更新：** 再次运行 `npx skills add flutter/agent-plugins --skill '*' --agent universal --yes` 会更新已安装的 skills。

---

### Q10: 能不能把官方 rules 合并到项目 AGENTS.md？

**A**: 可以，但不推荐。

- **分离的好处：** 官方 rules 更新时，只需重新下载对应 `.mdc` 文件；项目 AGENTS.md 保持稳定
- **合并的坏处：** 官方 rules 更新后，需要手动对比并合并到 AGENTS.md，容易遗漏或冲突

**建议：** 官方 rules 放 `.cursor/rules/`，项目 rules 放 `AGENTS.md` 或单独的 `.cursor/rules/flutter-core.mdc`。

---

## 💡 最佳实践

1. **官方插件 + 项目模板，各司其职**
   - 官方插件：SDK 能力（analyzer、测试、运行时、package skills）
   - 项目模板：业务规范（结构、状态管理、禁区）

2. **团队统一安装插件，降低 onboarding 成本**
   - 在团队文档中说明安装步骤（如本文 Step 2）
   - 新成员入职时，安装插件 + 拉取项目（AGENTS.md 随项目走）

3. **定期运行 `dart run skills@ get`**
   - 依赖更新后，package skills 也可能更新
   - 在 CI 或 pre-commit hook 中提示

4. **库作者：skills_lint 进 CI**
   - 在 GitHub Actions / GitLab CI 中加一步 `dart run skills_lint`
   - 发布前必须通过，避免无效 skills

5. **不要过度依赖 AI**
   - 官方插件增强 AI 能力，但最终代码质量仍需人工 review
   - 用 `flutter test`、`flutter analyze`、`flutter build` 验证

6. **核对官网，不编造命令**
   - 本文基于 2026-11-23 的官网，未来可能变化
   - 遇到命令报错时，先查 [docs.flutter.dev/ai](https://docs.flutter.dev/ai)

---

## 🔗 相关资源

### 官方文档（核对日期：2026-11-23）

- [Get started developing with AI](https://docs.flutter.dev/ai/get-started) — 官方安装指南
- [How Flutter AI tools work](https://docs.flutter.dev/ai/tools) — 六大组件详解（skills、MCP、rules、specialized agents、package skills、Developer Knowledge）
- [What's new in Flutter 3.50](https://flutter.dev/blog/whats-new-in-flutter-3-50) — 3.50 发布说明（包含 AI 工具链正式版）
- [Package skills](https://docs.flutter.dev/packages-and-plugins/using-packages) — 从依赖安装 skills
- [Ship skills with packages](https://docs.flutter.dev/packages-and-plugins/developing-packages) — 为包编写并发布 skills
- [flutter/agent-plugins](https://github.com/flutter/agent-plugins) — 官方 skills 与 rules 仓库
- [dart-lang/skills](https://github.com/dart-lang/skills) — Dart 官方 skills
- [package:skills_lint](https://pub.dev/packages/skills_lint) — skills 校验工具

### 本仓库资源

- [templates/AGENTS.md](../templates/AGENTS.md) — 项目 AI 协作模板
- [templates/cursor/rules/flutter-core.mdc](../templates/cursor/rules/flutter-core.mdc) — Cursor 项目 rules 模板
- [docs/context-engineering.md](../docs/context-engineering.md) — 上下文工程原理
- [workflows/semantics-accessibility.md](semantics-accessibility.md) — 无障碍适配工作流（与 a11y agent 互补）
- [prompts/ai-tooling/setup-official-flutter-ai-plugin.md](../prompts/ai-tooling/setup-official-flutter-ai-plugin.md) — 安装插件的提示词模板（本次新增）
- [prompts/ai-tooling/install-or-ship-package-skills.md](../prompts/ai-tooling/install-or-ship-package-skills.md) — 安装或发布 package skills 的提示词模板（本次新增）

---

**先装插件，再配规则，最后验证。官方插件与项目模板并存，各司其职。**
