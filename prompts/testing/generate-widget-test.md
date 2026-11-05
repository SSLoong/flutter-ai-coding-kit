# 提示词：生成 Widget 测试

**用途**：为现有 Flutter Widget 生成完整的单元测试用例

---

## 📋 适用场景

- 新建 Widget 未编写测试
- 重构后需补充测试覆盖
- 手动测试耗时，需自动化验证

---

## 📥 输入要求

在 Cursor 中准备：

1. **目标 Widget 源码**（如 `lib/widgets/product_card.dart`）
2. **依赖接口/服务**（如 ViewModel、Repository）
3. **（可选）现有测试示例**（作为风格参考）

---

## 💬 提示词模板

```markdown
我需要为以下 Flutter Widget 生成测试文件。

**目标 Widget**：`lib/widgets/[WIDGET_FILE_NAME].dart`

**要求**：

1. **测试文件路径**：`test/widgets/[WIDGET_FILE_NAME]_test.dart`

2. **依赖管理**：
   - 使用 `mocktail` mock 以下依赖：[列出依赖，如 UserRepository, AuthService]
   - 在 `setUp()` 中初始化 mock 对象

3. **测试场景覆盖**：
   - ✅ 正常渲染（显示正确文本、图片、按钮）
   - ✅ 加载状态（显示 loading indicator）
   - ✅ 错误状态（显示错误提示）
   - ✅ 用户交互（tap、scroll、input）
   - ✅ 边界条件（空数据、超长文本、网络异常）

4. **代码规范**：
   - 使用 `testWidgets()` 和 `group()`
   - 遵循 Arrange-Act-Assert 三段式
   - 用 `find.byType()` / `find.text()` / `find.byKey()` 定位元素
   - 用 `expect()` 验证结果
   - 交互用 `tester.tap()` / `tester.enterText()` / `tester.drag()`
   - 等待异步用 `await tester.pumpAndSettle()`

5. **完整性**：
   - 包含所有必要的 import
   - 可直接运行 `flutter test test/widgets/[WIDGET_FILE_NAME]_test.dart`

生成完整测试代码，不要省略任何部分。
```

---

## ✅ 验证步骤

生成后执行：

```bash
# 运行测试
flutter test test/widgets/[WIDGET_FILE_NAME]_test.dart

# 查看覆盖率
flutter test --coverage test/widgets/[WIDGET_FILE_NAME]_test.dart
```

**检查清单**：
- [ ] 测试全部通过（0 failed）
- [ ] 无 import 错误
- [ ] Mock 配置正确（无类型错误）
- [ ] 覆盖核心场景（至少 3 个 testWidgets）

---

## 🛠️ 常见调整

生成后可能需要手动微调：

1. **添加 Key**：AI 可能用 `find.byType()`，但多个同类型 Widget 时需 `Key`
   ```dart
   // Widget 代码
   ElevatedButton(key: Key('submit_button'), ...)
   
   // 测试代码
   await tester.tap(find.byKey(Key('submit_button')));
   ```

2. **Mock 复杂返回值**：
   ```dart
   when(() => mockRepo.fetchData()).thenAnswer(
     (_) async => Result.success(mockData),
   );
   ```

3. **等待动画结束**：
   ```dart
   await tester.pumpAndSettle(Duration(seconds: 1));
   ```

---

## 🔗 相关

- 工作流：[workflows/widget-test-generation.md](../../workflows/widget-test-generation.md)
- 修复脆弱测试：[fix-flaky-widget-test.md](fix-flaky-widget-test.md)
