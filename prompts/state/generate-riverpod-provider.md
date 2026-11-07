# 提示词：生成 Riverpod Provider

**用途**：为新功能生成符合最佳实践的 Riverpod Provider / Notifier

---

## 📋 适用场景

- 新功能开发，需要状态管理
- 添加数据层 Provider（异步加载、缓存）
- 添加依赖注入 Provider（Service、Repository）
- 添加计算状态 Provider（derived state）

---

## 📥 输入要求

在 Cursor 中准备：

1. **功能需求描述**（如"用户收藏列表管理"）
2. **数据模型定义**（如 `Favorite` 类）
3. **依赖的 Service/Repository 接口**（如 `FavoriteRepository`）
4. **（可选）现有 Provider 示例**（作为风格参考）

---

## 💬 提示词模板

```markdown
为以下功能生成 Riverpod Provider 代码。

**功能描述**：[功能名称和职责，如"管理用户收藏列表，支持添加/删除/查询"]

**数据模型**：
```dart
[粘贴或描述数据模型类，如：]
class Favorite {
  final String id;
  final String userId;
  final String itemId;
  final DateTime createdAt;

  Favorite({required this.id, required this.userId, required this.itemId, required this.createdAt});
}
```

**依赖接口**：
- `[RepositoryClass]`：[提供的方法，如 fetchFavorites(userId), addFavorite(userId, itemId), removeFavorite(favoriteId)]

**Provider 类型需求**：

根据以下场景，生成对应 Provider：

### 场景 1：依赖注入（单例服务）

**若需要全局单例服务（如 Repository、ApiClient、Database）**：

```dart
@riverpod
[ServiceClass] [serviceName]([ServiceName]Ref ref) {
  // 返回单例实例，可依赖其他 Provider
  return [ServiceClass]();
}
```

**示例输入**：
- 服务类：`FavoriteRepository`
- 初始化方式：`FavoriteRepository(ref.read(dioProvider))`

---

### 场景 2：异步数据加载（AsyncNotifier）

**若需要加载远程/本地异步数据，自动处理 loading/error 状态**：

```dart
@riverpod
class [FeatureName] extends _$[FeatureName] {
  @override
  Future<[DataType]> build([参数]) async {
    // 初始加载逻辑
    final repo = ref.read([repoProvider]);
    return await repo.[fetchMethod]([参数]);
  }

  // 刷新方法
  Future<void> refresh() async {
    state = const AsyncValue.loading();
    state = await AsyncValue.guard(() async {
      final repo = ref.read([repoProvider]);
      return await repo.[fetchMethod]([参数]);
    });
  }

  // 业务操作方法（如添加、删除、更新）
  Future<void> [operationName]([参数]) async {
    final repo = ref.read([repoProvider]);
    await repo.[operation]([参数]);
    ref.invalidateSelf(); // 触发重新加载
  }
}
```

**示例输入**：
- 功能名：`FavoriteList`
- 数据类型：`List<Favorite>`
- family 参数：`String userId`（用户 ID）
- 业务方法：`addFavorite(String itemId)`, `removeFavorite(String favoriteId)`

---

### 场景 3：同步状态管理（Notifier）

**若需要管理本地同步状态（如配置、表单、UI 状态）**：

```dart
@riverpod
class [StateName] extends _$[StateName] {
  @override
  [StateType] build() {
    // 返回初始状态
    return [initialValue];
  }

  void [updateMethod]([newValue]) {
    state = [newValue];
  }

  void [otherMethod]() {
    state = [transform current state];
  }
}
```

**示例输入**：
- 状态名：`FilterConfig`
- 状态类型：`FilterConfig` 类（含 category、priceRange、sortBy）
- 方法：`updateCategory(String category)`, `updatePriceRange(int min, int max)`, `reset()`

---

### 场景 4：计算状态（Provider）

**若需要基于其他 Provider 派生计算状态**：

```dart
@riverpod
[ComputedType] [computedName]([ComputedName]Ref ref) {
  final [dependency1] = ref.watch([provider1]);
  final [dependency2] = ref.watch([provider2]);

  // 计算逻辑
  return [computed result based on dependencies];
}
```

**示例输入**：
- 计算状态名：`favoriteCount`
- 依赖：`favoriteListProvider(userId)`
- 计算逻辑：返回收藏列表的长度

---

**生成要求**：

1. **文件路径**：`lib/providers/[feature_name]_provider.dart`

2. **包含完整 import 和 part 声明**：
   ```dart
   import 'package:riverpod_annotation/riverpod_annotation.dart';
   import 'package:flutter_riverpod/flutter_riverpod.dart';
   import '[依赖路径]';

   part '[文件名].g.dart';
   ```

3. **AsyncNotifier 处理错误**：
   - 用 `AsyncValue.guard` 自动捕获异常
   - 或手动 try-catch 并设置 `AsyncValue.error`

4. **family 参数**：
   - AsyncNotifier / Notifier 的 build 方法第一个参数自动成为 family 参数
   - 无参数时直接 `build()`

5. **autoDispose 策略**：
   - 默认自动释放（无需显式声明）
   - 若需要保持，用 `@Riverpod(keepAlive: true)` 或运行时 `ref.keepAlive()`

6. **注释**：
   - 每个 Provider 顶部加文档注释说明职责
   - 复杂方法加注释说明参数和行为

7. **输出内容**：
   - Provider 文件完整代码
   - 使用示例（在 Widget 中如何 watch/read）
   - 需要运行的命令（build_runner）
   - 测试建议（关键场景）

生成完整代码，不要省略 import 和 part。
```

---

## ✅ 验证步骤

生成代码后执行：

```bash
# 1. 运行代码生成
dart run build_runner build --delete-conflicting-outputs

# 2. 检查生成文件
ls lib/providers/[feature_name]_provider.g.dart

# 3. 分析代码
flutter analyze lib/providers/[feature_name]_provider.dart

# 4. 编写并运行测试
flutter test test/providers/[feature_name]_provider_test.dart
```

**检查清单**：
- [ ] .g.dart 文件生成成功
- [ ] 无编译错误或警告
- [ ] Provider 命名规范（小驼峰 + Provider 后缀）
- [ ] AsyncNotifier 正确处理 loading/error
- [ ] 测试覆盖主要场景

---

## 🛠️ 常见调整

生成后可能需要手动优化：

### 1. 添加防抖/节流

频繁调用的方法加防抖：

```dart
Timer? _debounceTimer;

void search(String query) {
  _debounceTimer?.cancel();
  _debounceTimer = Timer(const Duration(milliseconds: 300), () {
    _performSearch(query);
  });
}
```

### 2. 乐观更新

操作前先更新 UI，失败后回滚：

```dart
Future<void> toggleFavorite(String itemId) async {
  final previous = state.value ?? [];
  
  // 乐观更新
  state = AsyncValue.data([
    ...previous,
    Favorite(id: 'temp', userId: userId, itemId: itemId, createdAt: DateTime.now()),
  ]);

  try {
    await ref.read(favoriteRepositoryProvider).addFavorite(userId, itemId);
    ref.invalidateSelf(); // 重新加载真实数据
  } catch (error, stack) {
    // 回滚
    state = AsyncValue.data(previous);
    state = AsyncValue.error(error, stack);
  }
}
```

### 3. 分页加载

AsyncNotifier 管理分页状态：

```dart
@riverpod
class ProductList extends _$ProductList {
  int _page = 1;
  bool _hasMore = true;

  @override
  Future<List<Product>> build() async {
    _page = 1;
    _hasMore = true;
    return await _fetchPage(1);
  }

  Future<void> loadMore() async {
    if (!_hasMore || state.isLoading) return;

    final currentItems = state.value ?? [];
    state = AsyncValue.data(currentItems); // 保持当前数据，显示底部 loading

    try {
      _page++;
      final newItems = await _fetchPage(_page);
      
      if (newItems.isEmpty) {
        _hasMore = false;
      }
      
      state = AsyncValue.data([...currentItems, ...newItems]);
    } catch (error, stack) {
      _page--; // 回退页码
      state = AsyncValue.error(error, stack);
    }
  }

  Future<List<Product>> _fetchPage(int page) async {
    final repo = ref.read(productRepositoryProvider);
    return await repo.fetchProducts(page: page, pageSize: 20);
  }

  bool get hasMore => _hasMore;
}
```

### 4. 缓存过期策略

定时失效缓存：

```dart
@riverpod
class UserProfile extends _$UserProfile {
  Timer? _cacheTimer;

  @override
  Future<User> build(String userId) async {
    // 5 分钟后失效
    _cacheTimer?.cancel();
    _cacheTimer = Timer(const Duration(minutes: 5), () {
      ref.invalidateSelf();
    });

    final repo = ref.read(userRepositoryProvider);
    return await repo.fetchUser(userId);
  }

  @override
  void dispose() {
    _cacheTimer?.cancel();
    super.dispose();
  }
}
```

### 5. 依赖其他 Provider

计算或组合多个 Provider：

```dart
@riverpod
class CartSummary extends _$CartSummary {
  @override
  CartSummaryData build() {
    final cart = ref.watch(cartProvider);
    final prices = ref.watch(priceListProvider);

    final total = cart.items.fold<double>(
      0,
      (sum, item) => sum + (prices[item.productId] ?? 0) * item.quantity,
    );

    return CartSummaryData(
      itemCount: cart.items.length,
      totalPrice: total,
      discount: _calculateDiscount(total),
    );
  }

  double _calculateDiscount(double total) {
    if (total > 100) return total * 0.1;
    return 0;
  }
}
```

---

## 📝 使用示例模板

AI 生成 Provider 后，同时生成使用示例：

```dart
// 在 Widget 中使用

// 1. 监听数据变化（自动重建）
final dataAsync = ref.watch([providerName]([参数]));

return dataAsync.when(
  data: (data) => [显示数据],
  loading: () => CircularProgressIndicator(),
  error: (error, stack) => Text('Error: $error'),
);

// 2. 触发操作（不重建）
onPressed: () {
  ref.read([providerName]([参数]).notifier).[method]([参数]);
}

// 3. 手动刷新
onRefresh: () async {
  await ref.refresh([providerName]([参数]).future);
}

// 4. 监听但不重建（side-effect）
ref.listen([providerName]([参数]), (previous, next) {
  next.whenOrNull(
    error: (error, stack) {
      ScaffoldMessenger.of(context).showSnackBar(
        SnackBar(content: Text('Error: $error')),
      );
    },
  );
});
```

---

## 🧪 测试模板

AI 同时生成测试骨架：

```dart
import 'package:flutter_test/flutter_test.dart';
import 'package:mocktail/mocktail.dart';
import 'package:riverpod/riverpod.dart';

import 'package:my_app/providers/[feature]_provider.dart';
import 'package:my_app/repositories/[repo].dart';

class Mock[Repository] extends Mock implements [Repository] {}

void main() {
  late Mock[Repository] mockRepo;

  setUp(() {
    mockRepo = Mock[Repository]();
  });

  group('[ProviderName] Tests', () {
    test('loads data successfully', () async {
      // Arrange
      when(() => mockRepo.[fetchMethod]([参数])).thenAnswer(
        (_) async => [mockData],
      );

      final container = ProviderContainer(
        overrides: [
          [repoProvider].overrideWithValue(mockRepo),
        ],
      );

      // Act
      final result = await container.read([providerName]([参数]).future);

      // Assert
      expect(result, [expectedData]);
      verify(() => mockRepo.[fetchMethod]([参数])).called(1);

      container.dispose();
    });

    test('handles error correctly', () async {
      // Arrange
      when(() => mockRepo.[fetchMethod]([参数])).thenThrow(
        Exception('Network error'),
      );

      final container = ProviderContainer(
        overrides: [
          [repoProvider].overrideWithValue(mockRepo),
        ],
      );

      // Act
      final provider = container.read([providerName]([参数]));

      // Assert
      await expectLater(
        provider.future,
        throwsException,
      );

      container.dispose();
    });

    test('operation method works', () async {
      // Arrange
      when(() => mockRepo.[operationMethod]([参数])).thenAnswer(
        (_) async => null,
      );

      final container = ProviderContainer(
        overrides: [
          [repoProvider].overrideWithValue(mockRepo),
        ],
      );

      // Act
      await container.read([providerName]([参数]).notifier).[operationMethod]([参数]);

      // Assert
      verify(() => mockRepo.[operationMethod]([参数])).called(1);

      container.dispose();
    });
  });
}
```

---

## 🔗 相关

- 工作流：[workflows/riverpod-state-refactor.md](../../workflows/riverpod-state-refactor.md)
- 重构现有代码：[refactor-to-riverpod.md](refactor-to-riverpod.md)
- Riverpod 官方示例：[https://riverpod.dev/docs/essentials/first_request](https://riverpod.dev/docs/essentials/first_request)
