# 提示词：定义 Pigeon API

**用途**：从业务需求生成 Pigeon 定义文件和 `@ConfigurePigeon` 生成配置

---

## 📋 适用场景

- 新建 Dart ↔ Android/iOS 的类型安全接口
- 为已有功能补一份 `HostApi` / `FlutterApi`，替代手写方法名字符串
- 调整参数、枚举、错误码前，先改定义再重新生成

---

## 📥 输入要求

在 Cursor 中准备：

1. **调用清单**：每个方法谁发起（Dart 还是宿主）、参数、返回值、是否可空
2. **数据模型**：字段名、类型、哪些是枚举
3. **错误码**：宿主失败时 Dart 需要区分的 `code`（如 `unavailable`、`permission-denied`）
4. **工程路径**：
   - `pubspec.yaml` 的 `name`
   - Android `namespace`（`android/app/build.gradle` 或 `build.gradle.kts`）
   - 希望的输出路径（默认可沿用下方模板）
5. **运行约束**：Flutter ≥ 3.44，pigeon 30.0.4，只要平台通道

---

## 💬 提示词模板

```markdown
为下面的需求写一份 Pigeon 定义文件。只输出定义、pubspec 片段和生成命令，不要写宿主实现。

**需求**：[用一两段话描述能力，例如「Dart 读取电量和充电状态，宿主在电量变化时回调 Dart」]

**调用清单**：
- Dart → 宿主：
  - `[methodName]([params]) -> [return]`：[是否可空；是否需要等待系统回调]
- 宿主 → Dart：
  - `[methodName]([params]) -> [return]`：[Dart 侧如何处理]

**数据模型**：
- 枚举 `[EnumName]`：[case1, case2]
- 类 `[ClassName]`：
  - `[field]`: `[type]`，[必填或可空]

**错误码**：
- `[code]`：[何时抛出，details 里放什么]

**工程**：
- Dart 包名：`[pubspec_name]`
- Android package / namespace：`[android_namespace]`
- 定义文件：`pigeons/[api_name].dart`
- Dart 输出：`lib/src/[api_name].g.dart`
- Kotlin 输出：`android/app/src/main/kotlin/[android_namespace_path]/[ApiName].g.kt`
- Swift 输出：`ios/Runner/[ApiName].g.swift`

**版本与范围**：

- 使用 `pigeon: ^30.0.4`（Flutter 3.44 / Dart 3.12+）
- 只生成平台通道。不要设置 `useJni`、`useFfi`，不要生成 C++/Objective-C，除非我在上面点名
- 定义文件放在 `lib/` 外，顶部 `import 'package:pigeon/pigeon.dart';`
- 用 `@ConfigurePigeon(PigeonOptions(...))` 写死输出路径、`kotlinOptions: KotlinOptions(package: '[android_namespace]')` 和 `dartPackageName: '[pubspec_name]'`
- API 写成 `abstract class`，分别标注 `@HostApi()` 或 `@FlutterApi()`
- 文件中不要写方法体，不要使用增强枚举、`DateTime`、`Uri`、record、函数类型
- 自定义类字段只用 bool / int / double / String / typed data / List / Map / 本文件枚举和类，以及它们的可空形式
- 需要挂起等待宿主结果时，在 HostApi 方法上标 `@async`（生成 Kotlin `suspend` 与 Swift `async throws`）
- 只有在我明确要求回调风格时才用 `@asyncCallback`
- 耗时、但不需要 suspend 的 HostApi 方法标 `@TaskQueue(type: TaskQueueType.serialBackgroundThread)`
- 不要使用已废弃的 `dartHostTestHandler` 或 `dartTestOut`

**请输出**：

1. 完整的 `pigeons/[api_name].dart`（含全部 import）
2. `pubspec.yaml` 里要加的 `dev_dependencies` 片段：`pigeon: ^30.0.4`
3. 生成命令：`dart run pigeon --input pigeons/[api_name].dart`
4. 生成后我应打开哪些文件，核对哪些签名（Dart `Future`、Kotlin `Long`/`suspend`、Swift `Int64`/`async throws`、两端 `setUp`）
5. 哪些生成文件必须提交到 Git

如果需求更适合手写 MethodChannel（既有 channel 协议不能改、方法名动态）或 FFI（C/C++、裸指针），先说明原因并停下来，不要勉强生成 Pigeon 定义。
```

---

## ✅ 验证步骤

生成定义后执行：

```bash
flutter pub get
dart run pigeon --input pigeons/[api_name].dart
git status --short pigeons lib/src android/app/src/main/kotlin ios/Runner
```

**检查清单**：

- [ ] `dart run pigeon` 退出码为 0
- [ ] `lib/src/[api_name].g.dart`、Kotlin、Swift 三份生成物都出现在 `git status` 里
- [ ] 生成文件中的包名、`dartPackageName` 与工程一致
- [ ] HostApi 在 Dart 侧是返回 `Future` 的具体类；FlutterApi 是带 `setUp` 的抽象类
- [ ] 定义文件没有被 `lib/` 下的代码 import

---

## 🛠️ 常见问题

1. **生成器拒绝方法体或增强枚举**
   - 定义文件只留声明。逻辑放到宿主实现或 Dart 包装类。

2. **`*.g.dart` 没有出现在 git status**
   - `.gitignore` 忽略了 `*.g.dart`。为这个 Pigeon 输出加否定规则，或调整忽略范围。三端生成物都要提交。

3. **AI 写了 `flutter pub run pigeon`、`dartHostTestHandler` 或 `useFfi: true`**
   - 命令改为 `dart run pigeon --input ...`。测试不走 `dartHostTestHandler`。本文默认不要打开 Native Interop。

4. **Kotlin 输出没有 `package` 行，或 package 和目录不一致**
   - 在 `KotlinOptions(package: '...')` 里写 Android namespace，并让 `kotlinOut` 的目录与之对应，然后重新生成。

---

## 🔗 相关

- 工作流：[workflows/pigeon-platform-channels.md](../../workflows/pigeon-platform-channels.md)
- 实现与修复：[implement-and-fix-pigeon.md](implement-and-fix-pigeon.md)
- Pigeon 30.0.4：[https://pub.dev/packages/pigeon](https://pub.dev/packages/pigeon)
- 官方文档：[https://docs.flutter.dev/interop/pigeon](https://docs.flutter.dev/interop/pigeon)
