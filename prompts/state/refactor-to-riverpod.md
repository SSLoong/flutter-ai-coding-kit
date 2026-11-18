# 提示词：重构到 Riverpod

**用途**：将现有 StatefulWidget / Provider 代码迁移到 Riverpod 2.x

---

## 📋 适用场景

- StatefulWidget 中业务逻辑复杂，需要抽离
- 旧 Provider 包迁移到 Riverpod
- 状态需要跨多个 Widget 共享
- 需要提升可测试性（依赖注入）

---

## 📥 输入要求

在 Cursor 中准备：

1. **目标 Widget 源码**（如 `lib/pages/product_detail_page.dart`）
2. **相关 Service/Repository**（如 ProductRepository）
3. **现有状态变量列表**（State 类中的字段）
4. **（可选）现有 Provider 定义**（若从 Provider 包迁移）

---

## 💬 提示词模板

```markdown
将以下 Flutter 代码重构为使用 Riverpod 2.x。

**原文件**：`lib/[原文件路径]`

**现状分析**：
1. 当前使用 StatefulWidget，State 类中有以下状态：
   - `[状态变量1]`：[类型] - [用途]
   - `[状态变量2]`：[类型] - [用途]

2. 依赖以下外部服务：
   - `[Service1]`：[职责]
   - `[Repository1]`：[职责]

3. initState 中的初始化操作：
   - [列出初始化逻辑]

4. setState 调用点：
   - [列出触发重建的场景]

**重构要求**：

### 1. 生成 Provider 定义

**文件**：`lib/providers/[feature_name]_provider.dart`

- 使用 `riverpod_annotation` + `riverpod_generator`
- 依赖服务用单例 Provider：
  ```dart
  @riverpod
  [ServiceClass] [serviceName]([ServiceName]Ref ref) {
    return [ServiceClass](); // 或从其他 Provider 获取
  }
  ```

- 异步数据加载用 AsyncNotifier：
  ```dart
  @riverpod
  class [FeatureName] extends _$[FeatureName] {
    @override
    Future<[DataType]> build([参数]) async {
      final repo = ref.read([repoProvider]);
      return await repo.fetchData([参数]);
    }

    Future<void> refresh() async {
      state = const AsyncValue.loading();
      state = await AsyncValue.guard(() async {
        final repo = ref.read([repoProvider]);
        return await repo.fetchData([参数]);
      });
    }

    Future<void> [operationName]([参数]) async {
      final repo = ref.read([repoProvider]);
      await repo.[operation]([参数]);
      ref.invalidateSelf(); // 触发重新加载
    }
  }
  ```

- 同步状态用 Notifier：
  ```dart
  @riverpod
  class [StateName] extends _$[StateName] {
    @override
    [StateType] build() => [initialValue];

    void update([newValue]) {
      state = [newValue];
    }
  }
  ```

- 计算状态用 Provider：
  ```dart
  @riverpod
  [ComputedType] [computedName]([ComputedName]Ref ref) {
    final data = ref.watch([dataProvider]);
    return [computed logic based on data];
  }
  ```

### 2. 重构 Widget

- 将 StatefulWidget 改为：
  - **ConsumerWidget**：若不需要本地状态
  - **ConsumerStatefulWidget**：若仍需本地状态（如 TextEditingController、AnimationController）

- build 方法签名改为：
  ```dart
  @override
  Widget build(BuildContext context, WidgetRef ref) { ... }
  ```

- watch Provider 监听变化：
  ```dart
  final dataAsync = ref.watch([providerName]([参数]));

  return dataAsync.when(
    data: (data) => [显示数据的 Widget],
    loading: () => CircularProgressIndicator(),
    error: (error, stack) => Text('Error: $error'),
  );
  ```

- read Provider 触发操作（不监听）：
  ```dart
  onPressed: () {
    ref.read([providerName]([参数]).notifier).[method]();
  }
  ```

- 移除：
  - State 类定义
  - setState 调用
  - initState / dispose 中的数据加载逻辑（Riverpod 自动管理）

- 保留：
  - TextEditingController / AnimationController（需手动 dispose）
  - 纯 UI 本地状态（如展开/收起、焦点）可用 useState hook 或 StatefulWidget

### 3. 包含完整 import

```dart
// Provider 文件
import 'package:riverpod_annotation/riverpod_annotation.dart';
import 'package:flutter_riverpod/flutter_riverpod.dart';
import '[依赖路径]';

part '[文件名].g.dart';

// Widget 文件
import 'package:flutter/material.dart';
import 'package:flutter_riverpod/flutter_riverpod.dart';
import '[provider 路径]';
```

### 4. 输出内容

1. **新 Provider 文件完整代码**（`lib/providers/[feature]_provider.dart`）
2. **重构后的 Widget 文件完整代码**（原路径）
3. **需要在终端运行的命令**（code_gen）
4. **测试建议**（哪些场景需要验证）

生成所有代码，不要省略任何部分。
```

---

## ✅ 验证步骤

生成代码后执行：

```bash
# 1. 运行代码生成
dart run build_runner build --delete-conflicting-outputs

# 2. 检查分析器
flutter analyze

# 3. 运行测试（若已有）
flutter test

# 4. 运行应用验证
flutter run
```

**检查清单**：
- [ ] .g.dart 文件生成成功
- [ ] 无编译错误
- [ ] 分析器无警告
- [ ] 应用运行正常，核心流程验证通过
- [ ] 热重载工作正常

---

## 🛠️ 常见调整

生成后可能需要手动微调：

### 1. 依赖注入链

若 Service 依赖其他 Service：

```dart
@riverpod
UserRepository userRepository(UserRepositoryRef ref) {
  final dio = ref.read(dioProvider); // 依赖另一个 Provider
  return UserRepository(dio);
}
```

### 2. family 参数传递

AsyncNotifier 的 build 方法第一个参数自动成为 family 参数：

```dart
@riverpod
class ProductDetail extends _$ProductDetail {
  @override
  Future<Product> build(String productId) async { // productId 是 family 参数
    final repo = ref.read(productRepositoryProvider);
    return await repo.fetchProduct(productId);
  }
}

// 使用时传参
ref.watch(productDetailProvider('prod-123'))
```

### 3. keepAlive 控制

默认 autoDispose（无监听者时释放），若需要保持：

```dart
@Riverpod(keepAlive: true) // 全局保持
class GlobalConfig extends _$GlobalConfig { ... }

// 或运行时控制
@riverpod
class UserProfile extends _$UserProfile {
  @override
  Future<User> build(String userId) async {
    ref.keepAlive(); // 本次实例保持
    // ...
  }
}
```

### 4. 细粒度监听

只监听特定字段变化：

```dart
// ❌ 任何字段变化都重建
final user = ref.watch(userProvider);
Text(user.name)

// ✅ 只有 name 变化才重建
final name = ref.watch(userProvider.select((u) => u.name));
Text(name)
```

### 5. 错误处理增强

AsyncValue.guard 会捕获异常，也可自定义：

```dart
Future<void> updateData() async {
  state = const AsyncValue.loading();
  try {
    final repo = ref.read(repoProvider);
    final data = await repo.update();
    state = AsyncValue.data(data);
  } catch (error, stack) {
    state = AsyncValue.error(error, stack);
    // 可选：显示 SnackBar
    ref.read(snackbarServiceProvider).showError(error.toString());
  }
}
```

---

## 🔗 相关

- 工作流：[workflows/riverpod-state-refactor.md](../../workflows/riverpod-state-refactor.md)
- 生成新 Provider：[generate-riverpod-provider.md](generate-riverpod-provider.md)
- Riverpod 官方迁移指南：[https://riverpod.dev/docs/from_provider/motivation](https://riverpod.dev/docs/from_provider/motivation)
