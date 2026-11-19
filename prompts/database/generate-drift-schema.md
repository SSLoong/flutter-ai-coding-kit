# 提示词：生成 Drift Schema 与 DAO

**用途**：为新业务实体生成 Drift Table 定义、Database 类和基础 DAO 方法

---

## 📋 适用场景

- 新项目首次引入 Drift 本地数据库
- 已有 Drift 项目需新增表
- 从 sqflite 迁移到 Drift

---

## 📥 输入要求

准备以下信息：

1. **实体名称**（如 User、Product、Order）
2. **字段清单**：字段名、类型、约束（主键、唯一、非空、默认值）
3. **关系**（可选）：外键、一对多、多对多
4. **索引需求**（可选）：哪些字段需要索引

---

## 💬 提示词模板

```markdown
我需要在 Flutter 项目中用 Drift 创建本地数据库表。

**实体**：[实体名称，如 Product]

**字段定义**：
- [字段名]: [类型]，[约束说明]
  例如：
  - id: 整数，主键，自增
  - name: 字符串，非空
  - price: 浮点数，非空
  - description: 字符串，可选
  - categoryId: 整数，外键关联 Category 表
  - stockCount: 整数，非空，默认 0
  - createdAt: DateTime，非空
  - updatedAt: DateTime，非空

**索引**：
- [需要索引的字段，如 categoryId、name]

**要求**：

1. **Table 定义**：
   - 使用 Drift Dart 语法（继承 `Table` 类）
   - 每个字段用对应的列类型（IntColumn、TextColumn、RealColumn、DateTimeColumn 等）
   - 正确标注约束：`autoIncrement()`、`nullable()`、`unique()`、`withDefault()`
   - 如有外键，用 `references()` 或 `customConstraint()`

2. **Database 类**：
   - 使用 `@DriftDatabase` 注解，包含所有表
   - 包含 `schemaVersion` 字段（初始为 1）
   - 实现数据库连接逻辑（LazyDatabase + NativeDatabase）
   - 使用 `path_provider` 获取文档目录

3. **基础 DAO 方法**：
   在 Database 类中生成以下方法：
   - `getAll[Entity]s()` — 查询所有记录
   - `get[Entity]ById(int id)` — 根据主键查询
   - `insert[Entity]([Entity]Companion entity)` — 插入记录
   - `update[Entity]([Entity] entity)` — 更新记录
   - `delete[Entity](int id)` — 删除记录

4. **文件结构**：
   - `lib/database/tables.dart` — Table 定义
   - `lib/database/database.dart` — Database 类
   - 包含完整 import 和 part 声明

5. **代码生成说明**：
   - 说明如何运行 `build_runner` 生成 `.g.dart` 文件
   - 列出必要的 pubspec.yaml 依赖

生成完整可运行代码，不要省略任何部分。
```

---

## ✅ 验证步骤

生成后执行：

```bash
# 1. 确认依赖
flutter pub get

# 2. 运行代码生成
dart run build_runner build --delete-conflicting-outputs

# 3. 检查生成文件
ls lib/database/database.g.dart

# 4. 编译检查
flutter analyze lib/database/
```

**检查清单**：
- [ ] `.g.dart` 文件成功生成
- [ ] 无编译错误或类型警告
- [ ] Table 列定义符合业务逻辑
- [ ] Database 类包含所有基础 CRUD 方法

---

## 🛠️ 常见调整

生成后可能需要手动微调：

### 1. 添加复合索引

```dart
class Products extends Table {
  IntColumn get id => integer().autoIncrement()();
  TextColumn get name => text()();
  IntColumn get categoryId => integer()();
  
  @override
  List<Set<Column>> get uniqueKeys => [
    {name, categoryId}, // name + categoryId 组合唯一
  ];
}
```

### 2. 自定义约束（如外键）

```dart
class Orders extends Table {
  IntColumn get id => integer().autoIncrement()();
  IntColumn get userId => integer().customConstraint('REFERENCES users(id) ON DELETE CASCADE')();
}
```

### 3. 枚举类型转换

```dart
enum OrderStatus { pending, shipped, delivered }

class TypeConverters {
  static const orderStatus = EnumIndexConverter<OrderStatus>(OrderStatus.values);
}

class Orders extends Table {
  IntColumn get id => integer().autoIncrement()();
  IntColumn get status => intEnum<OrderStatus>()(); // 自动转换
}

// 在 Database 类中注册
@DriftDatabase(tables: [Orders])
class AppDatabase extends _$AppDatabase {
  @override
  int get schemaVersion => 1;
  
  // Drift 会自动处理 intEnum，无需额外配置
}
```

### 4. JSON 字段存储

```dart
import 'dart:convert';

class JsonMapConverter extends TypeConverter<Map<String, dynamic>, String> {
  const JsonMapConverter();
  
  @override
  Map<String, dynamic> fromSql(String fromDb) {
    return json.decode(fromDb) as Map<String, dynamic>;
  }
  
  @override
  String toSql(Map<String, dynamic> value) {
    return json.encode(value);
  }
}

class Products extends Table {
  IntColumn get id => integer().autoIncrement()();
  TextColumn get metadata => text().map(const JsonMapConverter())();
}
```

### 5. 添加默认值

```dart
class Products extends Table {
  IntColumn get id => integer().autoIncrement()();
  IntColumn get stockCount => integer().withDefault(const Constant(0))();
  DateTimeColumn get createdAt => dateTime().withDefault(currentDateAndTime)();
}
```

---

## 🔧 进阶场景

### 多表关联查询

```markdown
**追加提示词**：

现在需要添加关联查询：

1. `getProductsWithCategory()` — 返回产品及其关联的分类信息
2. `getOrdersByUser(int userId)` — 查询某用户的所有订单

使用 Drift 的 `join` 语法，返回自定义结果类。
```

### 批量操作与事务

```markdown
**追加提示词**：

新增批量方法：

1. `insertProducts(List<ProductsCompanion> products)` — 批量插入（事务）
2. `deleteProductsByCategory(int categoryId)` — 删除某分类下所有产品（事务）

使用 `transaction()` 确保原子性。
```

---

## 🔗 相关

- 工作流：[workflows/drift-local-database.md](../../workflows/drift-local-database.md)
- 迁移提示词：[add-drift-migration.md](add-drift-migration.md)
- Drift 官方文档：[https://drift.simonbinder.eu/](https://drift.simonbinder.eu/)
