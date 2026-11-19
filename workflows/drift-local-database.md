# Drift 本地数据库与 Schema 迁移工作流

**用 AI 辅助构建类型安全的 SQLite 本地持久化方案**

> 作者：Loong (SSLoong) | 日期：2026-11-17

---

## 📋 背景

直接使用 `sqflite` 需要手写 SQL 字符串，缺乏类型安全且易出错。Drift（前身 Moor）提供：
- ✅ **编译期类型检查**：Table 定义自动生成类型安全的查询 API
- ✅ **代码生成**：`.drift` 文件或 Dart 类 → DAO 和数据类
- ✅ **Schema 迁移管理**：`schemaVersion` + `MigrationStrategy` 统一升级逻辑
- ✅ **可测试性**：内存数据库 + 事务隔离，方便单元测试

AI 能快速生成 Table 定义、DAO 查询和迁移脚本，但**必须本地验证**（codegen + 测试）通过后才能合入。

---

## 🎯 目标

- ✅ 用 AI 生成 Drift Table、Database 类和 DAO 代码
- ✅ 配置 `build_runner` 自动生成 `.g.dart` 文件
- ✅ 实现 Schema 版本升级（新增列、新建表、数据迁移）
- ✅ 在单元测试中验证数据库逻辑（使用内存数据库）
- ✅ 避免常见陷阱（丢数据、忘升版本号、索引缺失）

---

## 🔧 适用场景

| 场景 | 适合度 | 说明 |
|------|--------|------|
| 新项目引入本地持久化 | ⭐⭐⭐⭐⭐ | 从零搭建 Drift 架构 |
| sqflite 迁移到 Drift | ⭐⭐⭐⭐ | 类型安全重构 |
| 已有表新增字段/索引 | ⭐⭐⭐⭐⭐ | Schema 迁移自动化 |
| 复杂查询与事务 | ⭐⭐⭐⭐ | AI 生成需人工审查性能 |
| 纯内存缓存 | ⭐⭐ | 优先考虑 Hive/Isar |

---

## 🚀 完整流程

### Step 1: 初始化依赖与配置

**人工操作**：在 `pubspec.yaml` 添加依赖（版本以最新稳定为准）：

```yaml
dependencies:
  drift: ^2.20.0
  sqlite3_flutter_libs: ^0.5.0
  path_provider: ^2.0.0
  path: ^1.8.0

dev_dependencies:
  drift_dev: ^2.20.0
  build_runner: ^2.4.0
```

**AI 提示词**（可复制）：

```markdown
我要在 Flutter 项目中集成 Drift 本地数据库。请生成：

1. `pubspec.yaml` 的依赖配置（drift、drift_dev、build_runner、sqlite3_flutter_libs、path_provider）
2. `build.yaml` 配置（如需自定义 build_runner 选项）
3. 说明各依赖的作用

使用当前稳定版本，不要使用已弃用的 moor 包。
```

**检查点**：
- [ ] `flutter pub get` 成功
- [ ] 依赖版本兼容（查看 pub.dev）

---

### Step 2: 设计 Table 与 Database

**人工操作**：明确业务需求，准备字段清单。例如用户表需要：`id (主键)`、`name`、`email`、`createdAt`。

**AI 提示词**（可复制）：

```markdown
为以下业务实体生成 Drift Table 定义和 Database 类：

**实体**：User
**字段**：
- id: 整数，主键，自增
- name: 字符串，非空
- email: 字符串，唯一约束
- avatarUrl: 字符串，可选
- createdAt: DateTime，非空

**要求**：
1. 使用 Drift 的 Dart Table 定义（不用 .drift 文件）
2. 生成 `@DriftDatabase` 注解的 AppDatabase 类
3. 包含完整 import
4. 说明如何运行 `build_runner` 生成代码

生成 `lib/database/tables.dart` 和 `lib/database/database.dart`。
```

**输出示例**（参考，实际以 AI 生成为准）：

```dart
// lib/database/tables.dart
import 'package:drift/drift.dart';

class Users extends Table {
  IntColumn get id => integer().autoIncrement()();
  TextColumn get name => text()();
  TextColumn get email => text().unique()();
  TextColumn get avatarUrl => text().nullable()();
  DateTimeColumn get createdAt => dateTime()();
}
```

```dart
// lib/database/database.dart
import 'package:drift/drift.dart';
import 'package:drift/native.dart';
import 'package:path_provider/path_provider.dart';
import 'package:path/path.dart' as p;
import 'dart:io';

import 'tables.dart';

part 'database.g.dart';

@DriftDatabase(tables: [Users])
class AppDatabase extends _$AppDatabase {
  AppDatabase() : super(_openConnection());

  @override
  int get schemaVersion => 1;

  // 查询方法示例
  Future<List<User>> getAllUsers() => select(users).get();
  Future<User?> getUserById(int id) => 
      (select(users)..where((u) => u.id.equals(id))).getSingleOrNull();
  Future<int> insertUser(UsersCompanion user) => into(users).insert(user);
  Future<bool> updateUser(User user) => update(users).replace(user);
  Future<int> deleteUser(int id) => 
      (delete(users)..where((u) => u.id.equals(id))).go();
}

LazyDatabase _openConnection() {
  return LazyDatabase(() async {
    final dbFolder = await getApplicationDocumentsDirectory();
    final file = File(p.join(dbFolder.path, 'app.db'));
    return NativeDatabase(file);
  });
}
```

**检查点**：
- [ ] Table 定义符合业务逻辑
- [ ] 主键、唯一约束、非空约束正确
- [ ] Database 类包含 `schemaVersion`

---

### Step 3: 运行代码生成

**人工操作**：

```bash
# 生成 .g.dart 文件
dart run build_runner build --delete-conflicting-outputs

# 若要持续监听文件变化
dart run build_runner watch --delete-conflicting-outputs
```

**常见错误**：
- ❌ `part 'database.g.dart' not found`：先运行 build_runner
- ❌ `Conflicting outputs`：加 `--delete-conflicting-outputs`
- ❌ 类型错误：检查 Table 列定义与 Companion 对象用法

**AI 提示词**（修复错误时使用）：

```markdown
build_runner 报错：

[粘贴错误信息]

请修复 Table 或 Database 定义，确保：
1. @DriftDatabase 注解正确
2. part 文件名匹配
3. 列类型定义符合 Drift 规范
```

---

### Step 4: 编写 DAO 与查询

**人工操作**：明确需要哪些查询（如按条件筛选、分页、排序）。

**AI 提示词**（可复制）：

```markdown
在 AppDatabase 中新增以下查询方法：

1. 按邮箱查找用户：`getUserByEmail(String email)`
2. 获取最近创建的 N 个用户（分页）：`getRecentUsers(int limit, int offset)`
3. 批量插入用户（事务）：`insertUsers(List<UsersCompanion> users)`
4. 更新用户名：`updateUserName(int id, String newName)`

**要求**：
- 使用 Drift 的类型安全查询 API（select、where、orderBy、limit）
- 批量操作用 `transaction()`
- 返回类型用 `Future<T>`
- 添加注释说明每个方法的用途
```

**输出示例**：

```dart
// 在 AppDatabase 类中添加

/// 根据邮箱查找用户
Future<User?> getUserByEmail(String email) {
  return (select(users)..where((u) => u.email.equals(email)))
      .getSingleOrNull();
}

/// 获取最近创建的用户（分页）
Future<List<User>> getRecentUsers(int limit, int offset) {
  return (select(users)
        ..orderBy([(u) => OrderingTerm.desc(u.createdAt)])
        ..limit(limit, offset: offset))
      .get();
}

/// 批量插入用户（事务保证原子性）
Future<void> insertUsers(List<UsersCompanion> userList) async {
  await transaction(() async {
    for (final user in userList) {
      await into(users).insert(user);
    }
  });
}

/// 更新用户名
Future<int> updateUserName(int id, String newName) {
  return (update(users)..where((u) => u.id.equals(id)))
      .write(UsersCompanion(name: Value(newName)));
}
```

---

### Step 5: Schema 迁移（新增列/表）

**场景**：已发布版本 schema v1，现在需要新增 `phoneNumber` 字段。

**人工操作**：
1. 修改 Table 定义（加新列）
2. 升级 `schemaVersion` → 2
3. 编写 `MigrationStrategy`

**AI 提示词**（可复制）：

```markdown
当前 User 表 schema 版本为 1。现需要新增字段 `phoneNumber`（字符串，可选）。

请帮我：
1. 修改 Users Table 定义，加入 `phoneNumber` 列
2. 更新 AppDatabase 的 `schemaVersion` 为 2
3. 实现 `MigrationStrategy`，在 `onUpgrade` 中执行 `ALTER TABLE` 添加列
4. 说明如何测试迁移（从 v1 升级到 v2）

完整代码，不要省略。
```

**输出示例**：

```dart
// 修改 lib/database/tables.dart
class Users extends Table {
  IntColumn get id => integer().autoIncrement()();
  TextColumn get name => text()();
  TextColumn get email => text().unique()();
  TextColumn get avatarUrl => text().nullable()();
  TextColumn get phoneNumber => text().nullable()(); // 新增
  DateTimeColumn get createdAt => dateTime()();
}

// 修改 lib/database/database.dart
@DriftDatabase(tables: [Users])
class AppDatabase extends _$AppDatabase {
  AppDatabase() : super(_openConnection());

  @override
  int get schemaVersion => 2; // 升级版本号

  @override
  MigrationStrategy get migration {
    return MigrationStrategy(
      onUpgrade: (Migrator m, int from, int to) async {
        if (from < 2) {
          // 从 v1 升级到 v2：添加 phoneNumber 列
          await m.addColumn(users, users.phoneNumber);
        }
      },
      beforeOpen: (details) async {
        // 开启外键约束（可选）
        await customStatement('PRAGMA foreign_keys = ON');
      },
    );
  }

  // ... 其他方法
}
```

**检查点**：
- [ ] `schemaVersion` 已递增
- [ ] `onUpgrade` 包含所有历史迁移步骤
- [ ] 重新运行 `build_runner build`
- [ ] 卸载 App 重装，或用迁移测试验证

---

### Step 6: 测试数据库逻辑

**人工操作**：编写单元测试，使用内存数据库隔离环境。

**AI 提示词**（可复制）：

```markdown
为 AppDatabase 生成单元测试，要求：

1. 使用内存数据库（`NativeDatabase.memory()`）
2. 测试插入、查询、更新、删除
3. 测试迁移（从 v1 升级到 v2）
4. 每个测试后关闭数据库

生成 `test/database/database_test.dart`，包含完整 import 和至少 5 个测试用例。
```

**输出示例**：

```dart
// test/database/database_test.dart
import 'package:drift/drift.dart';
import 'package:drift/native.dart';
import 'package:flutter_test/flutter_test.dart';
import 'package:my_app/database/database.dart';

// 创建内存数据库用于测试
AppDatabase _createTestDatabase() {
  return AppDatabase.override(NativeDatabase.memory());
}

void main() {
  late AppDatabase db;

  setUp(() {
    db = _createTestDatabase();
  });

  tearDown(() async {
    await db.close();
  });

  group('User CRUD Tests', () {
    test('insert and retrieve user', () async {
      // Arrange
      final user = UsersCompanion(
        name: Value('Alice'),
        email: Value('alice@example.com'),
        createdAt: Value(DateTime.now()),
      );

      // Act
      final id = await db.insertUser(user);
      final retrieved = await db.getUserById(id);

      // Assert
      expect(retrieved, isNotNull);
      expect(retrieved!.name, 'Alice');
      expect(retrieved.email, 'alice@example.com');
    });

    test('query user by email', () async {
      // Arrange
      await db.insertUser(UsersCompanion(
        name: Value('Bob'),
        email: Value('bob@example.com'),
        createdAt: Value(DateTime.now()),
      ));

      // Act
      final user = await db.getUserByEmail('bob@example.com');

      // Assert
      expect(user, isNotNull);
      expect(user!.name, 'Bob');
    });

    test('update user name', () async {
      // Arrange
      final id = await db.insertUser(UsersCompanion(
        name: Value('Charlie'),
        email: Value('charlie@example.com'),
        createdAt: Value(DateTime.now()),
      ));

      // Act
      await db.updateUserName(id, 'Charles');
      final updated = await db.getUserById(id);

      // Assert
      expect(updated!.name, 'Charles');
    });

    test('delete user', () async {
      // Arrange
      final id = await db.insertUser(UsersCompanion(
        name: Value('David'),
        email: Value('david@example.com'),
        createdAt: Value(DateTime.now()),
      ));

      // Act
      await db.deleteUser(id);
      final deleted = await db.getUserById(id);

      // Assert
      expect(deleted, isNull);
    });

    test('batch insert with transaction', () async {
      // Arrange
      final users = [
        UsersCompanion(
          name: Value('User1'),
          email: Value('user1@example.com'),
          createdAt: Value(DateTime.now()),
        ),
        UsersCompanion(
          name: Value('User2'),
          email: Value('user2@example.com'),
          createdAt: Value(DateTime.now()),
        ),
      ];

      // Act
      await db.insertUsers(users);
      final allUsers = await db.getAllUsers();

      // Assert
      expect(allUsers.length, 2);
    });
  });

  group('Migration Tests', () {
    test('migration from v1 to v2 adds phoneNumber column', () async {
      // 此测试需要模拟旧版数据库，实际项目中可用 Drift 的迁移测试工具
      // 这里简化验证新列存在
      final user = await db.getUserById(1);
      // phoneNumber 应可访问且为 null
      expect(user?.phoneNumber, isNull);
    });
  });
}

// 需要在 AppDatabase 中添加测试用构造函数
extension AppDatabaseTest on AppDatabase {
  static AppDatabase override(QueryExecutor executor) {
    return AppDatabase._internal(executor);
  }
}

// 在 database.dart 中添加
class AppDatabase extends _$AppDatabase {
  AppDatabase() : super(_openConnection());
  AppDatabase._internal(QueryExecutor e) : super(e);
  
  // ... 其他代码
}
```

**验证步骤**：

```bash
flutter test test/database/database_test.dart
```

**检查清单**：
- [ ] 所有测试通过
- [ ] 测试覆盖增删改查
- [ ] 迁移测试验证新列可用

---

### Step 7: 生产环境注意事项

**人工操作**：
1. **后台数据库初始化**（避免阻塞 UI）：
   ```dart
   // 使用单例 + lazy 初始化
   class DatabaseProvider {
     static AppDatabase? _instance;
     
     static Future<AppDatabase> get instance async {
       _instance ??= await _initDatabase();
       return _instance!;
     }
     
     static Future<AppDatabase> _initDatabase() async {
       // 后台创建连接
       return AppDatabase();
     }
   }
   ```

2. **关闭数据库**（App 退出时）：
   ```dart
   @override
   void dispose() {
     DatabaseProvider.instance.then((db) => db.close());
     super.dispose();
   }
   ```

3. **性能优化**：
   - 为常用查询字段添加索引
   - 大批量操作用事务
   - 避免 N+1 查询

**AI 提示词**（优化时使用）：

```markdown
当前 User 表查询 `getUserByEmail` 很慢（有 10 万条记录）。

请帮我：
1. 为 `email` 列添加索引
2. 修改 Table 定义（使用 `customConstraint` 或 Drift 索引语法）
3. 更新 schema 版本并编写迁移脚本
4. 说明如何验证索引生效（用 `EXPLAIN QUERY PLAN`）
```

---

## 🎨 可选：使用 .drift SQL 文件

若团队偏好写 SQL，可用 `.drift` 文件代替 Dart Table 定义。

**AI 提示词**：

```markdown
用 Drift SQL 文件语法重写 User 表定义。

生成 `lib/database/tables.drift`，包含：
- CREATE TABLE users
- 主键、唯一约束、非空约束
- 查询语句（getUserById、getAllUsers）

并更新 `database.dart` 引入 .drift 文件。
```

**示例**：

```sql
-- lib/database/tables.drift
CREATE TABLE users (
  id INTEGER PRIMARY KEY AUTOINCREMENT,
  name TEXT NOT NULL,
  email TEXT UNIQUE NOT NULL,
  avatar_url TEXT,
  phone_number TEXT,
  created_at INTEGER NOT NULL
);

getUserById: SELECT * FROM users WHERE id = ?;
getAllUsers: SELECT * FROM users ORDER BY created_at DESC;
```

```dart
// database.dart
import 'tables.drift';

@DriftDatabase(include: {'tables.drift'})
class AppDatabase extends _$AppDatabase {
  // ...
}
```

---

## ✅ 验证清单

完成后确认：

- [ ] `pubspec.yaml` 依赖配置正确
- [ ] `build_runner build` 成功生成 `.g.dart`
- [ ] Table 定义包含主键、约束、索引
- [ ] Database 类实现 `schemaVersion` 和 `MigrationStrategy`
- [ ] 单元测试通过（内存数据库）
- [ ] 迁移脚本在真实设备上验证无数据丢失
- [ ] DAO 方法有注释说明用途

---

## 🤔 FAQ

### Q1: Drift vs Hive vs Isar？
**A**:
- **Drift**: 类型安全 SQLite，适合关系数据、复杂查询、迁移管理
- **Hive**: 轻量 NoSQL，适合简单 KV 存储、设置项
- **Isar**: 高性能 NoSQL，适合大数据量、索引查询，但生态较新

### Q2: 何时不能直接 ALTER TABLE？
**A**:
- 删除列（SQLite 不支持，需重建表）
- 修改列类型（需数据迁移）
- 添加非空列且无默认值（已有数据会报错）

**解决方案**：创建临时表 → 复制数据 → 删除旧表 → 重命名。参考 [Drift 文档](https://drift.simonbinder.eu/docs/advanced-features/migrations/)。

### Q3: AI 生成的迁移脚本丢数据？
**A**:
1. **先测试迁移**：用内存数据库或测试设备验证
2. **备份数据**：关键操作前导出 SQLite 文件
3. **版本号递增**：确保每次修改 schema 都升 `schemaVersion`
4. **审查 AI 输出**：检查 `onUpgrade` 是否覆盖所有版本跨度

### Q4: Drift 与 freezed 如何配合？
**A**:
- **分离关注点**：Drift Table 用于数据库行，freezed 类用于业务域模型
- **映射转换**：编写 `toEntity()` / `fromEntity()` 方法
- **避免混用**：不要把 `@freezed` 类直接当表定义，会导致 build_runner 冲突

示例：
```dart
// 域模型（freezed）
@freezed
class UserEntity with _$UserEntity {
  const factory UserEntity({
    required int id,
    required String name,
    required String email,
  }) = _UserEntity;
}

// 数据库映射
extension UserMapper on User {
  UserEntity toEntity() => UserEntity(
    id: id,
    name: name,
    email: email,
  );
}
```

### Q5: 如何调试 SQL 执行？
**A**:
```dart
import 'package:drift/drift.dart';

AppDatabase() : super(_openConnection()) {
  // 打印所有 SQL 语句
  driftRuntimeOptions.dontWarnAboutMultipleDatabases = true;
}

LazyDatabase _openConnection() {
  return LazyDatabase(() async {
    final db = NativeDatabase(file, logStatements: true); // 启用日志
    return db;
  });
}
```

### Q6: 迁移时如何处理数据回填？
**A**:
```dart
onUpgrade: (m, from, to) async {
  if (from < 3) {
    // 先添加列（允许 null）
    await m.addColumn(users, users.status);
    
    // 回填默认值
    await customStatement(
      'UPDATE users SET status = ? WHERE status IS NULL',
      ['active'],
    );
  }
}
```

---

## 💡 最佳实践

1. **Schema 版本严格递增**
   - 每次修改表结构必须升版本号
   - 用注释标注每个版本改了什么

2. **迁移脚本向前兼容**
   - 保留所有历史迁移步骤（不要删除旧版本的 `if (from < N)`）
   - 新用户会从 v1 → v2 → v3 依次执行

3. **域模型与表分离**
   - Drift Table 专注数据库结构
   - 业务逻辑用 freezed/equatable 域模型
   - 用 extension 方法做映射

4. **测试覆盖迁移路径**
   - 写测试模拟从旧版本升级
   - 验证数据完整性（无丢失、类型正确）

5. **索引与性能**
   - 常用查询字段加索引（email、createdAt）
   - 用 `EXPLAIN QUERY PLAN` 验证索引生效
   - 大批量操作用 `transaction()`

6. **AI 输出人工审查要点**
   - 外键约束是否需要
   - 唯一索引是否合理
   - 迁移脚本是否覆盖边界情况（空表、大量数据）

---

## 🔗 相关资源

- 提示词模板：[prompts/database/generate-drift-schema.md](../prompts/database/generate-drift-schema.md)
- 迁移提示词：[prompts/database/add-drift-migration.md](../prompts/database/add-drift-migration.md)
- Drift 官方文档：[https://drift.simonbinder.eu/](https://drift.simonbinder.eu/)
- 迁移指南：[https://drift.simonbinder.eu/docs/advanced-features/migrations/](https://drift.simonbinder.eu/docs/advanced-features/migrations/)
- Freezed 工作流：（待 PR #11 合并后链接）

---

**Happy Coding! 🗄️**
