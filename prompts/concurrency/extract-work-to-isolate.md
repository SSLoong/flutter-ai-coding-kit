# 提示词：从现有代码提取耗时任务到 isolate

**用途**：识别并提取现有代码中可以移出 UI isolate 的耗时计算，生成可运行的 isolate 实现

---

## 📋 适用场景

- 已有代码因耗时计算导致 UI 卡顿
- DevTools Timeline 显示 UI 线程有长耗时事件
- 需要把 CPU 密集任务移到后台 isolate

---

## 📥 输入要求

在 Cursor 中准备：

1. **问题代码文件**（包含耗时操作的源文件）
2. **Timeline 数据**（可选，用于量化耗时）
3. **调用上下文**（代码如何被调用）

---

## 💬 提示词模板

```markdown
我需要把以下耗时计算移到后台 isolate，避免堵住 UI。

**现有代码**（位置：`[文件路径]`）：

```dart
[粘贴现有代码]
```

**问题描述**：
- 耗时操作：[具体是什么计算，如 JSON 解析、图片压缩]
- 预估耗时：[如有 Timeline 数据，说明耗时，如 200-500ms]
- 调用场景：[何时触发，如页面加载、按钮点击]

**要求**：

### 1. 选择合适的 isolate 方案

根据场景选择：
- **一次性计算**：用 `Isolate.run`（推荐）
- **需要 web 兼容**：用 `compute`
- **高频重复计算**：用长生命周期 isolate（`Isolate.spawn` + SendPort/ReceivePort）

**如不确定，默认用 `Isolate.run`。**

### 2. 提取可移动的计算

识别哪些代码可以移到 isolate：
- ✅ **可移动**：纯计算逻辑、数据转换、加密解密
- ❌ **不可移动**：访问 `BuildContext`、调用 `setState()`、访问 `rootBundle`

**只移动必要的计算部分，其他保持不变。**

### 3. 处理数据传递

确保传递的数据是**可发送的**：
- ✅ 可发送：`String`、`int`、`List`、`Map`、`Uint8List`
- ❌ 不可发送：闭包、Flutter UI 对象、平台通道对象

**如果参数复杂，用 Map 或自定义类包装。**

### 4. 生成完整代码

输出：
1. **修改后的完整文件代码**（包含所有必要的 import）
2. **Before/After 对比**（说明改动点）
3. **调用方式**（其他代码是否需要修改）
4. **注意事项**（如数据可发送性、web 兼容性）

**代码必须可以直接编译运行，不要省略 import 或类定义。**

---

**示例场景 1：JSON 解析**

```dart
// 现有代码（lib/services/data_service.dart）
Future<List<Route>> loadRoutes() async {
  final jsonString = await rootBundle.loadString('assets/routes.json');
  final List<dynamic> data = jsonDecode(jsonString); // ⚠️ 耗时 200-500ms
  return data.map((json) => Route.fromJson(json)).toList();
}
```

**要求**：
- 把 `jsonDecode` 和 `map` 移到 isolate
- 保持函数签名不变
- 使用 `Isolate.run`

---

**示例场景 2：图像压缩**

```dart
// 现有代码（lib/utils/image_helper.dart）
import 'package:image/image.dart' as img;

Future<Uint8List> compressImage(Uint8List imageBytes) async {
  final image = img.decodeImage(imageBytes); // ⚠️ 耗时 100-300ms
  if (image == null) throw ArgumentError('Invalid image');
  return Uint8List.fromList(img.encodeJpg(image, quality: 85)); // ⚠️ 耗时 50-200ms
}
```

**要求**：
- 把整个图像处理移到 isolate
- 使用 `compute` 确保 web 兼容
- 需要提取为顶层函数

---

**示例场景 3：数据库批量查询**

```dart
// 现有代码（lib/database/user_dao.dart）
Future<List<User>> searchUsers(String keyword) async {
  final db = await database;
  final List<Map<String, dynamic>> maps = await db.query(
    'users',
    where: 'name LIKE ?',
    whereArgs: ['%$keyword%'],
  ); // 查询很快
  
  // ⚠️ 耗时操作：复杂过滤和排序
  return maps
      .map((map) => User.fromJson(map))
      .where((user) => _complexFilter(user, keyword))
      .toList()
    ..sort((a, b) => _complexSort(a, b)); // 数万条数据时可能卡顿
}
```

**要求**：
- 数据库查询保持在主 isolate（需要数据库实例）
- 把 `map` + `where` + `sort` 移到 isolate
- 传递查询结果（List<Map>）到 isolate

---

**示例场景 4：批量任务**

```dart
// 现有代码（lib/services/image_service.dart）
Future<void> compressAllImages(List<String> imagePaths) async {
  for (final path in imagePaths) {
    final bytes = await File(path).readAsBytes();
    final compressed = await _compressImage(bytes); // ⚠️ 每张图耗时 100ms
    await File(path).writeAsBytes(compressed);
  }
}
```

**要求**：
- 文件读写保持在主 isolate
- 只把 `_compressImage` 移到 isolate
- 是否需要长生命周期 isolate？（如果图片很多，可以考虑复用 isolate）

```

---

## ✅ 验证步骤

AI 生成代码后执行：

```bash
# 1. 检查编译
flutter analyze

# 2. 运行应用
flutter run --profile

# 3. 用 DevTools 验证
# - 打开 Performance 面板
# - 录制 Timeline
# - 执行之前卡顿的操作
# - 确认 UI 线程不再有长耗时事件
```

**检查清单**：
- [ ] 代码能编译通过（无类型错误、无 import 缺失）
- [ ] 功能正常（数据正确、无业务逻辑错误）
- [ ] Performance Timeline 显示 UI 线程不再堵塞
- [ ] 测试覆盖关键路径（单元测试或手动测试）

---

## 🛠️ 常见调整

### 1. 闭包捕获外部变量

AI 可能生成捕获外部变量的闭包：

```dart
// ❌ 错误：闭包捕获 threshold
final threshold = 100;
await Isolate.run(() => data.where((x) => x > threshold).toList());
```

**修复**：改为顶层函数 + 显式参数

```dart
// ✅ 正确
await Isolate.run(() => _filterData(data, 100));

List<int> _filterData(List<int> data, int threshold) {
  return data.where((x) => x > threshold).toList();
}
```

---

### 2. `compute` 需要顶层函数

```dart
// ❌ 错误：实例方法不能传给 compute
class ImageService {
  Future<Uint8List> compress(Uint8List bytes) {
    return compute(_compress, bytes); // _compress 是实例方法，报错
  }
  
  Uint8List _compress(Uint8List bytes) { ... }
}
```

**修复**：改为顶层函数或静态方法

```dart
// ✅ 正确
class ImageService {
  Future<Uint8List> compress(Uint8List bytes) {
    return compute(_compressImage, bytes); // 顶层函数
  }
}

Uint8List _compressImage(Uint8List bytes) { ... }
```

---

### 3. 不可发送的对象

```dart
// ❌ 错误：File 对象不可发送
final file = File(path);
await Isolate.run(() => processFile(file)); // 报错
```

**修复**：只传递文件路径或字节数据

```dart
// ✅ 正确
final bytes = await File(path).readAsBytes();
await Isolate.run(() => processBytes(bytes));
```

---

### 4. 访问 rootBundle

```dart
// ❌ 错误：isolate 中无法访问 rootBundle
await Isolate.run(() async {
  final json = await rootBundle.loadString('assets/data.json'); // 报错
  return jsonDecode(json);
});
```

**修复**：先在主 isolate 加载

```dart
// ✅ 正确
final jsonString = await rootBundle.loadString('assets/data.json');
await Isolate.run(() => jsonDecode(jsonString));
```

---

### 5. 多参数传递

`compute` 只接受一个参数：

```dart
// ❌ 错误：compute 不支持多参数
await compute(_process, data, threshold); // 语法错误
```

**修复**：用 Map 或自定义类包装

```dart
// ✅ 正确
await compute(_process, {'data': data, 'threshold': threshold});

Result _process(Map<String, dynamic> params) {
  final data = params['data'] as List;
  final threshold = params['threshold'] as int;
  // ...
}
```

---

### 6. 异常处理

```dart
// 确保异常能传回主 isolate
try {
  final result = await Isolate.run(() {
    if (invalid) throw Exception('Invalid data');
    return process();
  });
} catch (e) {
  // isolate 中的异常会在这里捕获
  print('Error: $e');
}
```

---

## 🎯 输出质量标准

AI 生成的代码必须满足：

1. **完整性**：包含所有 import、类定义、方法签名
2. **可编译**：`flutter analyze` 无错误
3. **可运行**：功能正确，数据准确
4. **可验证**：说明如何用 Timeline 验证效果
5. **有注释**：关键改动点有注释说明

**检查方法**：
- 把生成的代码粘贴到项目中
- 运行 `flutter analyze`
- 执行 `flutter test`（如有测试）
- Profile 模式下测试性能

---

## 🔗 相关

- 工作流：[workflows/isolate-background-compute.md](../../workflows/isolate-background-compute.md)
- 修复报错：[fix-isolate-errors.md](fix-isolate-errors.md)
- 性能分析：[workflows/performance-timeline.md](../../workflows/performance-timeline.md)
- Flutter 文档：[Concurrency and isolates](https://docs.flutter.dev/perf/isolates)
