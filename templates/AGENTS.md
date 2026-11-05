# AGENTS.md 模板

**用途**：在 Flutter 项目根目录放置此文件，告知 AI 协作规则和项目约定

---

## 📝 使用方法

1. 将此模板复制到你的 Flutter 项目根目录：`cp templates/AGENTS.md YOUR_PROJECT/AGENTS.md`
2. 替换 `[占位符]` 为实际内容
3. 在 Cursor 中引用此文件（`@AGENTS.md`）让 AI 理解项目规范

---

## 📄 模板内容

```markdown
# AI 协作约定

本项目使用 AI（Cursor、Claude、GPT）辅助开发。请 AI 遵守以下规则。

---

## 📂 项目信息

- **项目名称**：[YOUR_APP_NAME]
- **Flutter 版本**：[3.19.0 / stable]
- **Dart 版本**：[3.3.0]
- **主要依赖**：
  - 状态管理：[flutter_bloc / riverpod / provider]
  - 路由：[go_router / auto_route]
  - 网络：[dio / http]
  - 本地存储：[hive / shared_preferences]

---

## 🏗️ 项目结构

```
lib/
├── main.dart               # 入口
├── app/                    # 全局配置（路由、主题）
├── features/               # 按功能分模块
│   ├── auth/
│   │   ├── data/           # 数据层（API、模型）
│   │   ├── domain/         # 业务逻辑（用例、仓库接口）
│   │   └── presentation/   # UI 层（Widget、状态）
│   └── home/
├── shared/                 # 共享组件、工具
│   ├── widgets/
│   ├── utils/
│   └── constants/
└── core/                   # 核心基础（网络、存储、依赖注入）

test/                       # 测试镜像 lib/ 结构
```

---

## ✅ 代码规范

### 1. 命名约定
- 文件名：`snake_case.dart`
- 类名：`PascalCase`
- 变量/函数：`camelCase`
- 常量：`kConstantName` 或 `UPPER_SNAKE_CASE`

### 2. Widget 规范
- 优先使用 `StatelessWidget`
- 状态管理用 `[YOUR_STATE_SOLUTION]`
- 复用组件放 `shared/widgets/`
- 业务组件放对应 feature 的 `presentation/widgets/`

### 3. 测试要求
- Widget 测试：`test/features/*/presentation/widgets/`
- 单元测试：`test/features/*/domain/` 和 `test/shared/utils/`
- Mock 使用 `mocktail`
- 覆盖率目标：≥ 80%

### 4. 依赖注入
- 使用 `[get_it / riverpod / provider]`
- 服务注册在 `core/di/injection.dart`

### 5. 错误处理
- 网络错误统一用 `Result<T>` 或 `Either<Failure, T>`
- UI 层捕获异常并显示用户友好提示

---

## 🚫 禁止操作

- ❌ 直接修改自动生成的文件（`*.g.dart`, `*.freezed.dart`）
- ❌ 在 Widget 中写业务逻辑（抽到 ViewModel/Bloc/Cubit）
- ❌ 硬编码字符串（用 `lib/shared/constants/strings.dart` 或 i18n）
- ❌ 不写测试就提交代码

---

## 🤖 AI 生成代码检查清单

每次生成代码后，AI 需确认：

- [ ] 遵循项目结构（文件放在正确目录）
- [ ] 遵循命名规范
- [ ] Import 正确（相对路径用 `package:` 形式）
- [ ] 有对应测试（或说明不需要测试的理由）
- [ ] 代码可编译通过（`flutter analyze` 无错误）
- [ ] 运行测试通过（`flutter test`）

---

## 📚 参考资源

- 路由配置：`lib/app/router/[app_router.dart]`
- 主题配置：`lib/app/theme/[app_theme.dart]`
- API 文档：`[docs/api.md]`
- 设计规范：`[Figma / docs/design.md]`

---

## 🔄 工作流

### 新增 Feature
1. 在 `lib/features/` 创建目录
2. 分 data / domain / presentation 三层
3. 写 UI 前先写测试（TDD）
4. 注册依赖（DI）
5. 添加路由（如需要）

### 修复 Bug
1. 先写复现 bug 的测试
2. 修复代码让测试通过
3. 确保未破坏现有测试

### 重构
1. 确保现有测试全通过
2. 重构代码
3. 再次运行所有测试

---

**更新日期**：[2026-11-05]
```

---

## 🔗 相关

- Cursor Rules：[templates/cursor/rules/flutter-core.mdc](cursor/rules/flutter-core.mdc)
- 上下文工程：[docs/context-engineering.md](../docs/context-engineering.md)
