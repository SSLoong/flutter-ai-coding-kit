# 提示词：修复 isolate 相关错误

**用途**：根据编译或运行时报错信息，修复 isolate 代码中的常见问题

---

## 📋 适用场景

- 代码使用 isolate 后出现编译错误
- 运行时报 "Invalid argument" 或序列化错误
- Isolate 中调用 Flutter API 失败
- 性能未改善或反而变慢

---

## 📥 输入要求

在 Cursor 中准备：

1. **报错信息**（完整的错误堆栈）
2. **问题代码**（涉及 isolate 的源文件）
3. **复现步骤**（可选，如何触发错误）

---

## 💬 提示词模板

```markdown
我在使用 isolate 时遇到以下错误，请帮我诊断并修复。

**错误信息**：

```
[粘贴完整的错误堆栈，包括错误类型、消息、堆栈跟踪]
```

**问题代码**（位置：`[文件路径]`）：

```dart
[粘贴相关代码]
```

**触发场景**：
- [如何触发错误，如：点击按钮 → 调用 isolate 函数 → 报错]

**要求**：

1. **诊断问题**：
   - 错误的根本原因是什么？
   - 是编译错误还是运行时错误？
   - 涉及哪些 isolate 限制或规则？

2. **提供修复方案**：
   - 给出修复后的完整代码（不要省略 import）
   - 说明 Before/After 的改动点
   - 解释为什么这样修复

3. **预防措施**：
   - 如何避免类似错误？
   - 相关的最佳实践？

---

**常见错误场景**（选择匹配的场景，或描述你的问题）：

### 场景 1: Invalid argument - 对象不可发送

**错误信息**：
```
Invalid argument(s): Illegal argument in isolate message: object is not sendable
```

**可能原因**：
- 传递了闭包、Flutter UI 对象、平台通道对象
- 传递了 `Future` 或 `Stream`（应先 await）

---

### 场景 2: 无法访问 Flutter 插件

**错误信息**：
```
MissingPluginException(No implementation found for method ...)
```

**可能原因**：
- 在 isolate 中直接调用插件，未注册 `BackgroundIsolateBinaryMessenger`
- 或该插件不支持后台 isolate

---

### 场景 3: rootBundle 不可用

**错误信息**：
```
Null check operator used on a null value
```

**代码**：
```dart
await Isolate.run(() async {
  final data = await rootBundle.loadString('assets/data.json');
});
```

**可能原因**：
- `rootBundle` 只在主 isolate 可用

---

### 场景 4: BuildContext 相关错误

**错误信息**：
```
Invalid argument(s): object is not sendable (BuildContext)
```

**代码**：
```dart
await Isolate.run(() {
  Navigator.of(context).push(...); // 试图在 isolate 中操作 UI
});
```

**可能原因**：
- UI 操作必须在主 isolate 执行

---

### 场景 5: compute 报错 "callback must be a top-level function"

**错误信息**：
```
Invalid argument(s): Illegal argument in isolate message: (object is a closure)
```

**代码**：
```dart
class MyService {
  Future<int> process(int value) {
    return compute(_helper, value); // _helper 是实例方法
  }
  
  int _helper(int value) => value * 2;
}
```

**可能原因**：
- `compute` 要求顶层函数或静态方法

---

### 场景 6: 性能未改善或更慢

**问题**：
- 移到 isolate 后，性能反而变差

**可能原因**：
- 任务太轻（<5ms），isolate 启动开销更大
- 频繁创建 isolate（应复用长生命周期 isolate）
- 传递了大量数据，拷贝开销大

---

### 场景 7: 循环依赖或状态丢失

**问题**：
- 在 isolate 中修改了变量，主 isolate 看不到变化

**代码**：
```dart
List<int> data = [1, 2, 3];
await Isolate.run(() {
  data.add(4); // 修改了副本，不会影响主 isolate
});
print(data.length); // 仍然是 3
```

**可能原因**：
- Isolate 是内存隔离的，修改不会反映到主 isolate

---

### 场景 8: Isolate.spawn 报错

**错误信息**：
```
Unhandled exception: Invalid argument(s): Illegal argument in isolate message
```

**代码**：
```dart
await Isolate.spawn(_worker, this); // 试图传递 this
```

**可能原因**：
- 传递了不可序列化的对象

---

### 场景 9: Web 平台不工作

**问题**：
- 代码在移动端正常，web 端卡顿依旧

**可能原因**：
- Web 平台不支持真正的 isolate，`Isolate.run` 退化为同步执行

---

### 场景 10: 测试中 isolate 报错

**错误信息**：
```
Bad state: No test is currently running
```

**可能原因**：
- 测试中使用 isolate 需要特殊处理

```

---

## ✅ 验证步骤

修复后执行：

```bash
# 1. 清理构建缓存
flutter clean

# 2. 重新安装依赖
flutter pub get

# 3. 检查编译
flutter analyze

# 4. 运行应用测试
flutter run --profile

# 5. 如有单元测试，执行测试
flutter test
```

**检查清单**：
- [ ] 错误不再出现
- [ ] 功能正常（数据正确）
- [ ] 性能有改善（用 Timeline 验证）
- [ ] 无新增 bug

---

## 🛠️ 常见修复模式

### 修复 1: 对象不可发送 → 只传数据

```dart
// ❌ 错误
final file = File(path);
await Isolate.run(() => process(file));

// ✅ 修复
final bytes = await File(path).readAsBytes();
await Isolate.run(() => process(bytes));
```

---

### 修复 2: 闭包捕获外部变量 → 顶层函数

```dart
// ❌ 错误
final threshold = 100;
await Isolate.run(() => data.where((x) => x > threshold).toList());

// ✅ 修复
await Isolate.run(() => _filter(data, 100));

List<int> _filter(List<int> data, int t) => data.where((x) => x > t).toList();
```

---

### 修复 3: compute 实例方法 → 顶层函数

```dart
// ❌ 错误
class Service {
  Future<int> calc(int x) => compute(_helper, x);
  int _helper(int x) => x * 2; // 实例方法
}

// ✅ 修复
class Service {
  Future<int> calc(int x) => compute(_helperFunc, x);
}

int _helperFunc(int x) => x * 2; // 顶层函数
```

---

### 修复 4: rootBundle 访问 → 主 isolate 加载

```dart
// ❌ 错误
await Isolate.run(() async {
  final json = await rootBundle.loadString('assets/data.json');
  return jsonDecode(json);
});

// ✅ 修复
final jsonString = await rootBundle.loadString('assets/data.json');
await Isolate.run(() => jsonDecode(jsonString));
```

---

### 修复 5: UI 操作 → 返回结果到主 isolate

```dart
// ❌ 错误
await Isolate.run(() {
  Navigator.push(...); // UI 操作不能在 isolate
});

// ✅ 修复
final result = await Isolate.run(() => compute());
// 回到主 isolate 后再操作 UI
Navigator.push(...);
```

---

### 修复 6: 插件调用 → 注册 BackgroundIsolateBinaryMessenger

```dart
// ❌ 错误
await Isolate.run(() async {
  final prefs = await SharedPreferences.getInstance(); // 报错
});

// ✅ 修复
final token = RootIsolateToken.instance!;
await Isolate.run(() async {
  BackgroundIsolateBinaryMessenger.ensureInitialized(token);
  final prefs = await SharedPreferences.getInstance(); // 现在可以用了
});
```

---

### 修复 7: 任务太轻 → 移除 isolate

```dart
// ❌ 过度优化：任务耗时 <1ms，isolate 启动反而慢
await Isolate.run(() => simpleCalc(x));

// ✅ 修复：轻量级计算直接执行
final result = simpleCalc(x);
```

---

### 修复 8: 频繁创建 isolate → 批量处理

```dart
// ❌ 低效
for (final item in items) {
  await Isolate.run(() => process(item)); // 每次 +1-5ms 开销
}

// ✅ 修复：批量处理
final results = await Isolate.run(() => items.map(process).toList());
```

---

### 修复 9: 状态共享 → 返回结果

```dart
// ❌ 错误：修改不会反映到主 isolate
List<int> data = [1, 2, 3];
await Isolate.run(() => data.add(4));
print(data.length); // 仍然是 3

// ✅ 修复：返回新数据
data = await Isolate.run(() {
  final newData = List<int>.from(data);
  newData.add(4);
  return newData;
});
print(data.length); // 现在是 4
```

---

### 修复 10: compute 多参数 → Map 包装

```dart
// ❌ 错误：语法错误
await compute(_process, arg1, arg2);

// ✅ 修复
await compute(_process, {'arg1': arg1, 'arg2': arg2});

Result _process(Map<String, dynamic> params) {
  final arg1 = params['arg1'];
  final arg2 = params['arg2'];
  // ...
}
```

---

## 🎯 诊断决策树

```
错误是编译时还是运行时？
  ├─ 编译时（syntax error, type error）
  │   └─ 检查：函数签名、import、闭包捕获
  │
  └─ 运行时
      ├─ "Invalid argument" / "not sendable"
      │   └─ 检查：传递的对象是否可序列化
      │
      ├─ "MissingPluginException"
      │   └─ 检查：是否需要 BackgroundIsolateBinaryMessenger
      │
      ├─ "Null check operator"
      │   └─ 检查：是否访问了 rootBundle / Flutter 绑定
      │
      └─ 性能未改善
          ├─ 任务太轻？ → 移除 isolate
          ├─ 频繁创建？ → 批量处理或长生命周期 isolate
          └─ 数据拷贝大？ → 减少传递数据量
```

---

## 📚 错误速查表

| 错误关键词 | 常见原因 | 修复方向 |
|-----------|---------|---------|
| `Invalid argument` / `not sendable` | 传递了闭包、UI 对象、Future | 只传基本类型、List、Map |
| `MissingPluginException` | 插件未在 isolate 中注册 | 用 `BackgroundIsolateBinaryMessenger` |
| `Null check operator` on rootBundle | 访问了主 isolate 专属 API | 先在主 isolate 加载资源 |
| `object is a closure` | `compute` 收到实例方法/闭包 | 改为顶层函数或静态方法 |
| 性能无改善或更慢 | 任务太轻或频繁创建 isolate | 移除 isolate 或批量处理 |
| 状态未更新 | isolate 修改了副本 | 返回结果到主 isolate |
| Web 平台卡顿依旧 | Web 不支持真正 isolate | 接受限制或用 Web Worker |

---

## 🔗 相关

- 工作流：[workflows/isolate-background-compute.md](../../workflows/isolate-background-compute.md)
- 提取任务：[extract-work-to-isolate.md](extract-work-to-isolate.md)
- Flutter 文档：[Concurrency and isolates](https://docs.flutter.dev/perf/isolates)
- Dart 文档：[SendPort.send](https://api.dart.dev/stable/dart-isolate/SendPort/send.html)
