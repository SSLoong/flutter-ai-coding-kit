# 提示词：审计 Wasm 兼容性与 JS interop

**用途**：根据 `pubspec.yaml`、条件导入和 `flutter build web --no-deprecated-js-interop` / `--wasm` 输出，判断项目能否上 Wasm，列出必须迁移的遗留 interop 与传递依赖，给出明确结论和验证步骤。

---

## 📋 适用场景

- 现有 Flutter Web 项目要升级到 Wasm 编译
- 检查项目代码与依赖是否兼容 `package:web` 与 `dart:js_interop`
- 识别传递依赖（pub 包）中的遗留 JS interop

---

## 📥 输入要求

在 Cursor 中准备以下上下文：

1. **pubspec.yaml**：打开并附上完整依赖列表（`dependencies` 与 `dev_dependencies`）
2. **构建报错输出**：运行以下命令并粘贴完整输出（包含依赖树 Context）
   ```bash
   flutter build web --wasm
   # 或
   flutter build web --no-deprecated-js-interop
   ```
3. **项目代码文件**：打开可能使用 `dart:html`、`dart:js` 的文件（如 `lib/**/*.dart`）

---

## 🎯 提示词模板

```markdown
我要审计 Flutter Web 项目的 Wasm 兼容性与 JS interop。

**Flutter 版本**：
[运行 `flutter --version` 并粘贴结果]

**目标浏览器**：
[如：Chrome/Edge 137+、Safari 26.2+、Firefox 147+，或只跑 Chromium]

**构建报错输出**：

[粘贴 `flutter build web --wasm` 或 `flutter build web --no-deprecated-js-interop` 的完整输出，包含依赖树 Context]

示例：
```
Error: The unavailable library 'dart:html' is imported through these packages:
    main.dart => package:my_app => dart:html
Detailed import paths for (some of) these imports:
    main.dart => package:my_app/lib/utils/browser_helper.dart => dart:html
```

**pubspec.yaml 依赖**：

```yaml
dependencies:
  flutter:
    sdk: flutter
  url_launcher: ^6.3.1
  shared_preferences: ^2.3.2
  # [粘贴其他依赖]

dev_dependencies:
  flutter_test:
    sdk: flutter
```

**分析要求**：

1. **识别不兼容库**：
   - 哪些项目代码文件直接使用了 `dart:html`、`dart:js`、`dart:js_util`、`package:js`？
   - 哪些传递依赖（pub 包）导致不兼容？

2. **判断迁移可行性**：
   - 传递依赖是否有 `package:web` 迁移版本？（检查 pub.dev 最新版本）
   - 项目代码使用的 API 能否迁移到 `package:web` 或 `dart:js_interop`？

3. **给出明确结论**：
   - **可上 Wasm**：无遗留库，或已有迁移路径
   - **必须先迁移**：有遗留库，需修改代码或等传递依赖更新
   - **需继续 JS fallback**：依赖深度使用遗留库且无迁移版本

4. **列出待办清单**：
   - 项目代码需修改哪些文件？
   - 传递依赖需升级到哪个版本？
   - 验证步骤是什么？

**输出格式**：

## 审计结果

### 不兼容库清单

**项目代码直接使用**：
- `lib/utils/browser_helper.dart`：导入 `dart:html`，使用 `window.location.href`
- [列出所有文件]

**传递依赖**：
- `flutter_web_plugins: ^0.0.1`：官方包，已迁移到 `package:web`，无问题
- `old_package: ^1.0.0`：使用 `dart:html`，pub.dev 最新版本 1.5.0 仍未迁移
- [列出所有问题依赖]

### 迁移方案

#### 1. 迁移项目代码（高优先级）

**文件**：`lib/utils/browser_helper.dart`

**Before**：
```dart
import 'dart:html' as html;

String getCurrentUrl() {
  return html.window.location.href;
}
```

**After**：
```dart
import 'package:web/web.dart' as web;

String getCurrentUrl() {
  return web.window.location.href;
}
```

**验证**：
1. 在 `pubspec.yaml` 添加 `web: ^1.1.0`
2. 重跑 `flutter build web --wasm` 确认无报错

#### 2. 升级传递依赖（若有迁移版本）

**依赖**：`url_launcher_web: ^2.3.3`

**操作**：
```bash
flutter pub upgrade url_launcher_web
```

#### 3. 条件导入（可选，迁移期兼容）

若需同时支持旧代码与新环境，用条件导入：

```dart
import 'browser_helper_fallback.dart'
  if (dart.library.js) 'browser_helper_legacy.dart'
  if (dart.library.js_interop) 'browser_helper_wasm.dart';
```

**browser_helper_legacy.dart**（旧环境）：
```dart
import 'dart:html' as html;
String getCurrentUrl() => html.window.location.href;
```

**browser_helper_wasm.dart**（新环境）：
```dart
import 'package:web/web.dart' as web;
String getCurrentUrl() => web.window.location.href;
```

### 验证步骤

1. 应用迁移代码
2. 添加 `web: ^1.1.0` 到 `pubspec.yaml`
3. 运行 `flutter pub get`
4. 重新构建：
   ```bash
   flutter build web --wasm
   # 或强制审计
   flutter build web --no-deprecated-js-interop
   ```
5. 确认无报错，`build/web/main.dart.*.wasm` 存在

### 结论

✅ **可上 Wasm**：需先迁移 1 个文件（`lib/utils/browser_helper.dart`），传递依赖已兼容。

或

❌ **需继续 JS fallback**：依赖 `old_package` 深度使用 `dart:html`，pub.dev 最新版本仍未迁移。需等上游更新或 fork 自行迁移。

或

⚠️ **必须先迁移**：有 3 个文件使用遗留库，2 个传递依赖需升级。预计工作量：1–2 天。
```

---

## ✅ 验证输出质量

AI 输出应包含：

- [ ] 明确列出不兼容的项目代码文件（文件路径 + 具体 API）
- [ ] 识别传递依赖的问题版本，并检查 pub.dev 是否有迁移版本
- [ ] 给出可落地的 Before/After 代码对比（不要只说"替换 dart:html"）
- [ ] 明确结论：可上 Wasm / 必须先迁移 / 需继续 JS fallback
- [ ] 列出验证步骤（可复制的命令）

若 AI 只给出空泛建议（如"优化兼容性"），追问：

```markdown
请明确指出：
1. 需要修改的具体文件路径和类名
2. 哪个 API 导致不兼容（如 `window.location.href`）
3. 给出 Before/After 代码对比
4. 传递依赖的具体版本号（当前 vs 目标）
```

---

## 🔗 相关资源

- Flutter Web Wasm 工作流：[workflows/flutter-web-wasm-and-caching.md](../../workflows/flutter-web-wasm-and-caching.md)
- 配置 Wasm 构建与缓存：[setup-wasm-build-and-cache.md](setup-wasm-build-and-cache.md)
- package:web 迁移指南：[https://dart.dev/interop/js-interop/package-web](https://dart.dev/interop/js-interop/package-web)
- dart:js_interop 文档：[https://dart.dev/interop/js-interop](https://dart.dev/interop/js-interop)

---

**先审计，再迁移，确保每一步都能构建通过。**
