# 提示词：生成 Dio API 客户端

**用途**：根据已有 API 说明生成类型安全的 Dio API client / service

---

## 📋 适用场景

- 新建项目需搭建网络层
- 已有 API 文档需快速生成客户端代码
- 重构敬落的网络请求为统一 service

---

## 📥 输入要求

在 Cursor 中准备：

1. **API 文档或接口说明**（路径、方法、请求/响应模型）
2. **认证方式**（Bearer Token / API Key / 无需认证）
3. **（可选）已有数据模型**（如 `User`、`Product` 类）
4. **（可选）现有 Dio 配置**（如 `DioClient` 单例）

---

## 💬 提示词模板

```markdown
我需要为以下 API 生成 Dio 客户端代码。

**API 说明**：

- Base URL: [YOUR_BASE_URL，如 https://api.example.com]
- 认证方式: [Bearer Token / API Key / 无需认证]

**接口列表**：

| 方法 | 路径 | 请求参数 | 响应 |
|------|------|----------|------|
| GET | /users/{id} | id: String | User 对象 |
| POST | /users | User 对象 | User 对象 |
| PUT | /users/{id} | id: String, User 对象 | User 对象 |
| DELETE | /users/{id} | id: String | void |

**要求**：

1. **文件结构**：
   - `lib/data/api/user_api_service.dart`（API 客户端类）
   - 使用现有 `DioClient.instance.dio`（如已存在）

2. **代码规范**：
   - 每个接口对应一个方法（如 `Future<User> getUser(String id)`）
   - 使用 Dio 5.x API（Duration 不是 int）
   - 异常处理：捕获 `DioException`，映射为 `ApiError`（如已有映射器）
   - 请求参数用命名参数或 Map，响应用类型化模型

3. **完整性**：
   - 包含所有必要的 import
   - 可直接运行，无需手动补充

生成完整代码，不要省略任何部分。
```

---

## ✅ 验证步骤

生成后执行：

```bash
# 检查语法
flutter analyze lib/data/api/

# 运行测试（如已生成）
flutter test test/data/api/user_api_service_test.dart
```

**检查清单**：
- [ ] 所有接口方法已生成
- [ ] 参数类型正确（String、int、自定义模型）
- [ ] 异常映射为域错误（如 `ApiError`）
- [ ] import 无误，可直接编译

---

## 🛠️ 常见调整

生成后可能需要手动微调：

### 1. 添加查询参数

AI 可能生成：

```dart
Future<List<User>> getUsers() async {
  final response = await dio.get('/users');
  return (response.data as List).map((e) => User.fromJson(e)).toList();
}
```

需调整为带分页参数：

```dart
Future<List<User>> getUsers({
  int page = 1,
  int pageSize = 20,
}) async {
  final response = await dio.get(
    '/users',
    queryParameters: {'page': page, 'page_size': pageSize},
  );
  return (response.data as List).map((e) => User.fromJson(e)).toList();
}
```

### 2. 请求体序列化

AI 可能忘记调用 `toJson()`：

```dart
Future<User> createUser(User user) async {
  final response = await dio.post(
    '/users',
    data: user.toJson(), // ✅ 必须序列化
  );
  return User.fromJson(response.data);
}
```

### 3. 统一错误映射

在每个方法加 try/catch（或在 Repository 层统一处理）：

```dart
Future<User> getUser(String id) async {
  try {
    final response = await dio.get('/users/$id');
    return User.fromJson(response.data);
  } on DioException catch (e) {
    throw ApiErrorMapper.mapDioException(e);
  }
}
```

### 4. 上传文件

对于 FormData 接口：

```dart
import 'package:dio/dio.dart';

Future<void> uploadAvatar(String userId, File file) async {
  final formData = FormData.fromMap({
    'user_id': userId,
    'file': await MultipartFile.fromFile(file.path, filename: 'avatar.jpg'),
  });

  await dio.post('/users/$userId/avatar', data: formData);
}
```

---

## 🔗 相关

- 工作流：[workflows/dio-network-layer.md](../../workflows/dio-network-layer.md)
- 统一错误处理：[unify-api-error-handling.md](unify-api-error-handling.md)
