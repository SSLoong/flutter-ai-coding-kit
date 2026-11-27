# 提示词:从原生 API 生成 FFIgen / JNIgen 绑定配置

**用途**:从已有 C/Objective-C 头文件或 Java/Kotlin API 生成 FFIgen 或 JNIgen 配置与绑定

---

## 📋 适用场景

- 绑定已有 C 库、Objective-C 框架、Java/Kotlin 类
- 为原生 API 生成 Dart 直接调用绑定
- 需要高性能、低延迟或大数据传输
- 在选型阶段,判断是否适合用 FFIgen/JNIgen

---

## 📥 输入要求

在 Cursor 中准备:

1. **原生 API 信息**:
   - 平台:iOS / Android / macOS / Linux / Windows
   - API 类型:C 库 / Objective-C 框架 / Swift 类(带 @objc)/ Java 类 / Kotlin 类 / JAR/AAR
   - 头文件路径或类的完全限定名
   - 需要绑定哪些类、方法、属性(不要全部生成,只生成需要的)

2. **工程路径**:
   - `pubspec.yaml` 的 `name`
   - Android `namespace`(如需 JNIgen)
   - 希望的输出路径(默认 `lib/src/xxx_bindings.g.dart`)

3. **使用场景**:
   - 是否需要大数据传输(>1MB)或高频调用(>10次/秒)?
   - 是否需要同步返回结果(不能等异步回调)?
   - 是否需要持有原生对象指针或句柄?

4. **版本约束**:
   - Flutter ≥ 3.50、Dart ≥ 3.14
   - ffigen 23.0.0、jnigen 1.1.0

---

## 💬 提示词模板

```markdown
为下面的原生 API 生成 FFIgen 或 JNIgen 绑定配置。先判断是否适合用直接互操作,再输出配置、依赖和生成命令,不要写包装代码。

**原生 API**:
- 平台:[iOS / Android / macOS / Linux / Windows]
- API 类型:[C 库 / Objective-C 框架 / Java 类 / Kotlin 类 / JAR/AAR]
- 具体 API:[类名或头文件路径,如 NSURLSession、android.widget.Toast、libsqlite3]
- 需要绑定的方法:[列出需要调用的方法或属性,不要全部生成]

**使用场景**:
[简述功能,如「iOS 上用 NSURLSession 发起网络请求」或「Android 上显示 Toast」]

- 是否需要大数据传输(>1MB)或高频调用(>10次/秒)?[是/否]
- 是否需要同步返回结果(不能等异步回调)?[是/否]
- 是否需要持有原生对象指针或句柄?[是/否]

**工程**:
- Dart 包名:`[pubspec_name]`
- Android namespace:`[android_namespace]`(如需 JNIgen)
- 配置脚本路径:`tool/ffigen.dart` 或 `tool/jnigen.dart`
- Dart 输出路径:`lib/src/[name]_bindings.g.dart`

**版本约束**:
- Flutter ≥ 3.50、Dart ≥ 3.14
- 使用 `ffigen: ^23.0.0` 或 `jnigen: ^1.1.0`
- 配置用 Dart 脚本,不要 YAML(已 deprecated)

**请先回答**:
1. 这个场景适合用 FFIgen/JNIgen 吗?还是更适合 Pigeon 平台通道或手写 MethodChannel?
2. 如果适合,用 FFIgen 还是 JNIgen?

**如果适合,请输出**:
1. `pubspec.yaml` 里要加的依赖片段(dependencies 和 dev_dependencies)
2. 完整的配置脚本(`tool/ffigen.dart` 或 `tool/jnigen.dart`)
   - 只生成需要的类和方法,不要全部生成
   - FFIgen:用 `visitors` 过滤需要的类
   - JNIgen:在 `input.classes` 里列出完全限定名
3. 生成命令:`dart run tool/ffigen.dart` 或 `dart run tool/jnigen.dart`
4. 生成后我应:
   - 打开哪个生成文件,核对哪些签名
   - 有哪些常见坑(内存、线程、平台差异)
   - 下一步如何包装 Layer 2 的惯用 Dart API

**如果不适合直接互操作**:
说明原因(如「这是新建跨平台 API,应该用 Pigeon」或「没有已有原生 API,无需生成绑定」),并停下来,不要勉强生成配置。
```

---

## ✅ 验证步骤

生成配置后执行:

```bash
flutter pub get
dart run tool/ffigen.dart  # 或 jnigen.dart
git status --short lib/src tool
```

**检查清单**:
- [ ] AI 先判断了选型,没有勉强生成不适合的绑定
- [ ] 配置脚本是 Dart(不是 YAML)
- [ ] 只生成了需要的类和方法(不是整个框架)
- [ ] 生成命令退出码为 0
- [ ] `lib/src/*_bindings.g.dart` 出现在 git status 里
- [ ] 生成文件中的包名与工程一致
- [ ] AI 说明了下一步如何包装 Layer 2 API

---

## 🛠️ 常见问题

### 1. AI 生成了 YAML 配置而不是 Dart 脚本

**原因**:YAML 配置已 deprecated,但 AI 可能基于旧文档。

**解决**:明确要求「配置用 Dart 脚本(tool/ffigen.dart),不要 YAML」。

---

### 2. AI 生成了整个框架的绑定,文件巨大

**原因**:没有用 `visitors` 过滤。

**解决**:在提示词里列出需要绑定的具体类和方法,要求「只生成需要的 API,减少生成文件大小」。

---

### 3. FFIgen 报错「找不到头文件」

**原因**:头文件路径不正确,或依赖缺失。

**解决**:
- iOS/macOS:检查 Xcode SDK 路径,通常在 `/Applications/Xcode.app/Contents/Developer/Platforms/iPhoneOS.platform/...`
- 自定义 C 库:确保头文件在 `input.entryPoints` 的路径下

---

### 4. JNIgen 报错「找不到类」

**原因**:类的完全限定名不正确,或 Gradle 依赖缺失。

**解决**:
- 确保 `input.classes` 里的类名是完全限定名(如 `android.widget.Toast`)
- 如果是 Gradle 依赖的类,确保 `androidSdk.addGradleDeps: true`
- 先执行 `flutter pub get`,让 JNIgen 能找到 Gradle 依赖

---

### 5. AI 建议用 Pigeon 而不是 FFIgen/JNIgen

**A**:听从建议。如果场景是新建跨平台 API,数据是结构化的,Pigeon 更简单且类型安全。FFIgen/JNIgen 适合绑定已有原生 API 或高性能场景。

---

### 6. C++ 库无法生成绑定

**原因**:FFIgen 只解析 C/Objective-C,不能直接解析 C++ 类。

**解决**:为 C++ 库写一个 `extern "C"` 包装,暴露 C 接口,然后用 FFIgen 生成绑定。

---

### 7. 纯 Swift API(无 @objc)无法生成绑定

**原因**:FFIgen 只解析 Objective-C 和带 `@objc` 注解的 Swift。

**解决**:
- 推荐:用 Pigeon 定义跨平台契约,Swift 侧实现 HostApi
- 实验性:[package:swiftgen](https://pub.dev/packages/swiftgen) 可以绑定纯 Swift,但官方标为 experimental

---

## 🔗 相关

- 工作流:[workflows/ffigen-jnigen-native-interop.md](../../workflows/ffigen-jnigen-native-interop.md)
- 包装与修复:[wrap-and-fix-native-bindings.md](wrap-and-fix-native-bindings.md)
- Pigeon 选型:[workflows/pigeon-platform-channels.md](../../workflows/pigeon-platform-channels.md)
- FFIgen 官方文档:[https://docs.flutter.dev/interop/ffigen](https://docs.flutter.dev/interop/ffigen)
- JNIgen 官方文档:[https://docs.flutter.dev/interop/jnigen](https://docs.flutter.dev/interop/jnigen)
