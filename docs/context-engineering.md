# Flutter 项目上下文工程实践

**如何让 AI 更懂你的 Flutter 项目**

> 作者：Loong (SSLoong) | 日期：2026-11-05

---

## 🎯 什么是上下文工程

**定义**：在与 AI 协作前，主动提供项目结构、依赖、规范等信息，让 AI 生成的代码更符合项目实际需求。

**核心理念**：AI 不是魔法，它需要"看见"才能"理解"。上下文越详细，输出质量越高。

---

## 🏗️ Flutter 项目的关键上下文

### 1. 项目结构与分层

**为什么重要**：让 AI 知道文件该放在哪个目录，避免混乱。

**提供方式**：

在 `AGENTS.md` 或提示词中明确：

```markdown
项目采用 Feature-First 结构：

lib/
├── features/
│   ├── auth/
│   │   ├── data/       # API、模型、数据源
│   │   ├── domain/     # 业务逻辑、用例
│   │   └── presentation/  # UI、状态管理
│   └── home/
├── shared/             # 跨 feature 共享组件
└── core/               # 基础设施（网络、DI、常量）

AI 生成代码时请遵循此结构。
```

**实战技巧**：在 Cursor 中用 `@folder` 引用整个目录结构，让 AI 看到实际文件组织。

---

### 2. 依赖清单（pubspec.yaml）

**为什么重要**：避免 AI 使用你项目中不存在的包，或生成不兼容的代码。

**提供方式**：

方式 A：引用文件
```markdown
@pubspec.yaml

基于 pubspec.yaml 中的依赖生成代码。
```

方式 B：明确列出关键依赖
```markdown
项目使用以下包：
- flutter_bloc: ^8.1.3（状态管理）
- dio: ^5.4.0（网络请求）
- go_router: ^13.0.0（路由）
- mocktail: ^1.0.0（测试 mock）

请用这些包，不要引入其他替代品。
```

**常见错误**：
- ❌ AI 用 `http` 但项目用 `dio`
- ❌ AI 用 `mockito` 但项目用 `mocktail`

---

### 3. 路由表与导航

**为什么重要**：让 AI 知道如何导航，避免生成错误的路由代码。

**提供方式**：

```markdown
项目使用 go_router，路由定义在 `lib/app/router/app_router.dart`：

- `/` → HomePage
- `/login` → LoginPage
- `/profile/:userId` → ProfilePage

跳转用：`context.go('/login')` 或 `context.push('/profile/123')`
```

或引用路由文件：
```markdown
@app/router/app_router.dart

基于现有路由生成导航代码。
```

---

### 4. 主题与设计系统

**为什么重要**：让 AI 生成的 UI 风格一致，符合设计规范。

**提供方式**：

```markdown
主题配置在 `lib/app/theme/app_theme.dart`：

- 主色：Colors.blue[700]
- 字体：Roboto
- 按钮样式：ElevatedButton 用 AppButtonStyle.primary
- 间距：用 AppSpacing（small: 8, medium: 16, large: 24）

生成 UI 时请遵循现有主题，不要硬编码颜色。
```

或提供设计规范文档：
```markdown
@docs/design_system.md

按照设计规范生成 UI 代码。
```

---

### 5. 状态管理模式

**为什么重要**：不同状态管理方案代码差异大，必须明确。

**提供方式**：

```markdown
项目使用 flutter_bloc：

- 业务逻辑在 Cubit/Bloc
- UI 用 BlocProvider + BlocBuilder/BlocConsumer
- 事件命名：XxxEvent（如 LoginRequested）
- 状态命名：XxxState（如 LoginLoading, LoginSuccess）

示例参考 `lib/features/auth/presentation/bloc/auth_bloc.dart`。
```

或引用示例文件：
```markdown
@features/auth/presentation/bloc/auth_bloc.dart

按照此 Bloc 模式生成新的状态管理代码。
```

---

### 6. API 接口与数据模型

**为什么重要**：让 AI 知道后端接口格式，生成匹配的网络请求和模型。

**提供方式**：

方式 A：提供 API 文档
```markdown
@docs/api.md

用户登录接口：
POST /api/auth/login
Body: { "email": "...", "password": "..." }
Response: { "token": "...", "user": { "id": "...", "name": "..." } }

基于此接口生成 Dart 模型和 API 调用代码。
```

方式 B：提供现有模型
```markdown
@features/auth/data/models/user_model.dart

按照 UserModel 的 JSON 序列化风格生成新模型。
```

---

### 7. 平台通道与原生交互（可选）

**为什么重要**：涉及 MethodChannel 时，AI 需要知道通道名称和方法签名。

**提供方式**：

```markdown
项目有平台通道 `com.example.app/native_util`：

- 方法 `getBatteryLevel()` → Future<int>
- 方法 `openSettings()` → Future<void>

生成调用代码时用此通道名。
```

---

## 💡 实战案例

### 案例 1：生成新 Feature 的完整代码

**场景**：为"产品列表"功能生成 data / domain / presentation 三层代码。

**上下文提供**：

```markdown
@AGENTS.md
@pubspec.yaml
@app/router/app_router.dart
@features/auth/  # 提供参考示例

需求：生成 features/product/ 包含：
1. data/models/product_model.dart（JSON 字段：id, name, price, imageUrl）
2. data/repositories/product_repository_impl.dart（调用 GET /api/products）
3. domain/entities/product.dart + domain/repositories/product_repository.dart
4. presentation/bloc/product_bloc.dart + presentation/pages/product_list_page.dart

遵循项目现有 feature 的结构和风格。
```

**结果**：AI 生成的代码结构一致、风格统一，依赖正确。

---

### 案例 2：修复跨模块 Bug

**场景**：登录后路由跳转失败。

**上下文提供**：

```markdown
@features/auth/presentation/bloc/auth_bloc.dart
@app/router/app_router.dart

问题：用户登录成功后，调用 `context.go('/home')` 报错 "Invalid route name"。

检查路由配置是否有 `/home`，若没有请告知正确路由。
```

**结果**：AI 发现路由表里是 `/` 而非 `/home`，给出正确跳转代码。

---

### 案例 3：补充测试

**场景**：为 `ProductCard` Widget 生成测试。

**上下文提供**：

```markdown
@lib/shared/widgets/product_card.dart
@test/features/auth/presentation/widgets/login_button_test.dart  # 作为风格参考

生成 test/shared/widgets/product_card_test.dart，覆盖：
1. 正常渲染（显示名称、价格、图片）
2. 点击卡片触发回调
3. 价格为 0 时显示"免费"

用 mocktail，风格对齐 login_button_test.dart。
```

**结果**：AI 生成的测试风格一致，mock 方式正确。

---

## 📝 最佳实践总结

### ✅ 做

1. **维护 `AGENTS.md`**：项目根目录放置协作规则，每次大改后更新
2. **引用真实文件**：用 `@file` 或 `@folder` 而非口头描述
3. **提供示例**：现有代码是最好的风格指南
4. **明确版本**：说明 Flutter/Dart 版本和关键依赖版本
5. **分步验证**：生成一部分代码 → 验证 → 再生成下一部分

### ❌ 不做

1. ❌ 假设 AI 知道你的项目结构（它不知道）
2. ❌ 模糊描述依赖（"用网络库" → 具体到 `dio: ^5.4.0`）
3. ❌ 不提供路由表就让 AI 写导航
4. ❌ 不看生成的 import 是否匹配项目依赖
5. ❌ 一次性让 AI 生成整个 feature（分步更可控）

---

## 🔗 相关工具

- **Cursor Rules**：项目级配置 → [templates/cursor/rules/flutter-core.mdc](../templates/cursor/rules/flutter-core.mdc)
- **AGENTS.md 模板**：[templates/AGENTS.md](../templates/AGENTS.md)
- **Widget 测试工作流**：[workflows/widget-test-generation.md](../workflows/widget-test-generation.md)

---

## 🎓 进阶技巧

### 1. 用 Mermaid 图解释架构

在 `AGENTS.md` 或提示词中用 Mermaid 流程图说明数据流：

```mermaid
graph LR
  UI[ProductListPage] --> Bloc[ProductBloc]
  Bloc --> Repo[ProductRepository]
  Repo --> API[ApiClient]
  API --> Backend[/api/products]
```

### 2. 提供反例

告诉 AI 什么**不要**做：

```markdown
❌ 不要在 Widget 中直接调用 Dio
❌ 不要用 Provider（项目用 flutter_bloc）
❌ 不要生成 *.g.dart（我们手动写模型）
```

### 3. 记录 AI 常犯的错误

在项目 `.cursor/rules` 中记录：

```markdown
AI 常见错误修正：
1. 生成 ListView 时忘记加 Key → 必须有 `key: Key(item.id)`
2. 用 `setState` 而非 Bloc → 统一用 Bloc emit
3. Mock 用 `mockito` → 项目统一用 `mocktail`
```

---

**总结**：上下文工程是 AI 协作的基石，投入 10 分钟整理上下文，能节省 1 小时调试时间。✨
