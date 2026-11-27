# FFIgen / JNIgen 直接原生互操作工作流

**从已有 C/Objective-C/Java/Kotlin API 生成 Dart 绑定,实现低延迟同步调用**

> 作者:Loong (SSLoong) | 日期:2026-11-21

本文对齐 pub.dev 上的 **ffigen 23.0.0** 与 **jnigen 1.1.0**（要求 Flutter 3.50 / Dart 3.14 及以上）和 Flutter 官方文档 [FFIgen](https://docs.flutter.dev/interop/ffigen)、[JNIgen](https://docs.flutter.dev/interop/jnigen)。先在本机执行 `flutter --version` 确认版本。

FFIgen 与 JNIgen 生成的是自底向上的镜像绑定,不是跨平台契约。生成后必须在本地编译、单测并验证线程模型。禁止把未跑通的绑定直接合入。

---

## 📋 背景

Pigeon 平台通道每次调用都要序列化、跨平台通道传递、反序列化,适合偶发的结构化调用。但当你需要:

- 调用已经存在的 C/Objective-C/Java/Kotlin 原生 API,而不是重新定义一份跨平台契约
- 大数据量传输(图像缓冲、音频帧)或低延迟场景(传感器轮询)
- 在 Dart 侧同步访问原生对象或内存

FFIgen(Foreign Function Interface generator)和 JNIgen(Java Native Interface generator)能从原生头文件或类直接生成 Dart 绑定,绕过平台通道的序列化开销,但代价是你必须在 Dart 侧管理原生内存生命周期、线程和平台差异。

AI 适合起草生成配置、包装惯用 Dart API,但原生绑定必须在本地重新生成、编译并验证,不要在 UI isolate 里跑重同步调用。

---

## 🎯 目标

- ✅ 判断该用 FFIgen/JNIgen 直接互操作、Pigeon 平台通道,还是手写 MethodChannel
- ✅ 从 C/Objective-C 头文件或 Java/Kotlin 类生成 FFIgen/JNIgen 配置并生成绑定
- ✅ 在生成绑定外包一层惯用 Dart API(官方 Layer 2),隐藏内存管理和线程约束
- ✅ 处理 Objective-C 引用循环、JNI 全局引用限制、同步调用不堵 UI isolate
- ✅ 用单元测试验证绑定调用、内存释放、线程模型

---

## 🔧 适用场景

| 场景 | 适合度 | 说明 |
|------|--------|------|
| 绑定已有 C/Objective-C/Java/Kotlin 库 | ⭐⭐⭐⭐⭐ | 官方推荐自底向上场景 |
| 大数据传输(图像、音频、传感器) | ⭐⭐⭐⭐⭐ | 无序列化开销,可直接操作原生内存 |
| 低延迟同步调用(不能等异步回调) | ⭐⭐⭐⭐⭐ | 可在非 UI isolate 同步调用原生 API |
| 持有原生对象指针或句柄 | ⭐⭐⭐⭐⭐ | 生成的类包装 JObject 或 FFI 指针 |
| 新建跨平台结构化 API | ⭐ | 应优先 Pigeon,它是自顶向下的跨平台契约 |
| 只需偶发调用,数据是枚举和数据类 | ⭐⭐ | Pigeon 平台通道更简单,无需管理内存 |
| 纯 Swift API(无 @objc 注解) | ⭐⭐ | package:swiftgen 仍是 experimental,生产用 Pigeon + Swift |
| C++ 类(不是 extern "C") | ⭐ | FFIgen 只解析 C/Objective-C,C++ 需先暴露 C 接口 |

**怎么选:**

- **FFIgen / JNIgen(本文)**:要绑定已存在的 C/Objective-C/Java/Kotlin API,或在后台 isolate 高频同步访问原生内存。Android 用 JNIgen 绑定 Java/Kotlin,iOS/macOS 用 FFIgen 绑定 Objective-C(含 `@objc` 注解的 Swift)或 C。生成的绑定不是跨平台的,是某一平台 API 的镜像。你要在 Dart 侧管理原生内存生命周期(`release()` / `using()` / `WeakReference`)和线程(UI isolate 禁止重同步调用)。
- **Pigeon 平台通道**:应用或插件里新的一组跨平台调用,数据是枚举、数据类、可空字段,调用频率是用户操作或偶发查询。Pigeon 生成的是自顶向下的契约,两侧签名一致,内存由平台通道管理。见 [Pigeon 工作流](pigeon-platform-channels.md)。
- **手写 MethodChannel**:对方已规定 channel 名、方法名,或方法名到运行时才知道。
- **Pigeon Native Interop(useJni / useFfi)**:官方博客(2026-11-12)和文档仍标为 preview,不作为默认。它不支持 `@TaskQueue`;多 Engine 路由在 Flutter 3.50 之前有已知限制。需要时先读 [native_interop_guide.md](https://github.com/flutter/packages/blob/main/packages/pigeon/native_interop_guide.md) 单独验证。

**与 Pigeon 的互补关系:**

- Pigeon 平台通道:自顶向下,一份 Dart schema 生成跨平台契约,内存自动管理。
- FFIgen / JNIgen:自底向上,从已有原生 API 生成镜像绑定,无序列化但要手动管理内存和线程。
- 两者不冲突:同一插件可以用 Pigeon 做结构化 API(如配置、状态查询),用 FFIgen/JNIgen 做高频数据传输(如视频帧回调)。

**耗时同步调用不堵 UI:**如果生成绑定的方法是同步的且耗时(如大文件解析),不要在 UI isolate 直接调用,要移到后台 isolate。参考 [Isolate 工作流](isolate-background-compute.md)。

---

## 🚀 完整流程

下面用「iOS 上调用系统 NSURLSession 发起网络请求」和「Android 上调用 Java Toast」贯穿流程。包名 `com.example.my_app` / `my_app` 换成你的。

### Step 0:准备上下文

**人工操作**:在 Cursor 里打开 `pubspec.yaml`、目标原生 API 的头文件(或类文件、JAR),以及 Android 的 `android/app/build.gradle`(如需 JNIgen)。确认 Flutter ≥ 3.50、Dart ≥ 3.14。

**判断场景**:
- 是否已有原生 API 的头文件或类文件?
- 是否需要大数据传输或低延迟同步调用?
- 是否需要持有原生对象指针或句柄?
- 是新建跨平台 API 还是绑定单一平台的已有 API?

如果是新建跨平台 API 且数据是结构化的,优先考虑 Pigeon。

---

### Step 1:判断选型并确认工具链

**AI 提示词**(可复制):

```markdown
我要在 Flutter 应用里调用原生 API,请先确认选型,不要写代码:

**需求**:
[简述功能,如「iOS 上用 NSURLSession 发起网络请求」或「Android 上显示 Toast」]

**原生 API**:
- 平台:[iOS / Android / 两者]
- 已有 API:[C 库 / Objective-C 框架 / Swift 类 / Java 类 / Kotlin 类 / JAR/AAR]
- 是否需要大数据传输(>1MB)或高频调用(>10次/秒)?
- 是否需要同步返回结果(不能等异步回调)?
- 是否需要在 Dart 侧持有原生对象指针?

**工程**:
- Flutter 版本:[执行 `flutter --version` 的结果]
- Dart 包名:[pubspec.yaml 的 name]
- Android namespace:[若需 JNIgen]

**请回答**:
1. 应该用 FFIgen、JNIgen、Pigeon 平台通道,还是手写 MethodChannel?
2. 如果用 FFIgen/JNIgen,需要哪些依赖?生成配置放哪?
3. 有哪些常见坑(内存、线程、平台差异)?
4. 生成绑定后,哪些部分需要包一层 Dart API(Layer 2)?

约束:ffigen 23.0.0、jnigen 1.1.0,优先 Dart 配置脚本(不要 YAML)。
```

**检查点**:AI 应明确回答选型理由,以及 FFIgen/JNIgen 是否适合当前场景。如果更适合 Pigeon,停下来不要勉强生成配置。

---

### Step 2:写 FFIgen 配置并生成(iOS / macOS / C 库)

**适用**:绑定 C 库、Objective-C 框架、或带 `@objc` 注解的 Swift 类。纯 Swift(无 `@objc`)用 package:swiftgen(experimental)或 Pigeon。

`pubspec.yaml` 增加依赖:

```yaml
dependencies:
  ffi: ^2.1.0

dev_dependencies:
  ffigen: ^23.0.0
```

**AI 提示词**(可复制):见 [prompts/interop/generate-native-bindings.md](../prompts/interop/generate-native-bindings.md)。

**配置示例**(绑定 iOS 的 Foundation 框架,`tool/ffigen.dart`):

```dart
import 'dart:io';
import 'package:ffigen/ffigen.dart';

Future<void> main() async {
  final packageRoot = Platform.script.resolve('../');
  final generator = FfiGenerator(
    output: Output(
      dart: DartOutput(
        path: packageRoot.resolve('lib/src/foundation_bindings.g.dart'),
      ),
    ),
    input: Input(
      entryPoints: [
        // Objective-C 头文件路径(Xcode SDK)
        '/Applications/Xcode.app/Contents/Developer/Platforms/iPhoneOS.platform/Developer/SDKs/iPhoneOS.sdk/System/Library/Frameworks/Foundation.framework/Headers/NSURLSession.h',
      ],
    ),
    visitors: [
      Visitor(
        // 只生成 NSURLSession 相关的类和方法
        objcinterface: (node) {
          if (node.name == 'NSURLSession' || node.name == 'NSURLSessionConfiguration') {
            node.isIncluded = true;
          }
        },
      ),
    ],
  );
  await generator.generate();
}
```

**Android C/C++ 库**(如绑定自己的 `.so`):用 FFIgen 解析 C 头文件,通过 `DynamicLibrary.open()` 加载,并用 build hooks 编译(见 `flutter create --template=package_ffi`)。C++ 必须先暴露 `extern "C"` 接口。

**生成**:

```bash
flutter pub get
dart run tool/ffigen.dart
```

生成后会出现 `lib/src/foundation_bindings.g.dart`,包含 `NSURLSession` 等类的 Dart 包装。

**关键约束**:
- FFIgen 解析 C/Objective-C 头文件,不能直接解析 C++ 类或模板。C++ 库需先写 `extern "C"` 包装。
- Objective-C 方法动态解析,FFIgen 自动映射 selector。验证时读生成绑定的方法签名。
- 生成期的 libclang 告警必须解决,否则运行时可能崩溃。
- macOS 上的 API 差异用 `Platform.isMacOS` 检查,Dart 无 C 宏。

---

### Step 3:写 JNIgen 配置并生成(Android)

**适用**:绑定 Java/Kotlin 类、JAR/AAR、Android SDK 包。

`pubspec.yaml` 增加依赖:

```yaml
dependencies:
  jni: ^0.11.0

dev_dependencies:
  jnigen: ^1.1.0
```

**AI 提示词**(可复制):见 [prompts/interop/generate-native-bindings.md](../prompts/interop/generate-native-bindings.md)。

**配置示例**(绑定 Android Toast,`tool/jnigen.dart`):

```dart
import 'dart:io';
import 'package:jnigen/jnigen.dart';

Future<void> main() async {
  final packageRoot = Platform.script.resolve('../');
  final generator = JniGenerator(
    output: Output(
      dart: DartOutput(
        path: packageRoot.resolve('lib/src/android_toast.g.dart'),
        structure: OutputStructure.singleFile,
      ),
    ),
    input: Input(
      classes: [
        'android.widget.Toast',
        'android.app.Activity',
        'java.lang.CharSequence',
      ],
      androidSdk: AndroidSdk(
        addGradleDeps: true,
        androidExample: packageRoot,
      ),
    ),
  );
  await generator.generate();
}
```

**生成**:

```bash
flutter pub get
# JNIgen 需要先 pub get Flutter app 才能找到 Gradle 依赖
dart run tool/jnigen.dart
```

生成后会出现 `lib/src/android_toast.g.dart`,包含 `Toast`、`Activity` 等类的 Dart 包装。

**关键约束**:
- JNIgen 使用 JNI 全局引用,有 50,000 个的硬限制。生成的对象必须调用 `.release()` 或用 `.using()` 自动释放。
- Android 上某些 API 必须在平台主线程调用,用 `dart:ui` 的 `runOnPlatformThread`。
- Dart 回调从 Java 来时运行在创建它的 isolate,跨线程等待可能死锁。
- 生成的类是 `extension type` on `JObject`,Dart 的 `is`/`as` 不检查 Java 运行时类型,用 `.isA(Foo.type)` 和 `.as(Foo.type)`。
- 核心 Java 类(如 `java.lang.Integer`、`java.util.List`)已由 `package:jni` 提供,不要在 classes 里重复生成。

---

### Step 4:包装惯用 Dart API(Layer 2)

生成的绑定(Layer 1)是原生 API 的镜像,签名和命名风格是原生的,暴露了内存管理细节。应在其外包一层惯用 Dart API(Layer 2),隐藏平台差异、自动管理内存、提供 async 接口。

**AI 提示词**(可复制):见 [prompts/interop/wrap-and-fix-native-bindings.md](../prompts/interop/wrap-and-fix-native-bindings.md)。

**Objective-C 包装示例**(隐藏 `NSURLSession` 的引用管理):

```dart
import 'dart:ffi';
import 'package:ffi/ffi.dart';
import 'package:my_app/src/foundation_bindings.g.dart';

class HttpClient {
  HttpClient() {
    final config = NSURLSessionConfiguration.defaultSessionConfiguration();
    _session = NSURLSession.sessionWithConfiguration_(config);
  }

  late final NSURLSession _session;

  Future<String> get(String url) async {
    // 在后台 isolate 调用同步的 Objective-C 方法
    return await Isolate.run(() {
      final nsUrl = NSURL.URLWithString_(url.toNSString());
      final request = NSURLRequest.requestWithURL_(nsUrl);
      
      // 同步等待(实际应用应异步,这里简化演示)
      final data = _session.dataTaskWithRequest_(request);
      
      // 转 Dart String
      return data.toStringDart();
    });
  }

  void dispose() {
    // Objective-C ARC 会处理释放,但显式清理是好习惯
    _session = null as dynamic;
  }
}
```

**Java 包装示例**(自动释放 JNI 引用):

```dart
import 'package:jni/jni.dart';
import 'package:my_app/src/android_toast.g.dart';

class ToastHelper {
  static void show(String message) {
    final activity = /* 获取 Activity,见下文 */;
    
    // using() 自动释放 JNI 引用
    activity.using((act) {
      final text = message.toJString();
      text.using((txt) {
        final toast = Toast.makeText(
          act.as(Context.type),
          txt.as(CharSequence.type),
          Toast.LENGTH_SHORT,
        );
        toast.using((t) => t.show());
      });
    });
  }
}
```

**关键点**:
- **Objective-C 内存**:Dart 闭包捕获原生包装对象时,用 `WeakReference` 打破引用循环。官方示例:[Objective-C memory guide](https://dart.dev/interop/objective-c-interop#memory-management)。
- **JNI 内存**:每个 `JObject` 持有全局引用,必须 `.release()` 或 `.using()`。实现接口时用 `WeakReference` 避免 Dart GC 和 Java GC 的循环引用。
- **线程**:iOS 的 UI 相关调用必须在主线程(`@MainActor`)。Android 的某些 API 要在平台主线程(`runOnPlatformThread`)。后台 isolate 可以同步调用原生方法,但不要在 UI isolate 里跑重同步调用。
- **错误处理**:原生异常不会自动变成 Dart 异常,要主动检查返回值或调用原生的错误获取方法。

---

### Step 5:处理平台特定约束

#### Objective-C(iOS / macOS)

**引用循环**:

```dart
// ❌ 错误:闭包捕获 self,导致循环引用
button.onTap = () {
  self.doSomething(); // self 强引用 button,button 强引用闭包
};

// ✅ 正确:用 WeakReference
final weakSelf = WeakReference(self);
button.onTap = () {
  weakSelf.target?.doSomething();
};
```

**运行时类型检查**:

```dart
// ❌ 错误:Dart 的 is/as 不检查 Objective-C 运行时类型
if (obj is NSString) { } // 静态类型,不检查运行时

// ✅ 正确:用 .isA() 和 .as()
if (obj.isA(NSString.classObject)) {
  final str = obj.as(NSString.type);
}
```

**多线程**:UI 相关调用必须在主线程。从后台回调 Dart 时,先回到 main actor:

```swift
Task { @MainActor in
  await dartCallback(result)
}
```

#### Java / Kotlin(Android)

**全局引用限制**:JNI 全局引用上限 50,000。大量对象时用 `.using()` 或显式 `.release()`:

```dart
// ❌ 错误:循环创建对象不释放
for (var i = 0; i < 100000; i++) {
  final obj = MyClass.new1(); // 泄漏全局引用
}

// ✅ 正确:自动释放
for (var i = 0; i < 100000; i++) {
  MyClass.new1().using((obj) {
    // 使用 obj
  }); // 自动释放
}
```

**调试内存泄漏**:设置 `Jni.captureStackTraceOnRelease = true`,日志会记录每次 `release()` 的调用栈,帮助定位 `DoubleReleaseError` 和 `UseAfterReleaseError`。

**线程约束**:JNI 调用在调用线程同步执行。平台主线程 API 要在 `runOnPlatformThread` 里调用;Dart 回调从 Java 来时运行在创建它的 isolate,跨线程等待可能死锁。

**获取 Activity**:

```dart
import 'dart:ui' show PlatformDispatcher;
import 'package:jni/jni.dart';

// Flutter 3.0+ 在 Android 上
final activity = androidActivity(PlatformDispatcher.instance.engineId!);
if (activity != null) {
  activity.using((act) {
    // 使用 Activity
  });
}
```

---

### Step 6:本地验证(必须)

**人工操作**:

```bash
# 1. 重新生成,避免手改生成物
dart run tool/ffigen.dart  # 或 jnigen.dart

# 2. Dart 分析
dart analyze lib/src/*_bindings.g.dart lib/your_wrapper.dart

# 3. 单元测试(不能完全覆盖原生调用,但可测包装逻辑)
flutter test test/your_wrapper_test.dart

# 4. 编译宿主
flutter build apk --debug  # Android
flutter build ios --no-codesign --debug  # iOS

# 5. 真机运行并验证
flutter run --profile
# 验证:调用生成绑定 → 检查返回值 → 确认内存未泄漏 → 线程正确
```

**检查清单**:
- [ ] 生成命令退出码为 0,生成文件已更新
- [ ] `dart analyze` 无 error(warning 可能是 libclang 告警,必须解决)
- [ ] 编译通过(`flutter build apk` / `flutter build ios`)
- [ ] 真机上调用生成绑定,返回值正确
- [ ] 内存:Objective-C 无循环引用,JNI 对象已释放(用 `.using()` 或显式 `.release()`)
- [ ] 线程:同步调用不在 UI isolate(或调用很快<5ms),UI 相关调用在平台主线程
- [ ] 原生 API 变更后,重新生成并重新测试

**常见失败**:
- `Invalid argument: <对象> cannot be sent`:尝试在 isolate 间传递不可序列化的原生对象。应只传递数据(String、List),不传递绑定对象。
- `DoubleReleaseError` / `UseAfterReleaseError`:JNI 对象被释放两次或释放后使用。检查 `.using()` 嵌套和手动 `.release()`。
- `Cannot call Objective-C method on nil`:Objective-C 对象已释放或为 nil。检查生命周期。
- UI 卡顿:同步调用耗时操作在 UI isolate。移到后台 isolate(见 [Isolate 工作流](isolate-background-compute.md))。
- 编译期 libclang 告警:头文件不完整或依赖缺失,必须解决,否则运行时可能崩溃。

---

### Step 7:迭代修复

生成器报错、编译错误、运行期崩溃,把**配置脚本、生成签名和完整报错**一起交给 AI。提示词用 [prompts/interop/wrap-and-fix-native-bindings.md](../prompts/interop/wrap-and-fix-native-bindings.md)。

修复原则:改配置脚本(如 `tool/ffigen.dart`)或包装代码,然后重新生成。生成文件头部写着 Autogenerated,不要在里面改签名。

---

## ✅ 验证清单

完成后确认:

- [ ] 配置脚本在 `tool/` 下,`ffigen`/`jnigen` 只出现在 `dev_dependencies`
- [ ] 生成绑定的 `*.g.dart` 提交到 Git
- [ ] 包装了 Layer 2 的惯用 Dart API,隐藏内存管理和平台差异
- [ ] Objective-C 闭包用 `WeakReference`,JNI 对象用 `.using()` 或 `.release()`
- [ ] 同步调用不在 UI isolate(或调用很快),UI 相关调用在平台主线程
- [ ] `flutter build apk` 与 `flutter build ios` 通过
- [ ] 真机上验证调用、返回值、内存、线程正确

---

## 🤔 FAQ

### Q1:FFIgen / JNIgen 和 Pigeon 有什么区别?

**A**:
- **Pigeon 平台通道**:自顶向下,一份 Dart schema 生成跨平台契约。数据序列化后跨平台通道传递,内存自动管理。适合新建跨平台 API,数据是枚举和数据类,调用频率是用户操作。
- **FFIgen / JNIgen**:自底向上,从已有原生 API 生成镜像绑定。无序列化,可同步调用,但要手动管理内存和线程。适合绑定已有 C/Objective-C/Java/Kotlin API,或大数据传输。
- **两者互补**:同一插件可以用 Pigeon 做结构化 API,用 FFIgen/JNIgen 做高频数据传输。

**简单判断**:
- 新建跨平台 API,数据是结构化的 → Pigeon
- 绑定已有原生 API,需要高性能或持有原生对象 → FFIgen/JNIgen

### Q2:生成的绑定可以在 UI isolate 直接调用吗?

**A**:视调用耗时而定。
- ✅ **可以**:调用很快(<5ms),如查询属性、简单计算。
- ❌ **不可以**:调用耗时(如大文件解析、图像处理),会堵住 UI 导致卡顿。

**正确做法**:耗时的同步调用移到后台 isolate:

```dart
final result = await Isolate.run(() {
  // 在后台 isolate 同步调用原生 API
  return nativeApi.heavyComputation(data);
});
```

参考 [Isolate 工作流](isolate-background-compute.md)。

### Q3:Objective-C 闭包捕获对象时为什么要用 WeakReference?

**A**:防止循环引用导致内存泄漏。

```dart
// ❌ 错误:self 强引用 button,button 的闭包强引用 self,循环引用
class MyView {
  late NSButton button;
  
  void setup() {
    button.onTap = () {
      self.doSomething(); // 捕获 self
    };
  }
}

// ✅ 正确:用 WeakReference 打破循环
void setup() {
  final weakSelf = WeakReference(this);
  button.onTap = () {
    weakSelf.target?.doSomething(); // 弱引用,可能为 null
  };
}
```

官方文档:[Objective-C memory management](https://dart.dev/interop/objective-c-interop#memory-management)。

### Q4:JNI 全局引用限制 50,000 是什么意思?

**A**:JNI 为每个从 Java 传到 Dart 的对象创建一个全局引用,防止 Java GC 回收。但全局引用数量有硬限制 50,000。

**如何避免**:
- 用 `.using()` 自动释放:

```dart
obj.using((o) {
  // 使用 o
}); // 自动释放
```

- 或显式调用 `.release()`:

```dart
final obj = MyClass.new1();
try {
  // 使用 obj
} finally {
  obj.release();
}
```

**调试**:设置 `Jni.captureStackTraceOnRelease = true`,日志会记录每次 `release()` 的调用栈。

### Q5:为什么 Dart 的 `is` 和 `as` 不能检查原生对象类型?

**A**:
- **Objective-C**:FFIgen 生成的类在 Dart 侧是静态类型,运行时都是 `Pointer<ObjCObject>`,Dart 的 `is`/`as` 只检查 Dart 类型,不检查 Objective-C 运行时类型。
- **Java/Kotlin**:JNIgen 生成的类是 `extension type` on `JObject`,编译后都是 `JObject`,Dart 的 `is`/`as` 不检查 Java 运行时类型。

**正确做法**:
- Objective-C:用 `.isA()` 和 `.as()`:

```dart
if (obj.isA(NSString.classObject)) {
  final str = obj.as(NSString.type);
}
```

- Java/Kotlin:用 `.isA(Foo.type)` 和 `.as(Foo.type)`:

```dart
if (obj.isA(MyClass.type)) {
  final myObj = obj.as(MyClass.type);
}
```

### Q6:C++ 库可以用 FFIgen 生成绑定吗?

**A**:❌ **不能直接生成**。FFIgen 只解析 C/Objective-C 头文件,不能解析 C++ 类、模板、命名空间。

**解决方案**:为 C++ 库写一个 `extern "C"` 包装,暴露 C 接口,然后用 FFIgen 生成绑定:

```cpp
// C++ 原始 API
class MyClass {
public:
  void doSomething();
};

// C 包装(my_wrapper.h)
#ifdef __cplusplus
extern "C" {
#endif

typedef void* MyClassHandle;
MyClassHandle my_class_create();
void my_class_do_something(MyClassHandle handle);
void my_class_destroy(MyClassHandle handle);

#ifdef __cplusplus
}
#endif
```

然后 FFIgen 解析 `my_wrapper.h`。

### Q7:纯 Swift API(无 @objc)可以用 FFIgen 生成绑定吗?

**A**:❌ **FFIgen 不支持**。FFIgen 只解析 Objective-C 和带 `@objc` 注解的 Swift。

**解决方案**:
- **推荐**:用 Pigeon 定义跨平台契约,Swift 侧实现 HostApi。
- **实验性**:Dart 团队的 [package:swiftgen](https://pub.dev/packages/swiftgen) 可以直接绑定纯 Swift,但官方标为 experimental,不建议生产使用。

### Q8:生成绑定后,原生 API 变更了怎么办?

**A**:重新生成。生成的绑定是原生 API 的镜像,原生签名变了,绑定必须重新生成才能匹配。

```bash
# 修改原生头文件或类后
dart run tool/ffigen.dart  # 或 jnigen.dart
# 重新编译和测试
flutter build apk --debug
flutter test
```

不要手改生成文件,会在下次生成时被覆盖。

### Q9:如何在后台 isolate 中使用生成的绑定?

**A**:可以,但要注意:
- **FFIgen(Objective-C/C)**:可以在任何 isolate 调用,但 UI 相关 API 必须在主线程。
- **JNIgen(Java/Kotlin)**:可以在任何 isolate 调用,但某些 Android API 要在平台主线程。

**示例**:

```dart
// 在后台 isolate 调用原生 API
final result = await Isolate.run(() {
  return myNativeApi.compute(data); // 同步调用
});
```

**约束**:
- 原生对象不能在 isolate 间传递,只能传递数据(String、int、List)。
- 每个 isolate 的 JNI 引用独立,不能在 isolate A 创建对象,在 isolate B 释放。

### Q10:生成的绑定如何单元测试?

**A**:生成绑定本身难以单元测试(依赖原生运行时),应测试包装后的 Layer 2 API。

**策略**:
- **Mock 包装层**:为 Layer 2 API 定义接口,单元测试 mock 接口,集成测试用真实绑定。

```dart
// 接口
abstract class HttpClient {
  Future<String> get(String url);
}

// 真实实现(用 FFIgen 绑定)
class NativeHttpClient implements HttpClient { }

// Mock 实现(用于单元测试)
class MockHttpClient implements HttpClient { }
```

- **集成测试**:在真机上测试生成绑定的调用、返回值、内存释放。

```bash
flutter test integration_test/native_bindings_test.dart
```

---

## 💡 最佳实践

### 1. 优先考虑是否真的需要 FFIgen/JNIgen

**决策树**:

```
是新建跨平台 API 吗?
  └─ 是 → ❌ 用 Pigeon,不要 FFIgen/JNIgen
  └─ 否 → 继续

已有原生 API 的头文件或类文件吗?
  └─ 否 → ❌ 没有原生 API,不需要生成绑定
  └─ 是 → 继续

需要大数据传输(>1MB)或高频调用(>10次/秒)吗?
  └─ 否 → 🤔 考虑 Pigeon,更简单
  └─ 是 → ✅ 用 FFIgen/JNIgen
```

### 2. 配置用 Dart 脚本,不要 YAML

```dart
// ✅ 推荐:Dart 脚本(tool/ffigen.dart)
import 'package:ffigen/ffigen.dart';
Future<void> main() async {
  final generator = FfiGenerator(...);
  await generator.generate();
}

// ❌ 避免:YAML 配置(已 deprecated)
// ffigen.yaml
```

YAML 配置已废弃,未来版本会移除。

### 3. 只生成需要的 API,减少生成文件大小

```dart
// ❌ 避免:生成整个框架
visitors: [
  Visitor(
    objcinterface: (node) => node.isIncluded = true, // 全部生成
  ),
],

// ✅ 推荐:只生成需要的类
visitors: [
  Visitor(
    objcinterface: (node) {
      if (node.name == 'NSURLSession' || node.name == 'NSURL') {
        node.isIncluded = true;
      }
    },
  ),
],
```

### 4. 包装 Layer 2 API,隐藏内存管理

```dart
// ❌ 避免:暴露生成绑定给业务代码
import 'package:my_app/src/bindings.g.dart';
final obj = MyNativeClass.new1();
obj.doSomething();
obj.release(); // 业务代码要管理内存

// ✅ 推荐:包装一层
class MyService {
  Future<void> doSomething() async {
    await Isolate.run(() {
      MyNativeClass.new1().using((obj) {
        obj.doSomething();
      }); // 自动释放
    });
  }
}
```

### 5. 用 WeakReference 和 using() 避免内存泄漏

```dart
// Objective-C:闭包用 WeakReference
final weakSelf = WeakReference(this);
button.onTap = () {
  weakSelf.target?.doSomething();
};

// JNI:用 using() 自动释放
obj.using((o) {
  o.doSomething();
});
```

### 6. 同步调用移到后台 isolate

```dart
// ❌ 避免:在 UI isolate 同步调用耗时操作
final result = nativeApi.heavyComputation(data); // 堵 UI

// ✅ 推荐:在后台 isolate 调用
final result = await Isolate.run(() {
  return nativeApi.heavyComputation(data);
});
```

### 7. 解决 libclang 告警

生成期的 libclang 告警必须解决:

```bash
dart run tool/ffigen.dart
# 如果出现 warning: implicit declaration of function 'xxx'
# 检查头文件路径、依赖是否完整
```

告警通常意味着头文件不完整,运行时可能崩溃。

### 8. 生成文件提交到 Git

```bash
git add lib/src/*_bindings.g.dart
git commit -m "生成 FFIgen/JNIgen 绑定"
```

不要依赖 CI 重新生成,生成配置可能变化。

### 9. 平台差异用运行时检查,不要 #if

```dart
// ❌ 错误:Dart 无 C 宏
#if TARGET_OS_IPHONE
  // iOS 专有代码
#endif

// ✅ 正确:运行时检查
if (Platform.isIOS) {
  // iOS 专有代码
} else if (Platform.isMacOS) {
  // macOS 专有代码
}
```

### 10. 调试内存问题

```dart
// JNI 内存泄漏调试
Jni.captureStackTraceOnRelease = true;
// 日志会记录每次 release() 的调用栈
```

---

## 🔗 相关资源

- 提示词:[prompts/interop/generate-native-bindings.md](../prompts/interop/generate-native-bindings.md) — 从原生 API 生成绑定配置
- 提示词:[prompts/interop/wrap-and-fix-native-bindings.md](../prompts/interop/wrap-and-fix-native-bindings.md) — 包装 Layer 2 API 与修复报错
- 互补工作流:[workflows/pigeon-platform-channels.md](pigeon-platform-channels.md) — Pigeon 平台通道
- 性能优化:[workflows/isolate-background-compute.md](isolate-background-compute.md) — 后台 isolate 计算
- Flutter 官方文档:[Native interop](https://docs.flutter.dev/interop)
- FFIgen 官方文档:[https://docs.flutter.dev/interop/ffigen](https://docs.flutter.dev/interop/ffigen)
- JNIgen 官方文档:[https://docs.flutter.dev/interop/jnigen](https://docs.flutter.dev/interop/jnigen)
- FFIgen 包(23.0.0):[https://pub.dev/packages/ffigen](https://pub.dev/packages/ffigen)
- JNIgen 包(1.1.0):[https://pub.dev/packages/jnigen](https://pub.dev/packages/jnigen)
- package:jni:[https://pub.dev/packages/jni](https://pub.dev/packages/jni)
- Objective-C 内存管理:[https://dart.dev/interop/objective-c-interop#memory-management](https://dart.dev/interop/objective-c-interop#memory-management)
- Objective-C 运行时类型:[https://dart.dev/interop/objective-c-interop#runtime-types](https://dart.dev/interop/objective-c-interop#runtime-types)
- Objective-C 线程:[https://dart.dev/interop/objective-c-interop#threading](https://dart.dev/interop/objective-c-interop#threading)
- JNIgen 线程指南:[https://github.com/dart-lang/native/blob/main/pkgs/jnigen/docs/threading.md](https://github.com/dart-lang/native/blob/main/pkgs/jnigen/docs/threading.md)
- JNI 对象生命周期:[https://github.com/dart-lang/native/blob/main/pkgs/jnigen/docs/lifecycle.md](https://github.com/dart-lang/native/blob/main/pkgs/jnigen/docs/lifecycle.md)
- JNIgen 调试指南:[https://github.com/dart-lang/native/blob/main/pkgs/jnigen/docs/debugging.md](https://github.com/dart-lang/native/blob/main/pkgs/jnigen/docs/debugging.md)
- Flutter 3.50 发布博客:[https://flutter.dev/blog/whats-new-in-flutter-3-50](https://flutter.dev/blog/whats-new-in-flutter-3-50)
- 稳定原生互操作博客(2026-11-12):[https://flutter.dev/blog/stable-native-interop](https://flutter.dev/blog/stable-native-interop)
- Dart 3.14 发布博客:[https://dart.dev/blog/announcing-dart-3-14](https://dart.dev/blog/announcing-dart-3-14)

---

**直连原生 API,榨干性能 🚀**
