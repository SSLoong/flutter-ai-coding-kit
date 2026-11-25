# Flutter Web：Wasm 编译、内容哈希缓存与 JS interop 审计工作流

**用 AI 升级 Flutter Web 到 WebAssembly、配置长期缓存策略，并审计迁移遗留 JS interop**

> 作者:Loong (SSLoong) | 日期:2026-11-25

本文对齐 **Flutter 3.50**（2026-11 发布）及官方文档核对日期 **2026-11-25**。先在本机执行 `flutter --version` 确认版本 ≥ 3.50；低于 3.50 时，`--web-content-hash`、`--no-deprecated-js-interop` 等特性不可用。

---

## 📋 背景

Flutter 3.50 把 Web 平台作为重要更新方向，带来三项关键能力：

1. **WebAssembly (Wasm) 编译**：`flutter build web --wasm` 同时产出 Wasm 与 JavaScript fallback。Chromium（Chrome/Edge 137+）已默认运行 Wasm；Safari 26.2+ 与 Firefox 147+ 已支持但需 opt-in 配置，旧版自动走 JS fallback。
2. **内容哈希缓存破坏**：`flutter build web --web-content-hash` 给编译产物和 assets 嵌入 SHA-256 内容哈希（如 `main.dart.HASH.js`、`logo.HASH.png`），hashed 文件可长期 `immutable` 缓存，`index.html` 与 `flutter_bootstrap.js` 需 `no-cache` 或 `must-revalidate`，并生成 `precache_manifest.json` 供自定义 Service Worker 使用。
3. **遗留 JS interop 审计**：`flutter build web --no-deprecated-js-interop` 会在出现 `dart:html`、`dart:js`、`dart:js_util`、`package:js` 等遗留库（含传递依赖）时失败。迁移目标：`dart:html` → `package:web`，`dart:js` / `package:js` → `dart:js_interop`，迁移期可用条件导入。

技术负责人需要一张可执行的决策表：**何时启用 Wasm、如何配置浏览器白名单、如何设置缓存头、如何迁移遗留 interop**。用 AI 协助审计依赖兼容性、改 bootstrap、写构建与 CDN 缓存方案，并在本地用官方命令验证。

---

## 🎯 目标

- ✅ 判断项目能否上 Wasm，或需继续 JS fallback
- ✅ 审计并迁移遗留 `dart:html`、`dart:js`、`package:js` 到 `package:web` 与 `dart:js_interop`
- ✅ 配置 web 初始化（Safari/Firefox opt-in，或明确只跑 Chromium）
- ✅ 使用 `--wasm` 与 `--web-content-hash` 构建
- ✅ 配置 CDN/静态托管缓存头（`immutable` vs `no-cache`）
- ✅ 本地与预发验证：构建成功、目标浏览器实际走 Wasm 还是 fallback、硬刷新与回访缓存是否正确

---

## 🔧 适用场景

| 场景 | 适合度 | 说明 |
|------|--------|------|
| 生产 Web 应用要提升性能 | ⭐⭐⭐⭐⭐ | Wasm 在 Chromium 上比 JS 快约 2–3 倍（官方博客基准），内容哈希缓存可大幅减少回访流量 |
| 已有 Web 目标，依赖里有 `dart:html` | ⭐⭐⭐⭐⭐ | 官方计划在未来版本从 SDK 移除遗留库，提前迁移避免破坏性变更 |
| 只需 Chromium（如内部工具、Chrome Extension） | ⭐⭐⭐⭐⭐ | 可直接启用 Wasm，无需 opt-in Safari/Firefox |
| 需兼容 Safari/Firefox，且愿意维护 opt-in 配置 | ⭐⭐⭐⭐ | 新版 Safari/Firefox 能跑 Wasm，旧版自动走 JS fallback |
| PWA 或需自定义 Service Worker | ⭐⭐⭐⭐⭐ | `precache_manifest.json` 配合 Workbox 管理缓存 |
| SEO 敏感（爬虫需服务端渲染） | ⭐⭐ | Flutter Web 客户端渲染；SEO 需配置预渲染服务或 SSR 方案 |
| 需要浏览器插件（Chrome Extension manifest v3） | ⭐⭐⭐ | Wasm 在 Extension 里工作，但需配置 CSP `nonce` 和 `wasmEvalCSP` |

**怎么选：**

- **上 Wasm + 内容哈希（本文）**：生产 Web 应用，目标用户在现代浏览器（Chrome 137+、Edge 137+、Safari 26.2+、Firefox 147+），或明确只跑 Chromium。依赖兼容 `package:web` 与 `dart:js_interop`，愿意为 Safari/Firefox 配置 opt-in。CDN 或静态托管已就绪。
- **继续纯 JS**：依赖深度使用 `dart:html` 且传递依赖无 `package:web` 迁移版本，或明确需兼容老旧浏览器（如 IE）。不用 `--wasm`，只用 `--web-content-hash` 也能受益。
- **本地 Web 开发不用 release 构建**：`flutter run -d chrome` 默认走 debug 模式 JS，性能不代表 Wasm。性能测试必须 `flutter run -d chrome --profile --wasm`。

---

## 🚀 完整流程

下面用「现有 Flutter 项目，已有移动端 + Web 目标，要升级到 Wasm + 内容哈希，并清掉遗留 `dart:html`」贯穿全流程。

### Step 1: 准备上下文

**人工操作**：

1. 确认 Flutter 版本：
   ```bash
   flutter --version
   # 必须 ≥ 3.50 才有 --web-content-hash 和 --no-deprecated-js-interop
   ```

2. 在 Cursor 里打开 `pubspec.yaml`、`web/index.html`、`web/flutter_bootstrap.js`（若有）。

3. 明确目标浏览器矩阵：
   - 只跑 Chromium（Chrome/Edge 137+）？
   - 需兼容 Safari 26.2+ 与 Firefox 147+，并愿意 opt-in？
   - 需兼容旧版 Safari/Firefox（自动 JS fallback）？

**AI 提示词**（可复制）：

```markdown
我要为 Flutter Web 项目升级到 Wasm 编译与内容哈希缓存。请先确认，不要写代码：

1. 我的目标浏览器是哪些？（如：Chrome/Edge 137+、Safari 26.2+、Firefox 147+，或只跑 Chromium）
2. 项目依赖（pubspec.yaml）和代码里是否有 dart:html、dart:js、dart:js_util、package:js？
3. 是否有传递依赖（pub 包）仍使用这些遗留库？
4. web/ 目录下 index.html 和 flutter_bootstrap.js（若有）是否已更新到 Flutter 3.22+ 的初始化写法？
5. 是否需要自定义 Service Worker（如 PWA）？

约束：Flutter 3.50，官方 Wasm 编译，不用实验特性。列出方案后再生成代码。
```

**检查点**：浏览器矩阵与依赖兼容性是否符合预期。可空性一旦写错，运行期会变成 404 或类型错误。

---

### Step 2: 审计 Wasm 兼容性与 JS interop

**人工操作**：

1. 普通构建（会自动跑 Wasm dry-run）：
   ```bash
   flutter build web
   ```
   若有不兼容库（如 `dart:html`），会看到非致命警告：
   ```
   Wasm dry run failed:
   Found incompatibilities with WebAssembly.

   package:my_app/main.dart 1:1 - dart:html unsupported (0)
   ```

2. 强制审计（构建失败时报告完整依赖树）：
   ```bash
   flutter build web --no-deprecated-js-interop
   ```
   或
   ```bash
   flutter build web --wasm
   ```
   若有遗留 interop，会报错并列出导入路径：
   ```
   Context: The unavailable library 'dart:html' is imported through these packages:

       main.dart => package:my_app => dart:html

   Detailed import paths for (some of) these imports:

       main.dart => package:my_app/main.dart => dart:html
   ```

**AI 提示词**（可复制，使用提示词模板：[prompts/web/audit-wasm-and-js-interop.md](../prompts/web/audit-wasm-and-js-interop.md)）：

```markdown
我用以下命令审计 Wasm 兼容性：

flutter build web --wasm

报错如下：

[粘贴完整错误输出，包含 Context 依赖树]

pubspec.yaml 依赖：

[粘贴 dependencies 和 dev_dependencies]

**分析要求**：

1. 哪些直接依赖使用了 dart:html / dart:js / package:js？
2. 哪些传递依赖（pub 包）导致不兼容？是否有 package:web 迁移版本？
3. 我的项目代码里哪些文件需要迁移？
4. 给出明确结论：「可上 Wasm / 必须先迁移 / 需继续 JS fallback」，并列出待办清单。

输出格式：
- 不兼容库清单（直接依赖 / 传递依赖 / 项目代码）
- 迁移目标库（package:web / dart:js_interop）
- 验证步骤
```

**AI 输出示例**（参考）：

```markdown
## 审计结果

### 不兼容库清单

**项目代码直接使用**：
- `lib/utils/browser_helper.dart`：导入 `dart:html`，使用 `window.location.href`

**传递依赖**：
- `flutter_web_plugins: ^0.0.1`（官方包，已迁移）
- `url_launcher_web: ^2.3.3`（已用 `package:web`）
- 无问题

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

#### 2. 条件导入（可选，迁移期兼容）

若需同时支持旧代码与新环境，用条件导入：

```dart
import 'browser_helper_fallback.dart'
  if (dart.library.js) 'browser_helper_legacy.dart'
  if (dart.library.js_interop) 'browser_helper_wasm.dart';
```

**结论**：✅ 可上 Wasm，需先迁移 1 个文件。
```

**检查点**：AI 是否明确指出代码位置、迁移目标库，并给出可落地的 Before/After 对比？若仅说"优化兼容性"等空泛建议，需追问具体方案。

---

### Step 3: 迁移遗留 JS interop

**人工操作**：

1. 添加 `package:web` 依赖：
   ```bash
   flutter pub add web
   ```

2. 应用 AI 生成的迁移代码，替换 `dart:html` 为 `package:web`，`dart:js` 为 `dart:js_interop`。

3. 重新构建验证：
   ```bash
   flutter build web --wasm
   # 或
   flutter build web --no-deprecated-js-interop
   ```

**AI 提示词**（可复制）：

```markdown
基于上述审计结果，请为以下文件生成迁移代码。

**目标文件**：`lib/utils/browser_helper.dart`

**迁移点**：
1. 替换 `dart:html` 为 `package:web`
2. 确保 API 对齐（如 `window.location.href`）

**要求**：
- 只改必须改的部分，保持现有逻辑不变
- 生成完整的修改后代码（包含 import）
- 注释说明每处迁移的原因

输出格式：
- 修改后的完整文件代码
- 关键改动说明（Before/After 对比）
```

**常见迁移模式**：

| 遗留库 | 目标库 | 示例 |
|--------|--------|------|
| `dart:html` | `package:web` | `import 'dart:html' as html;` → `import 'package:web/web.dart' as web;` |
| `dart:js` / `package:js` | `dart:js_interop` | `@JS()` 注解 → `@staticInterop` extension type |
| `dart:js_util.jsify()` | `dart:js_interop.jsify()` | `jsify(map)` → `map.jsify()` |

**检查点**：迁移后 `flutter build web --wasm` 无报错。

---

### Step 4: 配置 web 初始化（Safari/Firefox opt-in）

**背景**：
- **Chromium（Chrome/Edge 137+）**：默认跑 Wasm，无需配置。
- **Safari 26.2+ / Firefox 147+**：已支持 Wasm，但默认 `wasmAllowList` 把它们设为 `false`，需手动 opt-in。
- **Safari <= 26.1 / Firefox < 147**：运行时特性检测自动走 JS fallback。

**人工操作**：

1. 检查 `web/index.html` 是否已更新到 Flutter 3.22+ 的初始化写法。若未更新，运行：
   ```bash
   flutter create . --platforms web
   ```
   会重新生成 `web/index.html` 与默认 `web/flutter_bootstrap.js`（若存在会覆盖，注意备份自定义内容）。

2. 决定 opt-in 策略：
   - **只跑 Chromium**：无需改动，默认 `wasmAllowList: { blink: true, gecko: false, webkit: false }` 即可。
   - **启用 Safari/Firefox Wasm**：修改 `web/flutter_bootstrap.js` 或 `web/index.html` 的 `_flutter.loader.load()` 调用，增加 `wasmAllowList` 配置。

**AI 提示词**（可复制，使用提示词模板：[prompts/web/setup-wasm-build-and-cache.md](../prompts/web/setup-wasm-build-and-cache.md)）：

```markdown
我要配置 Flutter web 初始化，启用 Safari 26.2+ 和 Firefox 147+ 的 Wasm。

**目标浏览器**：
- Chrome/Edge 137+（默认启用）
- Safari 26.2+（需 opt-in）
- Firefox 147+（需 opt-in）
- 旧版自动 JS fallback

**要求**：
1. 修改 web/flutter_bootstrap.js（若有）或 web/index.html
2. 配置 wasmAllowList: { blink: true, webkit: true, gecko: true }
3. 保持 index.html 其他部分不变
4. 给出完整修改后代码

输出格式：
- 修改后的 web/flutter_bootstrap.js 或 web/index.html
- 配置说明
```

**配置示例**（`web/flutter_bootstrap.js`）：

```javascript
{{flutter_js}}
{{flutter_build_config}}

_flutter.loader.load({
  config: {
    wasmAllowList: {
      blink: true,     // Chrome/Edge 137+
      webkit: true,    // Safari 26.2+
      gecko: true,     // Firefox 147+
      unknown: false,  // 未知浏览器走 fallback
    },
  },
});
```

或在 `web/index.html` 里内联（若删除了 `flutter_bootstrap.js`）：

```html
<!DOCTYPE html>
<html>
<head>
  <meta charset="UTF-8">
  <meta name="viewport" content="width=device-width, initial-scale=1.0">
  <title>My Flutter App</title>
</head>
<body>
  <script>
    {{flutter_bootstrap_js}}
  </script>
  <script>
    _flutter.loader.load({
      config: {
        wasmAllowList: {
          blink: true,
          webkit: true,
          gecko: true,
        },
      },
    });
  </script>
</body>
</html>
```

**检查点**：配置后 `flutter build web --wasm` 成功，生成 `build/web/main.dart.wasm` 与 `build/web/main.dart.js`。

---

### Step 5: 使用 --wasm 与 --web-content-hash 构建

**人工操作**：

1. 构建 Wasm + 内容哈希：
   ```bash
   flutter build web --wasm --web-content-hash
   ```

2. 检查 `build/web/` 目录：
   - **Hashed 文件**（可长期缓存）：`main.dart.HASH.wasm`、`main.dart.HASH.mjs`、`main.dart.HASH.js`、`assets/logo.HASH.png`、`AssetManifest.HASH.bin` 等。
   - **Unhashed 文件**（需 no-cache）：`index.html`、`flutter_bootstrap.js`、`flutter.js`、`manifest.json`、`favicon.png`、`canvaskit/**` 等。
   - **Precache 清单**：`precache_manifest.json`（供自定义 Service Worker 使用）。

3. 验证哈希注入：
   ```bash
   cat build/web/flutter_bootstrap.js | grep mainWasmPath
   # 应显示类似 "mainWasmPath":"main.dart.a1b2c3d4.wasm"
   ```

**常见构建标志组合**：

| 场景 | 命令 | 说明 |
|------|------|------|
| 生产部署（Wasm + 哈希） | `flutter build web --wasm --web-content-hash` | 推荐 |
| 只用哈希，不用 Wasm | `flutter build web --web-content-hash` | 兼容老浏览器 |
| Wasm + Source Maps | `flutter build web --wasm --source-maps` | 错误监控（Sentry/Datadog） |
| Wasm + 保留符号名 | `flutter build web --wasm --no-strip-wasm` | 二进制增大约 47%，仅用于 staging/QA |
| Profile 模式（性能分析） | `flutter build web --wasm --profile` | Chrome DevTools 分析 |

**检查点**：
- [ ] `build/web/main.dart.HASH.wasm` 存在
- [ ] `build/web/main.dart.HASH.js` 存在（fallback）
- [ ] `flutter_bootstrap.js` 或 `index.html` 里能找到 `_flutter.buildConfig`
- [ ] `precache_manifest.json` 存在且包含 hashed 文件

---

### Step 6: 配置 CDN/静态托管缓存头

**背景**：
- **Hashed 文件**（`main.dart.HASH.wasm`、`logo.HASH.png`）：内容变化时文件名变化，可设置 `Cache-Control: public, max-age=31536000, immutable`（1 年）。
- **Unhashed 文件**（`index.html`、`flutter_bootstrap.js`）：内容变化时文件名不变，必须设置 `Cache-Control: no-cache` 或 `max-age=0, must-revalidate`，让浏览器每次都校验。

**人工操作**：

根据你的托管方式选择：

#### 6.1 Firebase Hosting

在 `firebase.json` 配置缓存头：

```json
{
  "hosting": {
    "public": "build/web",
    "ignore": ["firebase.json", "**/.*", "**/*.map"],
    "rewrites": [
      {
        "source": "**",
        "destination": "/index.html"
      }
    ],
    "headers": [
      {
        "source": "**/main.dart.*.{js,wasm,mjs}",
        "headers": [
          {
            "key": "Cache-Control",
            "value": "public, max-age=31536000, immutable"
          }
        ]
      },
      {
        "source": "assets/**",
        "headers": [
          {
            "key": "Cache-Control",
            "value": "public, max-age=31536000, immutable"
          }
        ]
      },
      {
        "source": "**/*.{html,json,js}",
        "headers": [
          {
            "key": "Cache-Control",
            "value": "no-cache"
          }
        ]
      },
      {
        "source": "**",
        "headers": [
          {
            "key": "Cross-Origin-Opener-Policy",
            "value": "same-origin"
          },
          {
            "key": "Cross-Origin-Embedder-Policy",
            "value": "credentialless"
          }
        ]
      }
    ]
  }
}
```

**重要**：
- `"ignore": ["**/*.map"]`：排除 source maps，不要公开上传到 CDN。
- `rewrites` 的 `"destination": "/index.html"` 是 SPA 路由，但需排除 `/assets/**`、`/canvaskit/**`、`/main.dart.*`，让 404 真正返回 404（不要让所有路径都返回 `index.html`）。Firebase 会自动处理这些，但若用其他 CDN，需明确配置。

#### 6.2 其他静态托管（Vercel、Netlify、Cloudflare Pages）

在各平台的配置文件（`vercel.json`、`netlify.toml`、`_headers`）里设置类似规则。

**AI 提示词**（可复制）：

```markdown
我要为 Flutter Web（已用 --web-content-hash）配置缓存头。

**托管方式**：[Firebase Hosting / Vercel / Netlify / Cloudflare Pages / Nginx]

**要求**：
1. Hashed 文件（main.dart.HASH.*、assets/**）：Cache-Control: public, max-age=31536000, immutable
2. Unhashed 文件（index.html、flutter_bootstrap.js、manifest.json）：Cache-Control: no-cache
3. COOP/COEP 头：Cross-Origin-Opener-Policy: same-origin, Cross-Origin-Embedder-Policy: credentialless
4. 排除 *.map 文件（不要公开上传）
5. SPA 路由重写到 /index.html，但排除 /assets/**、/canvaskit/**、/main.dart.*

输出格式：
- 完整配置文件（firebase.json / vercel.json / nginx.conf 等）
- 配置说明
```

**检查点**：
- [ ] Hashed 文件设置 `immutable`
- [ ] Unhashed 文件设置 `no-cache` 或 `must-revalidate`
- [ ] COOP/COEP 头已配置（Wasm 多线程渲染需要）
- [ ] `*.map` 文件已排除

---

### Step 7: 本地与预发验证

**人工操作**：

1. 本地启动 HTTP 服务器（COOP/COEP 需 HTTPS 或 localhost）：
   ```bash
   cd build/web
   python3 -m http.server 8000
   ```
   或用 `dhttpd`：
   ```bash
   dart pub global activate dhttpd
   dhttpd --host localhost --port 8000 --path build/web
   ```

2. 打开浏览器访问 `http://localhost:8000`，检查：
   - **Chrome/Edge 137+**：打开 DevTools Console，检查是否有 `dart2wasm` 或 Wasm 相关日志。运行：
     ```javascript
     console.log(typeof WebAssembly !== 'undefined' ? 'Wasm loaded' : 'JS fallback');
     ```
   - **Safari 26.2+ / Firefox 147+**（若已 opt-in）：同上。
   - **旧版浏览器**：应自动走 JS fallback，无报错。

3. 验证缓存：
   - **首次访问**：DevTools Network 面板，`main.dart.HASH.wasm` 应返回 `200` 或 `304`，`Cache-Control: immutable`。
   - **刷新（F5）**：`index.html` 应返回 `200`（从服务器校验），hashed 文件返回 `304` 或 `(disk cache)`。
   - **硬刷新（Ctrl+F5）**：所有文件重新下载，但 hashed 文件仍从缓存或返回 `304`。

4. 验证 Wasm 实际运行：
   在 Dart 代码里检查编译环境：
   ```dart
   const isRunningWithWasm = bool.fromEnvironment('dart.tool.dart2wasm');
   print('Running with Wasm: $isRunningWithWasm');
   ```
   或运行时检查：
   ```dart
   final isRunningWithWasm = identical(double.nan, double.nan);
   print('Running with Wasm: $isRunningWithWasm');
   ```

**AI 提示词**（验证异常时用）：

```markdown
我用以下命令构建并本地测试：

flutter build web --wasm --web-content-hash
python3 -m http.server 8000 --directory build/web

浏览器：Chrome 140

**问题**：
[描述问题，如：DevTools Console 显示 JS fallback，未走 Wasm / 缓存未生效，每次刷新都重新下载 / 404 错误]

**Network 面板截图或日志**：
[粘贴关键请求的 Status、Headers、Timing]

**要求**：
1. 判断是配置问题、浏览器问题，还是构建问题？
2. 给出修复方案

输出格式：
- 问题原因
- 修复步骤
```

**常见问题**：

| 症状 | 原因 | 修复 |
|------|------|------|
| Chrome 137+ 仍走 JS fallback | `wasmAllowList` 未配置 `blink: true` | 检查 `flutter_bootstrap.js` 或 `index.html` 的 `load()` 配置 |
| Safari 26.2+ 走 JS fallback | 默认 `webkit: false` | 增加 `wasmAllowList: { webkit: true }` |
| `SharedArrayBuffer` 警告 | COOP/COEP 头缺失 | 配置 `Cross-Origin-Opener-Policy: same-origin` 和 `Cross-Origin-Embedder-Policy: credentialless` |
| `main.dart.wasm` 404 | 未用 `--wasm` 构建 | 重跑 `flutter build web --wasm` |
| `logo.HASH.png` 404 | 代码里硬编码 `assets/logo.png` | 改用 `Image.asset('logo.png')` 或 `rootBundle` |
| 刷新后仍下载 hashed 文件 | `Cache-Control` 未设置 `immutable` | 检查 CDN/服务器配置 |

**检查清单**：
- [ ] Chrome/Edge 137+ 实际走 Wasm（Console 有 `dart2wasm` 相关日志或 Dart 侧 `bool.fromEnvironment('dart.tool.dart2wasm')` 为 `true`）
- [ ] Safari/Firefox（若 opt-in）走 Wasm 或自动 fallback
- [ ] 旧版浏览器走 JS fallback，无报错
- [ ] Hashed 文件回访时从缓存加载（`304` 或 `(disk cache)`）
- [ ] Unhashed 文件每次校验（`200` 或 `304`，不是 `(disk cache)`）
- [ ] COOP/COEP 头已生效，无 `SharedArrayBuffer` 警告

---

### Step 8: 配置自定义 Service Worker（可选，PWA 或复杂缓存策略）

**背景**：
- Flutter 3.50 起，官方不再生成 `flutter_service_worker.js`，改用 `precache_manifest.json` 供开发者自定义。
- 若需 PWA 离线支持、复杂缓存策略（如优先缓存、仅缓存特定路径），需自己写 Service Worker，并消费 `precache_manifest.json`。

**precache_manifest.json 格式**：

```json
{
  "version": 1,
  "entries": [
    {
      "url": "flutter_bootstrap.js",
      "hash": "a1b2c3d4",
      "size": 10432,
      "urlHashed": false
    },
    {
      "url": "main.dart.f215ce97.wasm",
      "hash": "f215ce97",
      "size": 1842104,
      "urlHashed": true
    }
  ]
}
```

**使用 Workbox 的示例**（`web/sw.js`）：

```javascript
importScripts('https://storage.googleapis.com/workbox-cdn/releases/7.0.0/workbox-sw.js');

// 读取 precache_manifest.json
fetch('precache_manifest.json')
  .then(response => response.json())
  .then(manifest => {
    const precacheList = manifest.entries.map(entry => ({
      url: entry.url,
      revision: entry.urlHashed ? null : entry.hash, // urlHashed 文件不需要 query param 版本
    }));

    workbox.precaching.precacheAndRoute(precacheList);
  });

// 其他策略...
workbox.routing.registerRoute(
  ({ request }) => request.destination === 'image',
  new workbox.strategies.CacheFirst({
    cacheName: 'images',
    plugins: [
      new workbox.expiration.ExpirationPlugin({
        maxEntries: 60,
        maxAgeSeconds: 30 * 24 * 60 * 60, // 30 天
      }),
    ],
  })
);
```

然后在 `web/index.html` 或 `main()` 里注册：

```javascript
if ('serviceWorker' in navigator) {
  navigator.serviceWorker.register('sw.js');
}
```

**AI 提示词**（可复制）：

```markdown
我要为 Flutter Web（已用 --web-content-hash）配置 PWA Service Worker。

**要求**：
1. 消费 precache_manifest.json
2. Hashed 文件（urlHashed: true）不需要 query param 版本
3. Unhashed 文件需 revision
4. 离线支持核心路由（如 /、/about）
5. 图片缓存策略：CacheFirst，最多 60 张，30 天过期

输出格式：
- web/sw.js 完整代码（Workbox 或原生 Service Worker API）
- 注册代码（index.html 或 main.dart）
```

**检查点**：
- [ ] `precache_manifest.json` 已生成
- [ ] Service Worker 已注册且激活
- [ ] 离线时核心路由可访问
- [ ] DevTools Application → Service Workers 显示已激活

---

## ✅ 验证清单

完成后确认：

- [ ] Flutter 版本 ≥ 3.50（`flutter --version`）
- [ ] 项目代码与依赖已迁移遗留 `dart:html`、`dart:js` 到 `package:web`、`dart:js_interop`
- [ ] `flutter build web --wasm` 成功，生成 `main.dart.HASH.wasm` 与 `main.dart.HASH.js`
- [ ] `web/flutter_bootstrap.js` 或 `web/index.html` 配置了 `wasmAllowList`（若需 Safari/Firefox opt-in）
- [ ] `flutter build web --web-content-hash` 成功，`flutter_bootstrap.js` 包含 `_flutter.buildConfig`
- [ ] CDN/静态托管配置了正确的 `Cache-Control` 头（hashed `immutable`，unhashed `no-cache`）
- [ ] COOP/COEP 头已配置（`same-origin` 与 `credentialless`）
- [ ] Chrome/Edge 137+ 实际走 Wasm（Dart 侧 `bool.fromEnvironment('dart.tool.dart2wasm')` 为 `true`）
- [ ] Safari/Firefox（若 opt-in）走 Wasm 或自动 fallback，旧版自动 fallback
- [ ] Hashed 文件回访时从缓存加载（`304` 或 `(disk cache)`）
- [ ] Unhashed 文件每次校验（`200` 或 `304`）
- [ ] 若需 PWA，Service Worker 已注册并消费 `precache_manifest.json`

---

## 🤔 FAQ

### Q1: `--web-content-hash` 失败，报 "index.html 引用 main.dart.js"？

**A**: Flutter 3.50+ 要求 `web/index.html` 使用新的初始化写法（`flutter_bootstrap.js` 或 `{{flutter_bootstrap_js}}`），不能硬编码 `main.dart.js` 或调用已废弃的 `loadEntrypoint()` 方法。

运行以下命令重新生成（会覆盖现有 `web/` 目录，注意备份自定义内容）：

```bash
flutter create . --platforms web
```

或手动修改 `web/index.html`，把旧的脚本块：

```html
<script>
  window.addEventListener('load', function(ev) {
    _flutter.loader.loadEntrypoint({
      entrypointUrl: "main.dart.js"
    });
  });
</script>
```

改为新的：

```html
<script src="flutter_bootstrap.js" async></script>
```

或内联：

```html
<script>
  {{flutter_bootstrap_js}}
</script>
```

---

### Q2: Chrome 137+ 仍走 JS fallback，未走 Wasm？

**A**: 检查以下几点：

1. **确认已用 `--wasm` 构建**：
   ```bash
   flutter build web --wasm
   ls build/web/main.dart.*.wasm  # 应存在
   ```

2. **检查 `wasmAllowList` 配置**：
   若自定义了 `flutter_bootstrap.js` 或 `index.html` 的 `load()` 调用，确认：
   ```javascript
   _flutter.loader.load({
     config: {
       wasmAllowList: { blink: true },  // Chrome/Edge
     },
   });
   ```
   默认就是 `blink: true`，若写错了（如 `blink: false`），会强制 fallback。

3. **检查浏览器版本**：
   ```javascript
   console.log(navigator.userAgent);
   ```
   Chrome/Edge < 137 不支持 WasmGC，会自动 fallback。

4. **检查 Dart 侧编译环境**：
   ```dart
   const isWasm = bool.fromEnvironment('dart.tool.dart2wasm');
   print('Wasm: $isWasm');  // 应为 true
   ```

---

### Q3: Safari/Firefox 走 JS fallback，明明配置了 `webkit: true` / `gecko: true`？

**A**: 分两种情况：

1. **Safari <= 26.1 / Firefox < 147**：这些版本不支持 WasmGC，`flutter_bootstrap.js` 会在运行时检测特性，自动 fallback。这是预期行为。
2. **Safari 26.2+ / Firefox 147+**：确认 `wasmAllowList` 配置在正确的 `load()` 调用里。若用了自定义 `onEntrypointLoaded` 回调，注意 `config` 对象不会自动转发到 `initializeEngine()`，需手动传递：
   ```javascript
   const flutterConfig = {
     wasmAllowList: { blink: true, webkit: true, gecko: true },
   };

   _flutter.loader.load({
     config: flutterConfig,
     onEntrypointLoaded: async function(engineInitializer) {
       // 必须再传一次 flutterConfig
       const appRunner = await engineInitializer.initializeEngine(flutterConfig);
       await appRunner.runApp();
     },
   });
   ```

---

### Q4: 部署后，新版本发布时用户还在旧页面，刷新后出现 404？

**A**: 这是 overlay deployment 场景。用户打开页面时是 release N-1，新版本 N 部署后，N-1 的 hashed 文件（如 `main.dart.OLD_HASH.wasm`）被删除，用户刷新或懒加载 asset 时找不到文件。

**解决方案**：

1. **保留至少一个旧版本的 hashed 文件**（N-1）24–72 小时，或用 CDN 的 object versioning。
2. **原子更新 `index.html` 和 `flutter_bootstrap.js`**（最后更新，确保 hashed 文件已上传）。
3. **SPA 路由排除 assets**：让 `/assets/**`、`/canvaskit/**`、`/main.dart.*` 返回真正的 404（不要重写到 `/index.html`），避免用户拿到 `200 text/html` 而非真实错误。

**Firebase Hosting 示例**（排除 assets 路由）：

```json
{
  "hosting": {
    "rewrites": [
      {
        "source": "**",
        "destination": "/index.html"
      }
    ]
  }
}
```

Firebase 会自动排除真实文件，但若用 Vercel/Netlify/Nginx，需明确配置 negative rewrite。

---

### Q5: `--web-content-hash` 能和 deferred loading（`deferred as`）一起用吗？

**A**: **目前不能**（Flutter 3.50）。官方文档明确：

> `--web-content-hash` does not yet support deferred imports (`deferred as`) in either JavaScript (`dart2js`) or WebAssembly (`dart2wasm` / `--enable-wasm-deferred-loading`) builds.

若项目使用 `import 'xxx.dart' deferred as yyy;`，构建时会报错。解决方案：

1. **短期**：去掉 deferred loading，或不用 `--web-content-hash`。
2. **长期**：等待官方支持（可关注 [Flutter issue tracker](https://github.com/flutter/flutter/issues)）。

---

### Q6: `*.map` 文件要不要上传 CDN？

**A**: **不要公开上传**。Source maps 包含未 minify 的符号名、类层次、文件路径，若公开部署，浏览器 DevTools 会自动下载，暴露内部结构。

**正确做法**：

1. **构建时生成 source maps**：
   ```bash
   flutter build web --wasm --source-maps
   ```
2. **CI/CD 里上传 `.map` 文件到错误监控服务**（Sentry、Datadog）的私有存储。
3. **排除 `*.map` 文件到公共 CDN**：
   - Firebase：`"ignore": ["**/*.map"]`
   - Nginx：`location ~ \.map$ { deny all; }`
   - Cloudflare：在 Transform Rules 里 block `*.map`

---

### Q7: COOP/COEP 头会影响第三方嵌入（如 iframe、跨域图片）吗？

**A**: 是的。`Cross-Origin-Embedder-Policy: credentialless` 或 `require-corp` 会阻止跨域资源加载（除非对方设置 `Cross-Origin-Resource-Policy` 或走 CORS）。

**影响**：
- **跨域图片/字体**：需对方服务器设置 `CORP: cross-origin` 或 CORS 头。
- **跨域 iframe**：需对方设置 `CORP: cross-origin` 或 `COEP: credentialless`。
- **第三方脚本**：需对方支持 COEP。

**解决方案**：
1. **优先用 `credentialless`**（比 `require-corp` 宽松，允许无凭据的跨域请求）。
2. **自托管第三方资源**（如 Google Fonts、CDN 图片）。
3. **若无法解决**，放弃多线程 Wasm（去掉 COOP/COEP 头），Wasm 会自动降级到单线程模式（性能略低，但仍比 JS 快）。

检查单线程模式警告：
```javascript
_flutter.loader.load({
  config: {
    suppressMultithreadingWarning: true,  // 抑制警告
  },
});
```

---

## 💡 最佳实践

1. **先审计，再迁移，再构建**
   - 用 `flutter build web --wasm` 或 `--no-deprecated-js-interop` 审计兼容性，明确待办清单后再动手。

2. **优先迁移项目代码，再等传递依赖**
   - 传递依赖（pub 包）的迁移版本可能还未发布，可用条件导入兼容过渡期。

3. **生产部署必须同时开启 `--wasm` 和 `--web-content-hash`**
   - Wasm 提升性能，内容哈希减少回访流量，两者互补。

4. **hashed 文件设置 `immutable`，unhashed 文件设置 `no-cache`**
   - 这是 HTTP 缓存的黄金法则。搞反了会导致缓存失效或无法更新。

5. **COOP/COEP 头必须配置（除非明确只用单线程 Wasm）**
   - 多线程 Wasm 渲染需要 `SharedArrayBuffer`，而 `SharedArrayBuffer` 需要这两个头。

6. **Source maps 只给错误监控服务，不要公开**
   - 在 CI/CD 里上传到 Sentry/Datadog，从 CDN 排除 `*.map`。

7. **overlay deployment 保留旧版本 hashed 文件 24–72 小时**
   - 避免用户在新版本部署时遇到 404。

8. **本地验证必须用 HTTP 服务器，不能直接打开 `index.html`**
   - `file://` 协议无法加载 Wasm，必须 `http://` 或 `https://`。

9. **性能测试必须用 `--profile` 或 `--release`，不要用 debug**
   - `flutter run -d chrome` 默认走 debug 模式 JS，性能不代表 Wasm。

10. **记录浏览器矩阵与 opt-in 决策**
    - 在项目 README 或 `.cursor/rules` 里记录目标浏览器与 `wasmAllowList` 配置，避免后续混乱。

---

## 🔗 相关资源

- 审计 Wasm 兼容性与 JS interop：[prompts/web/audit-wasm-and-js-interop.md](../prompts/web/audit-wasm-and-js-interop.md)
- 配置 Wasm 构建与缓存：[prompts/web/setup-wasm-build-and-cache.md](../prompts/web/setup-wasm-build-and-cache.md)
- 上下文工程：[docs/context-engineering.md](../docs/context-engineering.md)
- Flutter 3.50 发布博客：[https://flutter.dev/blog/whats-new-in-flutter-3-50](https://flutter.dev/blog/whats-new-in-flutter-3-50)
- WebAssembly 支持官方文档：[https://docs.flutter.dev/platform-integration/web/wasm](https://docs.flutter.dev/platform-integration/web/wasm)
- Web 部署官方文档：[https://docs.flutter.dev/deployment/web](https://docs.flutter.dev/deployment/web)
- Flutter web app 初始化：[https://docs.flutter.dev/platform-integration/web/initialization](https://docs.flutter.dev/platform-integration/web/initialization)
- package:web 迁移指南：[https://dart.dev/interop/js-interop/package-web](https://dart.dev/interop/js-interop/package-web)
- dart:js_interop 文档：[https://dart.dev/interop/js-interop](https://dart.dev/interop/js-interop)
- Dart WebAssembly 编译：[https://dart.dev/web/wasm](https://dart.dev/web/wasm)
- COOP/COEP 指南：[https://web.dev/articles/coop-coep](https://web.dev/articles/coop-coep)
- Flutter 3.50 Release Notes：[https://docs.flutter.dev/release/release-notes/release-notes-3.50.0](https://docs.flutter.dev/release/release-notes/release-notes-3.50.0)

---

**先审计，再迁移，再上 Wasm。让 Flutter Web 更快、更稳定。**
