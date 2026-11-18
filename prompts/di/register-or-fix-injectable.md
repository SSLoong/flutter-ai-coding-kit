# 提示词：注册或修复 injectable 依赖

**用途**：为新服务补充 injectable 注册、修复注册错误、或解决测试中的 DI 问题

---

## 📋 适用场景

- 新建 Service/Repository 需注册到 DI 容器
- build_runner 生成失败（找不到实现、循环依赖、注解错误）
- 测试中 mock 依赖不生效
- 接口绑定失败（找不到实现类）
- 环境切换后依赖解析错误

---

## 📥 输入要求

在 Cursor 中准备：

1. **目标服务类源码**（需要注册的类）
2. **错误信息**（如有 build_runner 报错，完整复制）
3. **依赖关系**（该服务依赖哪些其他服务）
4. **当前 `lib/injection.dart`**（用于检查配置）

---

## 💬 提示词模板

### 场景 1：为新服务添加 injectable 注册

```markdown
我需要为新建的服务类添加 injectable 注册。

**服务类信息**：
- 类名：[服务类名，如 PaymentService]
- 文件路径：`lib/[路径]/[文件名].dart`
- 生命周期：[单例/懒加载单例/工厂]
- 依赖的服务：[列出依赖，如 ApiClient, LocalStorage]

**是否有接口**：
- [ ] 是（接口名：[接口名]，需要依赖倒置）
- [ ] 否（直接注册实现类）

**要求**：

1. 为服务类添加正确的 injectable 注解：
   - 单例用 `@lazySingleton`
   - 工厂用 `@injectable`
   - 有接口用 `@LazySingleton(as: 接口类)`

2. 如果依赖其他服务，通过构造函数注入

3. 如果是接口绑定，同时生成接口定义（abstract class）

4. 提供完整的代码，包含所有必要的 import

5. 提供运行命令：
   ```bash
   dart run build_runner build --delete-conflicting-outputs
   ```

**示例**（参考）：

```dart
// 接口定义
abstract class PaymentService {
  Future<PaymentResult> processPayment(PaymentRequest request);
}

// 实现类（带注解）
import 'package:injectable/injectable.dart';
import '../domain/services/payment_service.dart';
import '../../core/api/api_client.dart';

@LazySingleton(as: PaymentService)
class PaymentServiceImpl implements PaymentService {
  final ApiClient _apiClient;

  PaymentServiceImpl(this._apiClient);

  @override
  Future<PaymentResult> processPayment(PaymentRequest request) async {
    final response = await _apiClient.post('/payments', request.toJson());
    return PaymentResult.fromJson(response.data);
  }
}
```

生成完整代码。
```

---

### 场景 2：修复 build_runner 生成错误

```markdown
我在运行 `dart run build_runner build` 时遇到错误。

**错误信息**：
```
[完整复制错误信息，包括堆栈跟踪]
```

**相关服务类**：
[列出报错涉及的类名和文件路径]

**要求**：

1. 分析错误原因（常见问题）：
   - ❌ 找不到实现类：检查注解和 import
   - ❌ 循环依赖：A → B → A
   - ❌ 抽象类未实现：`@Injectable(as: X)` 但无实现类
   - ❌ 冲突的注册：同一接口有多个实现但未指定环境
   - ❌ 漏调 configureDependencies()
   - ❌ injection.config.dart 路径错误

2. 提供修复方案（代码 + 解释）

3. 提供验证命令：
   ```bash
   # 清理后重新生成
   flutter clean
   flutter pub get
   dart run build_runner build --delete-conflicting-outputs
   ```

**输出示例**（参考）：

**问题**：循环依赖 UserService → AuthService → UserService

**原因**：
```dart
@lazySingleton
class UserService {
  final AuthService _authService;
  UserService(this._authService);
}

@lazySingleton
class AuthService {
  final UserService _userService; // ❌ 循环依赖
  AuthService(this._userService);
}
```

**解决方案 1**：引入中间层

```dart
// 创建 EventBus 解耦
@lazySingleton
class EventBus {
  final _controller = StreamController<Event>.broadcast();
  Stream<Event> get events => _controller.stream;
  void fire(Event event) => _controller.add(event);
}

@lazySingleton
class UserService {
  final EventBus _eventBus;
  UserService(this._eventBus) {
    _eventBus.events.listen((event) {
      if (event is UserLoggedOut) {
        // 处理登出
      }
    });
  }
}

@lazySingleton
class AuthService {
  final EventBus _eventBus;
  AuthService(this._eventBus);
  
  void logout() {
    _eventBus.fire(UserLoggedOut());
  }
}
```

**解决方案 2**：延迟初始化

```dart
@lazySingleton
class UserService {
  // 不在构造函数中注入 AuthService
  UserService();
  
  AuthService get authService => getIt<AuthService>(); // 用时获取
}
```

生成修复后的完整代码。
```

---

### 场景 3：为环境配置添加不同实现

```markdown
我需要为不同环境（dev/prod）注册不同的服务实现。

**服务接口**：[接口名，如 ApiClient]

**实现类**：
- Dev 环境：[类名，如 MockApiClient]（返回 mock 数据）
- Prod 环境：[类名，如 ApiClientImpl]（真实网络请求）

**要求**：

1. 定义环境常量（如果还没有）：
   ```dart
   abstract class Environment {
     static const dev = 'dev';
     static const prod = 'prod';
   }
   ```

2. 为不同实现添加环境注解：
   ```dart
   @LazySingleton(as: ApiClient, env: [Environment.dev])
   class MockApiClient implements ApiClient {}

   @LazySingleton(as: ApiClient, env: [Environment.prod])
   class ApiClientImpl implements ApiClient {}
   ```

3. 更新 `lib/injection.dart`，支持环境参数：
   ```dart
   void configureDependencies({String environment = Environment.prod}) {
     getIt.init(environment: environment);
   }
   ```

4. 提供运行命令：
   ```bash
   # Dev 环境
   flutter run --dart-define=ENV=dev

   # Prod 环境
   flutter run --dart-define=ENV=prod
   ```

生成完整代码（接口 + 两个实现 + 环境配置）。
```

---

### 场景 4：修复测试中 mock 不生效

```markdown
我在测试中注册了 mock 依赖，但 Widget/ViewModel 还是使用了真实服务。

**测试文件**：`test/[路径]/[文件名]_test.dart`

**问题现象**：
[描述问题，如："mock UserRepository 已注册，但测试中仍然调用真实网络请求"]

**当前测试代码**（关键部分）：
```dart
[粘贴 setUp() 和相关测试代码]
```

**要求**：

检查并修复以下常见问题：

1. **是否调用了 `getIt.reset()`？**
   ```dart
   setUp(() async {
     await getIt.reset(); // 必须在注册 mock 前重置
     mockUserRepository = MockUserRepository();
     getIt.registerSingleton<UserRepository>(mockUserRepository);
   });
   ```

2. **是否在 pumpWidget 之前注册 mock？**
   ```dart
   // ✅ 正确顺序
   setUp(() async {
     await getIt.reset();
     getIt.registerSingleton<UserRepository>(mockUserRepository); // 先注册
   });

   testWidgets('test', (tester) async {
     await tester.pumpWidget(...); // 后渲染
   });
   ```

3. **Widget 是否缓存了依赖？**
   ```dart
   // ❌ 错误：构造函数中获取（测试前已缓存）
   class MyWidget extends StatelessWidget {
     final repository = getIt<UserRepository>();
   }

   // ✅ 正确：build/initState 中获取
   class MyWidget extends StatelessWidget {
     @override
     Widget build(BuildContext context) {
       final repository = getIt<UserRepository>(); // 每次获取
     }
   }
   ```

4. **是否用了 @InjectableInit 初始化？**
   ```dart
   // ❌ 测试中不要调用 configureDependencies()
   setUp(() async {
     configureDependencies(); // 会注册真实服务
     getIt.registerSingleton<UserRepository>(mockUserRepository);
   });

   // ✅ 直接 reset 后手动注册 mock
   setUp(() async {
     await getIt.reset();
     getIt.registerSingleton<UserRepository>(mockUserRepository);
   });
   ```

提供修复后的完整测试代码。
```

---

## ✅ 验证步骤

### 新增服务注册后

```bash
# 1. 运行代码生成
dart run build_runner build --delete-conflicting-outputs

# 2. 检查生成文件
cat lib/injection.config.dart | grep [服务类名]

# 3. 运行应用验证
flutter run

# 4. 在代码中测试获取
final service = getIt<[服务类名]>();
```

### 修复错误后

```bash
# 1. 清理构建缓存
flutter clean

# 2. 重新安装依赖
flutter pub get

# 3. 重新生成
dart run build_runner build --delete-conflicting-outputs

# 4. 确认无报错
echo $?  # 应输出 0
```

### 测试 mock 修复后

```bash
# 运行测试
flutter test test/[路径]/[文件名]_test.dart

# 检查清单
- [ ] 测试全部通过
- [ ] 无网络请求（mock 生效）
- [ ] verify() 断言正确
```

---

## 🛠️ 常见调整

### 1. 接口绑定的多种写法

```dart
// ✅ 推荐（injectable 3.x）
@LazySingleton(as: UserRepository)
class UserRepositoryImpl implements UserRepository {}

// ✅ 也可（同等效果）
@Injectable(as: UserRepository)
@lazySingleton
class UserRepositoryImpl implements UserRepository {}

// ❌ 错误（缺少 as）
@lazySingleton
class UserRepositoryImpl implements UserRepository {} // 只会注册实现类，不会绑定接口
```

### 2. 多实现时必须指定环境

```dart
// ❌ 错误：同一接口多个实现，未指定环境
@LazySingleton(as: ApiClient)
class MockApiClient implements ApiClient {}

@LazySingleton(as: ApiClient)
class ApiClientImpl implements ApiClient {} // 报错：冲突

// ✅ 正确：用环境区分
@LazySingleton(as: ApiClient, env: [Environment.dev])
class MockApiClient implements ApiClient {}

@LazySingleton(as: ApiClient, env: [Environment.prod])
class ApiClientImpl implements ApiClient {}
```

### 3. @preResolve 用于异步初始化

```dart
@module
abstract class CoreModule {
  @preResolve
  Future<SharedPreferences> get prefs => SharedPreferences.getInstance();
}

// lib/injection.dart 需改为异步
@InjectableInit()
Future<void> configureDependencies() => getIt.init();

// main.dart
void main() async {
  WidgetsFlutterBinding.ensureInitialized();
  await configureDependencies(); // 等待异步初始化
  runApp(const MyApp());
}
```

### 4. @factoryMethod 用于复杂构造

```dart
@lazySingleton
class DatabaseService {
  final Database _db;
  
  DatabaseService._(this._db);

  @factoryMethod
  static Future<DatabaseService> create() async {
    final db = await openDatabase('app.db');
    return DatabaseService._(db);
  }
}
```

---

## 🔗 相关

- 工作流：[workflows/get-it-injectable-di.md](../../workflows/get-it-injectable-di.md)
- 初始化搭建：[setup-get-it-injectable.md](setup-get-it-injectable.md)
- injectable 文档：[https://pub.dev/packages/injectable](https://pub.dev/packages/injectable)
- get_it 文档：[https://pub.dev/packages/get_it](https://pub.dev/packages/get_it)
