# 提示词：修复不稳定测试

**用途**：诊断并修复偶发失败的 Flutter Widget 测试（flaky test）

---

## 📋 适用场景

- 测试在 CI 或本地偶尔失败
- 依赖时间、动画、异步操作的测试
- 报错信息为 "expected ... but found ..." 或超时

---

## 📥 输入要求

1. **失败的测试文件**（如 `test/widgets/product_list_test.dart`）
2. **错误日志**（完整 stack trace）
3. **测试代码片段**（至少包含失败的 `testWidgets()`）

---

## 💬 提示词模板

```markdown
以下 Widget 测试偶发失败，需要修复。

**测试文件**：`test/widgets/[TEST_FILE_NAME]_test.dart`

**错误信息**：
```
[粘贴完整错误日志，包括 stack trace]
```

**失败的测试代码**：
```dart
[粘贴具体 testWidgets() 代码]
```

**问题分析要求**：

1. 判断失败原因（选一个或多个）：
   - ⏱️ 异步操作未等待完成
   - 🎨 动画未结束就断言
   - 🔍 Finder 在 Widget 树更新前查找
   - ⏰ 依赖系统时钟（DateTime.now()）
   - 🎲 依赖随机数或外部状态
   - 🖼️ Golden 测试平台差异

2. 提供修复方案（可选多个）：
   - 加 `await tester.pumpAndSettle()` 等待动画
   - 用 `tester.runAsync()` 包裹异步操作
   - Mock `DateTime.now()` 或注入时钟
   - 用 `FakeAsync` 控制时间流逝
   - 添加 Key 精准定位 Widget
   - 增加 `tester.pump(Duration(...))` 逐帧推进

3. 生成修复后的**完整测试代码**（不要省略 import）

---

**期望输出**：

- 明确指出问题原因
- 提供修复后的测试代码
- 说明如何验证修复效果
```

---

## ✅ 验证步骤

修复后多次运行测试确认稳定性：

```bash
# 连续运行 10 次
for i in {1..10}; do flutter test test/widgets/[TEST_FILE_NAME]_test.dart; done

# 或使用 --repeat 参数（Flutter 3.10+）
flutter test --repeat=10 test/widgets/[TEST_FILE_NAME]_test.dart
```

**通过标准**：10 次运行全部 PASS，无偶发失败。

---

## 🛠️ 常见修复模式

### 1. 等待异步加载完成

**问题**：
```dart
await tester.pumpWidget(MyWidget());
expect(find.text('Loaded'), findsOneWidget); // ❌ 数据还在加载
```

**修复**：
```dart
await tester.pumpWidget(MyWidget());
await tester.pumpAndSettle(); // ✅ 等待所有异步和动画
expect(find.text('Loaded'), findsOneWidget);
```

---

### 2. 控制时间依赖

**问题**：
```dart
testWidgets('shows timestamp', (tester) async {
  await tester.pumpWidget(MyWidget()); // 依赖 DateTime.now()
  expect(find.text('2026-11-05'), findsOneWidget); // ❌ 明天就失败
});
```

**修复**：
```dart
testWidgets('shows timestamp', (tester) async {
  final mockClock = Clock.fixed(DateTime(2026, 11, 5));
  await tester.pumpWidget(MyWidget(clock: mockClock)); // ✅ 注入固定时钟
  expect(find.text('2026-11-05'), findsOneWidget);
});
```

---

### 3. 分步推进动画

**问题**：
```dart
await tester.tap(find.byKey(Key('open')));
expect(find.text('Content'), findsOneWidget); // ❌ 展开动画还在进行
```

**修复**：
```dart
await tester.tap(find.byKey(Key('open')));
await tester.pumpAndSettle(); // ✅ 等待动画结束
expect(find.text('Content'), findsOneWidget);
```

---

### 4. 处理网络请求

**问题**：
```dart
testWidgets('fetches data', (tester) async {
  await tester.pumpWidget(MyWidget());
  // ❌ 真实网络请求，速度不可控
});
```

**修复**：
```dart
testWidgets('fetches data', (tester) async {
  when(() => mockApi.getData()).thenAnswer((_) async => mockData);
  await tester.pumpWidget(MyWidget(api: mockApi)); // ✅ Mock 网络
  await tester.pumpAndSettle();
});
```

---

## 🔗 相关

- 工作流：[workflows/widget-test-generation.md](../../workflows/widget-test-generation.md)
- 生成测试：[generate-widget-test.md](generate-widget-test.md)
- Flutter 测试最佳实践：[https://docs.flutter.dev/testing/best-practices](https://docs.flutter.dev/testing/best-practices)
