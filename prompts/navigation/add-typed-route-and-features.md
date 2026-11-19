# 提示词：添加类型安全路由与功能

**用途**：为现有 go_router 项目添加类型安全路由（go_router_builder）或诊断常见路由问题

---

## 📋 适用场景

- 已使用 go_router 但路由定义是字符串（易出错）
- 需要添加 ShellRoute 或路由重定向
- 深链接、参数传递出现 bug
- 路由返回栈行为异常

---

## 📥 输入要求

在 Cursor 中准备：

1. **现有路由配置文件**（如 `lib/router/app_router.dart`）
2. **页面组件文件**（需要添加路由的页面）
3. **（可选）错误日志**（诊断问题时提供）

---

## 💬 提示词模板

选择以下场景之一：

---

### 场景 A：升级为类型安全路由

```markdown
我需要将现有 go_router 配置升级为类型安全路由（使用 go_router_builder）。

---

**当前路由文件**：`lib/router/app_router.dart`

**要求**：

1. **添加依赖**：
   ```yaml
   dependencies:
     go_router: ^[当前版本]
   
   dev_dependencies:
     go_router_builder: ^[最新版本]
     build_runner: ^[最新版本]
   ```

2. **定义类型安全路由类**：
   
   为以下页面创建路由类（继承 `GoRouteData`）：
   
   - [列出页面，如：HomePage、ProductDetailPage、SettingsPage]

   **示例格式**：
   ```dart
   @TypedGoRoute<HomeRoute>(path: '/home')
   class HomeRoute extends GoRouteData {
     @override
     Widget build(BuildContext context, GoRouterState state) {
       return const HomePage();
     }
   }

   @TypedGoRoute<ProductDetailRoute>(path: '/product/:id')
   class ProductDetailRoute extends GoRouteData {
     final String id;
     final String? category; // query 参数

     ProductDetailRoute({required this.id, this.category});

     @override
     Widget build(BuildContext context, GoRouterState state) {
       return ProductDetailPage(productId: id, category: category);
     }
   }
   ```

3. **更新 GoRouter 配置**：
   ```dart
   final appRouter = GoRouter(
     routes: $appRoutes, // ✅ 使用生成的路由
   );
   ```

4. **更新导航调用**：
   
   替换字符串路径为类型安全调用：
   
   | 原调用 | 新调用 |
   |--------|--------|
   | `context.push('/product/123')` | `ProductDetailRoute(id: '123').push(context)` |
   | `context.go('/home')` | `HomeRoute().go(context)` |

5. **生成代码**：
   ```bash
   flutter pub run build_runner build --delete-conflicting-outputs
   ```

**输出**：
- 完整的类型安全路由定义代码
- 更新后的导航调用示例
- 生成命令

```

---

### 场景 B：添加嵌套路由（ShellRoute）

```markdown
我需要为现有 go_router 项目添加嵌套路由，实现底部导航栏状态保持。

---

**需求描述**：

- 底部导航栏有 [列出 Tab，如：首页、搜索、个人中心]
- 每个 Tab 有独立的导航栈
- 切换 Tab 时保持各自页面状态（不重建）

**要求**：

1. 使用 `StatefulShellRoute.indexedStack` 配置
2. 为每个 Tab 创建一个 `StatefulShellBranch`
3. 包裹底部导航栏的 Scaffold

**输出**：
- 完整的 `StatefulShellRoute` 配置代码
- 带底部导航栏的 Scaffold Widget

```

---

### 场景 C：添加路由重定向（登录守卫）

```markdown
我需要为 go_router 添加全局路由守卫，实现登录检查。

---

**需求**：

- 未登录用户访问 [列出受保护路径，如：/profile、/settings] 时重定向到 `/login`
- 登录后跳转回原页面（通过 `?redirect=` 参数）
- 已登录用户访问 `/login` 重定向到 `/home`

**要求**：

1. 在 `GoRouter` 的 `redirect` 回调中实现
2. 使用 [AuthService / Provider / Riverpod] 获取登录状态
3. 处理重定向循环（确保有终止条件）

**输出**：
- 完整的 `redirect` 回调代码
- 登录页处理 `redirect` 参数的代码

```

---

### 场景 D：诊断路由问题

```markdown
我的 go_router 项目出现以下问题，需要诊断和修复：

---

**问题描述**：
[描述问题，如：深链接无法打开、参数传递失败、返回栈错误、重定向死循环]

**错误日志**：
```
[粘贴完整错误信息或日志]
```

**相关代码**：
```dart
[粘贴问题相关的路由配置或导航调用代码]
```

---

**分析要求**：

1. 判断问题原因（选一个或多个）：
   - 路径定义错误（缺少 `:paramName`）
   - 参数获取方式错误（`pathParameters` vs `queryParameters`）
   - 重定向逻辑错误（死循环或条件错误）
   - 深链接配置缺失（AndroidManifest.xml / Info.plist）
   - 嵌套路由配置错误（ShellRoute 使用不当）

2. 提供修复方案：
   - 修复后的路由配置代码
   - 需要更改的文件清单
   - 验证步骤

```

---

## ✅ 验证步骤

根据场景选择验证命令：

### 场景 A（类型安全路由）

```bash
# 生成代码
flutter pub run build_runner build --delete-conflicting-outputs

# 检查生成文件
ls lib/router/*.g.dart

# 编译检查
flutter analyze

# 运行应用
flutter run
```

**检查清单**：
- [ ] 生成的 `.g.dart` 文件无编译错误
- [ ] 所有导航调用改为类型安全 API
- [ ] 参数传递正确（编译时类型检查）

---

### 场景 B（嵌套路由）

```bash
# 运行应用
flutter run

# 测试步骤
1. 点击底部导航栏的各个 Tab
2. 在某个 Tab 进入子页面
3. 切换到其他 Tab
4. 切换回来，检查页面状态是否保持
```

**检查清单**：
- [ ] 底部导航栏显示正常
- [ ] 切换 Tab 不重建页面
- [ ] 各 Tab 导航栈独立
- [ ] 返回键在各 Tab 内正确返回

---

### 场景 C（路由守卫）

```bash
# 运行应用
flutter run

# 测试步骤
1. 未登录状态访问 /profile → 应重定向到 /login?redirect=/profile
2. 登录成功后 → 应跳转回 /profile
3. 已登录状态访问 /login → 应重定向到 /home
```

**检查清单**：
- [ ] 未登录访问受保护页面重定向正确
- [ ] 登录后返回原页面
- [ ] 无重定向死循环
- [ ] 公开页面不受影响

---

### 场景 D（诊断问题）

根据 AI 提供的修复方案执行验证步骤。

---

## 🛠️ 常见模式

### 1. 类型安全路由示例

```dart
// lib/router/app_router.dart

import 'package:flutter/material.dart';
import 'package:go_router/go_router.dart';

part 'app_router.g.dart';

// 首页路由
@TypedGoRoute<HomeRoute>(path: '/home')
class HomeRoute extends GoRouteData {
  @override
  Widget build(BuildContext context, GoRouterState state) {
    return const HomePage();
  }
}

// 详情页路由（带参数）
@TypedGoRoute<ProductDetailRoute>(path: '/product/:id')
class ProductDetailRoute extends GoRouteData {
  final String id;
  final String? category; // query 参数

  const ProductDetailRoute({required this.id, this.category});

  @override
  Widget build(BuildContext context, GoRouterState state) {
    return ProductDetailPage(productId: id, category: category);
  }
}

// 嵌套路由
@TypedGoRoute<SettingsRoute>(
  path: '/settings',
  routes: [
    TypedGoRoute<AccountSettingsRoute>(path: 'account'),
    TypedGoRoute<PrivacySettingsRoute>(path: 'privacy'),
  ],
)
class SettingsRoute extends GoRouteData {
  @override
  Widget build(BuildContext context, GoRouterState state) {
    return const SettingsPage();
  }
}

class AccountSettingsRoute extends GoRouteData {
  @override
  Widget build(BuildContext context, GoRouterState state) {
    return const AccountSettingsPage();
  }
}

class PrivacySettingsRoute extends GoRouteData {
  @override
  Widget build(BuildContext context, GoRouterState state) {
    return const PrivacySettingsPage();
  }
}

// GoRouter 配置
final appRouter = GoRouter(
  routes: $appRoutes, // 使用生成的路由
  debugLogDiagnostics: true,
);
```

**导航调用**：
```dart
// ✅ 类型安全
ProductDetailRoute(id: '123', category: 'books').push(context);

// ✅ 嵌套路由
AccountSettingsRoute().push(context);
```

---

### 2. 深链接调试

```dart
GoRouter(
  debugLogDiagnostics: true, // ✅ 开启日志
  routes: [...],
);
```

**查看日志**：
```
flutter: GoRouter.redirect: /product/123 => null
flutter: GoRouter.go: /product/123
```

**测试深链接**：

Android:
```bash
adb shell am start -W -a android.intent.action.VIEW \
  -d "myapp://product/123" com.example.myapp
```

iOS:
```bash
xcrun simctl openurl booted "myapp://product/123"
```

---

### 3. 参数传递最佳实践

| 参数类型 | 适用场景 | 示例 |
|---------|---------|------|
| 路径参数 | 必需的标识符 | `/product/:id` |
| query 参数 | 可选的过滤条件 | `/search?q=keyword&category=books` |
| extra | 临时的复杂对象（不支持深链接） | `context.push('/detail', extra: user)` |

**推荐**：尽量用路径参数和 query 参数，保证深链接可用。

---

## 🔗 相关

- 工作流：[workflows/gorouter-navigation.md](../../workflows/gorouter-navigation.md)
- 迁移到 go_router：[migrate-to-gorouter.md](migrate-to-gorouter.md)
- go_router_builder 文档：[https://pub.dev/packages/go_router_builder](https://pub.dev/packages/go_router_builder)
