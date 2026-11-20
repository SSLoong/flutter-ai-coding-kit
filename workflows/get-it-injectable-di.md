# get_it + injectable 依赖注入工作流

**用 AI 为 Flutter 项目搭建和维护依赖注入架构**

> 作者：Loong (SSLoong) | 日期：2026-11-18

---

## 📋 背景

依赖注入（Dependency Injection, DI）是解耦服务层与业务逻辑的核心模式。Flutter 中常用 **get_it**（Service Locator）+ **injectable**（代码生成）实现：
- **get_it**：运行时的服务定位器，负责注册和解析依赖
- **injectable**：编译时注解处理器，自动生成注册代码

**与 Riverpod 的区别**：
- **Riverpod**：状态管理 + 响应式依赖，适合 UI 层状态共享
- **get_it + injectable**：服务定位 + 依赖注入，适合服务层（Repository、API Client、本地存储）

**两者可以共存**：Riverpod Provider 内部可以使用 `getIt<T>()` 获取服务实例。

---

## 🎯 目标

- ✅ 为项目搭建 get_it + injectable 架构
- ✅ 用注解自动生成依赖注册代码
- ✅ 支持接口绑定（abstract → implementation）
- ✅ 支持多环境（dev/test/prod）配置
- ✅ 在测试中轻松替换 mock 实现

---

## 🔧 适用场景

| 场景 | 适合度 | 说明 |
|------|--------|------|
| 中大型项目服务层解耦 | ⭐⭐⭐⭐⭐ | Repository、API Client、存储层 |
| 需要接口/实现分离 | ⭐⭐⭐⭐⭐ | 依赖倒置原则，方便测试 |
| 多环境配置（dev/prod） | ⭐⭐⭐⭐ | 用 `@Environment` 区分 |
| 小型应用或纯 UI 项目 | ⭐⭐ | 可能过度设计 |

---

## 🚀 完整流程

### Step 1: 添加依赖

**人工操作**：编辑 `pubspec.yaml`，添加以下依赖：

```yaml
dependencies:
  # Service Locator
  get_it: ^8.0.0

  # 注解与代码生成
  injectable: ^3.0.0

dev_dependencies:
  # 代码生成器
  injectable_generator: ^3.0.0
  build_runner: ^2.4.0
```

运行安装：

```bash
flutter pub get
```

---

### Step 2: 初始化 get_it 配置

**AI 提示词**（可复制）：

```markdown
我需要为 Flutter 项目搭建 get_it + injectable 依赖注入架构。

**要求**：

1. 在 `lib/injection.dart` 创建初始化文件：
   - 定义全局 `GetIt` 实例 `getIt`
   - 用 `@InjectableInit` 注解标记初始化函数
   - 生成 `configureDependencies()` 函数用于注册

2. 在 `lib/main.dart` 的 `main()` 函数中：
   - 在 `runApp()` 之前调用 `configureDependencies()`
   - 确保依赖初始化完成后再启动应用

3. 生成完整可运行代码，包含所有必要的 import。
```

**输出示例**（参考）：

**lib/injection.dart**:

```dart
import 'package:get_it/get_it.dart';
import 'package:injectable/injectable.dart';

import 'injection.config.dart';

final getIt = GetIt.instance;

@InjectableInit(
  initializerName: 'init',
  preferRelativeImports: true,
  asExtension: true,
)
void configureDependencies() => getIt.init();
```

**lib/main.dart**:

```dart
import 'package:flutter/material.dart';
import 'injection.dart';

void main() {
  // 初始化依赖注入
  configureDependencies();
  
  runApp(const MyApp());
}

class MyApp extends StatelessWidget {
  const MyApp({super.key});

  @override
  Widget build(BuildContext context) {
    return MaterialApp(
      title: 'Flutter Demo',
      home: const HomePage(),
    );
  }
}
```

---

### Step 3: 为服务添加注解

**人工操作**：确定需要注入的服务类（如 Repository、API Client、本地存储）。

**AI 提示词**（可复制）：

```markdown
为以下服务类添加 injectable 注解，实现依赖注入。

**目标类**：[列出类名，如 UserRepository, ApiClient, LocalStorage]

**要求**：

1. **单例服务**（全局唯一实例）：
   - 用 `@lazySingleton` 注解（首次使用时创建）
   - 或 `@singleton` 注解（应用启动时创建）

2. **工厂模式**（每次获取新实例）：
   - 用 `@injectable` 注解

3. **接口绑定**（依赖倒置）：
   - 抽象类用 `@Injectable(as: 接口类)`
   - 实现类用 `@LazySingleton(as: 接口类)`

4. **构造函数注入**：
   - 所有依赖通过构造函数参数传入
   - injectable 会自动解析依赖

生成带注解的完整服务类代码。
```

**输出示例**（参考）：

**lib/core/api/api_client.dart**:

```dart
import 'package:dio/dio.dart';
import 'package:injectable/injectable.dart';

@lazySingleton
class ApiClient {
  final Dio _dio;

  ApiClient(this._dio) {
    _dio.options.baseUrl = 'https://api.example.com';
  }

  Future<Response> get(String path) => _dio.get(path);
  Future<Response> post(String path, dynamic data) => _dio.post(path, data: data);
}

// Dio 本身也需要注册
@module
abstract class DioModule {
  @lazySingleton
  Dio get dio => Dio();
}
```

**接口绑定示例**：

**lib/features/user/domain/repositories/user_repository.dart**:

```dart
import '../entities/user.dart';

abstract class UserRepository {
  Future<User> getUser(String id);
  Future<void> updateUser(User user);
}
```

**lib/features/user/data/repositories/user_repository_impl.dart**:

```dart
import 'package:injectable/injectable.dart';
import '../../domain/repositories/user_repository.dart';
import '../../domain/entities/user.dart';
import '../../../../core/api/api_client.dart';

@LazySingleton(as: UserRepository)
class UserRepositoryImpl implements UserRepository {
  final ApiClient _apiClient;

  UserRepositoryImpl(this._apiClient);

  @override
  Future<User> getUser(String id) async {
    final response = await _apiClient.get('/users/$id');
    return User.fromJson(response.data);
  }

  @override
  Future<void> updateUser(User user) async {
    await _apiClient.post('/users/${user.id}', user.toJson());
  }
}
```

---

### Step 4: 运行代码生成

**人工操作**：

```bash
# 清理旧生成文件并重新生成
dart run build_runner build --delete-conflicting-outputs
```

**检查点**：
- [ ] 生成文件 `lib/injection.config.dart` 已创建
- [ ] 无报错信息
- [ ] 生成文件包含所有注解的类的注册代码

**常见问题**：
- ❌ `@InjectableInit` 找不到：检查 `lib/injection.dart` 是否正确 import `injection.config.dart`
- ❌ 循环依赖错误：检查服务间的依赖关系，避免 A → B → A
- ❌ 找不到实现类：确认实现类已添加注解且 import 正确

---

### Step 5: 在代码中使用依赖

**人工操作**：在需要依赖的地方使用 `getIt<T>()`。

**AI 提示词**（可复制）：

```markdown
在以下场景中使用 getIt 获取依赖：

**场景 1**：在 Widget 中使用 Repository
- 在 State 或 StatelessWidget 中用 `getIt<UserRepository>()` 获取实例
- 不要在 Widget 构造函数中注入（违背 Flutter 设计模式）

**场景 2**：在 ViewModel/Cubit/Bloc 中注入依赖
- 通过构造函数注入，ViewModel 本身也用 injectable 注册

生成示例代码。
```

**输出示例**（参考）：

**在 Widget 中使用**：

```dart
import 'package:flutter/material.dart';
import '../../../injection.dart';
import '../../domain/repositories/user_repository.dart';

class UserProfilePage extends StatefulWidget {
  const UserProfilePage({super.key});

  @override
  State<UserProfilePage> createState() => _UserProfilePageState();
}

class _UserProfilePageState extends State<UserProfilePage> {
  late final UserRepository _userRepository;
  User? _user;

  @override
  void initState() {
    super.initState();
    _userRepository = getIt<UserRepository>();
    _loadUser();
  }

  Future<void> _loadUser() async {
    final user = await _userRepository.getUser('123');
    setState(() => _user = user);
  }

  @override
  Widget build(BuildContext context) {
    if (_user == null) return const CircularProgressIndicator();
    return Text(_user!.name);
  }
}
```

**在 ViewModel 中使用**（推荐方式）：

```dart
import 'package:injectable/injectable.dart';
import '../../domain/repositories/user_repository.dart';

@injectable
class UserViewModel {
  final UserRepository _userRepository;

  UserViewModel(this._userRepository);

  Future<User> loadUser(String id) => _userRepository.getUser(id);
}

// Widget 中使用 ViewModel
class UserProfilePage extends StatelessWidget {
  const UserProfilePage({super.key});

  @override
  Widget build(BuildContext context) {
    final viewModel = getIt<UserViewModel>();
    // ... 使用 viewModel
  }
}
```

---

### Step 6: 多环境配置

**适用场景**：dev 环境用 mock API，prod 环境用真实 API。

**AI 提示词**（可复制）：

```markdown
为项目配置多环境（dev、prod）依赖注入。

**要求**：

1. 定义环境常量：
   ```dart
   abstract class Environment {
     static const dev = 'dev';
     static const prod = 'prod';
   }
   ```

2. 为不同环境注册不同实现：
   - Dev 环境用 MockApiClient（`@Environment(Environment.dev)`）
   - Prod 环境用 ApiClient（`@Environment(Environment.prod)`）

3. 在 `configureDependencies()` 中传入环境参数：
   ```dart
   configureDependencies(environment: Environment.prod);
   ```

生成完整代码示例。
```

**输出示例**（参考）：

**lib/core/environment.dart**:

```dart
abstract class Environment {
  static const dev = 'dev';
  static const prod = 'prod';
}
```

**lib/core/api/mock_api_client.dart**:

```dart
import 'package:injectable/injectable.dart';
import 'api_client.dart';
import '../environment.dart';

@LazySingleton(as: ApiClient, env: [Environment.dev])
class MockApiClient implements ApiClient {
  @override
  Future<Response> get(String path) async {
    // 返回 mock 数据
    return Response(
      data: {'id': '123', 'name': 'Mock User'},
      statusCode: 200,
      requestOptions: RequestOptions(path: path),
    );
  }

  @override
  Future<Response> post(String path, dynamic data) async {
    return Response(
      data: {'success': true},
      statusCode: 200,
      requestOptions: RequestOptions(path: path),
    );
  }
}
```

**lib/core/api/api_client.dart**（真实实现）:

```dart
import 'package:dio/dio.dart';
import 'package:injectable/injectable.dart';
import '../environment.dart';

@LazySingleton(as: ApiClient, env: [Environment.prod])
class ApiClientImpl implements ApiClient {
  final Dio _dio;

  ApiClientImpl(this._dio) {
    _dio.options.baseUrl = 'https://api.example.com';
  }

  @override
  Future<Response> get(String path) => _dio.get(path);

  @override
  Future<Response> post(String path, dynamic data) => _dio.post(path, data: data);
}

// 抽象接口
abstract class ApiClient {
  Future<Response> get(String path);
  Future<Response> post(String path, dynamic data);
}
```

**lib/injection.dart**（更新）:

```dart
import 'package:get_it/get_it.dart';
import 'package:injectable/injectable.dart';

import 'injection.config.dart';

final getIt = GetIt.instance;

@InjectableInit(
  initializerName: 'init',
  preferRelativeImports: true,
  asExtension: true,
)
void configureDependencies({String environment = Environment.prod}) {
  getIt.init(environment: environment);
}
```

**lib/main.dart**（更新）:

```dart
import 'package:flutter/material.dart';
import 'injection.dart';
import 'core/environment.dart';

void main() {
  // 根据编译模式选择环境
  const env = String.fromEnvironment('ENV', defaultValue: Environment.prod);
  configureDependencies(environment: env);
  
  runApp(const MyApp());
}
```

**运行命令**：

```bash
# Dev 环境
flutter run --dart-define=ENV=dev

# Prod 环境
flutter run --dart-define=ENV=prod
```

---

### Step 7: 测试中使用 Mock

**人工操作**：在测试中需要替换真实服务为 mock。

**AI 提示词**（可复制）：

```markdown
为 Widget/ViewModel 测试配置 get_it mock 依赖。

**要求**：

1. 在每个测试的 `setUp()` 中：
   - 重置 GetIt：`await getIt.reset()`
   - 注册 mock 实现：`getIt.registerSingleton<UserRepository>(mockUserRepository)`

2. 在 `tearDown()` 中清理（可选）：
   - `await getIt.reset()`

3. 使用 mocktail 创建 mock 类

生成完整测试代码示例。
```

**输出示例**（参考）：

**test/features/user/presentation/user_profile_test.dart**:

```dart
import 'package:flutter_test/flutter_test.dart';
import 'package:mocktail/mocktail.dart';
import 'package:my_app/injection.dart';
import 'package:my_app/features/user/domain/repositories/user_repository.dart';
import 'package:my_app/features/user/domain/entities/user.dart';
import 'package:my_app/features/user/presentation/user_profile_page.dart';

class MockUserRepository extends Mock implements UserRepository {}

void main() {
  late MockUserRepository mockUserRepository;

  setUp(() async {
    // 重置 GetIt
    await getIt.reset();
    
    // 创建 mock
    mockUserRepository = MockUserRepository();
    
    // 注册 mock 到 GetIt
    getIt.registerSingleton<UserRepository>(mockUserRepository);
  });

  tearDown(() async {
    await getIt.reset();
  });

  group('UserProfilePage Tests', () {
    testWidgets('displays user name', (tester) async {
      // Arrange
      when(() => mockUserRepository.getUser('123'))
          .thenAnswer((_) async => User(id: '123', name: 'Alice'));

      // Act
      await tester.pumpWidget(
        const MaterialApp(home: UserProfilePage()),
      );
      await tester.pumpAndSettle();

      // Assert
      expect(find.text('Alice'), findsOneWidget);
      verify(() => mockUserRepository.getUser('123')).called(1);
    });
  });
}
```

**注意事项**：
- ⚠️ GetIt 不跨 isolate，单元测试每个文件独立运行没问题
- ⚠️ 集成测试（integration_test）中需在 `setUpAll()` 中初始化
- ⚠️ 测试中避免调用 `configureDependencies()`，手动注册 mock 即可

---

## ✅ 验证清单

完成后确认：

- [ ] `pubspec.yaml` 包含 get_it、injectable、injectable_generator、build_runner
- [ ] `lib/injection.dart` 正确配置 `@InjectableInit`
- [ ] `lib/injection.config.dart` 已生成且无报错
- [ ] `main.dart` 在 `runApp()` 前调用 `configureDependencies()`
- [ ] 服务类已添加 `@injectable`/`@lazySingleton`/`@singleton` 注解
- [ ] 接口绑定用 `@LazySingleton(as: Interface)` 正确配置
- [ ] 运行 `flutter run` 无报错，能成功获取依赖
- [ ] 测试中能正确 reset 并注册 mock

---

## 🤔 FAQ

### Q1: 何时用手写 get_it 注册，何时用 injectable？
**A**: 
- **小型项目（<10 个服务）**：手写 `getIt.registerSingleton()` 更直观
- **中大型项目**：用 injectable 自动生成，避免手动维护注册代码
- **可混用**：injectable 生成的代码调用 `getIt.init()` 后，可继续手动注册

### Q2: get_it 与 Riverpod 如何选择？
**A**: 
- **Riverpod**：响应式状态管理，UI 自动刷新，适合页面状态、表单、计数器
- **get_it**：服务定位器，适合无状态服务（Repository、API Client、本地存储）
- **推荐组合**：Riverpod Provider 内部用 `getIt<T>()` 获取 Repository

### Q3: @injectable、@lazySingleton、@singleton 有什么区别？
**A**: 
- **@injectable**：工厂模式，每次 `getIt<T>()` 创建新实例
- **@lazySingleton**：懒加载单例，首次使用时创建，全局唯一
- **@singleton**：饿汉单例，应用启动时立即创建

### Q4: 循环依赖如何解决？
**A**: 
- **问题**：A 依赖 B，B 依赖 A，injectable 报错
- **方案 1**：引入中间层（如 EventBus、Mediator）
- **方案 2**：用 `@factoryMethod` 延迟初始化
- **方案 3**：重新设计依赖关系，避免循环

### Q5: build_runner 报错 "Conflicting outputs"？
**A**: 
```bash
# 删除冲突文件后重新生成
dart run build_runner build --delete-conflicting-outputs
```

### Q6: 生成的代码找不到某个类？
**A**: 
- 检查该类是否添加了注解（`@injectable` 等）
- 检查 import 路径是否正确
- 尝试 `flutter clean && flutter pub get` 后重新生成

### Q7: 测试中修改了 mock，但 Widget 还是用旧的？
**A**: 
- 确认在 `setUp()` 中调用了 `await getIt.reset()`
- 确认在 `pumpWidget()` **之前** 注册了 mock
- 检查 Widget 是否缓存了依赖实例（应在 `initState()` 或 `build()` 中获取）

---

## 💡 最佳实践

### 1. 依赖层次分明

```
lib/
├── core/              # 核心服务（全局单例）
│   ├── api/           # ApiClient
│   ├── storage/       # LocalStorage
│   └── logger/        # Logger
├── features/          # 业务功能
│   └── user/
│       ├── domain/    # 接口定义（UserRepository）
│       ├── data/      # 实现（UserRepositoryImpl）
│       └── presentation/  # ViewModel（注入 Repository）
└── injection.dart     # DI 配置入口
```

### 2. 接口与实现分离

```dart
// domain/repositories/user_repository.dart (接口)
abstract class UserRepository {
  Future<User> getUser(String id);
}

// data/repositories/user_repository_impl.dart (实现)
@LazySingleton(as: UserRepository)
class UserRepositoryImpl implements UserRepository {
  // ...
}
```

### 3. 避免在 Widget 构造函数中注入

❌ **错误**：

```dart
class UserPage extends StatelessWidget {
  final UserRepository repository;
  const UserPage({required this.repository}); // 违背 Flutter 设计
}
```

✅ **正确**：

```dart
class UserPage extends StatelessWidget {
  const UserPage({super.key});

  @override
  Widget build(BuildContext context) {
    final repository = getIt<UserRepository>(); // 在 build 中获取
    // ...
  }
}
```

### 4. 测试优先设计

编写服务类时就考虑可测试性：
- 依赖通过构造函数注入
- 避免在构造函数中执行副作用
- 接口与实现分离

### 5. 环境配置外部化

```dart
// ❌ 硬编码
@LazySingleton(as: ApiClient)
class ApiClientImpl {
  ApiClientImpl() {
    _dio.options.baseUrl = 'https://api.example.com'; // 写死
  }
}

// ✅ 用环境变量或配置类
@module
abstract class ConfigModule {
  @lazySingleton
  AppConfig get config => AppConfig(
    apiBaseUrl: const String.fromEnvironment('API_URL', defaultValue: 'https://api.example.com'),
  );
}

@LazySingleton(as: ApiClient)
class ApiClientImpl {
  ApiClientImpl(AppConfig config) {
    _dio.options.baseUrl = config.apiBaseUrl;
  }
}
```

### 6. 记录注册日志（调试用）

```dart
void configureDependencies({String environment = Environment.prod}) {
  getIt.init(environment: environment);
  
  // 调试时打印已注册的服务
  if (kDebugMode) {
    print('📦 Registered services:');
    getIt.allFactories().forEach((factory) {
      print('  - ${factory.runtimeType}');
    });
  }
}
```

---

## 🔗 相关资源

- 提示词：[prompts/di/setup-get-it-injectable.md](../prompts/di/setup-get-it-injectable.md)
- 提示词：[prompts/di/register-or-fix-injectable.md](../prompts/di/register-or-fix-injectable.md)
- 上下文工程：[docs/context-engineering.md](../docs/context-engineering.md)
- get_it 官方文档：[https://pub.dev/packages/get_it](https://pub.dev/packages/get_it)
- injectable 官方文档：[https://pub.dev/packages/injectable](https://pub.dev/packages/injectable)
- Flutter 架构指南：[https://docs.flutter.dev/data-and-backend/state-mgmt/options](https://docs.flutter.dev/data-and-backend/state-mgmt/options)

---

**让服务层更清晰，测试更简单 🚀**
