# Dio 网络层与统一错误处理工作流

**用 AI 搭建可维护的 Dio 网络层，统一映射 HTTP 错误为域错误**

> 作者：Loong (SSLoong) | 日期：2026-11-15

---

## 📋 背景

手动配置 Dio 客户端容易遗漏超时、重试、鉴权逻辑，敬落的 try/catch 让错误处理不一致。AI 能快速生成规范的网络层骨架，但**必须理解 http 包 vs Dio 差异、技术层异常 vs 业务域错误映射，并本地验证通过**。

---

## 🎯 目标

- ✅ 搭建单例 Dio 客户端（baseUrl、超时、默认 headers）
- ✅ 实现 Interceptor：鉴权、日志、token 刷新
- ✅ DioException 统一映射为域错误（超时、无网、4xx/5xx、解析失败）
- ✅ 支持 CancelToken、重试/退避机制
- ✅ 用 http_mock_adapter 或自定义 Adapter 编写测试，不打真实网络

---

## 🔧 适用场景

| 场景 | 适合度 | 说明 |
|------|--------|------|
| 新项目网络层搭建 | ⭐⭐⭐⭐⭐ | 从 0 到 1 规范化 |
| 重构敬落的 try/catch | ⭐⭐⭐⭐⭐ | 收敛到统一错误处理 |
| 迁移 http 包到 Dio | ⭐⭐⭐⭐ | 需理解差异 |
| 复杂认证流程 | ⭐⭐⭐ | token 刷新需仔细处理队列 |

---

## 🚀 完整流程

### Step 1: 准备上下文

**人工操作**：在 Cursor 中打开或创建 `lib/core/network/` 目录，准备以下文件结构：

```
lib/core/network/
├── dio_client.dart          # 单例 Dio 配置
├── api_interceptor.dart     # 鉴权、日志、刷新 Interceptor
├── api_error.dart           # 域错误定义
└── api_error_mapper.dart    # DioException → ApiError 映射
```

**AI 提示词**（可复制）：

```markdown
我需要为 Flutter 项目搭建 Dio 网络层。请先确认：

1. API base URL 是什么？（如 https://api.example.com）
2. 需要哪些默认 headers？（如 Content-Type、Accept）
3. 认证方式是什么？（Bearer Token / API Key / 无需认证）
4. 是否需要 token 刷新机制？
5. 超时时间要求？（connect / receive / send timeout）

列出需要覆盖的错误场景（超时、无网、4xx、5xx、解析失败），再生成代码。
```

**检查点**：AI 列出的配置是否符合你的后端 API 规范？遗漏就明确补充。

---

### Step 2: 生成 Dio 客户端单例

**AI 提示词**（可复制）：

```markdown
生成 `lib/core/network/dio_client.dart`，要求：

1. **单例模式**：提供 `DioClient.instance.dio` 全局访问
2. **BaseOptions 配置**：
   - baseUrl: [YOUR_BASE_URL]
   - connectTimeout: 15 秒
   - receiveTimeout: 30 秒
   - headers: {'Content-Type': 'application/json', 'Accept': 'application/json'}
3. **拦截器**：预留 `addInterceptors()` 方法
4. **完整 import**：确保使用 Dio 5.x API（如 `Duration` 不是 `int`）

生成完整可运行代码，不要省略任何部分。
```

**输出示例**（参考，实际以 AI 生成为准）：

```dart
import 'package:dio/dio.dart';

/// Dio 客户端单例，统一管理网络请求配置
class DioClient {
  static final DioClient _instance = DioClient._internal();
  late final Dio dio;

  factory DioClient() => _instance;
  static DioClient get instance => _instance;

  DioClient._internal() {
    dio = Dio(
      BaseOptions(
        baseUrl: 'https://api.example.com',
        connectTimeout: const Duration(seconds: 15),
        receiveTimeout: const Duration(seconds: 30),
        headers: {
          'Content-Type': 'application/json',
          'Accept': 'application/json',
        },
      ),
    );
  }

  /// 添加拦截器（在 main.dart 或初始化时调用）
  void addInterceptors(List<Interceptor> interceptors) {
    dio.interceptors.addAll(interceptors);
  }
}
```

**检查点**：
- [ ] `connectTimeout` / `receiveTimeout` 使用 `Duration`（Dio 5.x 要求）
- [ ] `baseUrl` 正确无误
- [ ] 单例模式正确实现

---

### Step 3: 实现 API Interceptor

**AI 提示词**（可复制）：

```markdown
生成 `lib/core/network/api_interceptor.dart`，包含三个拦截器：

### 1. AuthInterceptor（鉴权）
- 从 SecureStorage 或 SharedPreferences 读取 token
- 在请求头添加 `Authorization: Bearer $token`
- 跳过不需要鉴权的路径（如 /auth/login）

### 2. LoggingInterceptor（日志）
- 打印请求 URL、method、headers（**禁止打印 Authorization / 敏感头**）
- 打印响应状态码、耗时
- 仅在 debug 模式启用

### 3. RefreshTokenInterceptor（token 刷新）
- 监听 401 错误
- 调用 /auth/refresh 接口刷新 token
- **避免并发刷新**：用队列或 lock 保证只刷新一次
- 刷新成功后重试原请求
- 刷新失败则清除 token 并跳转登录

要求：
- 使用 Dio 5.x API（如 `RequestInterceptorHandler`）
- 完整 import，可直接运行
- 注释清晰标注关键逻辑
```

**输出示例**（参考）：

```dart
import 'package:dio/dio.dart';
import 'package:flutter/foundation.dart';

/// 鉴权拦截器：为请求添加 Bearer Token
class AuthInterceptor extends Interceptor {
  final Future<String?> Function() getToken;

  AuthInterceptor({required this.getToken});

  @override
  void onRequest(
    RequestOptions options,
    RequestInterceptorHandler handler,
  ) async {
    // 跳过登录接口
    if (options.path.contains('/auth/login')) {
      return handler.next(options);
    }

    final token = await getToken();
    if (token != null) {
      options.headers['Authorization'] = 'Bearer $token';
    }

    handler.next(options);
  }
}

/// 日志拦截器：打印请求/响应（生产环境禁用）
class LoggingInterceptor extends Interceptor {
  @override
  void onRequest(RequestOptions options, RequestInterceptorHandler handler) {
    if (kDebugMode) {
      print('🌐 [${options.method}] ${options.uri}');
      // 不打印 Authorization 等敏感头
      final safeHeaders = Map<String, dynamic>.from(options.headers)
        ..remove('Authorization');
      print('📤 Headers: $safeHeaders');
    }
    handler.next(options);
  }

  @override
  void onResponse(Response response, ResponseInterceptorHandler handler) {
    if (kDebugMode) {
      print('✅ [${response.statusCode}] ${response.requestOptions.uri}');
    }
    handler.next(response);
  }

  @override
  void onError(DioException err, ErrorInterceptorHandler handler) {
    if (kDebugMode) {
      print('❌ [${err.response?.statusCode}] ${err.requestOptions.uri}');
      print('   Error: ${err.message}');
    }
    handler.next(err);
  }
}

/// Token 刷新拦截器：处理 401 并自动刷新
class RefreshTokenInterceptor extends Interceptor {
  final Dio dio;
  final Future<String?> Function() refreshToken;
  final void Function() onRefreshFailed;

  bool _isRefreshing = false;
  final List<Function> _pendingRequests = [];

  RefreshTokenInterceptor({
    required this.dio,
    required this.refreshToken,
    required this.onRefreshFailed,
  });

  @override
  void onError(DioException err, ErrorInterceptorHandler handler) async {
    if (err.response?.statusCode == 401) {
      // 避免并发刷新
      if (_isRefreshing) {
        // 加入队列等待
        _pendingRequests.add(() => _retry(err.requestOptions, handler));
        return;
      }

      _isRefreshing = true;

      try {
        final newToken = await refreshToken();
        if (newToken == null) {
          throw Exception('Refresh failed');
        }

        // 刷新成功，重试所有挂起的请求
        for (var request in _pendingRequests) {
          request();
        }
        _pendingRequests.clear();

        // 重试当前请求
        return _retry(err.requestOptions, handler);
      } catch (e) {
        // 刷新失败，清除 token 并导航到登录
        onRefreshFailed();
        return handler.next(err);
      } finally {
        _isRefreshing = false;
      }
    }

    handler.next(err);
  }

  Future<void> _retry(
    RequestOptions requestOptions,
    ErrorInterceptorHandler handler,
  ) async {
    try {
      final response = await dio.fetch(requestOptions);
      handler.resolve(response);
    } on DioException catch (e) {
      handler.next(e);
    }
  }
}
```

**检查点**：
- [ ] 不打印 Authorization header
- [ ] 刷新 token 时有并发保护
- [ ] 使用 `RequestInterceptorHandler` / `ErrorInterceptorHandler`（Dio 5.x）

---

### Step 4: 定义域错误与映射器

**AI 提示词**（可复制）：

```markdown
生成两个文件：

### 1. `lib/core/network/api_error.dart`（域错误定义）

定义 `ApiError` sealed class（或 enum + data class），包含：
- NetworkError（无网络连接）
- TimeoutError（请求超时）
- ServerError（5xx 错误）
- ClientError（4xx 错误，含 statusCode 和 message）
- UnauthorizedError（401，需重新登录）
- ParseError（JSON 解析失败）
- UnknownError（其他未知错误）

### 2. `lib/core/network/api_error_mapper.dart`（映射器）

提供静态方法 `ApiError mapDioException(DioException e)`，将 DioException 映射为上述域错误：
- `DioExceptionType.connectionTimeout` / `receiveTimeout` → TimeoutError
- `DioExceptionType.connectionError` → NetworkError
- `response.statusCode == 401` → UnauthorizedError
- `response.statusCode >= 400 && < 500` → ClientError
- `response.statusCode >= 500` → ServerError
- JSON 解析异常 → ParseError

要求：
- 使用 Dio 5.x API（`DioExceptionType` 枚举）
- 完整 import，可直接使用
```

**输出示例**（参考）：

```dart
// api_error.dart
sealed class ApiError {
  const ApiError();
}

class NetworkError extends ApiError {
  const NetworkError();
  @override
  String toString() => '网络连接失败，请检查网络设置';
}

class TimeoutError extends ApiError {
  const TimeoutError();
  @override
  String toString() => '请求超时，请稍后重试';
}

class ServerError extends ApiError {
  final int statusCode;
  final String message;

  const ServerError(this.statusCode, this.message);

  @override
  String toString() => '服务器错误 ($statusCode): $message';
}

class ClientError extends ApiError {
  final int statusCode;
  final String message;

  const ClientError(this.statusCode, this.message);

  @override
  String toString() => '请求错误 ($statusCode): $message';
}

class UnauthorizedError extends ApiError {
  const UnauthorizedError();
  @override
  String toString() => '登录已过期，请重新登录';
}

class ParseError extends ApiError {
  final String details;

  const ParseError(this.details);

  @override
  String toString() => '数据解析失败: $details';
}

class UnknownError extends ApiError {
  final String details;

  const UnknownError(this.details);

  @override
  String toString() => '未知错误: $details';
}
```

```dart
// api_error_mapper.dart
import 'package:dio/dio.dart';
import 'api_error.dart';

class ApiErrorMapper {
  static ApiError mapDioException(DioException e) {
    switch (e.type) {
      case DioExceptionType.connectionTimeout:
      case DioExceptionType.sendTimeout:
      case DioExceptionType.receiveTimeout:
        return const TimeoutError();

      case DioExceptionType.connectionError:
        return const NetworkError();

      case DioExceptionType.badResponse:
        final statusCode = e.response?.statusCode;
        final message = e.response?.data?['message'] ?? e.message ?? '未知错误';

        if (statusCode == 401) {
          return const UnauthorizedError();
        } else if (statusCode != null && statusCode >= 400 && statusCode < 500) {
          return ClientError(statusCode, message);
        } else if (statusCode != null && statusCode >= 500) {
          return ServerError(statusCode, message);
        }
        return UnknownError(message);

      case DioExceptionType.cancel:
        return const UnknownError('请求已取消');

      case DioExceptionType.badCertificate:
        return const UnknownError('证书验证失败');

      case DioExceptionType.unknown:
      default:
        if (e.error is FormatException) {
          return ParseError(e.error.toString());
        }
        return UnknownError(e.message ?? '未知错误');
    }
  }
}
```

**检查点**：
- [ ] 使用 `DioExceptionType` 枚举（Dio 5.x）
- [ ] 覆盖所有主要错误类型
- [ ] 域错误有用户友好的 `toString()`

---

### Step 5: 集成并测试

**人工操作**：在 `main.dart` 或初始化逻辑中配置拦截器：

```dart
import 'package:flutter_secure_storage/flutter_secure_storage.dart';

void setupDio() {
  final storage = FlutterSecureStorage();

  DioClient.instance.addInterceptors([
    AuthInterceptor(
      getToken: () async => await storage.read(key: 'access_token'),
    ),
    LoggingInterceptor(),
    RefreshTokenInterceptor(
      dio: DioClient.instance.dio,
      refreshToken: () async {
        // 调用刷新接口
        final response = await Dio().post('/auth/refresh');
        final newToken = response.data['access_token'];
        await storage.write(key: 'access_token', value: newToken);
        return newToken;
      },
      onRefreshFailed: () {
        // 跳转登录页
        // navigatorKey.currentState?.pushNamedAndRemoveUntil('/login', (_) => false);
      },
    ),
  ]);
}
```

**AI 提示词（生成测试）**：

```markdown
为网络层生成测试文件 `test/core/network/dio_client_test.dart`，要求：

1. 使用 `http_mock_adapter` mock Dio 请求（不打真实网络）
2. 测试场景：
   - ✅ 成功请求返回数据
   - ✅ 401 触发 token 刷新并重试
   - ✅ 超时映射为 TimeoutError
   - ✅ 无网映射为 NetworkError
   - ✅ 5xx 映射为 ServerError
3. 使用 `mocktail` mock token 获取/刷新逻辑
4. 验证 ApiErrorMapper 正确映射

完整 import，可运行 `flutter test` 或 `dart test`。
```

**验证步骤**：

```bash
# 运行测试
flutter test test/core/network/

# 或单独测试
dart test test/core/network/dio_client_test.dart
```

**检查清单**：
- [ ] 所有测试通过（0 failed）
- [ ] Mock 配置正确，无真实网络请求
- [ ] 错误映射符合预期

---

### Step 6: 可选功能 — 重试与退避

**适合场景**：临时网络抖动、服务端偶发错误（仅对幂等请求如 GET、PUT，POST 请勿盲目重试）。

**AI 提示词**：

```markdown
为 Dio 添加重试机制，要求：

1. 创建 `RetryInterceptor`：
   - 最多重试 3 次
   - 指数退避：第 1 次 1s，第 2 次 2s，第 3 次 4s
   - **仅重试 GET / PUT**（幂等请求）
   - 超时、网络错误可重试，4xx 不重试
2. 使用 `dio_smart_retry` 包或手动实现
3. 补充测试：验证重试次数和退避时间

完整代码 + 测试。
```

**检查点**：
- [ ] POST 请求不被重试
- [ ] 重试间隔符合指数退避
- [ ] 最终失败后正确抛出错误

---

### Step 7: 可选功能 — CancelToken

**适合场景**：用户快速切换页面、搜索防抖。

**AI 提示词**：

```markdown
示例如何在 Repository 中使用 CancelToken 取消请求：

```dart
class UserRepository {
  CancelToken? _cancelToken;

  Future<User> fetchUser(String id) async {
    // 取消上一次请求
    _cancelToken?.cancel('新请求已发起');
    _cancelToken = CancelToken();

    try {
      final response = await DioClient.instance.dio.get(
        '/users/$id',
        cancelToken: _cancelToken,
      );
      return User.fromJson(response.data);
    } on DioException catch (e) {
      if (e.type == DioExceptionType.cancel) {
        // 忽略取消错误
        throw const UnknownError('请求已取消');
      }
      throw ApiErrorMapper.mapDioException(e);
    }
  }

  void dispose() {
    _cancelToken?.cancel();
  }
}
```

补充测试验证取消逻辑。
```

---

## ✅ 验证清单

完成后确认：

- [ ] Dio 客户端使用 5.x API（Duration 不是 int）
- [ ] 拦截器不打印敏感信息（Authorization / token）
- [ ] RefreshTokenInterceptor 有并发保护
- [ ] ApiErrorMapper 覆盖所有 DioExceptionType
- [ ] 测试通过且不打真实网络
- [ ] POST 请求未被无脑重试
- [ ] 域错误有用户友好提示

---

## 🤔 FAQ

### Q1: http 包 vs Dio 的区别？
**A**:
- `http`：Flutter 官方基础 HTTP 库，轻量但需手动处理 JSON、重试、拦截
- `Dio`：社区流行库，内置拦截器、FormData、取消、进度监听，适合复杂场景

### Q2: 为什么不能直接在 UI 展示 DioException？
**A**:
- UI 层不应依赖网络库细节（明天换 http 包怎么办？）
- 域错误（ApiError）是业务语言，DioException 是技术实现
- 映射后可统一处理：UnauthorizedError → 跳登录，NetworkError → 提示检查网络

### Q3: RefreshTokenInterceptor 为什么需要队列？
**A**:
- 多个请求同时 401 时，若不加锁，会并发调用刷新接口
- 用 `_isRefreshing` flag + `_pendingRequests` 队列保证只刷新一次
- 刷新成功后批量重试队列中的请求

### Q4: POST 请求能重试吗？
**A**:
- **不建议**：POST 通常非幂等（如创建订单），重试可能导致重复提交
- **例外**：后端保证幂等性（如带幂等 key），可重试
- 安全做法：仅对 GET / PUT / DELETE 重试

### Q5: 如何在 CI 上跑网络测试？
**A**:
- **禁止真实网络**：用 `http_mock_adapter` 或自定义 `HttpClientAdapter`
- Mock 示例：
  ```dart
  final dioAdapter = DioAdapter(dio: dio);
  dioAdapter.onGet('/users/123', (server) {
    server.reply(200, {'id': '123', 'name': 'Alice'});
  });
  ```

### Q6: AI 生成的 Interceptor 不工作？
**A**:
- 检查是否调用 `dio.interceptors.add()`
- 确认拦截器顺序：Auth → Logging → Refresh
- 验证 `handler.next()` / `handler.resolve()` / `handler.reject()` 正确调用

---

## 💡 最佳实践

1. **上下文越详细，生成质量越高**
   - 附上现有 API 接口文档、错误码规范、鉴权流程

2. **分层清晰**
   - `DioClient`：纯配置，不含业务逻辑
   - `Interceptor`：横切关注点（鉴权、日志、刷新）
   - `ApiErrorMapper`：技术错误 → 业务错误
   - `Repository`：调用 Dio，捕获异常并映射

3. **测试先行**
   - 先写失败场景测试（401、超时、无网），再让 AI 实现

4. **记录常见错误**
   - 把修复过的 AI 错误加入项目 `.cursor/rules` 或 `AGENTS.md`

5. **禁止敏感信息泄露**
   - 日志拦截器明确排除 Authorization / Cookie / X-API-Key
   - 生产环境关闭 LoggingInterceptor

6. **刷新 token 需幂等**
   - 用 lock / flag 避免并发刷新
   - 刷新失败后清除 token 并导航登录，不要无限重试

---

## 🔗 相关资源

- 提示词模板：[prompts/networking/generate-dio-client.md](../prompts/networking/generate-dio-client.md)
- 统一错误处理：[prompts/networking/unify-api-error-handling.md](../prompts/networking/unify-api-error-handling.md)
- 上下文工程：[docs/context-engineering.md](../docs/context-engineering.md)
- Dio 官方文档：[https://pub.dev/packages/dio](https://pub.dev/packages/dio)
- http_mock_adapter：[https://pub.dev/packages/http_mock_adapter](https://pub.dev/packages/http_mock_adapter)

---

**高效搭建网络层 🌐**
