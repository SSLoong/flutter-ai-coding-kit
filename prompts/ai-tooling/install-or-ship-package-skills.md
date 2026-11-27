# 提示词：安装或发布 Package Skills

**用途**：分两个场景——(a) 从当前项目依赖安装 package skills；(b) 为即将发布的 Dart/Flutter 包编写并校验 skills

---

## 📋 适用场景

- **场景 A（消费者）**：项目依赖第三方包（如 Riverpod、Dio、go_router），想让 AI 掌握这些库的用法
- **场景 B（库作者）**：维护内部或开源的 Dart/Flutter 包，想为团队或社区提供 AI 指南

---

## 📥 输入要求

### 场景 A（安装）

在 Cursor 中准备：

1. **项目 `pubspec.yaml`**：列出当前依赖
2. **目标**：让 AI 掌握依赖库的用法

### 场景 B（发布）

在 Cursor 中准备：

1. **包目录结构**：包的根目录（包含 `lib/`、`pubspec.yaml`）
2. **目标**：为这个包编写 skills，让依赖它的项目的 AI 更懂这个库
3. **参考**：包的 README、API 文档、常见用法

---

## 💬 提示词模板

### 场景 A：从依赖安装 Package Skills

```markdown
从当前 Flutter 项目的依赖中安装 package skills。请按以下步骤：

**上下文**：

- 项目 `pubspec.yaml` 路径：[PATH_TO_PUBSPEC]
- 主要依赖：[列出关键依赖，如 riverpod、dio、go_router]

**任务**：

1. **发现并安装 skills**
   - 运行命令：`dart run skills@ get`
   - 说明这个命令做什么：
     - 扫描 `pubspec.yaml` 中的依赖
     - 从 pub.dev 下载带 skills 的包的 `skills/` 目录
     - 安装到项目的 `.agents/skills/` 目录

2. **验证安装**
   - 检查 `.agents/skills/` 目录，列出已安装的 package skills（如 `.agents/skills/riverpod/SKILL.md`）
   - 在 AI chat 中询问：`当前项目的依赖中有哪些 package skills？[依赖名] 的常见用法是什么？`

3. **解释 skills 的作用**
   - Package skills 是库作者编写的 AI 指南（SKILL.md），包含：
     - 安装步骤
     - 基本用法
     - 常见模式
     - 最佳实践
   - AI 在需要时会加载这些 skills（progressive disclosure）

4. **后续步骤**
   - 依赖更新后，再次运行 `dart run skills@ get` 更新 skills
   - 如果某个依赖没有 package skills，可以：
     - 参考本仓库提示词手写业务用法（如 `prompts/state/generate-riverpod-provider.md`）
     - 或在项目 AGENTS.md 中补充这个库的用法说明

**约束**：

- 命令是 `dart run skills@ get`（`@` 表示临时运行，不是项目依赖）
- 不是每个依赖都有 package skills；只有库作者发布了 skills 才会安装
- 不要编造 skills 内容；安装后查看 `.agents/skills/` 中的实际文件
```

---

### 场景 B：为自己的包编写并校验 Skills

```markdown
为当前 Dart/Flutter 包编写并校验 package skills，以便发布到 pub.dev 后让依赖它的项目的 AI 更懂这个库。

**上下文**：

- 包名：[PACKAGE_NAME]
- 包目录：[PATH_TO_PACKAGE_ROOT]
- 包的主要能力：[简述，如 "状态管理"、"网络请求封装"、"自定义 Widget 库"]

**任务**：

1. **创建 skills 目录结构**
   - 在包根目录（与 `pubspec.yaml` 同级）创建：
     ```
     [PACKAGE_NAME]/
     ├── lib/
     ├── pubspec.yaml
     └── skills/
         └── [PACKAGE_NAME]/
             └── SKILL.md
     ```
   - 注意：`skills/` 下必须有一个以包名命名的子目录

2. **编写 SKILL.md**
   - 格式：YAML frontmatter + Markdown 正文
   - frontmatter 包含：
     ```yaml
     ---
     name: [PACKAGE_NAME]_usage
     description: How to use [PACKAGE_NAME] for common tasks
     agent: universal
     ---
     ```
   - 正文包含：
     - **Installation**：如何添加依赖
     - **Basic Usage**：最简示例代码
     - **Common Patterns**：典型场景（如创建实例、错误处理、性能优化）
     - **References**：API 文档链接、官方仓库
   - 参考现有 package skills（如果有）或本包的 README

3. **添加 skills_lint 校验**
   - 在 `pubspec.yaml` 的 `dev_dependencies` 中添加：
     ```yaml
     dev_dependencies:
       skills_lint: ^1.0.0
     ```
   - 运行：
     ```bash
     dart pub get
     dart run skills_lint
     ```
   - 修复 linter 报错（常见错误：目录结构、frontmatter 缺少字段、`name` 与目录不一致）

4. **CI 集成**
   - 在 CI（GitHub Actions、GitLab CI）中添加一步：
     ```yaml
     - name: Lint skills
       run: dart run skills_lint
     ```
   - 确保发布前 skills 格式正确

5. **发布到 pub.dev**
   - 确认 `skills/` 目录未被 `.pubignore` 忽略
   - 运行 `dart pub publish --dry-run` 预检
   - 运行 `dart pub publish` 发布
   - 发布后，依赖这个包的项目运行 `dart run skills@ get` 时会自动安装 skills

6. **验证**
   - 创建一个测试项目，添加这个包为依赖
   - 运行 `dart run skills@ get`
   - 检查 `.agents/skills/[PACKAGE_NAME]/SKILL.md` 是否存在
   - 询问 AI：`[PACKAGE_NAME] 的常见用法是什么？`

**约束**：

- 目录结构必须是 `skills/<PACKAGE_NAME>/SKILL.md`，不能是 `skills/SKILL.md`
- frontmatter 必须包含 `name`、`description`、`agent`（`universal` 表示所有 AI 助手）
- 代码块用三个反引号，不要缩进
- 不要在 SKILL.md 中引用包外的文件（除非也在 skills 目录下）
- 通过 `dart run skills_lint` 后再发布
```

---

## ✅ 验证步骤

### 场景 A（安装）

执行后确认：

1. **命令成功**
   - `dart run skills@ get` 退出码为 0
   - 输出中列出已安装的 package skills

2. **目录存在**
   - 检查 `.agents/skills/` 目录，包含依赖的 skills（如 `.agents/skills/riverpod/SKILL.md`）

3. **AI 能用**
   - 询问 AI：`[依赖名] 的常见用法是什么？如何用 [依赖名] 实现 [某个功能]？`
   - AI 回答参考了 package skills 中的示例

---

### 场景 B（发布）

执行后确认：

1. **目录结构正确**
   - `skills/<PACKAGE_NAME>/SKILL.md` 存在
   - 不是 `skills/SKILL.md`（缺少包名子目录）

2. **linter 通过**
   - `dart run skills_lint` 退出码为 0，无报错

3. **发布成功**
   - `dart pub publish` 成功
   - pub.dev 上的包版本包含 `skills/` 目录（在 pub.dev 页面查看 Files 标签）

4. **验证可用**
   - 测试项目添加依赖并运行 `dart run skills@ get`
   - `.agents/skills/<PACKAGE_NAME>/SKILL.md` 出现
   - AI 询问这个包的用法时参考了 SKILL.md

---

## 🛠️ 常见问题

1. **`dart run skills@ get` 报错 "Package 'skills' not found"？**
   - 确认 Dart ≥ 3.14（`dart --version`）
   - 确认网络连接正常
   - 命令中的 `@` 不能省略（表示临时运行，不是项目依赖）

2. **安装后 `.agents/skills/` 目录为空或缺少某些依赖的 skills？**
   - 不是所有依赖都有 package skills
   - 只有库作者发布了 skills 的包才会被安装
   - 检查 pub.dev 上对应包的 Files 标签，确认是否包含 `skills/` 目录

3. **`dart run skills_lint` 报错 "skills/<PACKAGE_NAME>/SKILL.md not found"？**
   - 检查目录结构：必须是 `skills/<PACKAGE_NAME>/SKILL.md`，不能是 `skills/SKILL.md`
   - `<PACKAGE_NAME>` 必须与 `pubspec.yaml` 中的 `name` 一致

4. **frontmatter 缺少 `agent` 字段？**
   - 加上 `agent: universal`（表示所有 AI 助手都能用）

5. **发布后，别人安装不到 skills？**
   - 检查 `.pubignore` 是否忽略了 `skills/` 目录（移除忽略规则）
   - 确认 pub.dev 上的包版本包含 `skills/`（Files 标签）
   - 确认 `dart run skills_lint` 通过

6. **团队要不要给内部包发布 skills？**
   - **推荐发布**：即使是内部包，只要团队用 AI 辅助开发，skills 能让 AI 更懂这个库
   - **投入 vs 收益**：写一份 SKILL.md（1-2 小时）+ skills_lint 进 CI（10 分钟），换来全团队 AI 即刻掌握库用法
   - **不发布的情况**：库用法极简（只有 1-2 个 API），或团队不用 AI 辅助开发

---

## 🔗 相关

- 工作流：[workflows/flutter-ai-agent-plugins.md](../../workflows/flutter-ai-agent-plugins.md)
- 官方文档：[Package skills](https://docs.flutter.dev/packages-and-plugins/using-packages)、[Ship skills with packages](https://docs.flutter.dev/packages-and-plugins/developing-packages)
- skills_lint：[pub.dev/packages/skills_lint](https://pub.dev/packages/skills_lint)
- 官方 skills 仓库：[flutter/agent-plugins](https://github.com/flutter/agent-plugins)、[dart-lang/skills](https://github.com/dart-lang/skills)
