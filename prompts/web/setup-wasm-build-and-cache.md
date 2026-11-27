# 提示词：配置 Wasm 构建与缓存

**用途**：按官方文档生成或修改 web 初始化、构建命令（`--wasm`、`--web-content-hash`）、缓存头与 precache/Service Worker 接入方案，附检查清单。

---

## 📋 适用场景

- 配置 Flutter web 初始化（`flutter_bootstrap.js` 或 `index.html`）
- 设置 `wasmAllowList` opt-in Safari/Firefox
- 生成 CDN/静态托管缓存头配置（Firebase Hosting、Vercel、Nginx 等）
- 配置自定义 Service Worker 消费 `precache_manifest.json`

---

## 📥 输入要求

在 Cursor 中准备以下上下文：

1. **web/index.html**：打开当前文件
2. **web/flutter_bootstrap.js**（若有）：打开当前文件
3. **托管方式**：Firebase Hosting / Vercel / Netlify / Cloudflare Pages / Nginx / 其他
4. **目标浏览器**：Chrome/Edge 137+、Safari 26.2+、Firefox 147+，或只跑 Chromium

---

## 🎯 提示词模板

### 场景 1：配置 web 初始化（Safari/Firefox opt-in）

```markdown
我要配置 Flutter web 初始化，启用 Safari 26.2+ 和 Firefox 147+ 的 Wasm。

**目标浏览器**：
- Chrome/Edge 137+（默认启用）
- Safari 26.2+（需 opt-in）
- Firefox 147+（需 opt-in）
- 旧版自动 JS fallback

**当前文件**：
[粘贴 web/index.html 或 web/flutter_bootstrap.js 内容]

**要求**：

1. 修改 `web/flutter_bootstrap.js`（若有）或 `web/index.html`
2. 配置 `wasmAllowList: { blink: true, webkit: true, gecko: true }`
3. 保持 `index.html` 其他部分不变（如 meta 标签、favicon）
4. 若 `flutter_bootstrap.js` 不存在，给出是否需要创建的建议

**输出格式**：
- 修改后的完整文件代码
- 配置说明（哪些浏览器会走 Wasm，哪些会 fallback）
- 验证步骤

**输出示例**：

## 配置方案

### 方案：修改 web/flutter_bootstrap.js

**修改后的 web/flutter_bootstrap.js**：

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

**配置说明**：
- **Chrome/Edge 137+**：默认走 Wasm
- **Safari 26.2+**：已 opt-in，会走 Wasm
- **Firefox 147+**：已 opt-in，会走 Wasm
- **旧版浏览器**（Safari <= 26.1、Firefox < 147、其他）：运行时检测特性，自动 fallback 到 JS

**验证步骤**：
1. 保存文件
2. 构建：`flutter build web --wasm`
3. 本地测试：
   ```bash
   cd build/web
   python3 -m http.server 8000
   ```
4. 在 Chrome/Safari/Firefox 打开 `http://localhost:8000`，检查 Console 是否有 Wasm 相关日志
```

---

### 场景 2：生成 CDN/静态托管缓存头配置

```markdown
我要为 Flutter Web（已用 `--web-content-hash`）配置缓存头。

**托管方式**：[Firebase Hosting / Vercel / Netlify / Cloudflare Pages / Nginx]

**要求**：

1. **Hashed 文件**（`main.dart.HASH.*`、`assets/**`）：
   - `Cache-Control: public, max-age=31536000, immutable`

2. **Unhashed 文件**（`index.html`、`flutter_bootstrap.js`、`manifest.json`）：
   - `Cache-Control: no-cache`（或 `max-age=0, must-revalidate`）

3. **COOP/COEP 头**（Wasm 多线程渲染需要）：
   - `Cross-Origin-Opener-Policy: same-origin`
   - `Cross-Origin-Embedder-Policy: credentialless`

4. **排除 source maps**（不要公开上传）：
   - `*.map` 文件不上传到 CDN

5. **SPA 路由重写**：
   - 除 `/assets/**`、`/canvaskit/**`、`/main.dart.*` 外，其他路径重写到 `/index.html`
   - 真实文件（如 assets）返回 404，不要重写到 `index.html`

**输出格式**：
- 完整配置文件（`firebase.json` / `vercel.json` / `nginx.conf` 等）
- 配置说明
- 验证步骤

**输出示例**：

## 缓存头配置

### Firebase Hosting（firebase.json）

```json
{
  "hosting": {
    "public": "build/web",
    "ignore": [
      "firebase.json",
      "**/.*",
      "**/*.map"
    ],
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

**配置说明**：
1. **`ignore: ["**/*.map"]`**：排除 source maps，不上传到公共 CDN
2. **Hashed 文件**：`main.dart.HASH.*` 与 `assets/**` 设置 `immutable`，浏览器缓存 1 年
3. **Unhashed 文件**：`index.html`、`flutter_bootstrap.js`、`manifest.json` 设置 `no-cache`，每次校验
4. **COOP/COEP**：所有文件返回跨域隔离头，启用多线程 Wasm
5. **SPA 路由**：Firebase 自动处理真实文件，不会把 404 重写成 `index.html`

**验证步骤**：
1. 保存 `firebase.json`
2. 部署：`firebase deploy`
3. 访问线上 URL，打开 DevTools Network 面板
4. 检查 `main.dart.HASH.wasm` 的响应头：
   - `Cache-Control: public, max-age=31536000, immutable`
   - `Cross-Origin-Opener-Policy: same-origin`
   - `Cross-Origin-Embedder-Policy: credentialless`
5. 检查 `index.html` 的响应头：
   - `Cache-Control: no-cache`
6. 刷新页面，`main.dart.HASH.wasm` 应从缓存加载（`304` 或 `(disk cache)`）
```

---

### 场景 3：配置自定义 Service Worker（PWA 或复杂缓存策略）

```markdown
我要为 Flutter Web（已用 `--web-content-hash`）配置 PWA Service Worker。

**要求**：

1. 消费 `precache_manifest.json`
2. Hashed 文件（`urlHashed: true`）不需要 query param 版本
3. Unhashed 文件需 `revision`
4. 离线支持核心路由（如 `/`、`/about`）
5. 图片缓存策略：CacheFirst，最多 60 张，30 天过期

**输出格式**：
- `web/sw.js` 完整代码（Workbox 或原生 Service Worker API）
- 注册代码（`web/index.html` 或 `lib/main.dart`）
- 配置说明
- 验证步骤

**输出示例**：

## Service Worker 配置

### web/sw.js（Workbox）

```javascript
importScripts('https://storage.googleapis.com/workbox-cdn/releases/7.0.0/workbox-sw.js');

// 读取 precache_manifest.json
fetch('precache_manifest.json')
  .then(response => response.json())
  .then(manifest => {
    const precacheList = manifest.entries.map(entry => ({
      url: entry.url,
      // urlHashed 文件不需要 query param 版本
      revision: entry.urlHashed ? null : entry.hash,
    }));

    workbox.precaching.precacheAndRoute(precacheList);
  });

// 离线支持核心路由
workbox.routing.registerRoute(
  ({ request }) => request.mode === 'navigate',
  new workbox.strategies.NetworkFirst({
    cacheName: 'pages',
  })
);

// 图片缓存策略
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

### 注册 Service Worker（web/index.html）

在 `</body>` 之前增加：

```html
<script>
  if ('serviceWorker' in navigator) {
    navigator.serviceWorker.register('sw.js').then(registration => {
      console.log('Service Worker registered:', registration);
    }).catch(error => {
      console.error('Service Worker registration failed:', error);
    });
  }
</script>
```

**配置说明**：
1. **precache_manifest.json**：包含所有 hashed 与 unhashed 文件的清单
2. **urlHashed: true**：文件名自带哈希（如 `main.dart.HASH.wasm`），Workbox 不再加 `?__WB_REVISION__=xxx`
3. **urlHashed: false**：文件名未哈希（如 `index.html`），Workbox 用 `revision` 字段管理版本
4. **离线路由**：`NetworkFirst` 策略，优先网络，失败时用缓存
5. **图片缓存**：`CacheFirst` 策略，最多 60 张，30 天过期

**验证步骤**：
1. 保存 `web/sw.js` 和 `web/index.html`
2. 构建：`flutter build web --wasm --web-content-hash`
3. 本地测试：
   ```bash
   cd build/web
   python3 -m http.server 8000
   ```
4. 打开 `http://localhost:8000`，DevTools Application → Service Workers 应显示已激活
5. 离线模式（DevTools Network → Offline），刷新页面应仍能访问核心路由
6. 检查 DevTools Application → Cache Storage，应有 `workbox-precache-v2`、`pages`、`images` 等缓存
```

---

## ✅ 验证输出质量

AI 输出应包含：

- [ ] 完整的配置文件（可直接复制使用）
- [ ] 明确的配置说明（每个字段的作用）
- [ ] 验证步骤（可复制的命令或操作）
- [ ] 目标浏览器矩阵与 fallback 行为说明

若 AI 只给出空泛建议（如"优化缓存"），追问：

```markdown
请给出完整的配置文件（firebase.json / vercel.json / nginx.conf），包含：
1. Hashed 文件（main.dart.HASH.*、assets/**）的 Cache-Control 头
2. Unhashed 文件（index.html、flutter_bootstrap.js）的 Cache-Control 头
3. COOP/COEP 头
4. 排除 *.map 文件
5. SPA 路由重写规则
```

---

## 🔗 相关资源

- Flutter Web Wasm 工作流：[workflows/flutter-web-wasm-and-caching.md](../../workflows/flutter-web-wasm-and-caching.md)
- 审计 Wasm 兼容性：[audit-wasm-and-js-interop.md](audit-wasm-and-js-interop.md)
- Flutter web app 初始化：[https://docs.flutter.dev/platform-integration/web/initialization](https://docs.flutter.dev/platform-integration/web/initialization)
- Web 部署官方文档：[https://docs.flutter.dev/deployment/web](https://docs.flutter.dev/deployment/web)
- COOP/COEP 指南：[https://web.dev/articles/coop-coep](https://web.dev/articles/coop-coep)
- Workbox 官方文档：[https://developers.google.com/web/tools/workbox](https://developers.google.com/web/tools/workbox)

---

**按官方最佳实践配置，确保每一步都能验证通过。**
