# Isolate 与后台计算工作流

**用 AI 把耗时任务移出 UI isolate,保持应用流畅**

> 作者：Loong (SSLoong) | 日期：2026-11-20

---

## 📋 背景

Dart 应用默认运行在**单个 isolate**（类似线程但内存隔离）的事件循环中。当耗时计算（如 JSON 解析、图像处理、数据库查询）执行时间超过一帧间隔（16ms / 60fps），就会堵住主 isolate 导致 UI 卡顿。

**Isolate** 是 Dart 的并发模型，允许你在独立内存空间中并行执行代码，不会阻塞 UI 渲染。与 `async/await` 的区别：
- **`async/await`**：异步但仍在主 isolate 执行，不能把 CPU 密集计算移出 UI 线程
- **Isolate**：真正并行，在独立线程中运行，利用多核 CPU

**常见误区**：
- ❌ "我用了 `async/await`，性能应该没问题" → 异步不等于多线程，CPU 密集任务仍会堵 UI
- ❌ "所有耗时操作都要用 isolate" → 轻量级异步（如网络请求）用 `async` 即可
- ✅ "大数据解析、图像压缩等计算密集任务才需要 isolate"

---

## 🎯 目标

- ✅ 识别哪些任务需要移出 UI isolate
- ✅ 用 `Isolate.run` / `compute` 处理一次性耗时计算
- ✅ 理解 isolate 间消息传递与数据可发送性
- ✅ 在后台 isolate 中安全使用 Flutter 插件（可选）
- ✅ 用 DevTools Performance 验证 UI isolate 不再被堵塞

---

## 🔧 适用场景

| 场景 | 适合度 | 说明 |
|------|--------|------|
| JSON 解析（大文件，>1MB） | ⭐⭐⭐⭐⭐ | 数据结构复杂时 jsonDecode 可能耗时数百毫秒 |
| 图像缩放/压缩/滤镜 | ⭐⭐⭐⭐⭐ | 图像处理是典型 CPU 密集任务 |
| 数据库批量查询/计算 | ⭐⭐⭐⭐ | 大量数据聚合计算 |
| 加密/解密数据 | ⭐⭐⭐⭐⭐ | 密码学运算通常很耗 CPU |
| 复杂列表过滤/排序 | ⭐⭐⭐⭐ | 数万条数据的多条件筛选 |
| 文件解压缩/编码转换 | ⭐⭐⭐⭐⭐ | 压缩算法计算密集 |
| 轻量级异步（<5ms） | ⭐ | isolate 启动本身有开销（1-5ms），反而更慢 |
| 需要访问 BuildContext | ⭐ | isolate 无法访问 Flutter UI 对象 |
| 网络请求（I/O 等待） | ⭐ | I/O 操作用 `async/await` 即可，不需要 isolate |

---

## 🚀 完整流程

### Step 0: 识别性能瓶颈（必读）

**在优化前先确认问题**：用 DevTools Performance 采集 Timeline 数据（参考 [performance-timeline 工作流](performance-timeline.md)），确认：

- [ ] 问题出在 **UI 线程**（不是 Raster 线程）
- [ ] 存在单帧 Build/Event Handler 耗时 **>16ms** 的情况
- [ ] 耗时操作是 **CPU 计算**（不是网络等待或磁盘 I/O）

**反例**：如果 Timeline 显示网络请求回调慢，问题在服务器响应时间，不应该用 isolate。

---

### Step 1: 用 `Isolate.run` 移出简单计算（推荐）

**适用场景**：一次性、无状态的耗时计算（如解析 JSON、处理图片）。

**AI 提示词**（可复制，提示词模板：[prompts/concurrency/extract-work-to-isolate.md](../prompts/concurrency/extract-work-to-isolate.md)）：

```markdown
我需要把以下耗时计算移到后台 isolate，避免堵住 UI。

**现有代码**（位置：`lib/services/data_service.dart`）：

```dart
Future<List<Route>> parseRoutesFromJson(String jsonString) async {
  // 这个 jsonDecode 可能耗时 200-500ms（大文件）
  final List<dynamic> data = jsonDecode(jsonString);
  return data.map((json) => Route.fromJson(json)).toList();
}
```

**要求**：

1. 用 `Isolate.run` 把 JSON 解析移到后台 isolate
2. 保持函数签名和调用方式不变（其他代码无需改动）
3. 确保传递的数据是可发送的（sendable）
4. 生成完整代码，包含所有必要的 import

**输出格式**：
- 修改后的完整文件代码
- 改动说明（Before/After）
- 数据传递的注意事项
```

**输出示例**（参考）：

```dart
import 'dart:convert';
import 'dart:isolate';
import 'package:flutter/foundation.dart';

import '../models/route.dart';

// ✅ 优化后：JSON 解析在后台 isolate 执行
Future<List<Route>> parseRoutesFromJson(String jsonString) async {
  // Isolate.run 会：
  // 1. 创建新 isolate
  // 2. 在其中执行回调
  // 3. 返回结果并关闭 isolate
  return await Isolate.run(() {
    final List<dynamic> data = jsonDecode(jsonString);
    return data.map((json) => Route.fromJson(json as Map<String, dynamic>)).toList();
  });
}

// ❌ 优化前：在主 isolate 执行，堵 UI
// Future<List<Route>> parseRoutesFromJson(String jsonString) async {
//   final List<dynamic> data = jsonDecode(jsonString);
//   return data.map((json) => Route.fromJson(json)).toList();
// }
```

**关键改动**：

| Before | After | 说明 |
|--------|-------|------|
| `jsonDecode(jsonString)` 直接执行 | `Isolate.run(() => jsonDecode(...))` | 在后台 isolate 解析 |
| 堵住 UI 200-500ms | UI 保持流畅 | 主 isolate 继续渲染帧 |

**数据传递规则**：
- ✅ **可发送**：基本类型（String、int、bool）、List、Map、自定义可序列化类（如 `Route.fromJson`）
- ❌ **不可发送**：闭包捕获的外部变量、`BuildContext`、`Image`、`TextEditingController` 等 Flutter UI 对象

**Web 平台注意**：Flutter web 不支持真正的 isolate，`Isolate.run` 会退化为在主线程执行，但代码仍能编译。

---

### Step 2: 使用 `compute` 函数（兼容性更好）

**`compute` vs `Isolate.run`**：
- **`Isolate.run`**：Dart 3.0+ 原生 API，适合 Flutter 3.7+
- **`compute`**：Flutter 提供的封装，底层使用 `Isolate.run`（移动端）或主线程执行（web 端）
- **推荐**：新项目用 `Isolate.run`，需兼容旧版本或确保 web 编译通过用 `compute`

**AI 提示词**（可复制）：

```markdown
将以下代码改为用 `compute` 函数（确保 web 平台兼容性）。

**现有代码**：

```dart
Future<Uint8List> compressImage(Uint8List imageBytes) async {
  return await Isolate.run(() {
    return img.encodeJpg(img.decodeImage(imageBytes)!, quality: 85);
  });
}
```

**要求**：
1. 改为用 `compute` 函数
2. 提取顶层函数（compute 要求回调是顶层函数或静态方法）
3. 保持功能不变

生成完整代码。
```

**输出示例**（参考）：

```dart
import 'dart:typed_data';
import 'package:flutter/foundation.dart';
import 'package:image/image.dart' as img;

// ✅ 必须是顶层函数或静态方法（compute 要求）
Uint8List _compressImageInIsolate(Uint8List imageBytes) {
  final image = img.decodeImage(imageBytes);
  if (image == null) throw ArgumentError('Invalid image');
  return Uint8List.fromList(img.encodeJpg(image, quality: 85));
}

Future<Uint8List> compressImage(Uint8List imageBytes) async {
  // compute 在移动端创建 isolate，在 web 端退化为同步执行
  return await compute(_compressImageInIsolate, imageBytes);
}
```

**关键点**：
- `compute` 要求回调是**顶层函数**或**静态方法**（不能是闭包或实例方法）
- 只能传递**一个参数**（如需多个，用 Map 或自定义类包装）
- 在 web 平台会**同步执行**，但代码能编译运行

---

### Step 3: 长生命周期 isolate（进阶）

**适用场景**：
- 需要反复执行同一类计算（避免重复创建 isolate 的开销）
- 需要在后台 isolate 中维护状态
- 需要双向通信（如上传进度回调）

**⚠️ 注意**：长生命周期 isolate 需要手动管理生命周期和消息传递，复杂度较高。**大多数场景用 `Isolate.run` 或 `compute` 即可**。

**AI 提示词**（可复制）：

```markdown
我需要创建一个长生命周期的后台 isolate，用于批量处理图片压缩任务。

**需求**：
1. 主 isolate 可以持续发送图片数据到 worker isolate
2. Worker isolate 处理完后返回结果
3. 应用关闭时销毁 worker isolate

**要求**：
1. 用 `Isolate.spawn` + `SendPort`/`ReceivePort` 实现双向通信
2. Worker isolate 可以接收多次任务
3. 提供启动、发送任务、关闭 worker 的接口
4. 生成完整可运行代码，包含注释说明消息传递流程

生成代码示例。
```

**输出示例**（参考，实际由 AI 生成）：

```dart
import 'dart:async';
import 'dart:isolate';
import 'dart:typed_data';
import 'package:image/image.dart' as img;

class ImageWorker {
  Isolate? _isolate;
  SendPort? _sendPort;
  final _resultController = StreamController<Uint8List>.broadcast();

  Stream<Uint8List> get results => _resultController.stream;

  // 启动 worker isolate
  Future<void> start() async {
    final receivePort = ReceivePort();
    
    // 创建 isolate
    _isolate = await Isolate.spawn(_workerMain, receivePort.sendPort);
    
    // 获取 worker 的 SendPort（用于发送任务）
    _sendPort = await receivePort.first as SendPort;
    
    // 监听 worker 返回的结果
    final resultPort = ReceivePort();
    _sendPort!.send(resultPort.sendPort);
    resultPort.listen((result) {
      _resultController.add(result as Uint8List);
    });
  }

  // 发送任务到 worker
  void compressImage(Uint8List imageBytes) {
    _sendPort?.send(imageBytes);
  }

  // 关闭 worker
  void dispose() {
    _isolate?.kill(priority: Isolate.immediate);
    _isolate = null;
    _resultController.close();
  }

  // Worker isolate 的入口函数
  static void _workerMain(SendPort mainSendPort) {
    final workerReceivePort = ReceivePort();
    mainSendPort.send(workerReceivePort.sendPort);

    SendPort? resultPort;

    workerReceivePort.listen((message) {
      if (message is SendPort) {
        // 首次接收：保存结果返回端口
        resultPort = message;
      } else if (message is Uint8List) {
        // 处理图片压缩任务
        final image = img.decodeImage(message);
        if (image != null) {
          final compressed = Uint8List.fromList(img.encodeJpg(image, quality: 85));
          resultPort?.send(compressed);
        }
      }
    });
  }
}

// 使用示例
void main() async {
  final worker = ImageWorker();
  await worker.start();

  // 监听结果
  worker.results.listen((compressed) {
    print('压缩完成，大小：${compressed.length} bytes');
  });

  // 发送任务
  worker.compressImage(imageBytes1);
  worker.compressImage(imageBytes2);

  // 应用关闭时
  worker.dispose();
}
```

**关键点**：
- `ReceivePort` / `SendPort`：类似 Stream 的双向通信
- Worker 必须是**顶层函数**或**静态方法**
- 手动管理 isolate 生命周期（启动、关闭）
- 适合高频重复计算，避免频繁创建 isolate 的开销

**推荐库**：如果需要管理多个 worker，可以考虑 [worker_manager](https://pub.dev/packages/worker_manager) 等第三方包简化开发。

---

### Step 4: 在后台 isolate 中使用 Flutter 插件（可选）

**适用场景**：需要在后台 isolate 中调用原生平台 API（如加密、存储、推送通知）。

**⚠️ 限制**：
- ✅ **支持**：调用插件方法并等待返回结果（如 `SharedPreferences.getInstance()`）
- ❌ **不支持**：接收平台主动推送的消息（如 Firestore 实时监听）
- ❌ **不支持**：访问 `rootBundle` 加载资源文件

**AI 提示词**（可复制）：

```markdown
我需要在后台 isolate 中使用 `shared_preferences` 插件读取配置。

**需求**：
- 在后台 isolate 中调用 `SharedPreferences.getInstance()`
- 读取配置后返回主 isolate

**要求**：
1. 用 `BackgroundIsolateBinaryMessenger` 注册后台 isolate
2. 从主 isolate 传递 `RootIsolateToken`
3. 生成完整可运行代码

生成代码示例。
```

**输出示例**（参考）：

```dart
import 'dart:isolate';
import 'package:flutter/services.dart';
import 'package:shared_preferences/shared_preferences.dart';

// 主 isolate：启动后台 isolate
Future<bool> checkDebugMode() async {
  final rootToken = RootIsolateToken.instance!;
  return await Isolate.run(() => _readDebugFlag(rootToken));
}

// 后台 isolate：读取配置
Future<bool> _readDebugFlag(RootIsolateToken rootToken) async {
  // ✅ 注册后台 isolate，使其能调用平台通道
  BackgroundIsolateBinaryMessenger.ensureInitialized(rootToken);

  // 现在可以使用插件了
  final prefs = await SharedPreferences.getInstance();
  return prefs.getBool('isDebug') ?? false;
}
```

**关键步骤**：
1. 在主 isolate 获取 `RootIsolateToken.instance`
2. 传递 token 到后台 isolate
3. 调用 `BackgroundIsolateBinaryMessenger.ensureInitialized(rootToken)`
4. 之后可以正常使用插件

**限制**：只能**发起请求**并接收响应，不能接收平台主动推送的事件。

---

### Step 5: 验证优化效果（必须！）

**人工操作**：用 DevTools Performance 验证 UI isolate 不再被堵塞。

```bash
# 运行 profile 模式
flutter run --profile
```

1. **打开 DevTools Performance 面板**
2. **录制 Timeline**：执行之前卡顿的操作（如加载大 JSON）
3. **对比优化前后**：
   - 优化前：UI 线程出现 200ms+ 的长耗时事件
   - 优化后：UI 线程保持 <16ms，没有明显卡顿帧

**检查清单**：
- [ ] 卡顿帧耗时降低至 <16ms（60fps）
- [ ] Timeline 中看不到原耗时操作在 UI 线程执行
- [ ] 用户操作流畅，无明显卡顿感
- [ ] 功能测试通过，数据正确

**常见问题**：
- ❌ 优化后更慢了 → 可能任务太轻（<5ms），isolate 启动开销反而更大
- ❌ 数据不正确 → 检查传递的对象是否可发送，是否被意外修改
- ❌ 报错 "Invalid argument" → 尝试发送了不可序列化的对象（如闭包、UI 对象）

**验证步骤可参考**：[performance-timeline 工作流](performance-timeline.md#step-4-验证优化效果必须)

---

## ✅ 验证清单

完成后确认：

- [ ] 已用 DevTools Performance 确认问题出在 UI 线程的 CPU 计算
- [ ] 耗时操作已移到 `Isolate.run` 或 `compute` 中
- [ ] 传递的数据是可发送的（不含闭包、UI 对象）
- [ ] Profile 模式下 Timeline 显示 UI 线程不再被堵塞
- [ ] 功能测试通过，无回归 bug
- [ ] 如使用插件，已正确注册 `BackgroundIsolateBinaryMessenger`

---

## 🤔 FAQ

### Q1: `async/await` 和 isolate 有什么区别？
**A**：
- **`async/await`**：**异步但单线程**，适合 I/O 等待（网络请求、文件读取）
  - 执行过程：发起请求 → 让出控制权 → 等待响应 → 继续执行
  - **不会**把 CPU 密集计算移出主线程
- **Isolate**：**真正并行**，独立线程执行，适合 CPU 密集计算
  - 在多核 CPU 上可以与主线程同时运行

**简单判断**：
- 任务在**等待外部响应**（网络、磁盘）→ 用 `async/await`
- 任务在**高强度计算**（解析、加密、图像处理）→ 用 isolate

---

### Q2: 何时用 `Isolate.run`，何时用 `Isolate.spawn`？
**A**：
- **`Isolate.run`**（推荐）：
  - 一次性计算，完成后自动销毁
  - API 简单，类似 `async/await`
  - 适合 95% 的场景
- **`Isolate.spawn`**（进阶）：
  - 长生命周期，需要多次通信
  - 手动管理 SendPort/ReceivePort
  - 适合高频重复计算（避免频繁创建 isolate 开销）

**建议**：先用 `Isolate.run`，除非性能测试证明需要长生命周期 isolate。

---

### Q3: 为什么报错 "Invalid argument: <对象> cannot be sent"？
**A**：isolate 间只能传递**可发送**的对象。

**可发送**：
- ✅ 基本类型：`int`、`double`、`bool`、`String`
- ✅ 集合：`List`、`Map`、`Set`（元素也必须可发送）
- ✅ `Uint8List`、`ByteData` 等字节数据
- ✅ 自定义类（如果所有字段都可发送）

**不可发送**：
- ❌ 闭包捕获外部变量（如 `() => externalVar`）
- ❌ Flutter UI 对象（`BuildContext`、`Image`、`TextEditingController`）
- ❌ 平台通道对象（未序列化的原生对象）
- ❌ `Future`、`Stream`（应先 await 得到结果再传递）

**解决方案**：
- 只传递数据（String、List、Map），不传递对象引用
- 自定义类要确保所有字段都是可发送的

---

### Q4: Isolate 启动开销有多大？
**A**：创建 isolate 本身需要 **1-5ms**，因此：
- ✅ 任务耗时 **>10ms**：用 isolate 有收益
- ❌ 任务耗时 **<5ms**：isolate 开销反而更大，不如直接执行

**如何判断**：
- 先用 Timeline 测量任务耗时
- 如果任务很轻，考虑批量处理或放弃 isolate

---

### Q5: 如何在 isolate 中访问资源文件（assets）？
**A**：❌ **不能**在后台 isolate 中用 `rootBundle.loadString()`。

**原因**：`rootBundle` 绑定到主 isolate 的 Flutter 引擎。

**解决方案**：
1. 在主 isolate 加载资源：
   ```dart
   final jsonString = await rootBundle.loadString('assets/data.json');
   ```
2. 传递**字符串或字节数据**到 isolate：
   ```dart
   final result = await Isolate.run(() => parseJson(jsonString));
   ```

---

### Q6: 可以在 isolate 中修改 UI 吗？
**A**：❌ **不能**。Isolate 无法访问 Flutter UI 对象。

**原因**：Flutter 的渲染引擎绑定到主 isolate。

**正确做法**：
1. 在后台 isolate 计算结果
2. 返回结果到主 isolate
3. 在主 isolate 更新 UI（setState / Riverpod Provider）

---

### Q7: Web 平台不支持 isolate 怎么办？
**A**：
- **`Isolate.run`**：在 web 上退化为**主线程同步执行**，代码能编译但不会真正并行
- **`compute`**：同样在 web 上同步执行
- **Web Workers**：Dart web 平台未来可能支持，当前需用 JS 互操作手动实现

**建议**：
- 如果必须支持 web 且性能敏感，考虑用 WebAssembly 或服务端计算
- 对大多数应用，web 端性能足够，移动端用 isolate 优化即可

---

### Q8: Isolate 间可以共享数据吗？
**A**：❌ **不能**。Isolate 是**内存隔离**的，无法共享可变状态。

**消息传递时会发生**：
- **拷贝**：可变对象（List、Map）被复制到另一个 isolate
- **引用**：不可变对象（String）直接传引用（性能优化）

**场景**：
- 如果在 isolate A 修改了 List，isolate B 不会看到变化
- 需要结果时，通过消息传递返回

这是 Actor 模型的设计，确保线程安全。

---

### Q9: 如何调试 isolate 中的代码？
**A**：
- **DevTools**：可以看到所有 isolate 的 Timeline 事件
- **断点**：在 VSCode / Android Studio 中可以在 isolate 代码打断点
- **日志**：用 `print()` / `debugPrint()` 输出（会显示在控制台）

**Timeline 中查看**：
- 每个 isolate 有独立的 Timeline 轨道
- 可以看到 isolate 启动、执行、销毁的时间

---

### Q10: 多个 isolate 能同时运行吗？
**A**：✅ **可以**。Dart VM 会根据 CPU 核心数调度。

**注意**：
- 创建过多 isolate（如 100 个）会有内存和调度开销
- 一般情况下 2-4 个 worker isolate 足够
- 可以用 `IsolateNameServer` 管理多个命名 isolate

---

## 💡 最佳实践

### 1. 优先考虑是否真的需要 isolate

**决策树**：
```
任务耗时 <5ms？
  └─ 是 → ❌ 不用 isolate（开销反而更大）
  └─ 否 → 继续

任务是 I/O 等待（网络、文件）？
  └─ 是 → ❌ 用 async/await（不需要 isolate）
  └─ 否 → 继续

任务是 CPU 密集计算？
  └─ 是 → ✅ 用 isolate
```

### 2. 首选 `Isolate.run`，避免过度设计

```dart
// ✅ 推荐：简单直接
final result = await Isolate.run(() => heavyComputation(data));

// ❌ 避免：过早优化为长生命周期 isolate
// 除非性能测试证明需要
```

### 3. 只传递必要数据，减少拷贝开销

```dart
// ❌ 避免：传递整个对象
await Isolate.run(() => process(largeObject));

// ✅ 推荐：只传递需要的字段
await Isolate.run(() => process(largeObject.data));
```

### 4. 顶层函数优于闭包

```dart
// ❌ 闭包可能捕获外部变量，导致序列化失败
final threshold = 100;
await Isolate.run(() => filter(data, threshold)); // 可能报错

// ✅ 顶层函数 + 显式参数
await Isolate.run(() => _filterWithThreshold(data, 100));

List<int> _filterWithThreshold(List<int> data, int threshold) {
  return data.where((x) => x > threshold).toList();
}
```

### 5. 在 profile 模式下验证效果

```bash
# ✅ 正确：profile 模式测试性能
flutter run --profile

# ❌ 错误：debug 模式性能不可信
flutter run
```

### 6. 用 Timeline 确认 UI 不再堵塞

优化前后都要录制 Timeline，对比：
- UI 线程是否还有长耗时事件？
- 帧耗时是否降到 <16ms？

参考：[performance-timeline 工作流](performance-timeline.md)

### 7. 避免在 isolate 中访问 Flutter 绑定

```dart
// ❌ 错误：rootBundle 只在主 isolate 可用
await Isolate.run(() {
  final data = rootBundle.loadString('assets/data.json'); // 报错
});

// ✅ 正确：先在主 isolate 加载，再传递
final jsonString = await rootBundle.loadString('assets/data.json');
await Isolate.run(() => parseJson(jsonString));
```

### 8. 异常处理：isolate 中的异常会传回主 isolate

```dart
try {
  final result = await Isolate.run(() {
    if (invalidData) throw Exception('Invalid data');
    return compute();
  });
} catch (e) {
  // isolate 中的异常会在这里捕获
  print('Isolate error: $e');
}
```

### 9. 批量任务复用 isolate

如果需要处理大量任务：

```dart
// ❌ 低效：每次创建新 isolate
for (final image in images) {
  await Isolate.run(() => compress(image)); // 每次 1-5ms 开销
}

// ✅ 高效：批量处理或用长生命周期 isolate
final compressed = await Isolate.run(() {
  return images.map(compress).toList();
});
```

### 10. 单元测试时注意 isolate

```dart
// 单元测试中，isolate 会真实创建，测试时间较长
test('parse JSON', () async {
  final result = await parseJson(jsonString); // 可能 +10ms
  expect(result.length, 100);
});

// 可以 mock isolate 层，只测试纯逻辑
test('parse JSON (mocked)', () {
  final result = _parseJsonSync(jsonString); // 不启动 isolate
  expect(result.length, 100);
});
```

---

## 🔗 相关资源

- 提示词：[prompts/concurrency/extract-work-to-isolate.md](../prompts/concurrency/extract-work-to-isolate.md) — 提取耗时任务到 isolate
- 提示词：[prompts/concurrency/fix-isolate-errors.md](../prompts/concurrency/fix-isolate-errors.md) — 修复 isolate 相关报错
- 性能分析：[workflows/performance-timeline.md](performance-timeline.md) — 用 Timeline 定位性能瓶颈
- Flutter 官方文档：[Concurrency and isolates](https://docs.flutter.dev/perf/isolates)
- Dart 官方文档：[Isolates](https://dart.dev/language/isolates)
- API 文档：[compute()](https://api.flutter.dev/flutter/foundation/compute.html)
- API 文档：[Isolate.run()](https://api.dart.dev/stable/dart-isolate/Isolate/run.html)

---

**让 Flutter 应用更流畅，榨干多核 CPU 💪**
