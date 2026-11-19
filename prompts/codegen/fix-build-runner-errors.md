# 提示词：修复 build_runner 生成错误

**用途**：诊断和修复 freezed / json_serializable / build_runner 代码生成失败或静态分析错误

---

## 📋 适用场景

- `dart run build_runner build` 执行失败
- 生成后 `dart analyze` 报错
- `.freezed.dart` 或 `.g.dart` 未生成
- 运行时 JSON 序列化抛异常
- 升级依赖版本后生成代码不兼容

---

## 📥 输入要求

收集以下信息：

1. **完整错误信息**
   ```bash
   # 复制终端输出
   dart run build_runner build --delete-conflicting-outputs
   [完整错误堆栈]
   ```

2. **相关模型代码**
   - 打开报错的 `.dart` 文件（如 `lib/models/user.dart`）

3. **pubspec.yaml 依赖版本**
   ```yaml
   dependencies:
     freezed_annotation: ^X.X.X
     json_annotation: ^X.X.X
   dev_dependencies:
     build_runner: ^X.X.X
     freezed: ^X.X.X
     json_serializable: ^X.X.X
   ```

4. **Dart SDK 版本**
   ```bash
   dart --version
   ```

---

## 💬 提示词模板

### 通用诊断

```markdown
build_runner 代码生成失败，需要诊断并修复。

**错误信息**：
\`\`\`
[粘贴完整错误输出，包括堆栈跟踪]
\`\`\`

**报错文件**：`lib/models/[FILE_NAME].dart`

**依赖版本**：
\`\`\`yaml
freezed_annotation: ^[VERSION]
json_annotation: ^[VERSION]
build_runner: ^[VERSION]
freezed: ^[VERSION]
json_serializable: ^[VERSION]
\`\`\`

**请**：

1. **识别问题类型**：
   - Part 声明错误？
   - 类型不匹配？
   - 注解配置错误？
   - 依赖版本冲突？
   - 泛型或嵌套序列化问题？

2. **给出修复方案**（代码 + 说明）

3. **提供验证步骤**：
   \`\`\`bash
   # 修复后执行
   dart run build_runner clean
   dart run build_runner build --delete-conflicting-outputs
   dart analyze
   \`\`\`

如果是依赖版本问题，说明需要升级/降级哪些包。
```

---

### 常见错误专项修复

#### 错误类型 1: Part 声明问题

```markdown
错误信息包含 `part 'xxx.dart' not found` 或 `part of 'xxx.dart' must be a part of 'yyy.dart'`。

**当前代码**：
\`\`\`dart
[粘贴模型代码]
\`\`\`

**请检查并修复**：

1. Part 文件名是否与类文件名匹配？
   - 文件 `user.dart` → part 应为 `user.freezed.dart` 和 `user.g.dart`
   - 不是 `User.freezed.dart`（注意大小写）

2. Part 声明顺序（固定）：
   \`\`\`dart
   part 'xxx.freezed.dart';
   part 'xxx.g.dart';
   \`\`\`

3. 是否漏掉了某个 part？
   - 用了 `@freezed` 必须有 `.freezed.dart`
   - 用了 `fromJson/toJson` 必须有 `.g.dart`

给出修复后的完整 import 和 part 声明。
```

---

#### 错误类型 2: 冲突输出

```markdown
错误信息包含 `Conflicting outputs` 或 `existing outputs`.

**解决方案**：

\`\`\`bash
# 方案1：强制删除冲突文件（推荐）
dart run build_runner build --delete-conflicting-outputs

# 方案2：手动清理后重新生成
dart run build_runner clean
dart run build_runner build

# 方案3：删除特定生成文件
rm lib/models/*.g.dart lib/models/*.freezed.dart
dart run build_runner build
\`\`\`

**如果仍失败**，说明哪个文件被外部锁定（如 IDE 占用），关闭编辑器后重试。
```

---

#### 错误类型 3: 类型不匹配

```markdown
错误信息包含 `type 'Null' is not a subtype of type 'String'` 或 `A value of type 'X' can't be assigned to a variable of type 'Y'`。

**当前模型代码**：
\`\`\`dart
[粘贴模型定义]
\`\`\`

**JSON 样例**：
\`\`\`json
[粘贴实际 JSON]
\`\`\`

**请检查并修复**：

1. **可空性不匹配**：
   - JSON 字段可能为 null，但 Dart 类型非可空（`String` 应为 `String?`）
   - 字段未在 JSON 中出现，但 Dart 为 `required`（应改为可选或提供 `@Default`）

2. **类型错误**：
   - JSON 是数字但 Dart 是 `String`
   - JSON 是字符串但 Dart 是 `int`（需 `@JsonKey` 自定义转换）

3. **泛型未明确**：
   - ❌ `List items`（Dart 2 风格）
   - ✅ `List<OrderItem> items`

给出修复后的字段定义。
```

---

#### 错误类型 4: 泛型序列化

```markdown
错误信息涉及泛型类型（如 `ApiResponse<T>`）的 fromJson/toJson。

**当前代码**：
\`\`\`dart
[粘贴泛型模型代码]
\`\`\`

**请修复**：

泛型类 fromJson 需额外的类型转换参数：

\`\`\`dart
@freezed
class ApiResponse<T> with _$ApiResponse<T> {
  const factory ApiResponse({
    required int code,
    required T data,
  }) = _ApiResponse<T>;

  factory ApiResponse.fromJson(
    Map<String, dynamic> json,
    T Function(Object?) fromJsonT,  // 必须
  ) => _$ApiResponseFromJson(json, fromJsonT);
}

// 使用示例
final response = ApiResponse<User>.fromJson(
  jsonData,
  (data) => User.fromJson(data as Map<String, dynamic>),
);
\`\`\`

生成修复后的完整代码和使用示例。
```

---

#### 错误类型 5: 嵌套对象序列化

```markdown
错误信息涉及嵌套对象（如 `Order` 包含 `List<OrderItem>`）未正确序列化。

**当前模型代码**：
\`\`\`dart
[粘贴主对象和子对象代码]
\`\`\`

**请检查并修复**：

1. **子对象是否也是 freezed 或 JsonSerializable**？
   \`\`\`dart
   // ❌ 普通类不会自动序列化
   class OrderItem {
     final String productId;
     OrderItem(this.productId);
   }

   // ✅ 必须标注
   @freezed
   class OrderItem with _$OrderItem { ... }
   \`\`\`

2. **是否需要 explicitToJson**（如果用 @JsonSerializable）？
   \`\`\`dart
   @JsonSerializable(explicitToJson: true)
   class Order { ... }
   \`\`\`

3. **import 是否正确**？
   \`\`\`dart
   import 'order_item.dart';  // 必须导入子对象
   \`\`\`

给出修复后的完整代码。
```

---

#### 错误类型 6: 联合类型 JSON 判别

```markdown
错误信息涉及联合类型（union）的 fromJson 失败或判别字段缺失。

**当前代码**：
\`\`\`dart
[粘贴联合类型代码]
\`\`\`

**JSON 样例**：
\`\`\`json
[粘贴实际 JSON]
\`\`\`

**请修复**：

1. **默认判别方式**（runtimeType）：
   \`\`\`dart
   // 生成的 JSON 会包含 runtimeType 字段
   {"runtimeType": "success", "data": {...}}
   \`\`\`

2. **自定义判别字段**（如 API 用 `type` 或 `status`）：
   \`\`\`dart
   @Freezed(unionKey: 'type', unionValueCase: FreezedUnionCase.snake)
   class ApiResponse with _$ApiResponse {
     const factory ApiResponse.success(Data data) = Success;
     const factory ApiResponse.error(String message) = Error;
     
     factory ApiResponse.fromJson(Map<String, dynamic> json) 
       => _$ApiResponseFromJson(json);
   }

   // JSON 格式
   {"type": "success", "data": {...}}
   {"type": "error", "message": "Failed"}
   \`\`\`

3. **Fallback 分支**（可选）：
   \`\`\`dart
   @FreezedUnionValue('unknown')
   const factory ApiResponse.unknown() = Unknown;
   \`\`\`

给出修复后的完整代码。
```

---

#### 错误类型 7: 依赖版本冲突

```markdown
错误信息包含版本冲突（如 `version solving failed`）或 API 不兼容。

**当前 pubspec.yaml**：
\`\`\`yaml
[粘贴依赖部分]
\`\`\`

**请**：

1. **检查版本兼容性**：
   - freezed 4.x 需要 freezed_annotation ^2.4.0
   - json_serializable 6.x 需要 json_annotation ^4.8.0
   - build_runner 推荐最新稳定版

2. **推荐配置**（2026-11-16 稳定版本）：
   \`\`\`yaml
   dependencies:
     freezed_annotation: ^2.4.4
     json_annotation: ^4.9.0

   dev_dependencies:
     build_runner: ^2.17.1
     freezed: ^4.0.3
     json_serializable: ^6.15.2
   \`\`\`

3. **升级命令**：
   \`\`\`bash
   flutter pub upgrade freezed freezed_annotation json_serializable json_annotation build_runner
   flutter pub get
   dart run build_runner clean
   dart run build_runner build --delete-conflicting-outputs
   \`\`\`

如果有破坏性变更，说明需要修改代码的地方（如注解 API 变化）。
```

---

## ✅ 验证步骤

修复后执行完整验证流程：

```bash
# 1. 清理旧生成文件
dart run build_runner clean

# 2. 重新生成
dart run build_runner build --delete-conflicting-outputs

# 3. 静态分析
dart analyze

# 4. 运行测试（如有）
flutter test test/models/

# 5. 验证 JSON 序列化（手动测试或单元测试）
```

**检查清单**：
- [ ] build_runner 执行成功（无错误）
- [ ] `.freezed.dart` 和 `.g.dart` 已生成
- [ ] `dart analyze` 无错误或警告
- [ ] 单元测试通过（fromJson + toJson 往返）
- [ ] 实际 JSON 数据能正确解析

---

## 💡 预防措施

避免常见错误：

1. **代码生成前先 analyze**
   ```bash
   dart analyze lib/models/user.dart  # 先检查语法
   dart run build_runner build        # 再生成
   ```

2. **提交前验证生成文件是最新的**
   ```bash
   # CI 脚本
   dart run build_runner build --delete-conflicting-outputs
   git diff --exit-code  # 有变化说明忘记生成
   ```

3. **依赖版本固定**
   ```yaml
   # 团队统一版本，避免不一致
   freezed: 4.0.3  # 去掉 ^ 精确固定
   json_serializable: 6.15.2
   ```

4. **记录项目特定配置**
   - 如果需要 build.yaml 自定义配置，加注释说明原因
   - 文档化非标准的 Converter 和注解用法

---

## 🤔 常见问题

### Q1: 错误信息太长看不懂？
**A**: 先看第一条错误（通常是根本原因），忽略后续级联错误。关键词：
- `part ... not found` → Part 声明问题
- `type ... is not a subtype` → 类型不匹配
- `Conflicting outputs` → 旧文件冲突
- `version solving failed` → 依赖冲突

### Q2: 修复后仍然报错？
**A**: 
1. 重启 IDE（VSCode/Android Studio）
2. 清理 Dart/Flutter 缓存：`flutter clean && flutter pub get`
3. 删除 `.dart_tool/` 目录：`rm -rf .dart_tool && flutter pub get`
4. 检查是否有多个版本 SDK 或 Flutter 通道问题

### Q3: 能否禁用某个字段的生成？
**A**: 
```dart
@JsonKey(includeFromJson: false, includeToJson: false)
String? internalField,
```

### Q4: 升级依赖后 API 不兼容怎么办？
**A**: 
1. 查看 CHANGELOG：`https://pub.dev/packages/freezed/changelog`
2. 搜索 Breaking Changes
3. 常见迁移：
   - freezed 2.x → 4.x：注解 API 基本兼容，但 union 判别有细微变化
   - json_serializable 5.x → 6.x：泛型处理更严格

---

## 🔗 相关

- 工作流：[workflows/freezed-codegen.md](../../workflows/freezed-codegen.md)
- 生成模型提示词：[generate-freezed-model.md](generate-freezed-model.md)
- freezed 故障排查：[https://pub.dev/packages/freezed#troubleshooting](https://pub.dev/packages/freezed#troubleshooting)
- build_runner 文档：[https://pub.dev/packages/build_runner](https://pub.dev/packages/build_runner)
