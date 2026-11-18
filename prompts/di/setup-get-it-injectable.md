# 提示词：搭建 get_it + injectable 依赖注入

**用途**：为现有 Flutter 项目从零搭建 get_it + injectable 依赖注入架构

---

## 📋 适用场景

- 新项目需要建立服务层架构
- 现有项目服务层耦合严重，需解耦
- 需要接口与实现分离（依赖倒置原则）
- 需要多环境配置（dev/test/prod）

---

## 📥 输入要求

在 Cursor 中准备：

1. **项目根目录**（包含 `pubspec.yaml`）
2. **现有服务类列表**（如 ApiClient、Repository、LocalStorage）
3. **依赖关系说明**（哪些服务依赖其他服务）
4. **（可选）接口定义**（如果已有抽象类/接口）

---

## 💬 提示词模板

```markdown
我需要为 Flutter 项目搭建 get_it + injectable 依赖注入架构。

**项目信息**：
- 项目路径：[项目根目录路径]
- 需要注入的服务：[列出服务类名，如 ApiClient, UserRepository, LocalStorage]

**要求**：

### 1. 添加依赖

在 `pubspec.yaml` 中添加：

```yaml
dependencies:
  get_it: ^8.0.0
  injectable: ^3.0.0

dev_dependencies:
  injectable_generator: ^3.0.0
  build_runner: ^2.4.0
```

### 2. 创建初始化文件

在 `lib/injection.dart` 创建：

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

### 3. 修改 main.dart

在 `lib/main.dart` 的 `main()` 函数中，**在 runApp() 之前**调用：

```dart
void main() {
  configureDependencies();
  runApp(const MyApp());
}
```

### 4. 为服务添加注解

为以下服务类添加 injectable 注解：

**[服务名 1]**（如 ApiClient）：
- 类型：[单例/工厂/懒加载]
- 依赖：[列出依赖的其他服务]
- 是否有接口：[是/否]

**[服务名 2]**（如 UserRepository）：
- 类型：[单例/工厂/懒加载]
- 依赖：[列出依赖的其他服务]
- 是否有接口：[是/否，如果有，列出接口名]

**注解规则**：
- 全局单例用 `@lazySingleton`（推荐）或 `@singleton`
- 每次创建新实例用 `@injectable`
- 有接口的用 `@LazySingleton(as: 接口类)`
- 第三方库（如 Dio）用 `@module` + `@lazySingleton` 注册

**示例**：

```dart
// 无接口的单例
@lazySingleton
class ApiClient {
  final Dio _dio;
  ApiClient(this._dio);
}

// 有接口的单例
abstract class UserRepository {
  Future<User> getUser(String id);
}

@LazySingleton(as: UserRepository)
class UserRepositoryImpl implements UserRepository {
  final ApiClient _apiClient;
  UserRepositoryImpl(this._apiClient);
  
  @override
  Future<User> getUser(String id) async {
    // ...
  }
}

// 第三方库注册
@module
abstract class ThirdPartyModule {
  @lazySingleton
  Dio get dio => Dio();
}
```

### 5. 运行代码生成

执行命令：

```bash
dart run build_runner build --delete-conflicting-outputs
```

### 6. 验证

生成完整代码后，确认：

1. `lib/injection.config.dart` 已生成
2. 生成的文件中包含所有服务的注册代码
3. 无报错信息
4. 运行 `flutter run` 无异常

**输出要求**：

- 生成完整的服务类代码（包含所有必要的 import）
- 修改后的 `main.dart`
- 如需创建接口，也一并生成
- 提供完整的 build_runner 执行命令
```

---

## ✅ 验证步骤

生成后执行：

```bash
# 1. 安装依赖
flutter pub get

# 2. 运行代码生成
dart run build_runner build --delete-conflicting-outputs

# 3. 检查生成文件
cat lib/injection.config.dart

# 4. 运行应用验证
flutter run
```

**检查清单**：
- [ ] `pubspec.yaml` 包含所有依赖
- [ ] `lib/injection.dart` 正确配置
- [ ] `lib/injection.config.dart` 已生成且无语法错误
- [ ] `main.dart` 在 `runApp()` 前调用 `configureDependencies()`
- [ ] 所有服务类已添加注解
- [ ] 应用能正常启动，无 DI 相关报错

---

## 🛠️ 常见调整

生成后可能需要手动微调：

### 1. 接口绑定写法

AI 可能生成旧版写法，请统一为：

```dart
// ✅ 正确（injectable 3.x）
@LazySingleton(as: UserRepository)
class UserRepositoryImpl implements UserRepository {}

// ❌ 避免（旧版写法）
@Injectable(as: UserRepository)
@lazySingleton
class UserRepositoryImpl implements UserRepository {}
```

### 2. 第三方库注册

对于 Dio、SharedPreferences 等第三方库：

```dart
@module
abstract class CoreModule {
  @lazySingleton
  Dio get dio => Dio();

  @preResolve // 异步初始化
  Future<SharedPreferences> get prefs => SharedPreferences.getInstance();
}
```

使用 `@preResolve` 时，`configureDependencies()` 需改为异步：

```dart
@InjectableInit()
Future<void> configureDependencies() => getIt.init();

// main.dart
void main() async {
  WidgetsFlutterBinding.ensureInitialized();
  await configureDependencies();
  runApp(const MyApp());
}
```

### 3. 环境配置

如需多环境支持：

```dart
// lib/core/environment.dart
abstract class Environment {
  static const dev = 'dev';
  static const prod = 'prod';
}

// lib/injection.dart
void configureDependencies({String environment = Environment.prod}) {
  getIt.init(environment: environment);
}

// 服务类
@LazySingleton(as: ApiClient, env: [Environment.prod])
class ProdApiClient implements ApiClient {}

@LazySingleton(as: ApiClient, env: [Environment.dev])
class MockApiClient implements ApiClient {}
```

### 4. 循环依赖检查

如果 build_runner 报循环依赖错误：

```
Error: Circular dependency detected: A -> B -> A
```

解决方案：
- 引入中间层（如 EventBus）
- 用 `@factoryMethod` 延迟初始化
- 重新设计依赖关系

---

## 🔗 相关

- 工作流：[workflows/get-it-injectable-di.md](../../workflows/get-it-injectable-di.md)
- 注册和修复：[register-or-fix-injectable.md](register-or-fix-injectable.md)
- get_it 文档：[https://pub.dev/packages/get_it](https://pub.dev/packages/get_it)
- injectable 文档：[https://pub.dev/packages/injectable](https://pub.dev/packages/injectable)
