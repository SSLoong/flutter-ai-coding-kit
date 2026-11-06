# Golden 测试实践工作流

**用 AI 辅助生成 Flutter UI 快照测试，验证多尺寸和主题变体**

> 作者：Loong (SSLoong) | 日期：2026-11-06

---

## 📋 背景

Golden 测试（也称 UI 快照测试）通过捕获 Widget 渲染结果的图像快照，对比后续变化来发现意外的 UI 破坏。相比传统的 `expect(find.text('XXX'), findsOneWidget)` 断言，Golden 测试能捕捉**视觉层面**的回归（字体、颜色、布局、边距）。

AI 可以快速生成 Golden 测试骨架、覆盖多尺寸和主题变体，但**必须本地/CI 验证通过**，禁止无验证合入。

---

## 🎯 目标

- ✅ 为纯展示型 Widget 生成 `matchesGoldenFile` 测试
- ✅ 覆盖多尺寸（phone/tablet）、亮暗色模式、主题变体
- ✅ 建立 Golden 基线（`--update-goldens`）并纳入回归验证
- ✅ 排查 CI 上的字体/平台渲染差异

---

## 🔧 适用场景

| 场景 | 适合度 | 说明 |
|------|--------|------|
| 纯展示型 Widget（Card、Banner、Info 面板） | ⭐⭐⭐⭐⭐ | 无复杂交互，视觉一致性优先 |
| 多主题/暗色模式适配 | ⭐⭐⭐⭐⭐ | 快速验证 Brightness.dark |
| 响应式布局（phone/tablet/desktop） | ⭐⭐⭐⭐ | 不同尺寸快照对比 |
| 复杂交互 Widget（表单、动画） | ⭐⭐ | 优先用行为断言，Golden 作补充 |
| 动态数据渲染 | ⭐⭐⭐ | 需 mock 数据保证快照稳定性 |

**选择建议**：
- **优先 Golden**：静态展示、主题一致性、多尺寸适配场景
- **优先行为断言**：复杂交互、状态变化、业务逻辑验证
- **组合使用**：核心业务组件用行为断言 + Golden 覆盖视觉层

---

## 🚀 完整流程

### Step 1: 准备上下文

**人工操作**：在 Cursor 中打开目标 Widget 文件（如 `lib/widgets/product_info_card.dart`），附上依赖（如数据模型、主题配置）。

**AI 提示词**（可复制）：

```markdown
我需要为这个 Widget 生成 Golden 测试。请先分析：

1. 这是纯展示组件还是包含交互？
2. 依赖哪些外部数据（需 mock）？
3. 是否需要验证多主题（亮色/暗色）？
4. 是否需要验证多尺寸（手机/平板）？
5. 有哪些视觉状态变体（正常/加载/错误）？

列出需要快照的场景，再生成测试代码。
```

**检查点**：确认 AI 列出的场景覆盖核心视觉变体（至少包含亮暗色模式）。

---

### Step 2: 生成 Golden 测试骨架

**AI 提示词**（可复制）：

```markdown
基于上述场景，生成 `test/widgets/product_info_card_test.dart`，要求：

1. **基础结构**：
   - 使用 `testWidgets()` 和 `expectLater(..., matchesGoldenFile(...))`
   - 每个快照文件命名：`goldens/product_info_card_[场景描述].png`

2. **多尺寸支持**（必需）：
   - 用 `tester.view.physicalSize` 和 `tester.view.devicePixelRatio` 设置尺寸
   - 至少覆盖：手机（375x667 @2x）、平板（768x1024 @2x）

3. **多主题支持**（必需）：
   - 亮色模式：`ThemeData.light()` + `Brightness.light`
   - 暗色模式：`ThemeData.dark()` + `Brightness.dark`

4. **Mock 数据**：
   - 使用固定测试数据（避免随机值）
   - 用 `mocktail` mock 外部依赖

5. **完整性**：
   - 包含所有 import（含 `dart:ui` for FlutterView）
   - 添加 `setUp()` / `tearDown()` 恢复默认尺寸
   - 可直接运行 `flutter test --update-goldens`

生成完整代码，不要省略。
```

**输出示例**（参考，实际以 AI 生成为准）：

```dart
import 'package:flutter/material.dart';
import 'package:flutter_test/flutter_test.dart';
import 'package:my_app/widgets/product_info_card.dart';
import 'package:my_app/models/product.dart';

void main() {
  late Product testProduct;

  setUp(() {
    testProduct = Product(
      id: '123',
      name: 'Test Product',
      price: 99.99,
      description: 'A sample product for testing',
    );
  });

  group('ProductInfoCard Golden Tests', () {
    testWidgets('phone light mode', (tester) async {
      // 设置手机尺寸
      tester.view.physicalSize = const Size(375, 667);
      tester.view.devicePixelRatio = 2.0;

      await tester.pumpWidget(
        MaterialApp(
          theme: ThemeData.light(),
          home: Scaffold(
            body: ProductInfoCard(product: testProduct),
          ),
        ),
      );
      await tester.pumpAndSettle();

      await expectLater(
        find.byType(ProductInfoCard),
        matchesGoldenFile('goldens/product_info_card_phone_light.png'),
      );

      // 恢复默认尺寸
      addTearDown(() {
        tester.view.resetPhysicalSize();
        tester.view.resetDevicePixelRatio();
      });
    });

    testWidgets('phone dark mode', (tester) async {
      tester.view.physicalSize = const Size(375, 667);
      tester.view.devicePixelRatio = 2.0;

      await tester.pumpWidget(
        MaterialApp(
          theme: ThemeData.dark(),
          home: Scaffold(
            body: ProductInfoCard(product: testProduct),
          ),
        ),
      );
      await tester.pumpAndSettle();

      await expectLater(
        find.byType(ProductInfoCard),
        matchesGoldenFile('goldens/product_info_card_phone_dark.png'),
      );

      addTearDown(() {
        tester.view.resetPhysicalSize();
        tester.view.resetDevicePixelRatio();
      });
    });

    testWidgets('tablet light mode', (tester) async {
      tester.view.physicalSize = const Size(768, 1024);
      tester.view.devicePixelRatio = 2.0;

      await tester.pumpWidget(
        MaterialApp(
          theme: ThemeData.light(),
          home: Scaffold(
            body: ProductInfoCard(product: testProduct),
          ),
        ),
      );
      await tester.pumpAndSettle();

      await expectLater(
        find.byType(ProductInfoCard),
        matchesGoldenFile('goldens/product_info_card_tablet_light.png'),
      );

      addTearDown(() {
        tester.view.resetPhysicalSize();
        tester.view.resetDevicePixelRatio();
      });
    });

    testWidgets('tablet dark mode', (tester) async {
      tester.view.physicalSize = const Size(768, 1024);
      tester.view.devicePixelRatio = 2.0;

      await tester.pumpWidget(
        MaterialApp(
          theme: ThemeData.dark(),
          home: Scaffold(
            body: ProductInfoCard(product: testProduct),
          ),
        ),
      );
      await tester.pumpAndSettle();

      await expectLater(
        find.byType(ProductInfoCard),
        matchesGoldenFile('goldens/product_info_card_tablet_dark.png'),
      );

      addTearDown(() {
        tester.view.resetPhysicalSize();
        tester.view.resetDevicePixelRatio();
      });
    });
  });
}
```

---

### Step 3: 生成 Golden 基线（必须！）

**首次运行**时，需生成基准快照图像：

```bash
# 生成所有 Golden 基线
flutter test --update-goldens test/widgets/product_info_card_test.dart

# 检查生成的文件
ls -lh test/goldens/
```

**检查清单**：
- [ ] `test/goldens/` 目录下生成了对应的 `.png` 文件
- [ ] 快照文件大小合理（通常 10-100KB）
- [ ] 目视检查快照图像是否符合预期（可用图片查看器打开）

**重要**：Golden 基线文件需提交到 Git，供团队和 CI 使用。

---

### Step 4: 本地验证回归

**后续运行**时，测试会对比当前渲染与基线快照：

```bash
# 运行 Golden 测试（不更新基线）
flutter test test/widgets/product_info_card_test.dart

# 若 UI 变化是预期的，更新基线
flutter test --update-goldens test/widgets/product_info_card_test.dart
```

**检查清单**：
- [ ] 测试通过（0 failed）
- [ ] 无意外的快照差异
- [ ] 若有差异，确认是有意为之（如设计调整）后再更新

**常见问题与修复**：
- ❌ `Golden "XXX.png" does not exist`：首次需 `--update-goldens`
- ❌ 快照对比失败但目视无差异：可能是字体/渲染引擎差异（见 Step 5）
- ❌ `tester.view` 报错：确保 `flutter_test` 版本 ≥3.0

---

### Step 5: 排查 CI 上的 Golden 失败

Golden 测试在 **本地通过、CI 失败** 是常见问题，原因包括：

| 原因 | 表现 | 解决方案 |
|------|------|----------|
| **字体差异** | 文本位置/高度不同 | CI 用 `--platform chrome` 或 `flutter test --font-subset` |
| **Flutter SDK 版本** | 渲染引擎行为变化 | 统一 CI 与本地 Flutter 版本（如 3.27.2） |
| **操作系统差异** | macOS vs Linux 像素级差异 | 仅在 CI 环境生成基线，或用 Docker 统一 |
| **本地改 UI 未更新基线** | 预期差异 | 本地 `--update-goldens` 后提交 |
| **动画/异步未完成** | 部分渲染 | 确保 `await tester.pumpAndSettle()` |

**AI 辅助排查提示词**：

```markdown
我的 Golden 测试在 CI 上失败，本地通过。CI 错误信息：

[粘贴 CI 日志或失败截图]

请分析可能原因，并给出修复步骤（考虑字体、SDK 版本、平台差异）。
```

**人工检查点**：
1. 对比 CI 生成的失败截图（若有）与本地快照
2. 检查 CI 配置文件（如 `.github/workflows/test.yml`）的 Flutter 版本
3. 在本地用 `flutter test --platform chrome` 重现 CI 环境

---

### Step 6: 迭代改进

**若测试失败**，用以下提示词让 AI 修复：

```markdown
Golden 测试运行报错：

[粘贴错误信息]

请修复代码，确保：
1. `tester.view` 设置和恢复正确
2. Mock 数据稳定（无随机值）
3. 等待异步渲染完成
4. 快照文件路径正确
```

**若要新增场景**（如错误状态）：

```markdown
补充 Golden 测试场景：当数据加载失败时的错误提示 UI。

生成新的 `testWidgets` 并添加对应快照。
```

---

## ✅ 验证清单

完成后确认：

- [ ] 测试文件位于 `test/` 目录，路径对应 `lib/`
- [ ] 生成了 Golden 基线文件（`.png`）并提交到 Git
- [ ] 所有测试通过 `flutter test`（不带 `--update-goldens`）
- [ ] 覆盖至少 2 个尺寸 + 亮暗色模式（共 4 个快照）
- [ ] Mock 数据固定（无随机值、时间戳）
- [ ] CI 环境测试通过（或已排查差异原因）

---

## 🎨 Golden 测试 vs 行为断言

| 维度 | Golden 测试 | 传统行为断言 |
|------|-------------|--------------|
| **适用场景** | 纯展示、主题、多尺寸 | 交互、状态变化、业务逻辑 |
| **维护成本** | UI 变化需更新基线 | 断言相对稳定 |
| **失败信息** | 截图对比 | 具体断言点 |
| **CI 敏感度** | 字体/平台差异敏感 | 较稳定 |
| **覆盖深度** | 像素级视觉回归 | 功能行为正确性 |

**推荐策略**：
- **核心业务组件**：行为断言为主，Golden 补充视觉验证
- **纯展示组件**：Golden 优先，减少冗余断言
- **主题适配**：必须有 Golden 覆盖亮暗色
- **响应式布局**：Golden 验证多尺寸，行为断言验证交互

---

## 🤔 FAQ

### Q1: Golden 文件应该提交到 Git 吗？
**A**: **推荐提交**。理由：
- ✅ 团队和 CI 可直接使用基线
- ✅ PR review 可看到视觉变化（GitHub 会显示图片 diff）
- ❌ 但会增加仓库体积（单个快照 10-100KB）

**不提交的情况**：
- 团队约定每次 CI 重新生成基线（需配置 CI 脚本）
- 项目极度敏感体积（但测试快照通常占比小）

### Q2: 本地通过，CI 失败怎么办？
**A**: 按优先级排查：
1. **统一 Flutter 版本**：本地和 CI 用同一版本（如 `3.27.2`）
2. **用 Chrome 平台**：CI 配置 `flutter test --platform chrome`
3. **统一字体**：CI 和本地都加载 Roboto 等标准字体
4. **在 CI 环境生成基线**：用 CI artifact 下载快照，替换本地基线

### Q3: 改了 UI，Golden 大量失败正常吗？
**A**: **正常**。这正是 Golden 测试的价值：
- 📌 **有意改动**：设计调整后，运行 `--update-goldens` 更新基线
- ⚠️ **意外破坏**：未预期的变化，需检查代码回滚或修复

**流程**：
1. 改动前：所有 Golden 通过
2. 改动后：目视检查失败的快照差异
3. 确认无误后：`--update-goldens` 并提交新基线

### Q4: Golden 测试能捕获动画吗？
**A**: **部分支持**，但不推荐：
- ✅ 动画的**特定帧**可以快照（如起始/结束状态）
- ❌ 动画**过程**无法验证（需多帧快照，维护成本高）
- 推荐：动画用 `AnimationController` 测试 + 关键帧 Golden

### Q5: AI 生成的快照能直接信任吗？
**A**: **不能**。必须：
1. **目视检查**生成的快照图像（用图片查看器打开）
2. **本地运行**确认测试通过
3. **CI 验证**确保团队环境一致
4. **Code review**时检查快照合理性

---

## 💡 最佳实践

1. **上下文包含视觉信息**
   - 附上 Widget 源码、设计稿（如有）、主题配置

2. **固定测试数据**
   - 避免 `DateTime.now()`、`Random()`、动态加载图片
   - 用 mock 数据保证快照稳定

3. **快照文件命名清晰**
   - 格式：`goldens/[组件名]_[尺寸]_[主题]_[状态].png`
   - 示例：`product_card_phone_dark_loading.png`

4. **CI 配置明确**
   - 在 `.github/workflows/test.yml` 中指定 Flutter 版本
   - 考虑用 `--platform chrome` 统一渲染

5. **渐进式采用**
   - 先为 1-2 个核心展示组件试点
   - 验证团队流程后再扩展到更多组件

6. **记录平台差异**
   - 把 CI 修复经验写入 `AGENTS.md` 或项目 README

---

## 🔗 相关资源

- 提示词模板：[prompts/testing/generate-golden-test.md](../prompts/testing/generate-golden-test.md)
- 修复 Golden 差异：[prompts/testing/fix-golden-diff.md](../prompts/testing/fix-golden-diff.md)
- Widget 测试工作流：[workflows/widget-test-generation.md](widget-test-generation.md)
- Flutter 官方文档：[Golden file testing](https://docs.flutter.dev/cookbook/testing/widget/golden-file-testing)

---

**让 UI 回归测试更可靠！✨**
