# Semantics 无障碍工作流

**用 AI 为 Flutter 应用补充无障碍语义标注与测试**

> 作者：Loong (SSLoong) | 日期：2026-11-12

---

## 📋 背景

无障碍（Accessibility）是现代应用的基本质量标准，但手动为每个 Widget 添加语义标签（Semantics）耗时且容易遗漏。AI 能快速审计现有 UI、补充缺失的语义信息，并生成自动化测试来验证屏幕阅读器体验。

---

## 🎯 目标

- ✅ 为 Widget 补充 Semantics 标签、语义标签（semanticLabel）和交互提示
- ✅ 确保可点击目标尺寸 ≥48x48 像素
- ✅ 优化屏幕阅读器（TalkBack/VoiceOver）朗读体验
- ✅ 用 flutter_test 编写语义自动化测试
- ✅ 通过 Flutter 无障碍准则检查

---

## 🔧 适用场景

| 场景 | 适合度 | 说明 |
|------|--------|------|
| 已有 UI 需补无障碍 | ⭐⭐⭐⭐⭐ | 快速审计并修复 |
| 新功能开发 | ⭐⭐⭐⭐ | 一次性做对 |
| 图标按钮无文字 | ⭐⭐⭐⭐⭐ | 必须加 semanticLabel |
| 自定义绘制 Widget | ⭐⭐⭐ | 需手动标注语义 |
| 无障碍合规审查 | ⭐⭐⭐⭐⭐ | 自动化检测覆盖 |

---

## 🚀 完整流程

### Step 1: 审计现有 UI 的无障碍问题

**人工操作**：在 Cursor 中打开目标 Widget 文件（如 `lib/screens/product_detail_page.dart`），附上相关子组件。

**AI 提示词**（可复制）：

```markdown
请审计以下 Flutter Widget 的无障碍问题。

**检查项**：

1. **语义标签缺失**
   - 图标按钮（IconButton、FloatingActionButton）是否有 semanticLabel 或 Semantics 包裹？
   - 图片（Image）是否有 semanticLabel 或 Semantics 描述内容？
   - 自定义绘制 Widget 是否标注了语义？

2. **点击目标尺寸**
   - 可交互元素（按钮、链接）是否 ≥48x48 像素？
   - 太小的目标是否需要用 Semantics 扩大点击区域？

3. **屏幕阅读器体验**
   - 信息展示顺序是否符合逻辑（考虑遍历顺序）？
   - 是否有冗余语义（多个相同标签）？
   - 动态内容（加载、错误）是否能被朗读？

4. **语义合并与排除**
   - 装饰性元素（Divider、纯视觉 Container）是否需要 ExcludeSemantics？
   - 复杂组件是否需要 MergeSemantics 避免逐个朗读子元素？

**输出格式**：

- 列出所有发现的问题（按优先级排序）
- 每个问题给出具体代码位置和修复建议
```

**检查点**：AI 列出的问题是否覆盖关键交互元素？遗漏就明确补充。

---

### Step 2: 补充 Semantics 标签

**AI 提示词**（可复制）：

```markdown
基于上述审计结果，为代码补充 Semantics 标注。

**修复要求**：

1. **图标按钮**：加 `semanticLabel` 说明功能
   ```dart
   IconButton(
     icon: Icon(Icons.favorite),
     semanticLabel: '添加到收藏', // ✅ 添加描述
     onPressed: () {},
   )
   ```

2. **图片**：用 `Semantics` 包裹或在 `Image.semanticLabel` 添加描述
   ```dart
   Image.network(
     'https://example.com/product.jpg',
     semanticLabel: '商品正面图：红色运动鞋', // ✅ 描述图片内容
   )
   ```

3. **自定义交互区域**：用 `Semantics` 包裹并标注动作
   ```dart
   Semantics(
     label: '打开菜单',
     button: true,
     onTap: () => _openMenu(),
     child: CustomMenuIcon(),
   )
   ```

4. **装饰性元素**：用 `ExcludeSemantics` 排除
   ```dart
   ExcludeSemantics(
     child: Divider(), // ✅ 分割线不需要朗读
   )
   ```

5. **复杂卡片**：用 `MergeSemantics` 合并朗读
   ```dart
   MergeSemantics(
     child: Card(
       child: Column(
         children: [
           Text('标题'),
           Text('副标题'),
           Icon(Icons.arrow_forward),
         ],
       ),
     ),
   ) // ✅ 一次朗读完"标题 副标题"
   ```

6. **扩大点击区域**：对于小于 48x48 的元素
   ```dart
   Semantics(
     label: '删除',
     button: true,
     child: SizedBox(
       width: 48,
       height: 48,
       child: Center(
         child: Icon(Icons.delete, size: 24), // 视觉24，语义48
       ),
     ),
   )
   ```

生成修复后的完整代码，保持原有结构，只添加必要的 Semantics 标注。
```

**输出示例**（参考，实际以 AI 生成为准）：

```dart
// 修复前
IconButton(
  icon: Icon(Icons.share),
  onPressed: _share,
)

// 修复后
IconButton(
  icon: Icon(Icons.share),
  semanticLabel: '分享商品', // ✅ 添加语义标签
  onPressed: _share,
)

// 修复前
Container(
  width: 32,
  height: 32,
  child: GestureDetector(
    onTap: _delete,
    child: Icon(Icons.delete),
  ),
)

// 修复后
Semantics(
  label: '删除商品',
  button: true,
  child: SizedBox(
    width: 48, // ✅ 扩大到最小尺寸
    height: 48,
    child: Center(
      child: GestureDetector(
        onTap: _delete,
        child: Icon(Icons.delete),
      ),
    ),
  ),
)
```

---

### Step 3: 本地验证（真机测试必须！）

**人工操作**：

#### 3.1 使用屏幕阅读器测试

**Android (TalkBack)**:
```bash
# 1. 在设置 > 无障碍 > TalkBack 中启用
# 2. 运行应用
flutter run

# 3. 用手指滑动屏幕，听 TalkBack 朗读
# 4. 双击激活按钮，检查交互是否正常
```

**iOS (VoiceOver)**:
```bash
# 1. 在设置 > 辅助功能 > VoiceOver 中启用
# 2. 运行应用
flutter run

# 3. 用手指滑动屏幕，听 VoiceOver 朗读
# 4. 双击激活按钮，检查交互是否正常
```

**检查清单**：
- [ ] 所有图标按钮都能听到功能描述（不是"按钮"或空白）
- [ ] 图片有合理描述（不是文件名或 URL）
- [ ] 可交互元素能被聚焦且易于激活
- [ ] 朗读顺序符合视觉逻辑
- [ ] 装饰性元素不会被朗读

#### 3.2 使用 Flutter 无障碍准则测试

**修改测试文件**（如 `test/screens/product_detail_test.dart`）：

```dart
import 'package:flutter_test/flutter_test.dart';
import 'package:flutter/material.dart';
import 'package:my_app/screens/product_detail_page.dart';

void main() {
  testWidgets('meets accessibility guidelines', (tester) async {
    // Arrange
    await tester.pumpWidget(
      MaterialApp(
        home: ProductDetailPage(productId: '123'),
      ),
    );
    await tester.pumpAndSettle();

    // Act & Assert
    // ✅ 检查点击目标尺寸 ≥48x48
    await expectLater(tester, meetsGuideline(androidTapTargetGuideline));
    
    // ✅ 检查文本对比度 ≥4.5:1
    await expectLater(tester, meetsGuideline(textContrastGuideline));
    
    // ✅ 检查所有可交互元素有标签
    await expectLater(tester, meetsGuideline(labeledTapTargetGuideline));
  });
}
```

**运行测试**：

```bash
flutter test test/screens/product_detail_test.dart
```

**通过标准**：所有 `meetsGuideline` 断言通过，无失败。

---

### Step 4: 生成语义自动化测试

**AI 提示词**（可复制）：

```markdown
为修复后的 Widget 生成语义自动化测试。

**目标 Widget**：`lib/screens/[WIDGET_FILE_NAME].dart`

**测试要求**：

1. **测试文件路径**：`test/screens/[WIDGET_FILE_NAME]_semantics_test.dart`

2. **测试场景**：
   - ✅ 验证关键元素的 semanticLabel 正确
   - ✅ 验证可交互元素符合 `androidTapTargetGuideline`（≥48x48）
   - ✅ 验证文本对比度符合 `textContrastGuideline`（≥4.5:1）
   - ✅ 验证所有可点击元素有标签（`labeledTapTargetGuideline`）
   - ✅ 验证装饰性元素被 `ExcludeSemantics` 排除

3. **代码规范**：
   - 使用 `SemanticsHandle` 开启语义
   - 用 `tester.getSemantics(find.byType(...))` 获取语义节点
   - 用 `expect(semantics.label, '预期标签')` 验证
   - 用 `meetsGuideline(...)` 验证准则

4. **完整性**：
   - 包含所有必要的 import
   - 可直接运行 `flutter test test/screens/[WIDGET_FILE_NAME]_semantics_test.dart`

生成完整测试代码，不要省略任何部分。
```

**输出示例**（参考，实际以 AI 生成为准）：

```dart
import 'package:flutter_test/flutter_test.dart';
import 'package:flutter/material.dart';
import 'package:my_app/screens/product_detail_page.dart';

void main() {
  group('ProductDetailPage Semantics Tests', () {
    testWidgets('share button has correct semantic label', (tester) async {
      // Arrange
      await tester.pumpWidget(
        MaterialApp(
          home: ProductDetailPage(productId: '123'),
        ),
      );
      await tester.pumpAndSettle();

      // Act
      final shareFinder = find.byIcon(Icons.share);
      final semantics = tester.getSemantics(shareFinder);

      // Assert
      expect(semantics.label, '分享商品');
    });

    testWidgets('product image has semantic description', (tester) async {
      // Arrange
      await tester.pumpWidget(
        MaterialApp(
          home: ProductDetailPage(productId: '123'),
        ),
      );
      await tester.pumpAndSettle();

      // Act
      final imageFinder = find.byType(Image).first;
      final semantics = tester.getSemantics(imageFinder);

      // Assert
      expect(semantics.label, contains('商品'));
    });

    testWidgets('meets tap target size guideline', (tester) async {
      // Arrange
      await tester.pumpWidget(
        MaterialApp(
          home: ProductDetailPage(productId: '123'),
        ),
      );
      await tester.pumpAndSettle();

      // Assert
      await expectLater(tester, meetsGuideline(androidTapTargetGuideline));
    });

    testWidgets('meets text contrast guideline', (tester) async {
      // Arrange
      await tester.pumpWidget(
        MaterialApp(
          home: ProductDetailPage(productId: '123'),
        ),
      );
      await tester.pumpAndSettle();

      // Assert
      await expectLater(tester, meetsGuideline(textContrastGuideline));
    });

    testWidgets('meets labeled tap target guideline', (tester) async {
      // Arrange
      await tester.pumpWidget(
        MaterialApp(
          home: ProductDetailPage(productId: '123'),
        ),
      );
      await tester.pumpAndSettle();

      // Assert
      await expectLater(tester, meetsGuideline(labeledTapTargetGuideline));
    });

    testWidgets('decorative divider is excluded from semantics', (tester) async {
      // Arrange
      await tester.pumpWidget(
        MaterialApp(
          home: ProductDetailPage(productId: '123'),
        ),
      );
      await tester.pumpAndSettle();

      // Act
      final dividerFinder = find.byType(Divider).first;
      final hasSemanticsNode = tester.getSemantics(dividerFinder, nullOk: true);

      // Assert
      expect(hasSemanticsNode, isNull); // ✅ 装饰元素应无语义节点
    });
  });
}
```

---

### Step 5: 迭代改进

**若测试失败**，用以下提示词让 AI 修复：

```markdown
语义测试运行报错：

[粘贴错误信息]

请修复代码或测试，确保：
1. Semantics 标签与测试断言一致
2. 点击目标尺寸 ≥48x48
3. 文本对比度 ≥4.5:1
4. ExcludeSemantics 正确应用于装饰元素
```

**若要新增场景**（如动态内容朗读）：

```markdown
补充场景：当商品加载完成后，用 `SemanticsService.announce` 通知屏幕阅读器。

生成代码修复和对应测试。
```

---

## ✅ 验证清单

完成后确认：

- [ ] 所有图标按钮有 `semanticLabel` 或 `Semantics` 包裹
- [ ] 图片有语义描述
- [ ] 可交互元素 ≥48x48 像素
- [ ] 装饰性元素用 `ExcludeSemantics` 排除
- [ ] 通过真机屏幕阅读器测试（TalkBack 或 VoiceOver）
- [ ] 通过 Flutter 无障碍准则测试（`meetsGuideline`）
- [ ] 语义测试全部通过 `flutter test`

---

## 🤔 FAQ

### Q1: 什么元素需要添加 semanticLabel？
**A**: 
- **必须**：图标按钮（IconButton、FloatingActionButton）、纯图标交互、图片
- **可选**：已有文字的按钮（如 TextButton 的文字会自动成为标签）
- **不需要**：装饰性元素（Divider、Container、纯视觉间距）

### Q2: MergeSemantics 和 ExcludeSemantics 的区别？
**A**: 
- **MergeSemantics**：将子树的多个语义节点合并为一个，屏幕阅读器一次朗读完（如卡片）
- **ExcludeSemantics**：完全移除子树的语义信息，屏幕阅读器跳过（如装饰线）

### Q3: 如何测试动态内容的朗读？
**A**: 
- 使用 `SemanticsService.announce(message, textDirection)` 主动通知
- 测试中用 `tester.getSemantics()` 验证标签变化

```dart
// 通知屏幕阅读器
await SemanticsService.announce('商品已加载', TextDirection.ltr);

// 测试验证
final semantics = tester.getSemantics(find.byType(ProductCard));
expect(semantics.label, contains('已加载'));
```

### Q4: 自定义 Painter 怎么加语义？
**A**: 
- 用 `Semantics` 包裹 `CustomPaint`，手动标注 `label`、`value`、`button` 等属性

```dart
Semantics(
  label: '进度：75%',
  value: '75',
  child: CustomPaint(
    painter: CircularProgressPainter(progress: 0.75),
  ),
)
```

### Q5: 如何调试语义树？
**A**: 
- **模拟器/真机**：启用屏幕阅读器实际体验
- **开发工具**：
  ```bash
  # 在 Flutter DevTools 中查看 Semantics Tree
  flutter run --debug
  # 打开 DevTools > Inspector > 勾选 "Show Semantics Tree"
  ```
- **测试代码**：
  ```dart
  debugDumpSemanticsTree(); // 打印语义树结构
  ```

### Q6: CI 上如何运行无障碍测试？
**A**: 
- `meetsGuideline` 测试可在 CI 上运行（无需真机）
- 完整命令：
  ```bash
  flutter test --coverage test/**/*_semantics_test.dart
  ```

---

## 💡 最佳实践

1. **审计优先**
   - 先让 AI 审计现有代码，列出问题清单，再批量修复

2. **真机验证不可省略**
   - 自动化测试只能验证标签存在，屏幕阅读器体验必须真机试听

3. **标签要具体**
   - ❌ `semanticLabel: '按钮'`（太泛）
   - ✅ `semanticLabel: '添加到购物车'`（描述功能）

4. **合理使用 MergeSemantics**
   - 复杂卡片用 MergeSemantics 避免逐个朗读子元素
   - 但不要过度合并，丢失关键信息

5. **记录常见模式**
   - 把团队的语义标注规范加入 `.cursor/rules` 或 `AGENTS.md`

6. **定期回归测试**
   - UI 改动后重新运行语义测试，防止无障碍退化

---

## 🔗 相关资源

- 提示词模板：
  - [补充语义标签](../prompts/accessibility/add-semantics-labels.md)
  - [生成语义测试](../prompts/accessibility/generate-semantics-test.md)
- Flutter 官方文档：
  - [无障碍概览](https://docs.flutter.dev/ui/accessibility)
  - [无障碍测试](https://docs.flutter.dev/ui/accessibility/accessibility-testing)
  - [辅助技术](https://docs.flutter.dev/ui/accessibility/assistive-technologies)
  - [Semantics API](https://api.flutter.dev/flutter/widgets/Semantics-class.html)

---

**让每个用户都能平等地使用你的应用 ♿**
