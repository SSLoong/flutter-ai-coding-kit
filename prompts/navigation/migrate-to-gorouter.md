# 提示词：迁移到 go_router

**用途**：将 Flutter 应用从 Navigator 1.0 / 命名路由 / onGenerateRoute 迁移到 go_router

---

## 📋 适用场景

- 使用 `Navigator.pushNamed()` 的命名路由项目
- 使用 `onGenerateRoute` 的自定义路由项目
- 需要添加类型安全路由和深链接支持

---

## 📥 输入要求

在 Cursor 中准备：

1. **主路由文件**（如 `lib/main.dart` 或 `lib/routes.dart`）
2. **所有页面组件文件**（如 `lib/pages/*.dart`）
3. **（可选）现有的路由配置代码**（`routes: {}` 或 `onGenerateRoute`）

---

## 💬 提示词模板

```markdown
我需要将 Flutter 应用的导航系统从 [Navigator 1.0 / 命名路由 / onGenerateRoute] 迁移到 go_router。

---

### 第一步：分析现有路由

**当前路由配置文件**：`[文件路径，如 lib/routes.dart]`

请分析并输出：

1. **路由清单**（格式：路径 → 页面组件 → 参数）
   ```
   例如：
   - /home → HomePage（无参数）
   - /product/:id → ProductPage（参数：id）
   - /settings → SettingsPage（无参数）
   ```

2. **参数传递方式**
   - 路径参数（如 `/user/123`）
   - 命名参数（如 `arguments: {'userId': '123'}`）
   - query 参数（如 `?tab=profile`）

3. **现有守卫逻辑**
   - 是否有登录检查？
   - 是否有权限判断？
   - 在哪里实现的？

4. **嵌套导航**
   - 是否有底部导航栏（BottomNavigationBar）？
   - 是否有多级嵌套页面？

---

### 第二步：生成 go_router 配置

基于上述分析，生成 `lib/router/app_router.dart`：

**要求**：

1. **依赖声明**：
   ```yaml
   dependencies:
     go_router: ^[最新版本]
   ```

2. **GoRouter 配置**：
   - 定义所有路由（使用 `GoRoute`）
   - 路径参数用 `/:paramName` 格式
   - query 参数用 `state.uri.queryParameters` 获取

3. **重定向守卫**（如有登录逻辑）：
   - 在 `GoRouter.redirect` 回调中实现
   - 未登录访问受保护页面时重定向到 `/login?redirect=[原路径]`

4. **错误处理**：
   - 配置 `errorBuilder` 返回 404 页面

5. **嵌套路由**（如有底部导航）：
   - 使用 `ShellRoute` 或 `StatefulShellRoute`
   - 保持导航栏状态

**输出**：
- 完整的 `app_router.dart` 代码
- 包含所有必要的 import
- 至少 3 个示例路由配置

---

### 第三步：更新 MaterialApp

生成更新后的 `lib/main.dart`：

**变更**：
- `MaterialApp` 改为 `MaterialApp.router`
- 移除 `routes`、`onGenerateRoute`、`initialRoute`
- 添加 `routerConfig: appRouter`

**保留**：
- `theme`、`localizationsDelegates`、`title` 等其他配置不变

---

### 第四步：替换导航调用

批量替换以下文件中的导航 API：

`[列出需要修改的文件路径]`

**替换规则**：

| 原 API | 新 API |
|--------|--------|
| `Navigator.pushNamed(context, '/page')` | `context.push('/page')` |
| `Navigator.pushNamed(context, '/detail', arguments: id)` | `context.push('/detail/$id')` |
| `Navigator.pop(context)` | `context.pop()` |
| `Navigator.popUntil(context, ModalRoute.withName('/home'))` | `context.go('/home')` |

**注意**：
- `context.go()`: 替换整个路由栈（Web 模式）
- `context.push()`: 推入新路由（移动端模式）
- 根据原代码语义选择合适的 API

---

### 第五步：生成验证清单

输出一份测试清单，包括：

1. **基础验证**：
   - [ ] `flutter analyze` 通过
   - [ ] 应用启动正常
   - [ ] 所有页面可访问

2. **参数传递验证**：
   - [ ] 路径参数正确（如 `/product/123`）
   - [ ] query 参数正确（如 `?category=electronics`）

3. **守卫验证**：
   - [ ] 未登录访问受保护页面重定向到登录
   - [ ] 登录后可正常访问

4. **导航行为验证**：
   - [ ] 返回键行为正确
   - [ ] 嵌套导航状态保持

5. **深链接验证**：
   - [ ] Android: `adb shell am start -W -a android.intent.action.VIEW -d "myapp://[path]" [package]`
   - [ ] iOS: `xcrun simctl openurl booted "myapp://[path]"`

```

---

## ✅ 验证步骤

生成代码后执行：

```bash
# 添加依赖
flutter pub add go_router

# 代码分析
flutter analyze

# 运行应用
flutter run

# 测试所有路由路径
# （手动点击每个页面，验证导航和参数传递）
```

---

## 🛠️ 常见调整

### 1. 修复路径参数缺失

**问题**：
```dart
// ❌ 路径定义缺少参数
GoRoute(path: '/product', ...)
```

**修复**：
```dart
// ✅ 添加路径参数
GoRoute(
  path: '/product/:id',
  builder: (context, state) {
    final id = state.pathParameters['id']!;
    return ProductPage(productId: id);
  },
)
```

---

### 2. 处理复杂参数传递

**问题**：原代码用 `arguments` 传递多个参数

```dart
// ❌ 原代码
Navigator.pushNamed(context, '/detail', arguments: {
  'id': '123',
  'category': 'books',
});
```

**修复方案 A**：用 query 参数
```dart
// ✅ 方案 A
context.push('/detail/123?category=books');

// 页面中获取
final id = state.pathParameters['id']!;
final category = state.uri.queryParameters['category'];
```

**修复方案 B**：用 extra（不支持深链接）
```dart
// ⚠️ 方案 B（仅内部导航）
context.push('/detail', extra: {'id': '123', 'category': 'books'});

// 页面中获取
final args = state.extra as Map<String, String>;
```

---

### 3. 配置登录守卫

```dart
GoRouter(
  redirect: (context, state) {
    final authService = context.read<AuthService>();
    final isLoggedIn = authService.isLoggedIn;
    final isGoingToLogin = state.matchedLocation == '/login';

    // 未登录且不是去登录页，重定向
    if (!isLoggedIn && !isGoingToLogin) {
      if (_needsAuth(state.matchedLocation)) {
        return '/login?redirect=${Uri.encodeComponent(state.matchedLocation)}';
      }
    }

    // 已登录去登录页，重定向到首页
    if (isLoggedIn && isGoingToLogin) {
      return '/home';
    }

    return null; // 不重定向
  },
  routes: [...],
);

bool _needsAuth(String path) {
  return path.startsWith('/profile') || path.startsWith('/settings');
}
```

---

### 4. 配置底部导航（StatefulShellRoute）

```dart
StatefulShellRoute.indexedStack(
  builder: (context, state, navigationShell) {
    return Scaffold(
      body: navigationShell,
      bottomNavigationBar: BottomNavigationBar(
        currentIndex: navigationShell.currentIndex,
        onTap: (index) => navigationShell.goBranch(index),
        items: const [
          BottomNavigationBarItem(icon: Icon(Icons.home), label: '首页'),
          BottomNavigationBarItem(icon: Icon(Icons.search), label: '搜索'),
          BottomNavigationBarItem(icon: Icon(Icons.person), label: '我的'),
        ],
      ),
    );
  },
  branches: [
    StatefulShellBranch(
      routes: [
        GoRoute(
          path: '/home',
          builder: (context, state) => const HomePage(),
        ),
      ],
    ),
    StatefulShellBranch(
      routes: [
        GoRoute(
          path: '/search',
          builder: (context, state) => const SearchPage(),
        ),
      ],
    ),
    StatefulShellBranch(
      routes: [
        GoRoute(
          path: '/profile',
          builder: (context, state) => const ProfilePage(),
        ),
      ],
    ),
  ],
);
```

---

## 🔗 相关

- 工作流：[workflows/gorouter-navigation.md](../../workflows/gorouter-navigation.md)
- 添加类型安全路由：[add-typed-route-and-features.md](add-typed-route-and-features.md)
- go_router 官方文档：[https://pub.dev/packages/go_router](https://pub.dev/packages/go_router)
