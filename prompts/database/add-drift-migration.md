# 提示词：添加 Drift Schema 迁移

**用途**：在已有 Drift 数据库上安全地执行 Schema 变更（新增列、新建表、修改约束）

---

## 📋 适用场景

- 已发布应用需新增表字段
- 新建表或删除表
- 修改列类型或约束
- 添加索引或外键

---

## 📥 输入要求

在 Cursor 中准备：

1. **当前 Database 定义**（`lib/database/database.dart`）
2. **当前 Table 定义**（`lib/database/tables.dart`）
3. **当前 Schema 版本号**（`schemaVersion` 值）
4. **变更需求**（新增什么字段/表、改什么约束）

---

## 💬 提示词模板

```markdown
我需要对现有 Drift 数据库进行 Schema 迁移。

**当前状态**：
- Database 类：[Database 类名，如 AppDatabase]
- 当前 Schema 版本：[当前 schemaVersion 值，如 2]
- 涉及的表：[表名，如 Users]

**变更需求**：
[详细描述，例如：]
- 在 Users 表新增 `phoneNumber` 字段（字符串，可选）
- 新增 `Orders` 表，字段包括：id（主键）、userId（外键关联 Users）、amount（浮点）、status（枚举：pending/completed）、createdAt（DateTime）

**要求**：

1. **修改 Table 定义**：
   - 更新相关 Table 类（新增列或新建 Table）
   - 确保列定义完整（类型、约束、默认值）

2. **升级 Schema 版本**：
   - 将 `schemaVersion` 递增 1（如从 2 → 3）

3. **编写 MigrationStrategy**：
   - 在 `onUpgrade` 中处理从旧版本到新版本的迁移
   - 使用 `Migrator` API（`addColumn`、`createTable`、`customStatement`）
   - 按版本号分段处理（if (from < 3) { ... }）
   - 如果新增非空列，考虑数据回填

4. **迁移风险检查**：
   - 是否会丢失数据？
   - 是否需要数据迁移/转换？
   - 旧版本 App 打开新数据库会怎样？（提示升级）

5. **测试建议**：
   - 如何在测试中模拟从旧版本升级
   - 验证数据完整性

生成完整修改后的 `tables.dart` 和 `database.dart`，标注变更部分，不要省略现有代码。
```

---

## ✅ 验证步骤

生成后执行：

```bash
# 1. 重新生成 .g.dart
dart run build_runner build --delete-conflicting-outputs

# 2. 编译检查
flutter analyze lib/database/

# 3. 卸载 App 重装（测试全新安装）
flutter run

# 4. 从旧版本升级测试
# 方法 A: 用旧版 App 安装，再用新版覆盖安装
# 方法 B: 写单元测试模拟迁移（见下方示例）
```

**检查清单**：
- [ ] `schemaVersion` 已递增
- [ ] `onUpgrade` 包含新的迁移逻辑
- [ ] 新列/表在代码中可正常访问
- [ ] 旧用户升级后数据未丢失
- [ ] 新用户安装后数据库正常创建

---

## 🛠️ 常见场景

### 场景 1: 新增可选列

```dart
// 1. 修改 Table
class Users extends Table {
  IntColumn get id => integer().autoIncrement()();
  TextColumn get name => text()();
  TextColumn get email => text().unique()();
  TextColumn get phoneNumber => text().nullable()(); // 新增
  DateTimeColumn get createdAt => dateTime()();
}

// 2. 升级版本
@DriftDatabase(tables: [Users])
class AppDatabase extends _$AppDatabase {
  @override
  int get schemaVersion => 3; // 从 2 升到 3

  @override
  MigrationStrategy get migration {
    return MigrationStrategy(
      onUpgrade: (Migrator m, int from, int to) async {
        // 保留所有历史迁移
        if (from < 2) {
          // v1 → v2 的迁移（已有的，不要删除）
          await m.addColumn(users, users.avatarUrl);
        }
        
        if (from < 3) {
          // v2 → v3 的迁移（新增）
          await m.addColumn(users, users.phoneNumber);
        }
      },
    );
  }
}
```

---

### 场景 2: 新增非空列（需回填默认值）

```dart
// 1. 修改 Table（先设为可选，迁移后再改）
class Products extends Table {
  IntColumn get id => integer().autoIncrement()();
  TextColumn get name => text()();
  IntColumn get stockCount => integer().nullable()(); // 新增，先允许 null
}

// 2. 迁移脚本
@override
MigrationStrategy get migration {
  return MigrationStrategy(
    onUpgrade: (m, from, to) async {
      if (from < 4) {
        // 添加可选列
        await m.addColumn(products, products.stockCount);
        
        // 回填默认值
        await customStatement(
          'UPDATE products SET stock_count = ? WHERE stock_count IS NULL',
          [0],
        );
        
        // 可选：之后改为非空（需重建表，见场景 3）
      }
    },
  );
}
```

---

### 场景 3: 删除列或修改列类型（需重建表）

**注意**：SQLite 不支持 DROP COLUMN 和 ALTER COLUMN TYPE，必须重建表。

```dart
// 假设要删除 Users 表的 avatarUrl 列

@override
MigrationStrategy get migration {
  return MigrationStrategy(
    onUpgrade: (m, from, to) async {
      if (from < 5) {
        // 1. 创建临时表（只包含需要的列）
        await customStatement('''
          CREATE TABLE users_new (
            id INTEGER PRIMARY KEY AUTOINCREMENT,
            name TEXT NOT NULL,
            email TEXT UNIQUE NOT NULL,
            phone_number TEXT,
            created_at INTEGER NOT NULL
          )
        ''');
        
        // 2. 复制数据
        await customStatement('''
          INSERT INTO users_new (id, name, email, phone_number, created_at)
          SELECT id, name, email, phone_number, created_at FROM users
        ''');
        
        // 3. 删除旧表
        await customStatement('DROP TABLE users');
        
        // 4. 重命名新表
        await customStatement('ALTER TABLE users_new RENAME TO users');
      }
    },
  );
}
```

---

### 场景 4: 新建表

```dart
// 1. 新建 Table 定义
class Orders extends Table {
  IntColumn get id => integer().autoIncrement()();
  IntColumn get userId => integer()();
  RealColumn get amount => real()();
  TextColumn get status => text()();
  DateTimeColumn get createdAt => dateTime()();
}

// 2. 添加到 Database
@DriftDatabase(tables: [Users, Orders]) // 新增 Orders
class AppDatabase extends _$AppDatabase {
  @override
  int get schemaVersion => 6;

  @override
  MigrationStrategy get migration {
    return MigrationStrategy(
      onUpgrade: (m, from, to) async {
        // ... 保留之前所有迁移 ...
        
        if (from < 6) {
          // 创建新表
          await m.createTable(orders);
        }
      },
    );
  }
}
```

---

### 场景 5: 添加索引

```dart
// 在迁移中添加索引
if (from < 7) {
  await customStatement(
    'CREATE INDEX idx_users_email ON users(email)',
  );
  
  // 或复合索引
  await customStatement(
    'CREATE INDEX idx_orders_user_status ON orders(user_id, status)',
  );
}
```

---

## 🧪 测试迁移

### 方法 1: 单元测试模拟升级

```dart
// test/database/migration_test.dart
import 'package:drift/drift.dart';
import 'package:drift/native.dart';
import 'package:drift_dev/api/migrations.dart';
import 'package:flutter_test/flutter_test.dart';
import 'package:my_app/database/database.dart';

void main() {
  late SchemaVerifier verifier;

  setUpAll(() {
    verifier = SchemaVerifier(GeneratedHelper());
  });

  test('migration from v2 to v3 adds phoneNumber', () async {
    // 1. 创建 v2 数据库
    final connection = await verifier.startAt(2);
    final db = AppDatabase.forTesting(connection);

    // 2. 插入测试数据
    await db.into(db.users).insert(UsersCompanion(
      name: Value('Alice'),
      email: Value('alice@example.com'),
      createdAt: Value(DateTime.now()),
    ));

    // 3. 升级到 v3
    final migratedDb = AppDatabase.forTesting(
      await verifier.migrateAndValidate(connection, 3),
    );

    // 4. 验证数据完整性
    final users = await migratedDb.select(migratedDb.users).get();
    expect(users.length, 1);
    expect(users.first.name, 'Alice');
    expect(users.first.phoneNumber, isNull); // 新列应为 null

    await migratedDb.close();
  });
}

// 辅助类（需在 database.dart 中添加）
extension AppDatabaseTest on AppDatabase {
  static AppDatabase forTesting(QueryExecutor e) {
    return AppDatabase._internal(e);
  }
}
```

### 方法 2: 真机测试

```bash
# 1. 安装旧版本 App（schemaVersion = 2）
flutter run

# 2. 添加一些测试数据

# 3. 更新代码（schemaVersion = 3 + 迁移脚本）

# 4. 热重启（不卸载 App）
r  # 在 flutter run 终端按 r

# 5. 验证数据未丢失且新列可用
```

---

## ⚠️ 风险清单

完成迁移前检查：

- [ ] **版本号递增**：`schemaVersion` 是否加 1？
- [ ] **历史迁移保留**：旧的 `if (from < N)` 是否都还在？
- [ ] **数据丢失**：重建表时是否正确复制了所有数据？
- [ ] **非空约束**：新增非空列是否回填了默认值？
- [ ] **测试覆盖**：是否写了迁移测试或在真机验证？
- [ ] **外键约束**：新增外键是否在旧数据中合法？
- [ ] **索引重建**：重建表后是否重建了索引？

---

## 🛡️ 最佳实践

1. **迁移脚本只增不减**
   - 保留所有 `if (from < N)` 分支
   - 新用户会从 v1 依次升到最新版

2. **先在测试环境验证**
   - 用单元测试或测试设备验证迁移
   - 确认数据完整性再发布

3. **添加迁移日志**
   ```dart
   if (from < 3) {
     print('[Migration] Upgrading from v$from to v3: adding phoneNumber');
     await m.addColumn(users, users.phoneNumber);
   }
   ```

4. **备份关键数据**
   - 提供导出功能（导出 JSON 或 CSV）
   - 迁移失败时可恢复

5. **版本跨度大的迁移**
   - 如果跨多个版本（如 v1 → v5），考虑重建数据库
   - 提示用户备份或云端同步

---

## 🔗 相关

- 工作流：[workflows/drift-local-database.md](../../workflows/drift-local-database.md)
- Schema 生成：[generate-drift-schema.md](generate-drift-schema.md)
- Drift 迁移文档：[https://drift.simonbinder.eu/docs/advanced-features/migrations/](https://drift.simonbinder.eu/docs/advanced-features/migrations/)
