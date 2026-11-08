# go_router 导航改造工作流

**用 AI 将 Flutter 应用导航迁移至 go_router，实现类型安全路由**

> 作者：Loong (SSLoong) | 日期：2026-11-08

---

## 📋 背景

Flutter 原生的 Navigator 1.0 / 命名路由存在类型安全缺失、深链接配置复杂、路由守卫难实现等问题。go_router 提供声明式路由、类型安全、中间件、嵌套路由等现代特性，但**迁移涉及全局导航逻辑变更，必须验证所有路径和交互**。

---

## 🎯 目标

- ✅ 从 Navigator 1.0 / onGenerateRoute 迁移到 go_router
- ✅ 实现类型安全路由（推荐 go_router_builder）
- ✅ 配置路由重定向（登录守卫、权限检查）
- ✅ 支持嵌套导航（ShellRoute / StatefulShellRoute）
- ✅ 集成深链接与 URL 参数处理

---

## 🔧 适用场景

| 场景 | 适合度 | 说明 |
|------|--------|------|
| 命名路由迁移 | ⭐⭐⭐⭐⭐ | 路径字符串转类型安全 |
| onGenerateRoute 重构 | ⭐⭐⭐⭐ | 复杂逻辑转声明式配置 |
| 新增深链接支持 | ⭐⭐⭐⭐⭐ | go_router 原生支持 |
| 底部导航持久化 | ⭐⭐⭐⭐ | ShellRoute 完美适配 |
| 多步骤表单流程 | ⭐⭐⭐ | 子路由嵌套管理 |

---

## 🚀 完整流程

### Step 1: 分析现有导航结构

**人工操作**：在 Cursor 中打开主路由文件（如 `lib/main.dart`、`lib/routes.dart`），收集所有路由路径和页面组件。

**AI 提示词**（可复制）：

```markdown
我需要将现有 Flutter 应用的导航迁移到 go_router。请先分析当前路由结构：

1. 列出所有路由路径（命名路由或 onGenerateRoute 分支）
2. 识别需要传递的参数（路径参数、query 参数、额外数据）
3. 找出现有的导航守卫或权限检查逻辑
4/ 识别是否有嵌套导航（如 TabBar、BottomNavigationBar）
5. 列出深链接需求（如 /product/:id、/settings）

输出迁移前的路由清单和关键依赖。
```

**检查点**：确认 AI 列出的路由清单覆盖所有页面，未遗漏参数传递场景。

---

### Step 2: 生成 go_router 配置

**添加依赖**（人工操作）：

```bash
flutter pub add go_router
# 如需类型安全路由（推荐）
flutter pub add dev:go_router_builder
flutter pub add dev:build_runner
```

**AI 提示词**（可复制）：

```markdown
基于上述路由清单，生成 `lib/router/app_router.dart`，要求：

1. **路由定义**：
   - 使用 `GoRouter` 配置所有路由
   - 路径用 `/path/:paramName` 定义路径参数
   - query 参数用 `extra` 或 `state.uri.queryParameters` 处理

2. **重定向守卫**：
   - 在 `redirect` 回调中实现登录检查
   - 示例：未登录访问 `/profile` 重定向到 `/login`

3. **错误处理**：
   - 配置 `errorBuilder` 显示 404 页面

4. **嵌套路由**（如有底部导航）：
   - 使用 `ShellRoute` 包裹子路由
   - 保持底部 Tab 状态

生成完整可运行代码，包含：
- GoRouter 实例配置
- 至少 3 个示例路由（首页、详情页、设置页）
- 一个重定向守卫示例
```

**输出示例**（参考，实际以 AI 生成为准）：

```dart
import 'package:flutter/material.dart';
import 'package:go_router/go_router.dart';
import 'package:my_app/pages/home_page.dart';
import 'package:my_app/pages/product_detail_page.dart';
import 'package:my_app/pages/profile_page.dart';
import 'package:my_app/pages/login_page.dart';
import 'package:my_app/pages/not_found_page.dart';
import 'package:my_app/services/auth_service.dart';

final appRouter = GoRouter(
  initialLocation: '/home',
  redirect: (context, state) {
    final authService = AuthService.instance;
    final isLoggedIn = authService.isLoggedIn;
    final isGoingToLogin = state.matchedLocation == '/login';

    // 未登录且访问受保护页面，重定向到登录
    if (!isLoggedIn && !isGoingToLogin) {
      if (state.matchedLocation.startsWith('/profile')) {
        return '/login?redirect=${state.matchedLocation}';
      }
    }

    // 已登录访问登录页，重定向到首页
    if (isLoggedIn && isGoingToLogin) {
      return '/home';
    }

    return null; // 不重定向
  },
  routes: [
    GoRoute(
      path: '/home',
      name: 'home',
      builder: (context, state) => const HomePage(),
    ),
    GoRoute(
      path: '/product/:id',
      name: 'product-detail',
      builder: (context, state) {
        final id = state.pathParameters['id']!;
        final category = state.uri.queryParameters['category'];
        return ProductDetailPage(productId: id, category: category);
      },
    ),
    GoRoute(
      path: '/profile',
      name: 'profile',
      builder: (context, state) => const ProfilePage(),
      routes: [
        GoRoute(
          path: 'settings',
          name: 'settings',
          builder: (context, state) => const SettingsPage(),
        ),
      ],
    ),
    GoRoute(
      path: '/login',
      name: 'login',
      builder: (context, state) {
        final redirect = state.uri.queryParameters['redirect'];
        return LoginPage(redirectTo: redirect);
      },
    ),
  ],
  errorBuilder: (context, state) => NotFoundPage(path: state.matchedLocation),
);
```

---

### Step 3: 替换 MaterialApp 配置

**AI 提示词**：

```markdown
更新 `lib/main.dart`，用 `MaterialApp.router` 替换原有的 `MaterialApp`：

1. 移除 `routes`、`onGenerateRoute`、`initialRoute` 参数
2. 添加 `routerConfig: appRouter`（引用上一步生成的 GoRouter）
3. 保留 `theme`、`localizationsDelegates` 等其他配置

生成完整的 `main.dart` 代码。
```

**输出示例**：

```dart
import 'package:flutter/material.dart';
import 'package:my_app/router/app_router.dart';

void main() {
  runApp(const MyApp());
}

class MyApp extends StatelessWidget {
  const MyApp({super.key});

  @override
  Widget build(BuildContext context) {
    return MaterialApp.router(
      title: 'My App',
      theme: ThemeData(
        colorScheme: ColorScheme.fromSeed(seedColor: Colors.blue),
        useMaterial3: true,
      ),
      routerConfig: appRouter, // ✅ 使用 GoRouter
    );
  }
}
```

---

### Step 4: 重写页面导航调用

**人工操作**：全局搜索 `Navigator.push`、`Navigator.pushNamed` 等调用。

**AI 提示词**：

```markdown
替换所有导航调用为 go_router API：

| 原 API | go_router 替代 |
|--------|---------------|
| `Navigator.pushNamed(context, '/page')` | `context.go('/page')` 或 `context.push('/page')` |
| `Navigator.pushNamed(context, '/detail', arguments: id)` | `context.go('/detail/$id')` |
| `Navigator.pop(context)` | `context.pop()` |
| `Navigator.popUntil(...)` | `context.go('/home')` |

区别：
- `go()`: 替换当前路由栈（浏览器模式）
- `push()`: 推入新路由（移动端模式）
- `pop()`: 返回上一级

批量替换 `[文件路径列表]` 中的导航调用，保持语义一致。
```

---

### Step 5: 本地验证（必须！）

**人工操作**：

```bash
# 编译检查
flutter analyze

# 运行应用
flutter run
```

**检查清单**：
- [ ] 所有页面可正常访问（手动点击每个路由）
- [ ] 路径参数正确传递（如 `/product/123`）
- [ ] query 参数正确解析（如 `?category=electronics`）
- [ ] 重定向守卫生效（未登录访问受保护页面跳转登录）
- [ ] 返回栈行为符合预期（`context.pop()` 正确返回）
- [ ] 深链接可用（在浏览器或 `adb` 测试 `myapp://product/123`）
- [ ] 嵌套导航保持状态（底部 Tab 切换不重建页面）

**深链接测试**（Android）：

```bash
adb shell am start -W -a android.intent.action.VIEW \
  -d "myapp://product/123" com.example.myapp
```

**深链接测试**（iOS）：

```bash
xcrun simctl openurl booted "myapp://product/123"
```

**常见问题**：
- ❌ `Bad state: No GoRouter found in context`：确保 `MaterialApp.router` 配置了 `routerConfig`
- ❌ 路径参数为 null：检查路由定义是否用 `/:paramName`，获取时用 `state.pathParameters['paramName']`
- ❌ 重定向死循环：检查 `redirect` 回调逻辑，确保有终止条件
- ❌ 底部导航重建：改用 `ShellRoute` 包裹子路由

---

### Step 6: 类型安全路由（可选但推荐）

**适合有大量路由的项目**。使用 `go_router_builder` 生成类型安全代码。

**AI 提示词**：

```markdown
将 `lib/router/app_router.dart` 改造为类型安全路由：

1. **定义路由类**：
   - 每个页面创建一个继承 `GoRouteData` 的类
   - 用 `@TypedGoRoute` 注解声明路径

2. **示例**：
   ```dart
   @TypedGoRoute<ProductDetailRoute>(path: '/product/:id')
   class ProductDetailRoute extends GoRouteData {
     final String id;
     final String? category;

     ProductDetailRoute({required this.id, this.category});

     @override
     Widget build(BuildContext context, GoRouterState state) {
       return ProductDetailPage(productId: id, category: category);
     }
   }
   ```

3. **导航调用**：
   - `context.go()` 改为 `ProductDetailRoute(id: '123', category: 'electronics').go(context)`

4. **生成代码**：
   - 运行 `flutter pub run build_runner build --delete-conflicting-outputs`

输出完整的类型安全路由定义和使用示例。
```

**验证步骤**：

```bash
# 生成路由代码
flutter pub run build_runner build --delete-conflicting-outputs

# 检查生成的 .g.dart 文件
ls lib/router/*.g.dart

# 运行应用验证
flutter run
```

---

## ✅ 验证清单

完成后确认：

- [ ] 所有页面路由已迁移到 `GoRouter`
- [ ] `flutter analyze` 无错误
- [ ] 手动测试所有导航路径（包括嵌套路由）
- [ ] 路径参数和 query 参数正确传递
- [ ] 重定向守卫逻辑正常工作
- [ ] 深链接在 Android/iOS 上可用
- [ ] 返回栈行为符合预期
- [ ] 嵌套导航（如底部 Tab）状态保持

---

## 🤔 FAQ

### Q1: `context.go()` 和 `context.push()` 有什么区别？
**A**: 
- `go()`: 替换整个路由栈，适合 Web 浏览器模式（返回键返回上一个浏览记录）
- `push()`: 推入新路由到栈顶，适合移动端（返回键返回上一个页面）
- 建议：移动端用 `push()`，Web 端根据需求选择

### Q2: 如何传递复杂对象（如 User 实例）？
**A**: 
1. **推荐**：传递 ID，页面内重新加载
   ```dart
   context.go('/user/${user.id}');
   // 在页面中用 ID 从 Repository 获取
   ```
2. **临时方案**：用 `extra` 参数（不支持深链接）
   ```dart
   context.push('/user', extra: user);
   final user = GoRouterState.of(context).extra as User;
   ```

### Q3: 如何实现登录后返回原页面？
**A**: 
```dart
// 重定向时带上原路径
redirect: (context, state) {
  if (!isLoggedIn && state.matchedLocation != '/login') {
    return '/login?redirect=${Uri.encodeComponent(state.matchedLocation)}';
  }
  return null;
}

// 登录成功后跳转
final redirect = state.uri.queryParameters['redirect'];
if (redirect != null) {
  context.go(Uri.decodeComponent(redirect));
} else {
  context.go('/home');
}
```

### Q4: ShellRoute 和 StatefulShellRoute 有什么区别？
**A**: 
- `ShellRoute`: 简单嵌套布局（如带侧边栏的页面）
- `StatefulShellRoute`: 多个导航分支各自保持状态（如底部 Tab，切换不重建）
- 示例：
  ```dart
  StatefulShellRoute.indexedStack(
    builder: (context, state, navigationShell) {
      return ScaffoldWithNavBar(navigationShell: navigationShell);
    },
    branches: [
      StatefulShellBranch(routes: [/* 首页路由 */]),
      StatefulShellBranch(routes: [/* 搜索路由 */]),
      StatefulShellBranch(routes: [/* 个人路由 */]),
    ],
  );
  ```

### Q5: 如何调试路由问题？
**A**: 
```dart
GoRouter(
  debugLogDiagnostics: true, // ✅ 打印路由日志
  routes: [...],
);
```
查看控制台输出的路由匹配和重定向日志。

---

## 💡 最佳实践

1. **路径命名清晰**
   - ✅ `/product/:id` 
   - ❌ `/p/:i`（不直观）

2. **重定向守卫简洁**
   - 只在 `GoRouter` 的 `redirect` 中处理全局守卫
   - 页面级守卫用 `GoRoute.redirect`

3. **避免 extra 传递重对象**
   - 深链接无法携带 `extra`
   - 传 ID 后页面内加载更健壮

4. **测试深链接覆盖**
   - 每个路由都应支持直接通过 URL 访问
   - 在真机上测试 scheme（如 `myapp://`）

5. **渐进式迁移**
   - 先迁移主路由，验证通过后再迁移子路由
   - 保留原代码的备份分支

---

## 🔗 相关资源

- 提示词模板：
  - [迁移到 go_router](../prompts/navigation/migrate-to-gorouter.md)
  - [添加类型安全路由与功能](../prompts/navigation/add-typed-route-and-features.md)
- 上下文工程：[docs/context-engineering.md](../docs/context-engineering.md)
- go_router 官方文档：[https://pub.dev/packages/go_router](https://pub.dev/packages/go_router)
- 深链接配置指南：[https://docs.flutter.dev/ui/navigation/deep-linking](https://docs.flutter.dev/ui/navigation/deep-linking)

---

**Happy Routing! 🧭**
