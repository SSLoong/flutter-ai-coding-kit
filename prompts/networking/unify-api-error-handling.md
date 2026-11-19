# 提示词：统一 API 错误处理

**用途**：把敬落的 try/catch 或原始 DioException 收敛为统一域错误 + Interceptor 中央映射

---

## 📋 适用场景

- 代码中到处是原始 `catch (DioException e)`，错误处理不一致
- UI 直接依赖 `DioException`，违反分层原则
- 需要统一映射 HTTP 错误为用户友好提示

---

## 📥 输入要求

在 Cursor 中准备：

1. **现有网络请求代码**（Repository 或 Service 类）
2. **错误处理现状**（是否有域错误定义，或全是原始异常）
3. **（可选）现有 ApiError 定义**

---

## 💬 提示词模板

```markdown
我需要重构现有网络层的错误处理，实现统一映射。

**当前问题**：

1. 代码中有多处 `catch (DioException e)`，错误处理逻辑分散
2. UI 层直接依赖 `DioException`，违反分层
3. 缺少用户友好的错误提示

**重构目标**：

1. 定义域错误类型（`ApiError`），包含：
   - NetworkError（无网络）
   - TimeoutError（超时）
   - UnauthorizedError（401）
   - ClientError（4xx）
   - ServerError（5xx）
   - ParseError（JSON 解析失败）
   - UnknownError（其他）

2. 创建 `ApiErrorMapper`，统一将 `DioException` 映射为 `ApiError`

3. 重构现有代码：
   - Repository 层捕获 `DioException`，映射为 `ApiError` 后抛出
   - UI 层处理 `ApiError`，展示用户友好提示

**现有代码示例**（需重构）：

[粘贴你的 Repository 或 Service 代码]

**要求**：

1. **保持接口签名不变**（避免大面积改动）
2. **生成三个文件**：
   - `lib/core/network/api_error.dart`（域错误定义）
   - `lib/core/network/api_error_mapper.dart`（映射器）
   - 重构后的 Repository 代码
3. **补充测试**：验证映射逻辑正确
4. **使用 Dio 5.x API**（`DioExceptionType` 枚举）

生成完整代码 + 测试，不要省略。
```

---

## ✅ 验证步骤

生成后执行：

```bash
# 检查语法
flutter analyze lib/

# 运行测试
flutter test test/core/network/api_error_mapper_test.dart

# 运行重构后的 Repository 测试
flutter test test/data/repositories/
```

**检查清单**：
- [ ] 所有 `DioException` 已映射为 `ApiError`
- [ ] UI 层不再依赖 Dio 包
- [ ] 错误提示用户友好（中文 / 可本地化）
- [ ] 测试覆盖所有错误分支

---

## 🛠️ 常见调整

### 1. 补充业务错误码

后端返回业务错误码时，需扩展 `ClientError`：

```dart
class ClientError extends ApiError {
  final int statusCode;
  final String message;
  final String? errorCode; // 业务错误码

  const ClientError(this.statusCode, this.message, {this.errorCode});

  @override
  String toString() {
    if (errorCode != null) {
      return '[$errorCode] $message';
    }
    return '请求错误 ($statusCode): $message';
  }
}
```

映射时解析 `errorCode`：

```dart
case DioExceptionType.badResponse:
  final statusCode = e.response?.statusCode;
  final data = e.response?.data;
  final message = data?['message'] ?? '未知错误';
  final errorCode = data?['error_code']; // 解析业务码

  if (statusCode != null && statusCode >= 400 && statusCode < 500) {
    return ClientError(statusCode, message, errorCode: errorCode);
  }
  // ...
```

### 2. 本地化错误提示

在 `ApiError` 中添加本地化 key：

```dart
class NetworkError extends ApiError {
  const NetworkError();

  String localize(BuildContext context) {
    return AppLocalizations.of(context)!.errorNetwork;
  }
}
```

或用 `easy_localization` / `intl` 包。

### 3. Interceptor 统一处理

如果不想在每个 Repository 写 try/catch，可在 Interceptor 统一映射：

```dart
class ErrorMappingInterceptor extends Interceptor {
  @override
  void onError(DioException err, ErrorInterceptorHandler handler) {
    final apiError = ApiErrorMapper.mapDioException(err);
    // 包装成自定义 Exception
    handler.reject(
      DioException(
        requestOptions: err.requestOptions,
        error: apiError, // ApiError 作为 error 字段
        type: err.type,
      ),
    );
  }
}
```

Repository 中：

```dart
try {
  final response = await dio.get('/users');
  return User.fromJson(response.data);
} on DioException catch (e) {
  if (e.error is ApiError) {
    throw e.error as ApiError; // 直接抛域错误
  }
  throw ApiErrorMapper.mapDioException(e);
}
```

### 4. 测试 mock

用 `http_mock_adapter` mock 错误响应：

```dart
test('maps 401 to UnauthorizedError', () async {
  final dioAdapter = DioAdapter(dio: dio);
  dioAdapter.onGet(
    '/users/123',
    (server) => server.reply(401, {'message': 'Token expired'}),
  );

  try {
    await userRepository.getUser('123');
    fail('Should throw UnauthorizedError');
  } catch (e) {
    expect(e, isA<UnauthorizedError>());
  }
});
```

---

## 💡 重构步骤

1. **先定义域错误**（`ApiError` sealed class）
2. **创建映射器**（`ApiErrorMapper`）
3. **编写映射测试**（覆盖所有 `DioExceptionType`）
4. **重构一个 Repository**，验证可行性
5. **批量重构其他 Repository**
6. **UI 层改用 `ApiError` 展示提示**

---

## 🔗 相关

- 工作流：[workflows/dio-network-layer.md](../../workflows/dio-network-layer.md)
- 生成 API 客户端：[generate-dio-client.md](generate-dio-client.md)
