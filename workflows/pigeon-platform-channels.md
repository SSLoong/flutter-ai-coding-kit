# Pigeon 类型安全平台通道工作流

**用 AI 定义 HostApi / FlutterApi，生成 Dart、Kotlin、Swift，并在本地完成注册、编译和 mock 测试**

> 作者：Loong (SSLoong) | 日期：2026-11-19

本文对齐 pub.dev 上的 **pigeon 30.0.4**（要求 Flutter 3.44 / Dart 3.12 及以上）和 Flutter 官方文档 [Pigeon](https://docs.flutter.dev/interop/pigeon)。先在本机执行 `flutter --version`。低于 3.44 时，升级 Flutter 再套用下面的 `@async` 与 iOS 注册方式。

Pigeon 生成代码的签名会随版本变化。两侧（Dart 与宿主）必须用同一次、同一版本的生成结果。实现前打开生成文件，以生成出的接口为准；下文示例与 30.0.4 官方 example 的生成结果一致。

---

## 📋 背景

手写 `MethodChannel` 要在 Dart、Kotlin、Swift 三处维持方法名字符串和参数顺序。改一个字段就会在运行时变成 `PlatformException` 或类型转换崩溃。Pigeon 把接口写成一份 Dart 定义，再生成两侧的类型、编解码和 channel 名字。

AI 适合起草定义和宿主实现，但生成物必须在本地重新生成、编译，并用 fake/mock 测 Dart 侧调用。禁止把未跑通的生成代码直接合入。

---

## 🎯 目标

- ✅ 判断该用 Pigeon 平台通道、手写 MethodChannel，还是 FFI
- ✅ 写出 `HostApi`、`FlutterApi`、数据类、枚举、`@async` 与错误传递
- ✅ 生成 Dart / Kotlin / Swift，并在 Android、iOS 完成注册
- ✅ 在 Dart 测试里 fake 或 mock 生成出的 HostApi，不启动模拟器也能验证调用方
- ✅ 用 `flutter test` 与 `flutter build` 把「没提交生成物 / 没注册 / 类型不匹配」挡在合入之前

---

## 🔧 适用场景

| 场景 | 适合度 | 说明 |
|------|--------|------|
| 新建 Dart ↔ Kotlin/Swift 的结构化 API | ⭐⭐⭐⭐⭐ | 参数、返回值、枚举、可空字段都进定义文件 |
| 宿主在运行中回调 Dart | ⭐⭐⭐⭐⭐ | `@FlutterApi()`，由 Dart 实现 |
| 需要稳定错误码（无电量、无权限、超时） | ⭐⭐⭐⭐⭐ | 宿主抛 `FlutterError` / `PigeonError`，Dart 收 `PlatformException` |
| 已有 channel 协议，原生侧不能改 | ⭐ | 继续手写 `MethodChannel`，Pigeon 会生成另一套 channel 名 |
| 调用 C/C++ 库、持有裸指针 | ⭐ | 用 `ffigen` / `jnigen` 或 `dart:ffi` |
| 高频图像、音频、传感器缓冲 | ⭐⭐ | 平台通道每次都要序列化；先量化频率。Pigeon Native Interop 仍是 preview |

**怎么选：**

- **Pigeon 平台通道（本文）**：应用或插件里新的一组调用。数据是枚举、数据类、`List` / `Map`、可空字段，调用频率是用户操作或偶发系统查询。Android 用 Kotlin，iOS 用 Swift。窗口、桌面如果只要 Windows / Linux，同一份定义还可以生成 C++ / GObject；本文示例只写 Android 与 iOS。
- **手写 MethodChannel / EventChannel**：对方已经规定 channel 名、方法名和 codec（旧原生模块、第三方容器），或者方法名要到运行时才知道。Pigeon 的 channel 名固定为 `dev.flutter.pigeon.<dartPackageName>.<Api>.<method>`，并且使用 `StandardMessageCodec`。
- **FFI（`dart:ffi`、FFIgen、JNIgen）**：要绑定已经存在的 C / Objective-C / Java / Kotlin API，或在调用线程上同步访问原生内存。FFIgen（含 Objective-C）和 JNIgen 已稳定；它们生成的是某一侧原生 API 的绑定，不是一份跨平台契约。见 [FFIgen](https://docs.flutter.dev/interop/ffigen) 与 [JNIgen](https://docs.flutter.dev/interop/jnigen)。
- **Pigeon Native Interop（`useJni` / `useFfi`）**：官方文档和 2026-11-12 的 Flutter 博客仍把它标为 preview，不作为默认。它不支持 `@TaskQueue`；多 Engine 路由在 Flutter 3.50 之前还有已知限制。需要同步、高频或后台 isolate 直调宿主时，先读 [native_interop_guide.md](https://github.com/flutter/packages/blob/main/packages/pigeon/native_interop_guide.md)，单独验证，不要和本文的 `setUp(binaryMessenger, api)` 混用。

---

## 🚀 完整流程

下面用「电量」贯穿：Dart 向宿主要电量和快照（`BatteryHostApi`），宿主在电量变化时回调 Dart（`BatteryFlutterApi`）。包名 `com.example.my_app` / `my_app` 换成你的 `android` namespace 和 `pubspec.yaml` 的 `name`。

### Step 1: 准备上下文

**人工操作**：在 Cursor 里打开 `pubspec.yaml`、`android/app/build.gradle`（或 `.kts`）里的 `namespace`，以及 iOS 的 `AppDelegate.swift`。确认 Flutter ≥ 3.44。

**AI 提示词**（可复制）：

```markdown
我要为 Flutter 应用增加一条 Pigeon 平台通道。请先确认，不要写代码：

1. 哪些调用是 Dart → 宿主（HostApi）？哪些是宿主 → Dart（FlutterApi）？
2. 每个方法的参数、返回值、可空性？有没有枚举和数据类？
3. 哪些错误需要稳定 code（例如 unavailable、permission-denied）？
4. 宿主方法是立刻返回，还是要等待系统回调？后者才标 @async。
5. 有没有耗时 IO？耗时且不需要 suspend 的，标 @TaskQueue(type: TaskQueueType.serialBackgroundThread)。
6. 是否多 Engine / add-to-app？同一 Engine 上是否有多份实例（需要 messageChannelSuffix）？

约束：pigeon 30.0.4，平台通道，不启用 Native Interop。列出方案后再生成定义文件。
```

**检查点**：方向（谁实现、谁调用）和可空性是否符合预期。可空性一旦写错，运行期会变成 `null-error`。

---

### Step 2: 写定义并配置生成

定义文件放在 `lib/` 之外（例如 `pigeons/battery_api.dart`）。这个文件只声明类型和方法，不写方法体，也不要被应用代码 import。应用只 import 生成结果。

`pubspec.yaml` 增加 dev 依赖：

```yaml
dev_dependencies:
  flutter_test:
    sdk: flutter
  pigeon: ^30.0.4
```

**AI 提示词**（可复制）：见 [prompts/platform/define-pigeon-api.md](../prompts/platform/define-pigeon-api.md)。

**定义文件示例**（`pigeons/battery_api.dart`）：

```dart
import 'package:pigeon/pigeon.dart';

@ConfigurePigeon(
  PigeonOptions(
    dartOut: 'lib/src/battery_api.g.dart',
    dartOptions: DartOptions(),
    kotlinOut: 'android/app/src/main/kotlin/com/example/my_app/BatteryApi.g.kt',
    kotlinOptions: KotlinOptions(package: 'com.example.my_app'),
    swiftOut: 'ios/Runner/BatteryApi.g.swift',
    swiftOptions: SwiftOptions(),
    dartPackageName: 'my_app',
  ),
)
enum PowerSource { battery, ac, usb, unknown }

class BatterySnapshot {
  BatterySnapshot({
    required this.level,
    required this.source,
    this.isCharging,
  });

  /// 0–100。Dart int 在 Kotlin 是 Long，在 Swift 是 Int64。
  int level;
  PowerSource source;
  bool? isCharging;
}

@HostApi()
abstract class BatteryHostApi {
  /// 平台主线程上同步返回。实现必须短小。
  int getBatteryLevel();

  /// Kotlin 生成 suspend，Swift 生成 async throws。
  /// 协程默认仍从主线程启动；耗时 IO 不要只靠这个注解。
  @async
  BatterySnapshot readSnapshot();

  /// 后台串行队列。不要在实现里碰 UI，也不要在这条线程上直接调用 FlutterApi。
  @TaskQueue(type: TaskQueueType.serialBackgroundThread)
  String readDiagnostics();
}

@FlutterApi()
abstract class BatteryFlutterApi {
  void onBatteryChanged(BatterySnapshot snapshot);
}
```

定义规则（与 pigeon 30 的 README 一致）：

- 文件里不能有方法体。构造函数可以有，官方 example 也这样写。
- 自定义类的字段只能是平台通道支持的类型：`bool`、`int`、`double`、`String`、`Uint8List`、`Int32List`、`Int64List`、`Float64List`、`List`、`Map`、本文件里的枚举和类，以及它们的可空形式。
- 枚举必须是简单枚举。带构造函数、字段或方法的增强枚举会被生成器拒绝。
- `DateTime`、`Uri`、record、函数类型都不支持。时间用 `int` 毫秒。
- 继承只允许空的 `sealed` 父类，而且仅 Swift / Kotlin / Dart 生成器支持。数据模型保持扁平。
- `@async` 是默认的异步风格：Kotlin `suspend`，Swift `async throws`。只有 Java、Objective-C、C++、GObject 会把 `@async` 和 `@asyncCallback` 都生成成回调。
- 需要回调形态时才用 `@asyncCallback`（Kotlin `(Result<T>) -> Unit`，Swift completion）。回调必须正好调用一次。
- `@TaskQueue` 只对平台通道有效。和 Native Interop 一起用会在生成期报错。
- 从 pigeon 28 起，Kotlin / Swift 上的 `@FlutterApi` 方法默认就是 `suspend` / `async throws`，即使 Dart 声明的返回值不是 `Future`。
- 不要加 `dartHostTestHandler`。pigeon 26.1.0 起该参数已废弃，测试改为 fake / mock 生成出的 Dart API。

---

### Step 3: 生成代码

**人工操作**：

```bash
flutter pub get
dart run pigeon --input pigeons/battery_api.dart
```

`@ConfigurePigeon` 已经写了输出路径。命令行参数会覆盖同名字段；路径以定义文件为准时，不要再传一套 `--dart_out`。

生成后应出现：

- `lib/src/battery_api.g.dart`
- `android/app/src/main/kotlin/com/example/my_app/BatteryApi.g.kt`
- `ios/Runner/BatteryApi.g.swift`

打开这三份文件，核对签名后再写实现。30.0.4 官方 example 对应的形态是：

| 位置 | 期望签名 |
|------|----------|
| Dart `BatteryHostApi` | 具体类，构造参数 `binaryMessenger`、`messageChannelSuffix`；方法都返回 `Future` |
| Dart `BatteryFlutterApi` | 抽象类，外加 `static void setUp(BatteryFlutterApi? api, {BinaryMessenger? binaryMessenger, String messageChannelSuffix = ''})` |
| Kotlin | `interface BatteryHostApi`：`fun getBatteryLevel(): Long`、`suspend fun readSnapshot(): BatterySnapshot`、`fun readDiagnostics(): String`；`companion object` 里 `setUp(binaryMessenger, api, messageChannelSuffix = "")` |
| Kotlin 错误类型 | 生成 `class FlutterError(code, message, details) : RuntimeException()` |
| Swift | `protocol BatteryHostApi`：`throws -> Int64`、`async throws -> BatterySnapshot`；注册类 `BatteryHostApiSetup.setUp(binaryMessenger:api:messageChannelSuffix:)` |
| Swift 错误类型 | 生成 `PigeonError`。Swift 实现里抛它，不要抛 `FlutterError` |

`kotlinOptions.package` 必须和 `BatteryApi.g.kt` 顶部的 `package` 行、目录 `kotlin/com/example/my_app/` 一致。`dartPackageName` 会写进 channel 名：`dev.flutter.pigeon.my_app.BatteryHostApi.getBatteryLevel`。不要手改这些名字。

**构建配置：**

- Android：文件放在 `android/app/src/main/kotlin/<package>/` 后，Flutter Gradle Plugin 会把它编进 app。平台通道不需要额外 sourceSet，也不需要 CMake。
- `@async` 的 Kotlin 生成物会 `import kotlinx.coroutines`。官方 example 没有单独声明这条依赖。若 `compileDebugKotlin` 报 `Unresolved reference 'kotlinx'`，再给 app 模块加上与当前 Kotlin 匹配的 `kotlinx-coroutines-android`。
- iOS：`ios/Runner/*.swift` 在使用同步目录（`PBXFileSystemSynchronizedRootGroup`）的工程里会自动进 Runner target。旧工程要在 Xcode 里把 `BatteryApi.g.swift` 加进 Runner target，不要加进测试 target。纯 Swift 生成物不需要 Bridging Header。

**提交生成物。** `pigeons/battery_api.dart` 和三份生成文件都进 Git。很多 `.gitignore` 含有 `*.g.dart`，会把 Pigeon 的 Dart 输出一并忽略。生成后执行 `git status`，确认三个生成文件都是待提交，而不是被忽略。CI 不应依赖「再跑一次 pigeon」才编得过，因为两侧版本必须一致，漏提交一侧就会在运行时对不上。

---

### Step 4: Android 实现 HostApi 并注册

**AI 提示词**（可复制）：见 [prompts/platform/implement-and-fix-pigeon.md](../prompts/platform/implement-and-fix-pigeon.md)。

**实现示例**（`android/app/src/main/kotlin/com/example/my_app/MainActivity.kt`）。`FlutterError`、`BatteryHostApi`、`BatterySnapshot`、`PowerSource`、`BatteryFlutterApi` 与 `MainActivity` 同包，由 `BatteryApi.g.kt` 生成，不需要再 import。

```kotlin
package com.example.my_app

import android.content.Context
import android.os.BatteryManager
import io.flutter.embedding.android.FlutterActivity
import io.flutter.embedding.engine.FlutterEngine

class BatteryHostApiImpl(private val context: Context) : BatteryHostApi {
    override fun getBatteryLevel(): Long {
        val manager = context.getSystemService(Context.BATTERY_SERVICE) as BatteryManager
        val level = manager.getIntProperty(BatteryManager.BATTERY_PROPERTY_CAPACITY)
        if (level < 0) {
            throw FlutterError("unavailable", "Battery level is unavailable", null)
        }
        return level.toLong()
    }

    override suspend fun readSnapshot(): BatterySnapshot {
        val level = getBatteryLevel()
        val manager = context.getSystemService(Context.BATTERY_SERVICE) as BatteryManager
        val charging = manager.isCharging
        return BatterySnapshot(
            level = level,
            source = if (charging) PowerSource.AC else PowerSource.BATTERY,
            isCharging = charging,
        )
    }

    override fun readDiagnostics(): String {
        // 此方法在 Pigeon 的后台串行 TaskQueue 上执行。
        return "level=${getBatteryLevel()}"
    }
}

private var batteryFlutterApi: BatteryFlutterApi? = null

class MainActivity : FlutterActivity() {
    override fun configureFlutterEngine(flutterEngine: FlutterEngine) {
        super.configureFlutterEngine(flutterEngine)
        val messenger = flutterEngine.dartExecutor.binaryMessenger
        BatteryHostApi.setUp(messenger, BatteryHostApiImpl(this))
        batteryFlutterApi = BatteryFlutterApi(messenger)
    }
}

suspend fun notifyBatteryChanged(snapshot: BatterySnapshot) {
    batteryFlutterApi?.onBatteryChanged(snapshot)
}
```

要点：

- Dart `int` 实现成 Kotlin `Long`，枚举 `PowerSource.ac` 实现成 `PowerSource.AC`。
- 同步方法和 `@async` 方法直接 `throw FlutterError(code, message, details)`。生成代码会抓成 `PlatformException`。`details` 只能是 codec 支持的类型。
- 其他异常也会被包成错误列表（类名 + `toString()`），Dart 侧拿不到你想要的稳定 code。业务失败一律抛 `FlutterError`。
- `setUp(..., null)` 会卸掉 handler。Activity 销毁或 add-to-app 里 Engine 销毁时调用它。
- 插件形态把同一对调用放进 `FlutterPlugin`：`onAttachedToEngine` 里 `setUp(binding.binaryMessenger, impl)`，`onDetachedFromEngine` 里 `setUp(binding.binaryMessenger, null)`。

同一条 Engine 上要挂多份实例时，两侧使用同一个非空后缀：

```kotlin
BatteryHostApi.setUp(
    messenger,
    BatteryHostApiImpl(this),
    messageChannelSuffix = "player-1",
)
```

```dart
final api = BatteryHostApi(messageChannelSuffix: 'player-1');
```

空后缀是默认实例。后缀会变成 channel 名末尾的 `.$suffix`。

---

### Step 5: iOS 实现 HostApi 并注册

当前 Flutter 模板使用 UIScene。应用级通道在 `FlutterImplicitEngineDelegate.didInitializeImplicitFlutterEngine` 里注册，messenger 取 `engineBridge.applicationRegistrar.messenger()`。插件仍在 `FlutterPlugin.register(with:)` 里用 `registrar.messenger()`。

**实现与注册示例**（`ios/Runner/AppDelegate.swift`）。生成的协议、`PigeonError`、`BatteryHostApiSetup`、`BatteryFlutterApi` 都在 Runner 模块里。

```swift
import Flutter
import UIKit

final class BatteryHostApiImpl: BatteryHostApi {
  func getBatteryLevel() throws -> Int64 {
    let device = UIDevice.current
    device.isBatteryMonitoringEnabled = true
    let raw = device.batteryLevel
    if raw < 0 {
      throw PigeonError(
        code: "unavailable",
        message: "Battery level is unavailable",
        details: "battery-monitor-disabled"
      )
    }
    return Int64((raw * 100).rounded())
  }

  func readSnapshot() async throws -> BatterySnapshot {
    let level = try getBatteryLevel()
    switch UIDevice.current.batteryState {
    case .charging, .full:
      return BatterySnapshot(level: level, source: PowerSource.ac, isCharging: true)
    case .unplugged:
      return BatterySnapshot(level: level, source: PowerSource.battery, isCharging: false)
    default:
      return BatterySnapshot(level: level, source: PowerSource.unknown, isCharging: nil)
    }
  }

  func readDiagnostics() throws -> String {
    let level = try getBatteryLevel()
    return "level=\(level)"
  }
}

@main
@objc class AppDelegate: FlutterAppDelegate, FlutterImplicitEngineDelegate {
  private let batteryApi = BatteryHostApiImpl()
  private var batteryFlutterApi: BatteryFlutterApi?

  override func application(
    _ application: UIApplication,
    didFinishLaunchingWithOptions launchOptions: [UIApplication.LaunchOptionsKey: Any]?
  ) -> Bool {
    return super.application(application, didFinishLaunchingWithOptions: launchOptions)
  }

  func didInitializeImplicitFlutterEngine(_ engineBridge: FlutterImplicitEngineBridge) {
    GeneratedPluginRegistrant.register(with: engineBridge.pluginRegistry)
    let messenger = engineBridge.applicationRegistrar.messenger()
    BatteryHostApiSetup.setUp(binaryMessenger: messenger, api: batteryApi)
    batteryFlutterApi = BatteryFlutterApi(binaryMessenger: messenger)
  }

  func notifyBatteryChanged(snapshot: BatterySnapshot) async throws {
    guard let batteryFlutterApi else { return }
    try await batteryFlutterApi.onBatteryChanged(snapshot: snapshot)
  }
}
```

工程里如果已经有 `didInitializeImplicitFlutterEngine`，只把 `setUp` 和 `BatteryFlutterApi(...)` 两行并进去，不要再写一个 `AppDelegate`。

插件注册：

```swift
import Flutter
import UIKit

public class BatteryPlugin: NSObject, FlutterPlugin {
  public static func register(with registrar: FlutterPluginRegistrar) {
    BatteryHostApiSetup.setUp(
      binaryMessenger: registrar.messenger(),
      api: BatteryHostApiImpl()
    )
  }
}
```

要点：

- Swift 用 `Int64`，枚举 case 与 Dart 同名：`PowerSource.battery`、`PowerSource.ac`。
- 业务错误抛 `PigeonError(code:message:details:)`。`FlutterError` 不符合 `Swift.Error`，pigeon 19 起生成代码不再用它做 Swift 错误类型。
- `@MainActor` 标在生成的 async FlutterApi 上（pigeon 29.0.1 起）。从后台回调进 Dart 时，先回到 main actor 再 `await`。
- 卸载：`BatteryHostApiSetup.setUp(binaryMessenger: messenger, api: nil)`。

---

### Step 6: Dart 侧调用，并实现 FlutterApi

业务代码依赖生成类，但把 `BatteryHostApi` 从构造函数注入，测试才能替换它。

**调用与回调示例**（`lib/battery_client.dart`）：

```dart
import 'package:flutter/services.dart';

import 'package:my_app/src/battery_api.g.dart';

class BatteryClient {
  BatteryClient(this._api);

  final BatteryHostApi _api;

  Future<int> getBatteryLevel() => _api.getBatteryLevel();

  Future<BatterySnapshot> readSnapshot() => _api.readSnapshot();

  Future<String> readDiagnostics() => _api.readDiagnostics();
}

class BatteryEvents implements BatteryFlutterApi {
  BatteryEvents(this.onChanged);

  final void Function(BatterySnapshot snapshot) onChanged;

  @override
  void onBatteryChanged(BatterySnapshot snapshot) {
    onChanged(snapshot);
  }
}

String messageForBatteryError(Object error) {
  if (error is PlatformException) {
    return '${error.code}: ${error.message ?? ''}';
  }
  return error.toString();
}
```

**注册 FlutterApi**（`lib/main.dart` 的启动段）：

```dart
import 'package:flutter/material.dart';

import 'package:my_app/battery_client.dart';
import 'package:my_app/src/battery_api.g.dart';

Future<void> main() async {
  WidgetsFlutterBinding.ensureInitialized();
  BatteryFlutterApi.setUp(
    BatteryEvents((snapshot) {
      debugPrint('battery ${snapshot.level} via ${snapshot.source.name}');
    }),
  );
  runApp(const MyApp());
}

class MyApp extends StatelessWidget {
  const MyApp({super.key});

  @override
  Widget build(BuildContext context) {
    return const MaterialApp(home: BatteryPage());
  }
}

class BatteryPage extends StatefulWidget {
  const BatteryPage({super.key});

  @override
  State<BatteryPage> createState() => _BatteryPageState();
}

class _BatteryPageState extends State<BatteryPage> {
  final BatteryClient _client = BatteryClient(BatteryHostApi());
  String _label = 'unknown';

  Future<void> _reload() async {
    try {
      final snapshot = await _client.readSnapshot();
      setState(() => _label = '${snapshot.level}%');
    } on PlatformException catch (error) {
      setState(() => _label = messageForBatteryError(error));
    }
  }

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      body: Center(child: Text(_label)),
      floatingActionButton: FloatingActionButton(
        onPressed: _reload,
        child: const Icon(Icons.battery_std),
      ),
    );
  }
}
```

要点：

- Dart 侧每次调用都是 `Future`，宿主方法写不写 `@async` 都不改变这一点。平台通道没有同步的 Dart 返回值。
- `BatteryFlutterApi.setUp` 必须发生在宿主第一次回调之前。热重载 / 热重启后在 `main` 里再次 `setUp`，用新实例替换旧 handler。
- 页面或 Engine 退出时调用 `BatteryFlutterApi.setUp(null)` 卸掉 Dart handler。
- Dart 实现里如果要把稳定错误码回给宿主，抛 `PlatformException(code: ..., message: ..., details: ...)`。其他异常会被生成代码包成 `PlatformException(code: 'error', message: e.toString())`。
- 宿主返回 `null` 给非空返回值时，Dart 抛 `PlatformException(code: 'null-error')`。

---

### Step 7: 用 fake / mock 测试生成接口

pigeon 26.1.0 起废弃 `dartHostTestHandler` 和 `dartTestOut`。应用测试 fake 或 mock 生成出的 `BatteryHostApi`，并直接调用你写的 `BatteryFlutterApi` 实现。序列化本身由 Pigeon 的测试覆盖；应用测试覆盖「调用方如何使用这条接口」。

fake 继承生成类并覆盖方法。方法没有被调用到真实 channel，因此不需要模拟器，也不会出现 `MissingPluginException`。

**测试示例**（`test/battery_client_test.dart`）：

```dart
import 'package:flutter/services.dart';
import 'package:flutter_test/flutter_test.dart';
import 'package:my_app/battery_client.dart';
import 'package:my_app/src/battery_api.g.dart';

class FakeBatteryHostApi extends BatteryHostApi {
  FakeBatteryHostApi({this.level = 80, this.error});

  int level;
  PlatformException? error;
  int readSnapshotCalls = 0;

  @override
  Future<int> getBatteryLevel() async {
    final pending = error;
    if (pending != null) {
      throw pending;
    }
    return level;
  }

  @override
  Future<BatterySnapshot> readSnapshot() async {
    readSnapshotCalls += 1;
    final pending = error;
    if (pending != null) {
      throw pending;
    }
    return BatterySnapshot(
      level: level,
      source: PowerSource.ac,
      isCharging: true,
    );
  }

  @override
  Future<String> readDiagnostics() async => 'level=$level';
}

void main() {
  test('readSnapshot returns the generated data class', () async {
    final api = FakeBatteryHostApi(level: 42);
    final client = BatteryClient(api);

    final snapshot = await client.readSnapshot();

    expect(snapshot.level, 42);
    expect(snapshot.source, PowerSource.ac);
    expect(snapshot.isCharging, isTrue);
    expect(api.readSnapshotCalls, 1);
  });

  test('PlatformException from the host api is preserved', () async {
    final api = FakeBatteryHostApi(
      error: PlatformException(
        code: 'unavailable',
        message: 'Battery level is unavailable',
      ),
    );
    final client = BatteryClient(api);

    await expectLater(
      client.getBatteryLevel(),
      throwsA(
        isA<PlatformException>().having(
          (error) => error.code,
          'code',
          'unavailable',
        ),
      ),
    );
  });

  test('BatteryEvents forwards snapshots', () {
    BatterySnapshot? received;
    final events = BatteryEvents((snapshot) => received = snapshot);
    final snapshot = BatterySnapshot(
      level: 7,
      source: PowerSource.usb,
      isCharging: true,
    );

    events.onBatteryChanged(snapshot);

    expect(received, snapshot);
  });
}
```

可选：项目里已有 `mocktail` 时，用 `class MockBatteryHostApi extends Mock implements BatteryHostApi {}`，再 `when(() => api.getBatteryLevel()).thenAnswer((_) async => 42)`。`implements` 的 mock 不会走生成类的 channel。断言异步错误用 `expectLater`，因为生成方法返回 `Future`。

不要在 `flutter test` 里调用 `BatteryFlutterApi.setUp`，除非先 `TestWidgetsFlutterBinding.ensureInitialized()` 并准备 `BinaryMessenger`。那是在测 channel 接线，不是在测生成接口的调用方。接线放到真机或集成测试。

---

### Step 8: 本地验证（必须）

**人工操作**：

```bash
# 1. 重新生成，避免手改生成物
dart run pigeon --input pigeons/battery_api.dart

# 2. Dart 分析 + 单元测试（不编译宿主）
dart analyze lib/src/battery_api.g.dart lib/battery_client.dart lib/main.dart
flutter test test/battery_client_test.dart

# 3. 编译宿主，确认 Kotlin / Swift 实现和生成接口一致
flutter build apk --debug
flutter build ios --no-codesign --debug
```

只跑 `flutter test` 证明不了宿主已经注册、类型已经对齐。`flutter test` 不编译 `android/` 和 `ios/`。

**检查清单**：

- [ ] `dart run pigeon` 退出码为 0，三个生成文件都更新
- [ ] `git status` 能看到生成文件（没有被 `*.g.dart` 忽略）
- [ ] `flutter test test/battery_client_test.dart` 全部通过
- [ ] `flutter build apk --debug` 通过（Kotlin `Long` / 枚举大写 / `setUp` 签名）
- [ ] `flutter build ios --no-codesign --debug` 通过（`Int64` / `PigeonError` / `async throws`）
- [ ] 真机或模拟器上调用一次 `readSnapshot()`，失败时能看到 `unavailable` 而不是 `channel-error`
- [ ] 宿主触发一次 `onBatteryChanged`，Dart 回调收到同一份 `level` 和 `source`

**常见失败**：

- `MissingPluginException` 或 `Unable to establish connection on channel`：宿主没有对这个 Engine 的 messenger 调用 `setUp`，或 Dart / 宿主的 `messageChannelSuffix` 不一致
- `null-error`：非空返回值在宿主侧成了 null，或枚举 raw 对不上
- Kotlin `ClassCastException` / `Long` vs `Int`：把 Dart `int` 写成了 `Int`
- Swift `Cannot convert value of type 'Int' to 'Int64'`：电量计算后没有转成 `Int64`
- `Unresolved reference 'kotlinx'`：补上 coroutines，或暂时去掉 `@async` 再生成（同步方法的生成物不引用 coroutines）

---

### Step 9: 迭代修复

生成器报错、Kotlin/Swift 编译错误、或运行期 `PlatformException`，把**定义文件、生成签名和完整报错**一起交给 AI。提示词用 [prompts/platform/implement-and-fix-pigeon.md](../prompts/platform/implement-and-fix-pigeon.md)。

修复原则：改 `pigeons/*.dart` 或宿主实现，然后重新 `dart run pigeon`。生成文件头部写着 Autogenerated，不要在里面改签名。

---

## ✅ 验证清单

完成后确认：

- [ ] 定义文件在 `lib/` 外，`pigeon` 只出现在 `dev_dependencies`
- [ ] `@HostApi()` 与 `@FlutterApi()` 方向正确，错误用 `FlutterError` / `PigeonError` / `PlatformException`
- [ ] `@async` 与 `@TaskQueue` 的选择和线程模型一致
- [ ] Android 在 `configureFlutterEngine`（或插件的 `onAttachedToEngine`）里 `setUp`
- [ ] iOS 在 `didInitializeImplicitFlutterEngine`（或插件的 `register(with:)`）里 `setUp`
- [ ] Dart 在 `main` 里 `BatteryFlutterApi.setUp`，退出时 `setUp(null)`
- [ ] 生成的 `.g.dart`、`.g.kt`、`.g.swift` 与定义文件一起提交，且来自同一版 pigeon
- [ ] `flutter test` 用 fake 或 mock 覆盖成功路径和 `PlatformException`
- [ ] `flutter build apk` 与 `flutter build ios` 通过
- [ ] 多 Engine 时每个 Engine 用自己的 `binaryMessenger` 注册；同一 Engine 的多实例才用 `messageChannelSuffix`

---

## 🤔 FAQ

### Q1: 生成物忘了提交，CI 或同事拉下来编不过？

**A**: 把 `pigeons/battery_api.dart`、`lib/src/battery_api.g.dart`、`BatteryApi.g.kt`、`BatteryApi.g.swift` 全部提交。检查 `.gitignore` 里的 `*.g.dart`、`*.g.kt`。Pigeon 的输出和 `build_runner` 的 `*.g.dart` 撞名时，为 Pigeon 路径加否定规则，例如 `!lib/src/battery_api.g.dart`。

只提交 Dart 生成物、不提交 Kotlin/Swift（或反过来）时，channel 编解码可能已经变了，表现是运行期崩溃或字段错位，而不是编译错误。pub.dev 说明：两侧必须用同一版本 Pigeon 生成，混用版本是未定义行为。不要把生成代码放进插件的公共 API，也不要把 Dart 生成物和宿主生成物拆到两个独立发版的包里。

### Q2: 调用就报 MissingPluginException / channel-error？

**A**: 这是「没注册」或「注册到了另一个 messenger」。

- Android 应用：`configureFlutterEngine` 里对 `flutterEngine.dartExecutor.binaryMessenger` 调用 `BatteryHostApi.setUp`。
- Android 插件：`onAttachedToEngine` 注册，`onDetachedFromEngine` 传 `null` 卸掉。
- iOS 应用（UIScene 模板）：写在 `didInitializeImplicitFlutterEngine`，使用 `engineBridge.applicationRegistrar.messenger()`。写在 `application:didFinishLaunchingWithOptions:` 里注册不上这条隐式 Engine。迁移说明见 [UIScene adoption](https://docs.flutter.dev/release/breaking-changes/uiscenedelegate)。
- iOS 插件：`registrar.messenger()`。
- FlutterApi：Dart `main` 里 `WidgetsFlutterBinding.ensureInitialized()` 之后立刻 `setUp`。宿主在这之前回调，会得到 connection error。
- `messageChannelSuffix` 必须逐字相同。默认实例不要只在一侧填后缀。
- 热重启之后 Dart handler 会丢，再次 `setUp`。

### Q3: 类型不匹配，编译失败或运行期 ClassCastException？

**A**: 对照生成文件，不要对照定义文件里的 Dart 类型直接写宿主。

| Dart 定义 | Kotlin | Swift |
|-----------|--------|-------|
| `int` | `Long` | `Int64` |
| `bool` | `Boolean` | `Bool` |
| `double` | `Double` | `Double` |
| `String?` | `String?` | `String?` |
| 枚举 `battery` | `PowerSource.BATTERY` | `PowerSource.battery` |
| 非空返回值 | 禁止返回 null | 禁止返回 nil |

可空字段在定义里就是 `T?`。不要在宿主把可空值塞进非空参数。枚举不要写成增强枚举，也不要手改生成枚举的 `raw`。

改了字段顺序或类型后，必须重新生成三端。手改 `*.g.kt` / `*.g.swift` 会在下次生成时被覆盖，并造成两端不一致。

### Q4: 卡顿、死锁，或后台线程调用 Flutter 崩溃？

**A**:

- 没标 `@async`、也没标 `@TaskQueue` 的 HostApi 跑在平台主线程。实现里禁止网络、磁盘和锁等待。
- `@async` 让 Kotlin 用 `suspend`、Swift 用 `async throws`。Kotlin 生成代码用 `Dispatchers.Main` 启动这个 suspend 方法。它解决的是「宿主需要挂起等结果」，不是自动丢到后台。
- 耗时且不需要挂起的读取，用 `@TaskQueue(type: TaskQueueType.serialBackgroundThread)`。该线程禁止访问 UI。
- 在 TaskQueue 或自建后台线程上需要回调 Dart 时，先回到主线程：Kotlin 切到 `Dispatchers.Main` 再调用 `BatteryFlutterApi`；Swift 在 main actor 上 `await`。生成的 Swift FlutterApi 带 `@MainActor`。
- `@asyncCallback` 的 completion / `Result` 回调正好调用一次。漏调会使 Dart 侧的 `Future` 一直挂起；调两次是未定义行为。
- `@TaskQueue` 不能和 Native Interop 一起生成。

### Q5: 多 Engine、add-to-app 时串到了另一个 Engine？

**A**: 每个 `FlutterEngine` 有自己的 `BinaryMessenger`。注册、创建 `BatteryFlutterApi`、Dart 里的 `setUp` 都绑在这个 Engine 上。

```kotlin
fun attachBattery(engine: FlutterEngine, context: Context) {
    BatteryHostApi.setUp(
        engine.dartExecutor.binaryMessenger,
        BatteryHostApiImpl(context),
    )
}

fun detachBattery(engine: FlutterEngine) {
    BatteryHostApi.setUp(engine.dartExecutor.binaryMessenger, null)
}
```

Dart 入口（每个 Engine 的 entrypoint）都要执行 `BatteryFlutterApi.setUp(...)`。Engine 销毁前 `setUp(null)`，避免宿主再往已经死掉的 isolate 发消息。

同一 Engine 里的多份播放器、多份视图才使用 `messageChannelSuffix`。后缀不能代替「每个 Engine 各注册一次」。

平台通道按 messenger 隔离。Pigeon 30 的 Native Interop 是另一套模型：`register` 要带上 Engine（Kotlin 的 binding / `FlutterEngine`，Swift 的 registrar），`unregister` 取代 `register(null)`，原生 `@FlutterApi` 用 `getInstance(binding)` / `getInstance(registrar:)` 查找，`HostApi.getInstance(name)` 已删除。Flutter 3.50 之前，Native Interop 还有多 Engine 误绑到错误实例的已知问题。add-to-app 生产路径用本文的平台通道。

### Q6: 测试里 fake 没写到，一跑就 MissingPluginException？

**A**: `FakeBatteryHostApi` 必须覆盖被测方法。没覆盖的方法会走生成类里的 `BasicMessageChannel`。单元测试默认没有宿主实现。

业务类通过构造函数接收 `BatteryHostApi`。测试传入 fake，生产传入 `BatteryHostApi()`。不要在测试里启用 `dartHostTestHandler`。

### Q7: 生成器报 “method or function definitions” 或增强枚举？

**A**: 定义文件只保留 import、`@ConfigurePigeon`、枚举、数据类和抽象 API。把计算逻辑挪到 `lib/battery_client.dart` 或宿主实现。枚举改回没有字段和方法的简单枚举。

---

## 💡 最佳实践

1. **先定方向和错误码，再生成**
   - HostApi 与 FlutterApi 写反时，生成代码仍然能编过，调用方向会整条反掉。

2. **实现前读生成签名**
   - `Long` / `Int64`、枚举大小写、`suspend` / `async throws`、`setUp` 的参数顺序，都以本次生成文件为准。

3. **生成物与定义一起提交**
   - 生成命令写进 README 或 `tool/generate.sh`，方便人重新生成；CI 仍使用已提交的生成物。

4. **Dart 调用包一层可替换的客户端**
   - `BatteryClient(BatteryHostApi api)` 让单元测试不碰 Engine。

5. **主线程只做转发**
   - 查询系统服务若可能等待，放到 `@TaskQueue` 或 suspend 内部的后台 dispatcher，回调 Dart 时回到主线程。

6. **不要把 Pigeon 生成类当作插件的公共 API**
   - 官方稳定性说明：生成代码允许破坏性变更。对外暴露你自己的 Dart 类。

---

## 🔗 相关资源

- 从需求生成定义：[prompts/platform/define-pigeon-api.md](../prompts/platform/define-pigeon-api.md)
- 实现宿主与按报错修复：[prompts/platform/implement-and-fix-pigeon.md](../prompts/platform/implement-and-fix-pigeon.md)
- 上下文工程：[docs/context-engineering.md](../docs/context-engineering.md)
- Pigeon 包（30.0.4）：[https://pub.dev/packages/pigeon](https://pub.dev/packages/pigeon)
- 更新日志：[https://pub.dev/packages/pigeon/changelog](https://pub.dev/packages/pigeon/changelog)
- Flutter 官方文档：[https://docs.flutter.dev/interop/pigeon](https://docs.flutter.dev/interop/pigeon)
- FFIgen：[https://docs.flutter.dev/interop/ffigen](https://docs.flutter.dev/interop/ffigen)
- JNIgen：[https://docs.flutter.dev/interop/jnigen](https://docs.flutter.dev/interop/jnigen)
- Pigeon Native Interop 指南（preview）：[https://github.com/flutter/packages/blob/main/packages/pigeon/native_interop_guide.md](https://github.com/flutter/packages/blob/main/packages/pigeon/native_interop_guide.md)
- 官方 example（`@ConfigurePigeon`、`@async`、`FlutterError` / `PigeonError`）：[https://github.com/flutter/packages/blob/main/packages/pigeon/example/README.md](https://github.com/flutter/packages/blob/main/packages/pigeon/example/README.md)
- iOS UIScene 注册迁移：[https://docs.flutter.dev/release/breaking-changes/uiscenedelegate](https://docs.flutter.dev/release/breaking-changes/uiscenedelegate)
- 测试里如何替代插件通道：[https://docs.flutter.dev/testing/plugins-in-tests](https://docs.flutter.dev/testing/plugins-in-tests)

---

**先生成，再注册，再测试。**
