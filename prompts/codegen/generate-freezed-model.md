# 提示词：生成 freezed 数据模型

**用途**：根据 API 文档、JSON 样例或业务需求生成 freezed + json_serializable 数据模型

---

## 📋 适用场景

- 对接新 API 接口，需快速生成 Dart 模型
- 已有 JSON 响应，需转为类型安全的数据类
- 重构旧代码，从手写模型迁移到 freezed
- 设计复杂业务模型（联合类型、嵌套对象）

---

## 📥 输入要求

在 Cursor 中准备以下材料之一：

1. **JSON 样例**（推荐）
   ```json
   {
     "user_id": "123",
     "display_name": "Alice",
     "email": "alice@example.com",
     "avatar_url": null,
     "created_at": "2026-11-16T10:00:00Z"
   }
   ```

2. **API 文档截图或描述**
   - 字段名称、类型、是否必填、格式说明

3. **已有 Dart 类**（需迁移到 freezed）
   ```dart
   class User {
     final String id;
     final String name;
     String? avatarUrl;
     User(this.id, this.name, {this.avatarUrl});
   }
   ```

4. **业务需求描述**
   - 例：「商品信息，包含 ID、名称、价格、库存（整数）、可选图片列表」

---

## 💬 提示词模板

### 基础模型生成

```markdown
我需要生成一个 freezed + json_serializable 数据模型。

**场景**：[描述，如「用户信息 API 响应」、「订单详情」]

**JSON 样例**（如有）：
\`\`\`json
[粘贴真实 JSON]
\`\`\`

**要求**：

1. **文件路径**：`lib/models/[model_name].dart`

2. **依赖配置**（已在 pubspec.yaml）：
   - freezed_annotation: ^2.4.4
   - json_annotation: ^4.9.0
   - build_runner: ^2.17.1 (dev)
   - freezed: ^4.0.3 (dev)
   - json_serializable: ^6.15.2 (dev)

3. **代码结构**：
   \`\`\`dart
   import 'package:freezed_annotation/freezed_annotation.dart';

   part '[model_name].freezed.dart';
   part '[model_name].g.dart';

   @freezed
   class [ModelName] with _$[ModelName] {
     const factory [ModelName]({
       // 字段列表
     }) = _[ModelName];

     factory [ModelName].fromJson(Map<String, dynamic> json) 
       => _$[ModelName]FromJson(json);
   }
   \`\`\`

4. **字段规则**：
   - 根据 JSON 推断类型（`String`、`int`、`double`、`bool`、`List<T>`）
   - 可为 null 的字段用 `String?`、`int?` 等
   - JSON 蛇形命名（如 `user_id`）映射到 Dart 驼峰（`userId`）用 `@JsonKey(name: 'user_id')`
   - 日期字段 ISO8601 字符串 → `DateTime`
   - 需要默认值的字段用 `@Default(value)`

5. **完整性**：
   - 包含所有必要的 import
   - part 声明不能漏（两个 part）
   - 可直接运行 `dart run build_runner build --delete-conflicting-outputs`

**输出**：
1. 先列出字段清单（字段名、Dart 类型、是否可空、默认值、JSON 映射），供我确认
2. 确认后生成完整代码
```

---

### 联合类型（Union）生成

```markdown
我需要生成一个 freezed 联合类型（union）模型。

**场景**：[描述，如「API 响应结果包装」、「加载状态」、「多态业务实体」]

**分支说明**：
- [分支1名称]：[字段描述]
- [分支2名称]：[字段描述]
- [分支3名称]：[字段描述]

**示例（如有）**：
\`\`\`json
// 成功响应
{"status": "success", "data": {"id": "123", "name": "Item"}}

// 错误响应
{"status": "error", "message": "Not found"}
\`\`\`

**要求**：

1. **文件路径**：`lib/models/[model_name].dart`

2. **联合类型语法**：
   \`\`\`dart
   @freezed
   class Result<T> with _$Result<T> {
     const factory Result.success(T data) = Success<T>;
     const factory Result.error(String message, {int? code}) = Error<T>;
     
     factory Result.fromJson(
       Map<String, dynamic> json,
       T Function(Object?) fromJsonT,
     ) => _$ResultFromJson(json, fromJsonT);
   }
   \`\`\`

3. **判别字段**（如 JSON 有 type/status 字段）：
   \`\`\`dart
   @Freezed(unionKey: 'status', unionValueCase: FreezedUnionCase.snake)
   class ApiResponse with _$ApiResponse { ... }
   \`\`\`

4. **使用示例**（在代码注释里）：
   \`\`\`dart
   // 模式匹配
   result.when(
     success: (data) => print(data),
     error: (msg, code) => print('Error: $msg'),
   );
   \`\`\`

生成完整代码和使用示例。
```

---

### 复杂嵌套模型生成

```markdown
我需要生成嵌套的 freezed 模型（主对象包含子对象）。

**主对象**：[名称，如 Order]
**子对象**：[名称，如 OrderItem、Address]

**JSON 样例**：
\`\`\`json
{
  "order_id": "ORD-001",
  "items": [
    {"product_id": "P1", "quantity": 2, "price": 29.99},
    {"product_id": "P2", "quantity": 1, "price": 49.99}
  ],
  "shipping_address": {
    "street": "123 Main St",
    "city": "Beijing",
    "postal_code": "100000"
  }
}
\`\`\`

**要求**：

1. **文件组织**：
   - 选项A（推荐）：每个模型一个文件
     - `lib/models/order.dart`
     - `lib/models/order_item.dart`
     - `lib/models/address.dart`
   - 选项B：相关模型放一个文件（如果很小）
     - `lib/models/order.dart`（包含所有）

2. **嵌套字段**：
   \`\`\`dart
   @freezed
   class Order with _$Order {
     const factory Order({
       required String orderId,
       required List<OrderItem> items,  // 嵌套列表
       required Address shippingAddress,  // 嵌套对象
     }) = _Order;

     factory Order.fromJson(Map<String, dynamic> json) 
       => _$OrderFromJson(json);
   }
   \`\`\`

3. **依赖引用**（如多文件）：
   \`\`\`dart
   import 'order_item.dart';
   import 'address.dart';
   \`\`\`

生成所有需要的模型文件，标注文件路径。
```

---

## ✅ 验证步骤

生成后执行：

```bash
# 1. 代码生成
dart run build_runner build --delete-conflicting-outputs

# 2. 静态分析
dart analyze lib/models/[model_name].dart

# 3. 单元测试（可让 AI 一起生成）
flutter test test/models/[model_name]_test.dart
```

**检查清单**：
- [ ] 生成成功，无错误输出
- [ ] `.freezed.dart` 和 `.g.dart` 文件已创建
- [ ] `dart analyze` 无问题
- [ ] 字段类型与 JSON 匹配（可空性、类型）
- [ ] JSON 映射正确（蛇形 ↔ 驼峰）
- [ ] 测试通过（fromJson + toJson 往返一致）

---

## 🛠️ 常见调整

生成后可能需要手动微调：

### 1. 自定义日期格式

```dart
// 如果 JSON 是时间戳毫秒而非 ISO8601 字符串
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
    @TimestampConverter() required DateTime timestamp,
  }) = _Event;
  // ...
}
```

### 2. 忽略某些 JSON 字段

```dart
@JsonKey(includeFromJson: false, includeToJson: false)
String? internalField,
```

### 3. 计算字段（不序列化）

```dart
@freezed
class Product with _$Product {
  const Product._();  // 允许添加方法
  
  const factory Product({
    required double price,
    required int quantity,
  }) = _Product;

  factory Product.fromJson(Map<String, dynamic> json) => _$ProductFromJson(json);

  // 计算属性（不会序列化）
  double get totalPrice => price * quantity;
}
```

### 4. 泛型类型参数

```dart
// 使用时
final response = ApiResponse<List<User>>.fromJson(
  json,
  (data) => (data as List).map((e) => User.fromJson(e)).toList(),
);
```

---

## 🤔 常见问题

### Q1: AI 生成的字段类型不对？
**A**: 提供更详细的 JSON 样例，包含实际值：
- ❌ `"count": null` → AI 可能误判为 `String?`
- ✅ `"count": 42` → 明确是 `int`

如果 JSON 样例里某字段确实是 null，在需求里明确说明类型。

### Q2: 日期字段生成失败？
**A**: 明确日期格式：
- ISO8601 字符串（`2026-11-16T10:00:00Z`）→ 可直接用 `DateTime`
- 时间戳毫秒 → 需自定义 `TimestampConverter`
- 其他格式 → 提供转换逻辑

### Q3: 能一次生成多个关联模型吗？
**A**: 可以，但建议：
- 先生成主模型，确认结构
- 再生成子模型，逐个验证
- 最后组合测试

避免一次生成太多代码难以排查问题。

---

## 🔗 相关

- 工作流：[workflows/freezed-codegen.md](../../workflows/freezed-codegen.md)
- 修复生成错误：[fix-build-runner-errors.md](fix-build-runner-errors.md)
