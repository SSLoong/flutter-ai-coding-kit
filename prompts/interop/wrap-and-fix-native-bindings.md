# 提示词:包装原生绑定与修复报错

**用途**:把 FFIgen/JNIgen 生成的绑定包成惯用 Dart API,或根据生成/编译/内存/线程报错修复

---

## 📋 适用场景

- 生成绑定后,需要包装一层惯用 Dart API(官方 Layer 2)
- 生成期或编译期报错,需要调整配置或修复代码
- 运行期内存泄漏、崩溃、线程错误,需要修复生命周期或线程模型
- 想隐藏原生 API 的内存管理和平台差异,提供 async 接口

---

## 📥 输入要求

在 Cursor 中准备:

1. **生成的绑定文件**:`lib/src/*_bindings.g.dart`
2. **配置脚本**(如有报错):`tool/ffigen.dart` 或 `tool/jnigen.dart`
3. **完整报错信息**:
   - 生成期报错:命令行输出的完整错误
   - 编译期报错:`flutter build` 的完整日志
   - 运行期报错:崩溃堆栈、Timeline 卡顿截图
4. **当前代码**(如需包装):调用生成绑定的现有代码

---

## 💬 提示词模板

### 场景 A:包装惯用 Dart API(Layer 2)

```markdown
我有以下 FFIgen/JNIgen 生成的绑定,需要包装一层惯用 Dart API(官方 Layer 2),隐藏内存管理和平台差异。

**生成的绑定**(位置:`lib/src/[name]_bindings.g.dart`):
[贴上生成文件的关键签名,或完整文件如果不长]

**原生 API 用途**:
[简述功能,如「调用 NSURLSession 发起网络请求」或「显示 Android Toast」]

**要求**:
1. 包装后的 API 要符合 Dart 惯用法:
   - 方法名用小驼峰(camelCase)
   - 返回 `Future` 而不是同步调用(如果原生方法可能耗时)
   - 自动管理内存:Objective-C 用 `WeakReference`,JNI 用 `.using()` 或 `.release()`
   - 隐藏平台差异:iOS/Android 的差异在包装层处理
2. 对于耗时的同步调用,要在后台 isolate 执行,不堵 UI
3. 对于 UI 相关的原生调用(如 Toast、UI 属性),要在平台主线程执行
4. 生成完整的包装类代码,包含:
   - 构造函数(如需初始化原生对象)
   - 包装后的方法(async 接口、内存管理)
   - dispose 方法(释放资源)
   - 必要的 import

**输出格式**:
- 包装类的完整代码
- 使用示例
- 内存管理说明(哪些对象需要释放、如何释放)
- 线程约束说明(哪些方法必须在主线程、哪些可以在后台)

**平台**:[iOS / Android / 两者],工具:[FFIgen / JNIgen]
```

---

### 场景 B:修复生成期报错

```markdown
我在生成 FFIgen/JNIgen 绑定时遇到报错,需要调整配置或修复代码。

**配置脚本**(位置:`tool/ffigen.dart` 或 `tool/jnigen.dart`):
[贴上完整配置脚本]

**生成命令**:
`dart run tool/ffigen.dart` 或 `dart run tool/jnigen.dart`

**完整报错信息**:
```
[贴上命令行输出的完整错误,包括 libclang 告警、类找不到、头文件缺失等]
```

**要求**:
1. 分析报错原因
2. 给出修复后的配置脚本(完整的 `tool/ffigen.dart` 或 `tool/jnigen.dart`)
3. 说明为什么这样改,以及还需要做什么(如安装依赖、修改头文件路径)

**平台**:[iOS / Android],工具:[FFIgen / JNIgen],Flutter 版本:[执行 `flutter --version`]
```

---

### 场景 C:修复编译期报错

```markdown
生成 FFIgen/JNIgen 绑定后,编译失败,需要修复代码或配置。

**生成的绑定**(位置:`lib/src/[name]_bindings.g.dart`):
[如果文件不长,贴上完整内容;否则贴上报错涉及的部分]

**调用代码**(位置:`lib/[file].dart`):
[贴上调用生成绑定的代码]

**编译命令**:
`flutter build apk --debug` 或 `flutter build ios --no-codesign --debug`

**完整报错信息**:
```
[贴上完整编译日志,包括文件名、行号、错误类型]
```

**要求**:
1. 分析报错原因(是配置问题、依赖缺失,还是调用代码错误)
2. 给出修复方案:
   - 如果是配置问题,给出修复后的配置脚本并重新生成
   - 如果是依赖缺失,给出要加的 `pubspec.yaml` 片段
   - 如果是调用代码错误,给出修复后的调用代码
3. 说明为什么这样改

**平台**:[iOS / Android],工具:[FFIgen / JNIgen]
```

---

### 场景 D:修复运行期内存/线程错误

```markdown
调用 FFIgen/JNIgen 绑定时,遇到内存泄漏、崩溃或线程错误,需要修复生命周期或线程模型。

**调用代码**(位置:`lib/[file].dart`):
[贴上调用生成绑定的完整代码,包括构造、调用、释放]

**运行期问题**:
[选择一个或多个]:
- [ ] 内存泄漏(Objective-C 循环引用、JNI 全局引用未释放)
- [ ] 崩溃(DoubleReleaseError、UseAfterReleaseError、Cannot call method on nil)
- [ ] UI 卡顿(同步调用耗时操作在 UI isolate)
- [ ] 线程错误(UI API 在后台线程调用、JNI 跨 isolate 错误)

**完整报错信息或现象**:
```
[如果崩溃,贴上堆栈;如果卡顿,描述现象并贴上 Timeline 截图链接]
```

**要求**:
1. 分析问题原因(内存管理、线程模型、isolate 使用)
2. 给出修复后的调用代码(完整的 Dart 文件)
3. 说明:
   - 哪些对象需要 `WeakReference` / `.using()` / `.release()`
   - 哪些调用要在后台 isolate,哪些要在平台主线程
   - 如何验证修复有效(内存未泄漏、UI 不卡顿)

**平台**:[iOS / Android],工具:[FFIgen / JNIgen]
```

---

## ✅ 验证步骤

### 包装后验证

```bash
# 1. Dart 分析
dart analyze lib/your_wrapper.dart

# 2. 单元测试(测试包装逻辑,不依赖原生运行时)
flutter test test/your_wrapper_test.dart

# 3. 真机运行
flutter run --profile
# 验证:调用 → 返回值正确 → 内存未泄漏 → UI 不卡顿
```

### 修复后验证

```bash
# 生成期报错:重新生成
dart run tool/ffigen.dart  # 或 jnigen.dart
# 应该退出码为 0,无报错

# 编译期报错:重新编译
flutter build apk --debug  # 或 ios
# 应该编译通过

# 运行期报错:真机测试
flutter run --profile
# 验证:不崩溃、内存不泄漏(用 DevTools Memory)、UI 不卡顿(用 Timeline)
```

**检查清单**:
- [ ] 生成命令退出码为 0(生成期报错)
- [ ] 编译通过(编译期报错)
- [ ] 真机调用返回值正确(运行期错误)
- [ ] 内存:Objective-C 无循环引用,JNI 对象已释放
- [ ] 线程:同步调用不在 UI isolate(或调用很快),UI 相关调用在平台主线程
- [ ] Timeline 显示无卡顿帧(运行期性能)

---

## 🛠️ 常见问题

### 1. AI 给的包装代码没有管理内存

**提示词改进**:明确要求「对于 Objective-C 闭包,用 WeakReference;对于 JNI 对象,用 .using() 或 .release()」。

---

### 2. AI 给的包装代码在 UI isolate 同步调用耗时操作

**提示词改进**:说明「如果原生方法可能耗时(如文件 I/O、大数据解析),要在后台 isolate 执行,用 Isolate.run()」。

---

### 3. libclang 告警没有解决

**原因**:头文件路径不正确或依赖缺失。

**解决**:
- 检查 `input.entryPoints` 的路径
- 补全头文件依赖(如 Foundation.framework、UIKit.framework)
- 如果是自定义 C 库,确保所有引用的头文件都存在

---

### 4. JNI 报错「Global reference limit exceeded」

**原因**:创建了超过 50,000 个 JNI 全局引用未释放。

**解决**:
- 用 `.using()` 自动释放:

```dart
obj.using((o) {
  // 使用 o
}); // 自动释放
```

- 或循环中显式释放:

```dart
for (var i = 0; i < 100000; i++) {
  final obj = MyClass.new1();
  try {
    // 使用 obj
  } finally {
    obj.release();
  }
}
```

---

### 5. Objective-C 报错「Cannot call method on nil」

**原因**:Objective-C 对象已释放或为 nil。

**解决**:
- 检查对象生命周期,确保对象在使用期间未释放
- 用 `WeakReference` 时,检查 `.target` 是否为 null:

```dart
weakSelf.target?.doSomething(); // 可能为 null
```

---

### 6. UI 卡顿,Timeline 显示长耗时事件

**原因**:同步调用耗时操作在 UI isolate。

**解决**:移到后台 isolate:

```dart
final result = await Isolate.run(() {
  return nativeApi.heavyComputation(data);
});
```

参考 [Isolate 工作流](../../workflows/isolate-background-compute.md)。

---

### 7. Android Toast 不显示或崩溃

**原因**:Toast 必须在平台主线程调用。

**解决**:用 `dart:ui` 的 `runOnPlatformThread`:

```dart
import 'dart:ui' show runOnPlatformThread;

runOnPlatformThread(() {
  Toast.makeText(...).show();
});
```

或在 Kotlin 侧用 `runOnUiThread`。

---

## 🔗 相关

- 工作流:[workflows/ffigen-jnigen-native-interop.md](../../workflows/ffigen-jnigen-native-interop.md)
- 生成配置:[generate-native-bindings.md](generate-native-bindings.md)
- 后台计算:[workflows/isolate-background-compute.md](../../workflows/isolate-background-compute.md)
- Objective-C 内存管理:[https://dart.dev/interop/objective-c-interop#memory-management](https://dart.dev/interop/objective-c-interop#memory-management)
- JNI 对象生命周期:[https://github.com/dart-lang/native/blob/main/pkgs/jnigen/docs/lifecycle.md](https://github.com/dart-lang/native/pkgs/jnigen/docs/lifecycle.md)
