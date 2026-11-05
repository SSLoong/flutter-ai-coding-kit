# Widget 测试生成工作流

**用 AI 为 Flutter Widget 和业务层自动生成测试用例**

> 作者：Loong (SSLoong) | 日期：2026-11-05

---

## 📋 背景

手写 Widget 测试耗时且容易遗漏边界场景。AI 能快速生成基础测试骨架和常见用例，但**必须本地验证**通过才能合入代码。

---

## 🎯 目标

- ✅ 为已有 Widget 生成符合 Arrange-Act-Assert 模式的测试
- ✅ 覆盖正常路径、边界条件、用户交互
- ✅ 集成 mocktail/mockito 模拟依赖
- ✅ 可选生成 Golden 测试（UI 快照）

---

## 🔧 适用场景

| 场景 | 适合度 | 说明 |
|------|--------|------|
| 无测试的业务 Widget | ⭐⭐⭐⭐⭐ | 从 0 到 1 |
| 重构后补测试 | ⭐⭐⭐⭐ | 验证行为不变 |
| 复杂状态交互 | ⭐⭐⭐ | 需人工调整 mock |
| 纯 UI 展示组件 | ⭐⭐⭐⭐⭐ | Golden 测试效果好 |

---

## 🚀 完整流程

### Step 1: 准备上下文

**人工操作**：在 Cursor 中打开目标 Widget 文件（如 `lib/widgets/user_card.dart`），附上相关依赖（如 ViewModel、Service）。

**AI 提示词**（可复制）：

```markdown
我需要为这个 Widget 生成测试。请先确认：

1. 这个 Widget 的职责是什么？
2. 有哪些输入参数？
3. 依赖哪些外部服务或 Provider？
4. 有哪些用户交互（tap、scroll、input）？

列出需要覆盖的测试场景，再生成代码。
```

**检查点**：AI 列出的场景是否覆盖你的预期？遗漏就明确补充。

---

### Step 2: 生成测试骨架

**AI 提示词**（可复制）：

```markdown
基于上述场景，生成 `test/widgets/user_card_test.dart`，要求：

- 使用 `flutter_test` 包
- 每个测试用 `testWidgets()`
- Arrange-Act-Assert 三段式
- 用 `mocktail` mock 依赖（如 UserService）
- 用 `find` + `expect` 验证 UI 元素
- 用 `tester.tap()` / `tester.enterText()` 模拟交互
- 测试组用 `group()` 分类

生成完整可运行代码，不要省略 import。
```

**输出示例**（参考，实际以 AI 生成为准）：

```dart
import 'package:flutter_test/flutter_test.dart';
import 'package:flutter/material.dart';
import 'package:mocktail/mocktail.dart';
import 'package:my_app/widgets/user_card.dart';
import 'package:my_app/services/user_service.dart';

class MockUserService extends Mock implements UserService {}

void main() {
  late MockUserService mockUserService;

  setUp(() {
    mockUserService = MockUserService();
  });

  group('UserCard Widget Tests', () {
    testWidgets('displays user name and avatar', (tester) async {
      // Arrange
      when(() => mockUserService.getUser('123'))
          .thenAnswer((_) async => User(id: '123', name: 'Alice'));

      // Act
      await tester.pumpWidget(
        MaterialApp(
          home: UserCard(userId: '123', userService: mockUserService),
        ),
      );
      await tester.pumpAndSettle();

      // Assert
      expect(find.text('Alice'), findsOneWidget);
      expect(find.byType(CircleAvatar), findsOneWidget);
    });

    testWidgets('shows loading indicator initially', (tester) async {
      // Arrange
      when(() => mockUserService.getUser('123'))
          .thenAnswer((_) async => Future.delayed(Duration(seconds: 1)));

      // Act
      await tester.pumpWidget(
        MaterialApp(
          home: UserCard(userId: '123', userService: mockUserService),
        ),
      );

      // Assert
      expect(find.byType(CircularProgressIndicator), findsOneWidget);
    });

    testWidgets('handles tap on card', (tester) async {
      // Arrange
      bool tapped = false;
      when(() => mockUserService.getUser('123'))
          .thenAnswer((_) async => User(id: '123', name: 'Bob'));

      // Act
      await tester.pumpWidget(
        MaterialApp(
          home: UserCard(
            userId: '123',
            userService: mockUserService,
            onTap: () => tapped = true,
          ),
        ),
      );
      await tester.pumpAndSettle();
      await tester.tap(find.byType(UserCard));

      // Assert
      expect(tapped, isTrue);
    });
  });
}
```

---

### Step 3: 本地验证（必须！）

**人工操作**：

```bash
# 运行生成的测试
flutter test test/widgets/user_card_test.dart

# 查看覆盖率（可选）
flutter test --coverage
genhtml coverage/lcov.info -o coverage/html
open coverage/html/index.html  # macOS
```

**检查清单**：
- [ ] 所有测试通过（0 failed）
- [ ] 无 import 错误或 mock 问题
- [ ] 覆盖率达到预期（建议 ≥80%）
- [ ] 测试运行时间合理（单个 Widget 测试 <5s）

**常见问题**：
- ❌ `MissingPluginException`：需要用 `TestWidgetsFlutterBinding.ensureInitialized()`
- ❌ Mock 类型不匹配：检查 `when()` 的返回类型
- ❌ Finder 找不到 Widget：加 `await tester.pumpAndSettle()` 等待动画
- ❌ Golden 测试失败：首次运行需 `flutter test --update-goldens`

---

### Step 4: 迭代改进

**若测试失败**，用以下提示词让 AI 修复：

```markdown
测试运行报错：

[粘贴错误信息]

请修复测试代码，确保：
1. Import 正确
2. Mock 返回值类型匹配
3. Finder 能找到 Widget（必要时加 Key）
```

**若要新增场景**（如错误处理）：

```markdown
补充测试场景：当 userService.getUser() 抛异常时，显示错误提示。

生成新的 `testWidgets`，加入现有 group。
```

---

## 🎨 可选：生成 Golden 测试

**适合纯展示型 Widget**（无复杂交互）。

**AI 提示词**：

```markdown
为 UserCard 生成 Golden 测试，验证 UI 快照在不同状态下的一致性。

场景：
1. 正常加载后的样子
2. 加载中状态
3. 错误状态

使用 `matchesGoldenFile('goldens/user_card_XXX.png')`。
```

**验证步骤**：

```bash
# 首次生成基准图
flutter test --update-goldens test/widgets/user_card_test.dart

# 后续验证
flutter test test/widgets/user_card_test.dart
```

---

## ✅ 验证清单

完成后确认：

- [ ] 测试文件位于 `test/` 目录，路径对应 `lib/`
- [ ] 所有测试通过 `flutter test`
- [ ] 使用 mocktail/mockito mock 依赖
- [ ] 测试遵循 Arrange-Act-Assert 结构
- [ ] 覆盖正常路径 + 至少一个边界场景
- [ ] Golden 测试（如有）基准图已提交

---

## 🤔 FAQ

### Q1: AI 生成的 mock 代码报错？
**A**: 确认依赖版本：
- `mocktail: ^1.0.0` (推荐) 或 `mockito: ^5.0.0` + `build_runner`
- 检查 mock 类是否正确 extend `Mock` 并 implement 目标接口

### Q2: 测试太慢怎么办？
**A**: 
- 避免真实网络请求，用 mock
- 减少 `pump()` 次数，用 `pumpAndSettle()` 一次性等待
- 拆分成多个小测试文件

### Q3: Golden 测试在 CI 上失败？
**A**: 
- 使用 `flutter test --platform chrome` 统一渲染引擎
- 提交 goldens/ 到 Git
- CI 环境保持一致（Flutter SDK 版本、字体）

### Q4: AI 能生成集成测试吗？
**A**: 
- Widget 测试 ✅（本工作流）
- Integration 测试 ⚠️（需明确流程，可另开工作流）
- Patrol/Maestro ⚠️（复杂交互建议手动编写关键场景）

---

## 💡 最佳实践

1. **上下文越详细，生成质量越高**
   - 附上 Widget 源码、依赖接口、甚至已有测试示例

2. **验证标准前置**
   - 告诉 AI "测试必须通过 `flutter test` 且覆盖率 ≥80%"

3. **小步快跑**
   - 先生成 1-2 个测试验证可行，再批量生成

4. **记录常见错误**
   - 把修复过的 AI 错误加入项目 `.cursor/rules` 或 `AGENTS.md`

5. **人工审查 Assert**
   - AI 可能漏关键断言（如状态变化、回调调用），需补充

---

## 🔗 相关资源

- 提示词模板：[prompts/testing/generate-widget-test.md](../prompts/testing/generate-widget-test.md)
- 修复脆弱测试：[prompts/testing/fix-flaky-widget-test.md](../prompts/testing/fix-flaky-widget-test.md)
- 上下文工程：[docs/context-engineering.md](../docs/context-engineering.md)
- Flutter 官方测试文档：[https://docs.flutter.dev/testing](https://docs.flutter.dev/testing)

---

**Happy Testing! 🧪**
