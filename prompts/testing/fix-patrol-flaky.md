# 提示词：修复不稳定的 Patrol 测试

**用途**：诊断并修复间歇性失败的 Patrol 端到端测试（Flaky Tests）

---

## 📋 适用场景

- Patrol 测试间歇性通过/失败
- 错误信息提示"找不到元素"或"超时"
- 本地通过但 CI 失败
- 不同设备上表现不一致

---

## 📥 输入要求

在 Cursor 中准备：

1. **失败的测试代码**：`integration_test/[文件名]_test.dart`
2. **错误日志**：完整的 `patrol test` 输出或 CI 日志
3. **失败截图/视频**（如有）：Patrol 自动生成的失败快照
4. **可复现步骤**：如"运行 3 次中有 1 次失败"

---

## 💬 提示词模板

```markdown
我有一个 Patrol 端到端测试间歇性失败，需要诊断并修复。

**测试代码**：
```dart
[粘贴失败的测试代码片段，尤其是失败的步骤]
```

**错误信息**：
```
[粘贴完整错误日志，包括堆栈跟踪]
```

**失败频率**：[如"10 次运行中失败 3 次"、"仅在 CI 环境失败"、"Android 上失败但 iOS 正常"]

**失败场景**：[如"点击按钮时找不到元素"、"等待页面加载超时"]

**请分析可能原因并提供修复方案，考虑以下因素**：

1. **异步时序问题**
   - 是否需要 `waitUntilVisible()` 等待元素加载？
   - 动画是否完成？（`pumpAndSettle()` 不足）
   - 网络请求是否延迟？

2. **Finder 定位问题**
   - Key 是否唯一？（多个同名 Widget）
   - 元素是否在视口外？（需滚动）
   - Semantics label 是否正确？

3. **平台差异**
   - iOS/Android 表现不同？
   - CI 环境性能较低导致超时？

4. **状态污染**
   - 前一个测试是否影响后续测试？
   - 是否需要 `tearDown()` 清理？

生成修复后的测试代码和建议的测试策略。
```

---

## 🔍 常见原因与修复方案

### 1️⃣ 异步等待不足

**症状**：错误 `Finder 找不到元素` 或 `Expected to find: Widget, actual: 0`

**原因**：元素还在加载/动画中，测试就尝试交互

**修复**：

```dart
// ❌ 不稳定：立即查找可能失败
await $(#submitButton).tap();

// ✅ 稳定：显式等待元素可见
await $(#submitButton).waitUntilVisible();
await $(#submitButton).tap();

// ✅ 或增加自定义等待时间
await $.tester.pumpAndSettle(Duration(seconds: 2));
await $(#submitButton).tap();
```

---

### 2️⃣ 网络请求时序不确定

**症状**：数据加载慢导致测试超时

**修复**：

```dart
// ❌ 不稳定：依赖真实网络请求
patrolTest('加载商品列表', ($) async {
  await $(#refreshButton).tap();
  await $(#productList).waitUntilVisible();  // 可能超时
});

// ✅ 稳定：Mock 网络请求或使用测试环境
// 方法 1: 在 main.dart 中注入 mock 数据
void main() {
  if (const String.fromEnvironment('ENV') == 'test') {
    runApp(MyApp(api: MockApi()));
  } else {
    runApp(MyApp(api: RealApi()));
  }
}

// 方法 2: 增加合理的超时时间
await $(#productList).waitUntilVisible(
  timeout: Duration(seconds: 10),
);
```

---

### 3️⃣ 动画未完成

**症状**：页面跳转或组件动画期间点击失败

**修复**：

```dart
// ❌ 不稳定：动画期间点击
await $(#nextPageButton).tap();
await $(#targetWidget).tap();  // 可能页面还在切换

// ✅ 稳定：等待动画完成
await $(#nextPageButton).tap();
await $.pumpAndSettle();  // 等待所有动画和帧完成
await $(#targetWidget).waitUntilVisible();
await $(#targetWidget).tap();
```

---

### 4️⃣ 元素在视口外

**症状**：长列表底部的元素找不到

**修复**：

```dart
// ❌ 不稳定：元素可能在屏幕外
await $(#bottomButton).tap();

// ✅ 稳定：先滚动到元素位置
await $(#bottomButton).scrollTo();
await $(#bottomButton).tap();

// ✅ 或在特定列表中滚动
await $(ListView).containing($(#bottomButton)).scrollTo();
```

---

### 5️⃣ Key 冲突或不唯一

**症状**：`Expected to find 1, actual: 2` 或点击了错误的元素

**修复**：

```dart
// ❌ 不稳定：多个相同 Key 的 Widget
ListView(
  children: [
    ProductCard(key: Key('productCard')),  // 冲突！
    ProductCard(key: Key('productCard')),  // 冲突！
  ],
)

// ✅ 稳定：使用唯一 Key
ListView.builder(
  itemBuilder: (context, index) {
    return ProductCard(key: Key('productCard_$index'));
  },
)

// 测试中使用
await $(Key('productCard_0')).tap();  // 点击第一个
```

---

### 6️⃣ 原生权限弹窗时序

**症状**：`$.native.grantPermission()` 调用时弹窗未出现

**修复**：

```dart
// ❌ 不稳定：立即处理权限可能太早
await $(#requestLocationButton).tap();
await $.native.grantPermissionWhenInUse();  // 弹窗可能还没出现

// ✅ 稳定：等待原生弹窗出现
await $(#requestLocationButton).tap();
await Future.delayed(Duration(milliseconds: 500));  // 等待系统弹窗
await $.native.grantPermissionWhenInUse();

// ✅ 或使用 Patrol 的自动等待
await $(#requestLocationButton).tap();
await $.native.tap(Selector(textContains: 'Allow'));  // 自动等待原生元素
```

---

### 7️⃣ CI 环境性能差异

**症状**：本地通过，CI 环境超时

**修复**：

```dart
// CI 环境增加 timeout
patrolTest(
  '长流程测试',
  ($) async {
    // 测试逻辑
  },
  config: PatrolTesterConfig(
    timeout: Duration(minutes: 5),  // CI 环境增加超时
  ),
);

// 或在 patrol.toml 配置
// [test]
// timeout = "5m"
```

---

### 8️⃣ 状态污染（前一个测试影响后续）

**症状**：单独运行通过，批量运行失败

**修复**：

```dart
void main() {
  late AppDatabase database;

  setUp(() async {
    // 每个测试前重置状态
    database = await setupTestDatabase();
  });

  tearDown(() async {
    // 清理
    await database.close();
  });

  patrolTest('测试 A', ($) async {
    // 使用干净的 database
  });

  patrolTest('测试 B', ($) async {
    // 使用干净的 database，不受测试 A 影响
  });
}
```

---

## ✅ 验证修复效果

修复后需验证稳定性：

```bash
# 连续运行 10 次，检查是否仍失败
for i in {1..10}; do
  echo "===== Run $i ====="
  patrol test -t integration_test/[文件名]_test.dart
  if [ $? -ne 0 ]; then
    echo "Failed on run $i"
    break
  fi
done
```

**通过标准**：
- 连续运行 10 次全部通过
- 不同设备（iOS/Android）均通过
- CI 环境通过

---

## 💡 预防性最佳实践

1. **默认使用 `waitUntilVisible()`**
   ```dart
   // 养成习惯：交互前先等待
   await $(#button).waitUntilVisible();
   await $(#button).tap();
   ```

2. **Mock 网络请求**
   - 使用 `mockito` 或 `http_mock_adapter` 模拟 API
   - 端到端测试不应依赖真实后端稳定性

3. **使用唯一且语义化的 Key**
   ```dart
   // ❌ 不好
   TextField(key: Key('field1'))
   
   // ✅ 好
   TextField(key: Key('emailInputField'))
   ```

4. **视口外元素先滚动**
   ```dart
   await $(#element).scrollTo();
   await $(#element).tap();
   ```

5. **CI 环境增加超时**
   - CI 机器性能通常低于本地，设置 1.5-2 倍超时

6. **隔离测试状态**
   - 每个测试独立启动应用（`app.main()`）
   - 使用 `setUp()` / `tearDown()` 清理数据

---

## 🔗 相关

- 工作流：[workflows/patrol-ui-automation.md](../../workflows/patrol-ui-automation.md)
- 生成 Patrol 测试：[generate-patrol-test.md](generate-patrol-test.md)
- Widget 测试修复：[fix-flaky-widget-test.md](fix-flaky-widget-test.md)
