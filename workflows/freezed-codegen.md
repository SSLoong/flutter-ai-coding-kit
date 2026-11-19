# freezed 数据模型与代码生成工作流

**用 AI 生成不可变数据模型与 JSON 序列化代码**

> 作者：Loong (SSLoong) | 日期：2026-11-16

---

## 📋 背景

手写 Dart 数据类需要大量样板代码：构造函数、copyWith、相等性判断、toString、JSON 序列化。`freezed` + `json_serializable` 通过代码生成自动化这些工作，但必须理解**何时用、何时不用**，并掌握生成流程和常见坑。

---

## 🎯 目标

- ✅ 理解 freezed、json_serializable、Dart 3 sealed class 的选型边界
- ✅ 用 AI 根据 API 文档或 JSON 样例快速生成模型
- ✅ 正确配置依赖与 build_runner
- ✅ 验证生成代码通过 `dart analyze` 和单元测试
- ✅ 掌握联合类型（union）、自定义 Converter、泛型等高级用法

---

## 🔧 适用场景

| 场景 | 推荐方案 | 说明 |
|------|----------|------|
| 复杂业务模型（需 copyWith、等价性） | ⭐⭐⭐⭐⭐ freezed + json_serializable | 不可变性 + 模式匹配 + JSON |
| API 响应/请求模型 | ⭐⭐⭐⭐⭐ freezed + json_serializable | 自动序列化，减少错误 |
| 联合类型（如 `Result<T>`、状态机） | ⭐⭐⭐⭐⭐ freezed union | 类型安全的枚举分支 |
| 简单 DTO（只需 JSON 转换） | ⭐⭐⭐⭐ json_serializable | 不需要 freezed 额外开销 |
| 轻量配置类（3 个字段以内） | ⭐⭐⭐ Dart 3 record | 手写即可，无需生成 |
| 密封类型（穷尽性检查） | ⭐⭐⭐⭐ Dart 3 sealed class | 原生语法，无需 freezed |
| 本地缓存/数据库模型 | ⭐⭐⭐ 看持久化方案 | Hive/Drift/Isar 有各自注解 |

**选型原则**（技术负责人视角）：
- **优先 Dart 3 原生能力**：record、sealed class 够用时不引入生成依赖
- **freezed 的价值**：自动 copyWith、深度不可变、联合类型模式匹配
- **json_serializable 的价值**：类型安全的 JSON 转换，减少手写序列化错误
- **不要过度工程**：3 字段以内的纯数据类手写更快

---

## 🚀 完整流程

### Step 1: 配置依赖

**人工操作**：在 `pubspec.yaml` 中添加依赖。

```yaml
dependencies:
  freezed_annotation: ^2.4.4  # 注解定义（运行时需要）
  json_annotation: ^4.9.0     # JSON 注解（运行时需要）

dev_dependencies:
  build_runner: ^2.17.1       # 代码生成器
  freezed: ^4.0.3             # freezed 生成器
  json_serializable: ^6.15.2  # JSON 生成器
```

**说明**：
- `freezed_annotation` 和 `json_annotation` 是 **dependencies**（运行时需要注解）
- `build_runner`、`freezed`、`json_serializable` 是 **dev_dependencies**（开发时生成代码）

**安装依赖**：
```bash
flutter pub get
```

---

### Step 2: 准备上下文

**人工操作**：在 Cursor 中准备以下材料之一：

1. **API 文档**（如 Swagger、接口文档）
2. **JSON 响应样例**（复制一段真实返回）
3. **已有 Dart 类**（需要迁移到 freezed）
4. **业务需求描述**（如「用户信息，包含 id、姓名、邮箱、可选头像 URL」）

**AI 提示词**（可复制）：

```markdown
我需要为以下场景生成 freezed + json_serializable 数据模型。

**场景**：[描述，如「用户信息 API 响应」]

**JSON 样例**（如有）：
\`\`\`json
{
  "id": "123",
  "name": "Alice",
  "email": "alice@example.com",
  "avatar_url": null,
  "created_at": "2026-11-16T10:00:00Z"
}
\`\`\`

**要求**：
1. 生成 `lib/models/user.dart`
2. 使用 `@freezed` 注解
3. 字段类型根据 JSON 推断（如 `String?` 表示可空）
4. 支持 `fromJson` / `toJson`
5. 日期字段用 `DateTime`，需自定义 `JsonConverter`（如有）
6. 包含完整 import 和 part 声明

先列出字段清单确认类型，再生成完整代码。
```

**检查点**：AI 列出的字段类型（`String`、`int?`、`DateTime`）是否符合预期？命名风格是否与项目一致（驼峰 vs 蛇形）？

---

### Step 3: 生成模型代码

**AI 提示词**（可复制）：

```markdown
基于上述需求，生成完整的 freezed 数据模型。

**文件路径**：`lib/models/[MODEL_NAME].dart`

**代码规范**：

1. **Import 顺序**：
   \`\`\`dart
   import 'package:freezed_annotation/freezed_annotation.dart';
   import 'package:json_annotation/json_annotation.dart';
   \`\`\`

2. **Part 声明**（两个 part，顺序固定）：
   \`\`\`dart
   part '[model_name].freezed.dart';
   part '[model_name].g.dart';
   \`\`\`

3. **类定义**：
   \`\`\`dart
   @freezed
   class User with _$User {
     const factory User({
       required String id,
       required String name,
       String? avatarUrl,  // 可空字段
       @Default(false) bool isActive,  // 默认值
     }) = _User;

     factory User.fromJson(Map<String, dynamic> json) => _$UserFromJson(json);
   }
   \`\`\`

4. **字段命名映射**（JSON 蛇形 → Dart 驼峰）：
   \`\`\`dart
   @JsonKey(name: 'avatar_url')
   String? avatarUrl,
   \`\`\`

5. **自定义转换**（如 ISO8601 时间戳）：
   \`\`\`dart
   @JsonKey(fromJson: _dateTimeFromJson, toJson: _dateTimeToJson)
   DateTime createdAt,

   // 在类外部定义（或使用全局 Converter）
   DateTime _dateTimeFromJson(String json) => DateTime.parse(json);
   String _dateTimeToJson(DateTime date) => date.toIso8601String();
   \`\`\`

6. **联合类型**（如需要）：
   \`\`\`dart
   @freezed
   class Result<T> with _$Result<T> {
     const factory Result.success(T data) = Success<T>;
     const factory Result.error(String message) = Error<T>;
     
     factory Result.fromJson(
       Map<String, dynamic> json,
       T Function(Object?) fromJsonT,
     ) => _$ResultFromJson(json, fromJsonT);
   }
   \`\`\`

生成完整代码，不要省略 import 和 part。
```

**输出示例**（参考，实际以 AI 生成为准）：

```dart
import 'package:freezed_annotation/freezed_annotation.dart';

part 'user.freezed.dart';
part 'user.g.dart';

@freezed
class User with _$User {
  const factory User({
    required String id,
    required String name,
    required String email,
    @JsonKey(name: 'avatar_url') String? avatarUrl,
    @JsonKey(name: 'created_at') required DateTime createdAt,
  }) = _User;

  factory User.fromJson(Map<String, dynamic> json) => _$UserFromJson(json);
}
```

---

### Step 4: 运行代码生成

**人工操作**：

```bash
# 一次性生成（推荐用于 CI 和首次生成）
dart run build_runner build --delete-conflicting-outputs

# 监听模式（开发时自动重新生成）
dart run build_runner watch --delete-conflicting-outputs
```

**参数说明**：
- `build`：一次性生成，完成后退出
- `watch`：监听文件变化自动重新生成（保持运行）
- `--delete-conflicting-outputs`：自动覆盖冲突文件（避免手动删除 `.g.dart`）

**成功输出示例**：
```
[INFO] Generating build script completed, took 412ms
[INFO] Creating build script snapshot... completed, took 8.2s
[INFO] Building new asset graph completed, took 1.2s
[INFO] Checking for unexpected pre-existing outputs. completed, took 1ms
[INFO] Running build completed, took 3.4s
[INFO] Caching finalized dependency graph completed, took 89ms
[INFO] Succeeded after 3.5s with 4 outputs (8 actions)
```

**生成文件**：
- `lib/models/user.freezed.dart`（freezed 生成的 copyWith、==、toString 等）
- `lib/models/user.g.dart`（json_serializable 生成的 fromJson/toJson）

---

### Step 5: 本地验证（必须！）

**人工操作**：

```bash
# 1. 静态分析
dart analyze

# 2. 单元测试（编写基础 JSON 序列化测试）
flutter test test/models/user_test.dart
```

**验证清单**：
- [ ] `dart analyze` 无错误（0 issues）
- [ ] `.freezed.dart` 和 `.g.dart` 已生成
- [ ] 模型可正常 `fromJson` / `toJson`
- [ ] `copyWith` 工作正常
- [ ] 联合类型（如有）模式匹配编译通过

**单元测试示例**（可让 AI 生成）：

```dart
import 'package:flutter_test/flutter_test.dart';
import 'package:my_app/models/user.dart';

void main() {
  group('User Model Tests', () {
    test('fromJson deserializes correctly', () {
      final json = {
        'id': '123',
        'name': 'Alice',
        'email': 'alice@example.com',
        'avatar_url': null,
        'created_at': '2026-11-16T10:00:00Z',
      };

      final user = User.fromJson(json);

      expect(user.id, '123');
      expect(user.name, 'Alice');
      expect(user.avatarUrl, isNull);
      expect(user.createdAt, DateTime.parse('2026-11-16T10:00:00Z'));
    });

    test('toJson serializes correctly', () {
      final user = User(
        id: '456',
        name: 'Bob',
        email: 'bob@example.com',
        createdAt: DateTime.parse('2026-11-16T12:00:00Z'),
      );

      final json = user.toJson();

      expect(json['id'], '456');
      expect(json['name'], 'Bob');
      expect(json['avatar_url'], isNull);
      expect(json['created_at'], '2026-11-16T12:00:00Z');
    });

    test('copyWith updates specific fields', () {
      final user = User(
        id: '789',
        name: 'Charlie',
        email: 'charlie@example.com',
        createdAt: DateTime.now(),
      );

      final updated = user.copyWith(name: 'Charles');

      expect(updated.id, '789');  // 不变
      expect(updated.name, 'Charles');  // 已更新
      expect(updated.email, 'charlie@example.com');  // 不变
    });

    test('equality works correctly', () {
      final user1 = User(
        id: '1',
        name: 'Test',
        email: 'test@example.com',
        createdAt: DateTime.parse('2026-01-01T00:00:00Z'),
      );

      final user2 = User(
        id: '1',
        name: 'Test',
        email: 'test@example.com',
        createdAt: DateTime.parse('2026-01-01T00:00:00Z'),
      );

      expect(user1, equals(user2));
      expect(user1.hashCode, equals(user2.hashCode));
    });
  });
}
```

---

### Step 6: 迭代修复

**若生成失败或 analyze 报错**，用以下提示词让 AI 修复：

```markdown
代码生成或静态分析报错：

[粘贴完整错误信息]

请修复模型代码，确保：
1. Part 声明正确（`.freezed.dart` 和 `.g.dart` 都有）
2. JsonKey 映射正确（蛇形 vs 驼峰）
3. 泛型类型明确（如 `List<String>` 不能写成 `List`）
4. 嵌套对象正确序列化（需 `@JsonSerializable(explicitToJson: true)` 或 freezed 默认处理）
5. 自定义 Converter 签名正确

修复后重新运行 `dart run build_runner build --delete-conflicting-outputs`。
```

**常见错误与解决**：

| 错误 | 原因 | 解决 |
|------|------|------|
| `part 'xxx.dart' not found` | 未生成或路径错误 | 检查 part 文件名，重新运行 build_runner |
| `Conflicting outputs` | 旧生成文件冲突 | 加 `--delete-conflicting-outputs` |
| `The getter 'copyWith' isn't defined` | `.freezed.dart` 未生成 | 确认 `@freezed` 注解存在，重新生成 |
| `type 'Null' is not a subtype of type 'String'` | JSON 字段实际为 null，但类型非可空 | 改为 `String?` |
| `fromJsonT` 参数错误（泛型联合类型） | 泛型序列化需手动传递 | 参考官方文档 Generic 部分 |
| `Missing @JsonSerializable` | 嵌套类未标注 | 嵌套类也需 `@freezed` 或 `@JsonSerializable` |

---

## 🎨 高级用法

### 1. 联合类型（Union / Sealed）

**场景**：状态机、Result 类型、多态 API 响应。

```dart
@freezed
class LoadingState<T> with _$LoadingState<T> {
  const factory LoadingState.idle() = Idle<T>;
  const factory LoadingState.loading() = Loading<T>;
  const factory LoadingState.success(T data) = Success<T>;
  const factory LoadingState.error(String message) = Error<T>;
}

// 使用模式匹配
final state = LoadingState<User>.success(user);
state.when(
  idle: () => Text('Ready'),
  loading: () => CircularProgressIndicator(),
  success: (data) => Text(data.name),
  error: (msg) => Text('Error: $msg'),
);
```

### 2. 自定义 JsonConverter

**场景**：非标准格式（如时间戳毫秒、颜色十六进制）。

```dart
class TimestampConverter implements JsonConverter<DateTime, int> {
  const TimestampConverter();

  @override
  DateTime fromJson(int json) => DateTime.fromMillisecondsSinceEpoch(json);

  @override
  int toJson(DateTime object) => object.millisecondsSinceEpoch;
}

@freezed
class Event with _$Event {
  const factory Event({
    required String title,
    @TimestampConverter() required DateTime timestamp,
  }) = _Event;

  factory Event.fromJson(Map<String, dynamic> json) => _$EventFromJson(json);
}
```

### 3. 嵌套对象与泛型

```dart
@freezed
class ApiResponse<T> with _$ApiResponse<T> {
  const factory ApiResponse({
    required int code,
    required String message,
    required T data,
  }) = _ApiResponse<T>;

  factory ApiResponse.fromJson(
    Map<String, dynamic> json,
    T Function(Object?) fromJsonT,
  ) => _$ApiResponseFromJson(json, fromJsonT);
}

// 使用
final response = ApiResponse<User>.fromJson(
  jsonDecode(responseBody),
  (json) => User.fromJson(json as Map<String, dynamic>),
);
```

---

## 🤔 FAQ

### Q1: freezed 和 json_serializable 必须一起用吗？

**A**: 不是。
- **只用 json_serializable**：如果只需 JSON 转换，不需要 copyWith / 不可变性
  ```dart
  @JsonSerializable()
  class User {
    final String id;
    final String name;
    User(this.id, this.name);
    factory User.fromJson(Map<String, dynamic> json) => _$UserFromJson(json);
    Map<String, dynamic> toJson() => _$UserToJson(this);
  }
  ```
- **只用 freezed**：如果不需要 JSON，只要不可变性和 copyWith
  ```dart
  @freezed
  class Config with _$Config {
    const factory Config({required String apiUrl}) = _Config;
  }
  ```

### Q2: 生成文件要提交到 Git 吗？

**A**: **推荐提交**。
- **提交**的好处：拉代码即可运行，无需先生成；代码审查能看到变化
- **不提交**的风险：CI 必须先跑 build_runner；团队成员可能忘记生成
- **行业实践**：Flutter 官方和多数项目**提交** `.g.dart` 和 `.freezed.dart`
- `.gitignore` 里**不要加** `*.g.dart` 和 `*.freezed.dart`

### Q3: build_runner 很慢怎么办？

**A**: 
1. **用 watch 模式**：开发时保持运行，避免频繁全量构建
2. **只生成特定文件**：`dart run build_runner build --build-filter=lib/models/user.dart`
3. **清理缓存**：`dart run build_runner clean` 再重新构建
4. **升级依赖**：确保 build_runner 和生成器是最新版本

### Q4: 能否与 Riverpod / Bloc / GetX 一起用？

**A**: **完全可以**。freezed 只负责数据模型，与状态管理方案无关。
```dart
// Riverpod 示例
@riverpod
class UserNotifier extends _$UserNotifier {
  @override
  LoadingState<User> build() => const LoadingState.idle();

  Future<void> fetchUser(String id) async {
    state = const LoadingState.loading();
    try {
      final user = await _repo.getUser(id);  // 返回 freezed User
      state = LoadingState.success(user);
    } catch (e) {
      state = LoadingState.error(e.toString());
    }
  }
}
```

### Q5: freezed 生成的代码能手动改吗？

**A**: **不能**。`.freezed.dart` 和 `.g.dart` 是自动生成的，每次 build_runner 会覆盖。
- **扩展功能**：用 extension 或辅助方法
  ```dart
  extension UserX on User {
    String get displayName => avatarUrl != null ? '$name 👤' : name;
  }
  ```
- **自定义逻辑**：在原始 `@freezed` 类里加 private 构造函数和方法（见官方文档 Custom methods）

### Q6: 联合类型 JSON 序列化失败？

**A**: 联合类型默认用 `runtimeType` 判别，需手动配置或传递判别字段。
```dart
@Freezed(unionKey: 'type', unionValueCase: FreezedUnionCase.snake)
class Shape with _$Shape {
  const factory Shape.circle(double radius) = Circle;
  const factory Shape.rectangle(double width, double height) = Rectangle;
  
  factory Shape.fromJson(Map<String, dynamic> json) => _$ShapeFromJson(json);
}

// JSON 格式
{"type": "circle", "radius": 5.0}
{"type": "rectangle", "width": 10.0, "height": 20.0}
```

### Q7: 能否在 CI 里验证生成文件是最新的？

**A**: 可以，避免开发者忘记生成就提交。
```yaml
# .github/workflows/ci.yml
- name: Check generated files are up-to-date
  run: |
    dart run build_runner build --delete-conflicting-outputs
    git diff --exit-code  # 如果有变化则失败
```

### Q8: Dart 3 sealed class 和 freezed union 该选哪个?

**A**: 
- **Dart 3 sealed class**：原生语法，编译时穷尽性检查，无需依赖，适合简单状态枚举
  ```dart
  sealed class Result<T> {}
  class Success<T> extends Result<T> { final T data; Success(this.data); }
  class Error<T> extends Result<T> { final String msg; Error(this.msg); }
  ```
- **freezed union**：自动生成 when/map、copyWith、等价性，适合复杂多态业务模型且需 JSON
- **选型**：如果不需要 JSON 序列化且分支简单 → sealed class；否则 → freezed

---

## 💡 最佳实践

1. **选型先于工具**
   - 3 字段以内轻量类：手写 Dart 3 record
   - 需密封穷尽检查但无 JSON：sealed class
   - 复杂业务模型需 copyWith + JSON：freezed + json_serializable
   - 只需 JSON 序列化：json_serializable

2. **上下文越详细，生成质量越高**
   - 提供真实 JSON 样例（包含 null 和边界情况）
   - 注明字段含义和业务约束（如「用户 ID，非空唯一」）
   - 附上相关模型（如 User 引用 Address）

3. **验证标准前置**
   - 告诉 AI「代码必须通过 `dart analyze` 且包含单元测试」
   - 要求生成 fromJson/toJson 的测试用例

4. **团队约定**
   - 统一命名风格（JSON 蛇形 vs Dart 驼峰映射）
   - 生成文件是否提交 Git（推荐提交）
   - build.yaml 配置（如全局 `explicit_to_json: true`）

5. **CI 集成**
   - 在 CI 里验证生成文件是最新的（避免忘记生成）
   - 或者 CI 里统一跑一次 build_runner（适合不提交生成文件的团队）

6. **边界清晰**
   - 本工作流只负责**数据模型与 JSON 序列化**
   - 网络请求层（Dio Interceptor）见 [PR #10](https://github.com/SSLoong/flutter-ai-coding-kit/pull/10)
   - 模型在 Repository 之下使用，但本工作流不涉及具体 Repository 实现

---

## 🔗 相关资源

- 生成模型提示词：[prompts/codegen/generate-freezed-model.md](../prompts/codegen/generate-freezed-model.md)
- 修复生成错误：[prompts/codegen/fix-build-runner-errors.md](../prompts/codegen/fix-build-runner-errors.md)
- freezed 官方文档：[https://pub.dev/packages/freezed](https://pub.dev/packages/freezed)
- json_serializable 官方文档：[https://pub.dev/packages/json_serializable](https://pub.dev/packages/json_serializable)
- Dart 3 sealed class：[https://dart.dev/language/class-modifiers#sealed](https://dart.dev/language/class-modifiers#sealed)

---

**让数据模型生成更智能 🚀**
