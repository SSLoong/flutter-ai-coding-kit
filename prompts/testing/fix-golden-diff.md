# 提示词：修复 Golden 测试差异

**用途**：排查和修复 Golden 测试快照对比失败的问题

---

## 📋 适用场景

- Golden 测试本地通过，CI 失败
- 更新 UI 后快照对比失败
- 字体/平台渲染差异导致失败
- 动画/异步未完成导致快照不一致

---

## 📥 输入要求

准备以下信息：

1. **失败的测试文件**（如 `test/widgets/my_widget_test.dart`）
2. **错误日志**（`flutter test` 输出）
3. **（若有）CI 失败截图或 diff 图片**
4. **预期行为描述**（UI 应该长什么样）

---

## 💬 提示词模板

### 场景 1: 本地通过，CI 失败

```markdown
我的 Golden 测试在本地通过，但 CI 上失败。

**测试文件**：`test/widgets/[WIDGET_FILE_NAME]_test.dart`

**CI 错误日志**：
[粘贴 CI 失败日志]

**环境信息**：
- 本地 Flutter 版本：[如 3.27.2]
- CI Flutter 版本：[如 3.27.2 或未知]
- 本地操作系统：[如 macOS 14]
- CI 操作系统：[如 Ubuntu 22.04]

**请分析可能原因并给出修复方案，考虑：**
1. 字体差异（本地 vs CI）
2. Flutter SDK 版本不一致
3. 渲染引擎差异（macOS vs Linux）
4. 是否需要统一使用 `--platform chrome`
```

### 场景 2: 修改 UI 后快照失败

```markdown
我修改了 Widget 的 UI，Golden 测试现在失败了。

**测试文件**：`test/widgets/[WIDGET_FILE_NAME]_test.dart`

**错误信息**：
[粘贴错误日志，如 "Golden 'goldens/xxx.png' differs by X%"]

**UI 变更描述**：
[描述你改了什么，如："增加了边距"、"修改了字体大小"]

**问题**：
1. 这些失败是预期的吗（UI 有意变更）？
2. 如何验证新的快照是否正确？
3. 更新基线后需要检查什么？

请给出验证和更新基线的步骤。
```

### 场景 3: 快照渲染不完整

```markdown
Golden 测试生成的快照显示 Widget 未完全渲染（部分元素缺失或动画未完成）。

**测试代码片段**：
```dart
testWidgets('my test', (tester) async {
  await tester.pumpWidget(MaterialApp(home: MyWidget()));
  await tester.pumpAndSettle(); // ← 可能这里有问题
  
  await expectLater(
    find.byType(MyWidget),
    matchesGoldenFile('goldens/my_widget.png'),
  );
});
```

**现象**：
[描述，如："图片未加载"、"动画停在中间帧"]

请修复测试代码，确保快照捕获完整的渲染状态。
```

### 场景 4: 随机或动态数据导致失败

```markdown
Golden 测试每次运行快照都不同，因为 Widget 使用了动态数据。

**测试文件**：`test/widgets/[WIDGET_FILE_NAME]_test.dart`

**Widget 中使用的动态数据**：
[如：DateTime.now(), Random(), 网络图片 URL]

**当前测试代码**：
```dart
// 贴上现有测试代码
```

请修改测试，用固定的 mock 数据替换动态值，确保快照稳定。
```

---

## 🔧 常见修复方案

### 1. 统一 CI 和本地环境

**问题**：字体、渲染引擎差异

**解决方案**：

```yaml
# .github/workflows/test.yml
jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      - uses: subosito/flutter-action@v2
        with:
          flutter-version: '3.27.2'  # 与本地一致
      
      - name: Run Golden tests
        run: flutter test --platform chrome  # 统一渲染引擎
```

**提示词**：
```markdown
我需要修改 CI 配置，使 Golden 测试使用 Chrome 平台渲染引擎。

当前 CI 配置文件：[粘贴 .github/workflows/test.yml]

请修改配置，添加 `--platform chrome` 参数。
```

---

### 2. 更新 Golden 基线

**问题**：UI 有意修改后，快照对比失败

**解决方案**：

```bash
# Step 1: 更新基线
flutter test --update-goldens test/widgets/my_widget_test.dart

# Step 2: 目视检查新快照
open test/goldens/my_widget_*.png

# Step 3: 验证测试通过
flutter test test/widgets/my_widget_test.dart

# Step 4: 提交新基线
git add test/goldens/
git commit -m "更新：my_widget Golden 基线（UI 调整）"
```

**检查清单**：
- [ ] 新快照目视检查符合预期
- [ ] 所有相关 Golden 测试通过
- [ ] Git diff 显示的快照变化合理
- [ ] PR review 时附上快照对比说明

---

### 3. 固定测试数据

**问题**：动态数据导致快照不稳定

**修复前**：
```dart
testWidgets('date display', (tester) async {
  await tester.pumpWidget(
    MaterialApp(
      home: Scaffold(
        body: Text('Today: ${DateTime.now()}'),  // ❌ 动态
      ),
    ),
  );
  // ...
});
```

**修复后**：
```dart
testWidgets('date display', (tester) async {
  final fixedDate = DateTime(2026, 11, 6);  // ✅ 固定
  
  await tester.pumpWidget(
    MaterialApp(
      home: Scaffold(
        body: Text('Today: $fixedDate'),
      ),
    ),
  );
  // ...
});
```

**提示词**：
```markdown
修改测试代码，将所有动态数据替换为固定值：

[粘贴测试代码]

需要固定的动态值：
- DateTime.now() → DateTime(2026, 11, 6)
- Random().nextInt(100) → 42
- 网络图片 → 本地 asset 或 mock
```

---

### 4. 等待渲染完成

**问题**：快照捕获时 Widget 未完全渲染

**修复前**：
```dart
testWidgets('async widget', (tester) async {
  await tester.pumpWidget(MaterialApp(home: AsyncWidget()));
  // ❌ 立即快照，异步加载未完成
  await expectLater(..., matchesGoldenFile(...));
});
```

**修复后**：
```dart
testWidgets('async widget', (tester) async {
  await tester.pumpWidget(MaterialApp(home: AsyncWidget()));
  await tester.pumpAndSettle();  // ✅ 等待所有动画和异步
  await expectLater(..., matchesGoldenFile(...));
});
```

**特殊情况**（动画不停止）：
```dart
// 等待固定时间
await tester.pump(Duration(milliseconds: 500));

// 或多次 pump 到特定帧
for (int i = 0; i < 10; i++) {
  await tester.pump(Duration(milliseconds: 100));
}
```

---

### 5. 在 CI 环境重新生成基线

**问题**：本地和 CI 无法统一渲染

**解决方案**（适用于 GitHub Actions）：

```yaml
# .github/workflows/update-goldens.yml
name: Update Golden Files

on:
  workflow_dispatch:  # 手动触发

jobs:
  update:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      - uses: subosito/flutter-action@v2
        with:
          flutter-version: '3.27.2'
      
      - run: flutter test --update-goldens --platform chrome
      
      - name: Upload Golden files
        uses: actions/upload-artifact@v3
        with:
          name: goldens
          path: test/goldens/
```

**流程**：
1. 触发 CI workflow 生成 Golden
2. 下载 artifact 中的快照文件
3. 替换本地 `test/goldens/` 目录
4. 提交新基线

---

## ✅ 验证清单

修复后确认：

- [ ] 测试在本地通过（`flutter test`）
- [ ] 测试在 CI 通过
- [ ] 快照文件目视检查符合预期
- [ ] 若更新了基线，Git diff 合理
- [ ] 测试数据固定（无动态值）
- [ ] 渲染等待充分（`pumpAndSettle` 或指定延迟）

---

## 🔗 相关

- 工作流：[workflows/golden-test-practice.md](../../workflows/golden-test-practice.md)
- 生成 Golden 测试：[generate-golden-test.md](generate-golden-test.md)
- Widget 测试修复：[fix-flaky-widget-test.md](fix-flaky-widget-test.md)
