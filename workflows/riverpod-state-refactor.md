# Riverpod 状态重构工作流

**用 AI 将 Flutter 状态管理从 setState 或旧方案迁移到 Riverpod**

> 作者：Loong (SSLoong) | 日期：2026-11-07

---

## 📋 背景

随着 Flutter 应用复杂度增长，setState 和旧的 Provider 方案会带来状态共享困难、测试不便、性能问题等挑战。Riverpod 提供了编译时安全、自动依赖管理、易测试的现代状态管理方案。AI 能帮助快速识别重构点、生成 Provider/Notifier 代码，但**必须经过测试验证**才能合入。

---

## 🎯 目标

- ✅ 从 StatefulWidget + setState 迁移到 ConsumerWidget / ConsumerStatefulWidget
- ✅ 将业务逻辑从 Widget 中抽离到 Provider / Notifier
- ✅ 建立类型安全的依赖注入体系
- ✅ 可测试：用 ProviderContainer override mock 依赖
- ✅ 符合 Riverpod 2.x 最佳实践（autoDispose、family、AsyncNotifier）

---

## 🔧 适用场景

| 场景 | 适合度 | 说明 |
|------|--------|------|
| StatefulWidget 业务逻辑重 | ⭐⭐⭐⭐⭐ | 状态、网络请求、计算混在一起 |
| 跨 Widget 状态共享 | ⭐⭐⭐⭐⭐ | InheritedWidget / Provider 繁琐 |
| 难以测试的 Widget | ⭐⭐⭐⭐⭐ | mock 依赖困难 |
| 性能优化需求 | ⭐⭐⭐⭐ | 细粒度重建、autoDispose 释放 |
| 简单表单状态 | ⭐⭐ | setState 可能更简单 |
| 一次性临时状态 | ⭐ | 不必引入 Riverpod |

---

## 🚀 完整流程

### Step 0: 决策边界（何时用 Riverpod）

**判断标准**（AI 提示词）：

```markdown
分析以下 Widget/Feature，判断是否应该使用 Riverpod：

**代码路径**：`[文件路径]`

**判断维度**：
1. 状态是否需要跨多个 Widget 共享？
2. 有异步数据加载（网络请求、数据库）吗？
3. 业务逻辑是否复杂（>50 行计算/转换）？
4. 是否需要独立测试业务逻辑？
5. 是否需要细粒度控制重建范围？

如果 3 个及以上为"是"，建议 Riverpod；否则说明理由。
```

**检查点**：AI 的建议是否合理？若是简单表单或一次性状态，继续用 setState 更高效。

---

### Step 1: 梳理现有状态与依赖

**人工操作**：在 Cursor 中打开目标文件（如 `lib/pages/user_profile_page.dart`），附上相关 Service/Repository。

**AI 提示词**（可复制）：

```markdown
分析这个 StatefulWidget 的状态管理，列出：

1. **State 类中的状态变量**（如 `_isLoading`, `_userData`）
2. **依赖的外部服务**（如 UserRepository, AuthService）
3. **生命周期操作**（initState 中的初始化、dispose 中的清理）
4. **setState 调用点**（触发重建的位置）
5. **状态作用域**（仅本 Widget 用还是需要共享）

输出表格，标注每个状态的建议方案：
- 本地状态 → 保留 setState 或 StateProvider
- 异步数据 → AsyncNotifierProvider
- 计算状态 → Provider / computed
- 依赖注入 → Provider
```

**输出示例**：

| 状态/依赖 | 当前实现 | 作用域 | 建议方案 |
|----------|---------|-------|---------|
| `_isLoading` | `bool` in State | 本 Widget | 保留 setState 或移入 Notifier |
| `_userData` | `User?` in State | 本 Widget + 子组件 | AsyncNotifierProvider |
| `UserRepository` | 构造函数注入 | 全局 | Provider（单例） |
| `_errorMessage` | `String?` in State | 本 Widget | 移入 AsyncNotifier 的 error |

**检查点**：确认分类合理，特别是哪些状态真正需要 Riverpod。

---

### Step 2: 生成 Provider 定义

**AI 提示词**（可复制）：

```markdown
基于上述分析，生成 Riverpod Provider 代码。

**目标文件**：`lib/providers/user_profile_provider.dart`

**要求**：

1. **依赖 Provider**（单例服务）：
   ```dart
   @riverpod
   UserRepository userRepository(UserRepositoryRef ref) {
     return UserRepository(); // 或从 Dio 等初始化
   }
   ```

2. **数据 Provider**（异步加载，自动释放）：
   ```dart
   @riverpod
   class UserProfile extends _$UserProfile {
     @override
     Future<User> build(String userId) async {
       final repo = ref.read(userRepositoryProvider);
       return await repo.fetchUser(userId);
     }

     Future<void> refresh() async {
       state = const AsyncValue.loading();
       state = await AsyncValue.guard(() async {
         final repo = ref.read(userRepositoryProvider);
         return await repo.fetchUser(userId);
       });
     }

     Future<void> updateProfile(String name) async {
       final repo = ref.read(userRepositoryProvider);
       await repo.updateUser(userId, name);
       ref.invalidateSelf(); // 触发重新 build
     }
   }
   ```

3. **使用 riverpod_annotation + riverpod_generator**（推荐）
4. **AsyncNotifier 处理异步**（不用手动管理 loading/error）
5. **family 传参**（如 userId）
6. **autoDispose 自动释放**（默认行为）

生成完整代码，包含 import 和 part 声明。
```

**输出示例**：

```dart
import 'package:riverpod_annotation/riverpod_annotation.dart';
import 'package:my_app/repositories/user_repository.dart';
import 'package:my_app/models/user.dart';

part 'user_profile_provider.g.dart';

// 依赖注入：UserRepository 单例
@riverpod
UserRepository userRepository(UserRepositoryRef ref) {
  return UserRepository();
}

// 数据 Provider：异步加载用户信息（带 family 参数）
@riverpod
class UserProfile extends _$UserProfile {
  @override
  Future<User> build(String userId) async {
    final repo = ref.read(userRepositoryProvider);
    return await repo.fetchUser(userId);
  }

  Future<void> refresh() async {
    state = const AsyncValue.loading();
    state = await AsyncValue.guard(() async {
      final repo = ref.read(userRepositoryProvider);
      return await repo.fetchUser(arg); // arg 是 family 参数
    });
  }

  Future<void> updateName(String newName) async {
    final repo = ref.read(userRepositoryProvider);
    await repo.updateUser(arg, newName);
    ref.invalidateSelf();
  }
}
```

**检查点**：
- [ ] import 正确（`riverpod_annotation`, `flutter_riverpod`）
- [ ] part 声明匹配文件名
- [ ] family 参数正确传递（build 的第一个参数）
- [ ] ref.read 用于不监听，ref.watch 用于监听变化

---

### Step 3: 运行代码生成

**人工操作**：

```bash
# 安装依赖（首次）
flutter pub add flutter_riverpod riverpod_annotation
flutter pub add -d riverpod_generator build_runner

# 运行 code_gen
dart run build_runner build --delete-conflicting-outputs

# 或 watch 模式（开发时）
dart run build_runner watch
```

**检查清单**：
- [ ] 生成 `.g.dart` 文件无报错
- [ ] 分析器无警告（`flutter analyze`）
- [ ] Provider 名称符合规范（小驼峰 + Provider 后缀）

**常见错误**：
- ❌ `part 'xxx.g.dart'` 路径错误 → 确保与文件名匹配
- ❌ `@riverpod` 导入错误 → 检查 `import 'package:riverpod_annotation/riverpod_annotation.dart'`
- ❌ 生成失败 → 运行 `dart run build_runner clean` 后重试

---

### Step 4: 重构 Widget 使用 Provider

**AI 提示词**（可复制）：

```markdown
将 StatefulWidget 重构为使用 Riverpod Provider。

**原文件**：`lib/pages/user_profile_page.dart`
**Provider**：`lib/providers/user_profile_provider.dart`

**要求**：

1. **改为 ConsumerWidget 或 ConsumerStatefulWidget**：
   - 无需本地状态 → ConsumerWidget
   - 仍需本地状态（如 TextEditingController）→ ConsumerStatefulWidget

2. **在 build 方法中 watch Provider**：
   ```dart
   @override
   Widget build(BuildContext context, WidgetRef ref) {
     final userAsync = ref.watch(userProfileProvider('user123'));

     return userAsync.when(
       data: (user) => Text(user.name),
       loading: () => CircularProgressIndicator(),
       error: (err, stack) => Text('Error: $err'),
     );
   }
   ```

3. **用 ref.read 触发方法**（不监听变化）：
   ```dart
   onPressed: () {
     ref.read(userProfileProvider('user123').notifier).updateName('Alice');
   }
   ```

4. **移除**：
   - State 类
   - setState 调用
   - initState / dispose 中的数据加载（Riverpod 自动管理）

5. **保留**：
   - 纯 UI 本地状态（如动画控制器、焦点）可用 useState hook 或保留 StatefulWidget
   - 表单输入用 TextEditingController（dispose 仍需手动）

生成重构后的完整 Widget 代码。
```

**输出示例**：

```dart
import 'package:flutter/material.dart';
import 'package:flutter_riverpod/flutter_riverpod.dart';
import 'package:my_app/providers/user_profile_provider.dart';

class UserProfilePage extends ConsumerWidget {
  final String userId;

  const UserProfilePage({Key? key, required this.userId}) : super(key: key);

  @override
  Widget build(BuildContext context, WidgetRef ref) {
    final userAsync = ref.watch(userProfileProvider(userId));

    return Scaffold(
      appBar: AppBar(title: const Text('User Profile')),
      body: userAsync.when(
        data: (user) => Center(
          child: Column(
            mainAxisAlignment: MainAxisAlignment.center,
            children: [
              Text('Name: ${user.name}', style: Theme.of(context).textTheme.headlineSmall),
              const SizedBox(height: 16),
              ElevatedButton(
                onPressed: () {
                  ref.read(userProfileProvider(userId).notifier).updateName('New Name');
                },
                child: const Text('Update Name'),
              ),
            ],
          ),
        ),
        loading: () => const Center(child: CircularProgressIndicator()),
        error: (error, stack) => Center(child: Text('Error: $error')),
      ),
      floatingActionButton: FloatingActionButton(
        onPressed: () {
          ref.invalidate(userProfileProvider(userId)); // 刷新
        },
        child: const Icon(Icons.refresh),
      ),
    );
  }
}
```

**检查点**：
- [ ] Widget 继承 ConsumerWidget / ConsumerStatefulWidget
- [ ] build 方法签名包含 WidgetRef ref
- [ ] watch 用于监听，read 用于触发方法
- [ ] AsyncValue.when 处理 loading/error/data
- [ ] 无 setState 调用
- [ ] 入口改为 ProviderScope（见 Step 5）

---

### Step 5: 更新应用入口

**人工操作**：确保 `main.dart` 用 ProviderScope 包裹。

```dart
import 'package:flutter/material.dart';
import 'package:flutter_riverpod/flutter_riverpod.dart';
import 'package:my_app/app.dart';

void main() {
  runApp(
    const ProviderScope(
      child: MyApp(),
    ),
  );
}
```

---

### Step 6: 编写测试（必须！）

**AI 提示词**（可复制）：

```markdown
为重构后的 Widget 和 Provider 生成测试。

**要求**：

1. **Provider 测试**（`test/providers/user_profile_provider_test.dart`）：
   ```dart
   test('loads user data', () async {
     final container = ProviderContainer(
       overrides: [
         userRepositoryProvider.overrideWithValue(MockUserRepository()),
       ],
     );

     when(() => mockRepo.fetchUser('123')).thenAnswer(
       (_) async => User(id: '123', name: 'Alice'),
     );

     final provider = container.read(userProfileProvider('123').future);
     final user = await provider;

     expect(user.name, 'Alice');
     container.dispose();
   });
   ```

2. **Widget 测试**（`test/pages/user_profile_page_test.dart`）：
   ```dart
   testWidgets('displays user name', (tester) async {
     await tester.pumpWidget(
       ProviderScope(
         overrides: [
           userProfileProvider('123').overrideWith(
             (ref) => User(id: '123', name: 'Bob'),
           ),
         ],
         child: MaterialApp(home: UserProfilePage(userId: '123')),
       ),
     );

     await tester.pumpAndSettle();
     expect(find.text('Name: Bob'), findsOneWidget);
   });
   ```

3. **覆盖**：
   - 正常加载
   - loading 状态
   - error 状态
   - 交互（按钮触发更新）

生成完整测试代码，包含 mocktail 配置。
```

**验证步骤**：

```bash
flutter test test/providers/user_profile_provider_test.dart
flutter test test/pages/user_profile_page_test.dart
```

**检查清单**：
- [ ] 所有测试通过
- [ ] ProviderContainer override 正确
- [ ] Mock 返回值类型匹配
- [ ] dispose container 避免内存泄漏

---

### Step 7: 本地运行验证

**人工操作**：

```bash
# 运行应用
flutter run

# 热重载测试功能
# 检查：
# - 数据正常加载
# - loading/error 状态正确
# - 交互（刷新、更新）生效
# - 无控制台报错/警告
```

**检查清单**：
- [ ] 冷启动无异常
- [ ] 热重载不丢状态（Riverpod 自动保持）
- [ ] 交互响应正常
- [ ] 内存无泄漏（反复进出页面观察）

---

## ✅ 验证清单

完成后确认：

- [ ] 所有 Provider 使用 @riverpod 注解，生成 .g.dart 文件
- [ ] Widget 改为 ConsumerWidget / ConsumerStatefulWidget
- [ ] 测试全部通过（flutter test）
- [ ] 分析器无警告（flutter analyze）
- [ ] 应用运行正常，核心流程验证通过
- [ ] 文档/注释更新（说明 Provider 职责）

---

## 🤔 FAQ

### Q1: 何时用 Provider vs Notifier vs AsyncNotifier？
**A**: 
- **Provider**：只读数据、计算值、单例服务（如 Dio、Repository）
- **Notifier**：可变状态，同步操作（如计数器、表单状态）
- **AsyncNotifier**：异步数据加载（网络请求、数据库），自动处理 loading/error

### Q2: watch vs read vs listen 的区别？
**A**: 
- **watch**：监听变化，Provider 更新时重建 Widget（只能在 build 中用）
- **read**：一次性读取，不监听（用于事件处理、initState）
- **listen**：监听但不重建，用于 side-effect（如显示 SnackBar）

### Q3: autoDispose 是什么？何时需要？
**A**: 
- 默认行为：Provider 无监听者时自动释放（清理缓存、关闭连接）
- 需要保持：用 `@Riverpod(keepAlive: true)` 或 `ref.keepAlive()`
- 适合：列表数据、临时详情页（离开即释放）
- 不适合：全局配置、用户信息（需要持久）

### Q4: family 是什么？如何用？
**A**: 
- 作用：Provider 带参数（如用户 ID、查询条件）
- 语法：`@riverpod` 生成器的 build 方法第一个参数自动成为 family 参数
  ```dart
  @riverpod
  class UserProfile extends _$UserProfile {
    @override
    Future<User> build(String userId) async { ... } // userId 是 family 参数
  }
  
  // 使用
  ref.watch(userProfileProvider('123'))
  ref.watch(userProfileProvider('456')) // 不同参数独立缓存
  ```

### Q5: 如何避免循环依赖？
**A**: 
- ❌ Provider A watch Provider B，B watch A
- ✅ 拆分共同依赖到 C，A 和 B 都 watch C
- ✅ 用 `ref.listen` 单向响应，而非双向 watch
- 工具：`flutter pub run riverpod_lint` 检测循环

### Q6: 旧代码用 Provider 包，迁移到 Riverpod 麻烦吗？
**A**: 
- Riverpod 与 Provider 包不兼容，需要完整迁移
- 策略：
  1. 新功能用 Riverpod
  2. 旧模块按优先级逐个迁移
  3. 过渡期可能混用（不推荐长期）
- 迁移成本：中小型应用 1-3 天，大型应用需分阶段

### Q7: 何时用 riverpod_generator vs 手写 Provider？
**A**: 
- **推荐 generator**（现代写法）：
  - 类型安全，编译时检查
  - 自动生成 Provider 名称
  - 支持 family 和 autoDispose
- **手写**（legacy）：
  - 简单场景（单个值）
  - 不想依赖 build_runner
  - 动态创建 Provider（极少见）

### Q8: 性能优化建议？
**A**: 
1. **细粒度 watch**：只 watch 需要的字段
   ```dart
   // ❌ 整个对象变化都重建
   final user = ref.watch(userProvider);
   
   // ✅ 只监听 name 字段
   final name = ref.watch(userProvider.select((u) => u.name));
   ```
2. **autoDispose**：临时数据自动释放
3. **Provider 拆分**：避免大 Provider，按职责拆小
4. **避免在 build 中 read**：会漏监听，改用 watch

---

## 💡 最佳实践

1. **明确边界**
   - 不是所有状态都需要 Riverpod
   - 简单本地状态（如 checkbox）保留 setState

2. **Provider 职责单一**
   - 一个 Provider 管理一类数据（用户、商品、订单）
   - 不要做成大杂烩

3. **测试先行**
   - 迁移前写测试覆盖现有行为
   - 重构后测试保持通过

4. **增量迁移**
   - 从叶子节点（无依赖的 Widget）开始
   - 逐步向上迁移，保持应用可运行

5. **统一规范**
   - Provider 文件统一放 `lib/providers/`
   - 命名规范：`xxxProvider`（生成器自动）
   - 注释说明职责和参数

6. **记录迁移经验**
   - 遇到的坑和解决方案加入 `.cursor/rules`
   - 更新团队文档

---

## 🔗 相关资源

- 提示词模板：
  - [prompts/state/refactor-to-riverpod.md](../prompts/state/refactor-to-riverpod.md)
  - [prompts/state/generate-riverpod-provider.md](../prompts/state/generate-riverpod-provider.md)
- 上下文工程：[docs/context-engineering.md](../docs/context-engineering.md)
- Riverpod 官方文档：[https://riverpod.dev](https://riverpod.dev)
- Riverpod 代码生成：[https://riverpod.dev/docs/concepts/about_code_generation](https://riverpod.dev/docs/concepts/about_code_generation)

---

**让状态管理更优雅 🎯**
