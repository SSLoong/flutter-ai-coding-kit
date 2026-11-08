# Flutter AI Coding Kit

**面向 Flutter 开发者的 AI 辅助编程实践指南**

> 姊妹仓库：[ios-ai-coding-kit](https://github.com/SSLoong/ios-ai-coding-kit) — iOS/Swift AI 工作流合集

## 📖 关于本仓库

这是一个收集 Flutter/Dart 开发中与 AI（Cursor、Claude、GPT）协作的实战工作流、提示词模板和最佳实践的开源仓库。目标是让你用 AI 更高效、更可靠地完成：

- 📝 Widget 测试与单元测试生成
- 🏗️ 状态管理重构（Riverpod、Bloc）
- 🧭 路由与导航优化
- 🎨 主题与 UI 一致性
- ⚡ 性能分析与优化
- ♿ 无障碍适配

**定位**：日更或高频更新，持续沉淀实战经验，欢迎 PR 和讨论。

---

## 📂 目录结构

```
flutter-ai-coding-kit/
├── workflows/          # 完整工作流文档（从场景→步骤→验证）
├── prompts/            # 可复用提示词模板库
├── templates/          # 项目模板与配置示例（.cursor/rules、AGENTS.md）
├── examples/           # 最小示例代码片段
└── docs/               # 原理与最佳实践文章
```

---

## 🚀 快速开始

1. **浏览 workflows/**：找到你当前场景（如补测试、重构状态管理）
2. **复制 prompts/**：拿到可直接使用的提示词片段
3. **套用 templates/**：在项目里配置 Cursor Rules 或 AGENTS.md
4. **看 docs/**：理解上下文工程、验证标准前置等核心理念

---

## ✅ 推荐工作流

| 工作流 | 状态 | 说明 |
|--------|------|------|
| [Widget 测试生成](workflows/widget-test-generation.md) | ✅ | 用 AI 为业务组件自动生成测试用例 |
| Riverpod 状态重构 | 🚧 | 从 setState 迁移到 Riverpod |
| [go_router 导航改造](workflows/gorouter-navigation.md) | ✅ | 类型安全路由 + AI 辅助迁移 |
| Golden 测试实践 | 🚧 | UI 快照测试与多尺寸验证 |
| Patrol UI 自动化 | 🚧 | 端到端测试流程 |
| Flavor 多环境配置 | 🚧 | dev/staging/prod 环境隔离 |
| 性能 Timeline 分析 | 🚧 | AI 辅助解读性能数据 |
| Semantics 无障碍 | 🚧 | 可访问性标签与测试 |

---

## 💡 核心理念

### 1. 上下文工程优先
把项目结构、依赖、规范喂给 AI，而不是让它猜。参考 [docs/context-engineering.md](docs/context-engineering.md)。

### 2. 验证标准前置
先约定测试通过标准（`flutter test --coverage`、Golden 无变化），再让 AI 生成代码。

### 3. 迭代改进
AI 第一版输出通常需人工校正；记录问题改进提示词，下次更准确。

---

## 📅 日更说明

本仓库采用**日更或高频更新**模式：
- 每次提交聚焦一个实战工作流或提示词改进
- Issue 区欢迎提问和场景建议
- PR 欢迎补充你的 Flutter AI 协作经验

---

## 🤝 贡献方式

1. Fork 本仓库
2. 创建 feature 分支：`git checkout -b feature/your-workflow`
3. 提交：`git commit -m "新增：XXX 工作流"`
4. Push：`git push origin feature/your-workflow`
5. 提交 PR

所有贡献者将列入致谢。

---

## 📄 License

MIT License - 详见 [LICENSE](LICENSE)

Copyright (c) 2026 Loong (SSLoong)

---

## 📧 联系

- GitHub: [@SSLoong](https://github.com/SSLoong)
- Issues: [flutter-ai-coding-kit/issues](https://github.com/SSLoong/flutter-ai-coding-kit/issues)

---

**一起让 Flutter 开发更智能 🚀**
