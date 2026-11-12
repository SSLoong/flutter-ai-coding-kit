# 提示词：生成 Semantics 语义测试

**用途**：为 Flutter Widget 生成无障碍语义自动化测试

---

## 📋 适用场景

- 验证语义标签正确性
- 检查点击目标尺寸合规（≥48x48）
- 确保文本对比度达标（≥4.5:1）
- 验证装饰元素被正确排除
- 回归测试无障碍特性

---

## 📥 输入要求

在 Cursor 中准备：

1. **目标 Widget 源码**（如 `lib/widgets/product_card.dart`）
2. **已添加的 Semantics 标注**（作为测试依据）
3. **（可选）现有测试文件**（作为风格参考）

---

## 💬 提示词模板

```markdown
我需要为以下 Flutter Widget 生成语义测试文件。

**目标 Widget**：`lib/widgets/[WIDGET_FILE_NAME].dart`

**要求**：

1. **测试文件路径**：`test/widgets/[WIDGET_FILE_NAME]_semantics_test.dart`

2. **测试场景覆盖**：
   - ✅ 验证关键元素的 semanticLabel 正确
   - ✅ 验证图标按钮、图片有语义描述
   - ✅ 验证可交互元素符合 `androidTapTargetGuideline`（≥48x48）
   - ✅ 验证文本对比度符合 `textContrastGuideline`（≥4.5:1）
   - ✅ 验证所有可点击元素有标签（`labeledTapTargetGuideline`）
   - ✅ 验证装饰性元素被 `ExcludeSemantics` 排除（无语义节点）
   - ✅ 验证 `MergeSemantics` 正确合并子节点

3. **代码规范**：
   - 使用 `flutter_test` 包
   - 用 `testWidgets()` 和 `group()`
   - 用 `tester.getSemantics(find.byType(...))` 获取语义节点
   - 用 `expect(semantics.label, '预期标签')` 验证标签
   - 用 `meetsGuideline(...)` 验证无障碍准则
   - 用 `tester.getSemantics(..., nullOk: true)` 验证元素被排除

4. **完整性**：
   - 包含所有必要的 import
   - 可直接运行 `flutter test test/widgets/[WIDGET_FILE_NAME]_semantics_test.dart`

**测试代码模板参考**：

```dart
import 'package:flutter_test/flutter_test.dart';
import 'package:flutter/material.dart';
import 'package:my_app/widgets/product_card.dart';

void main() {
  group('ProductCard Semantics Tests', () {
    testWidgets('favorite button has correct semantic label', (tester) async {
      // Arrange
      await tester.pumpWidget(
        MaterialApp(
          home: Scaffold(
            body: ProductCard(
              productId: '123',
              name: 'Test Product',
              price: 99.0,
            ),
          ),
        ),
      );
      await tester.pumpAndSettle();

      // Act
      final favoriteFinder = find.byIcon(Icons.favorite);
      final semantics = tester.getSemantics(favoriteFinder);

      // Assert
      expect(semantics.label, '添加到收藏');
    });

    testWidgets('product image has semantic description', (tester) async {
      // Arrange
      await tester.pumpWidget(
        MaterialApp(
          home: Scaffold(
            body: ProductCard(
              productId: '123',
              name: 'Test Product',
              imageUrl: 'https://example.com/product.jpg',
            ),
          ),
        ),
      );
      await tester.pumpAndSettle();

      // Act
      final imageFinder = find.byType(Image);
      final semantics = tester.getSemantics(imageFinder);

      // Assert
      expect(semantics.label, isNotEmpty);
      expect(semantics.label, contains('商品'));
    });

    testWidgets('meets tap target size guideline', (tester) async {
      // Arrange
      await tester.pumpWidget(
        MaterialApp(
          home: Scaffold(
            body: ProductCard(productId: '123'),
          ),
        ),
      );
      await tester.pumpAndSettle();

      // Assert
      await expectLater(
        tester,
        meetsGuideline(androidTapTargetGuideline),
      );
    });

    testWidgets('meets text contrast guideline', (tester) async {
      // Arrange
      await tester.pumpWidget(
        MaterialApp(
          home: Scaffold(
            body: ProductCard(productId: '123'),
          ),
        ),
      );
      await tester.pumpAndSettle();

      // Assert
      await expectLater(
        tester,
        meetsGuideline(textContrastGuideline),
      );
    });

    testWidgets('meets labeled tap target guideline', (tester) async {
      // Arrange
      await tester.pumpWidget(
        MaterialApp(
          home: Scaffold(
            body: ProductCard(productId: '123'),
          ),
        ),
      );
      await tester.pumpAndSettle();

      // Assert
      await expectLater(
        tester,
        meetsGuideline(labeledTapTargetGuideline),
      );
    });

    testWidgets('decorative divider is excluded from semantics', (tester) async {
      // Arrange
      await tester.pumpWidget(
        MaterialApp(
          home: Scaffold(
            body: ProductCard(productId: '123'),
          ),
        ),
      );
      await tester.pumpAndSettle();

      // Act
      final dividerFinder = find.byType(Divider);
      final semanticsNode = tester.getSemantics(
        dividerFinder,
        nullOk: true, // ✅ 不抛异常
      );

      // Assert
      expect(semanticsNode, isNull); // ✅ 装饰元素应无语义节点
    });

    testWidgets('merged semantics combines child labels', (tester) async {
      // Arrange
      await tester.pumpWidget(
        MaterialApp(
          home: Scaffold(
            body: ProductCard(
              productId: '123',
              name: 'Test Product',
              price: 99.0,
            ),
          ),
        ),
      );
      await tester.pumpAndSettle();

      // Act
      final cardFinder = find.byType(Card);
      final semantics = tester.getSemantics(cardFinder);

      // Assert
      expect(semantics.label, contains('Test Product'));
      expect(semantics.label, contains('99'));
    });
  });
}
```

生成完整测试代码，覆盖上述所有场景。
```

---

## ✅ 验证步骤

生成后执行：

```bash
# 运行语义测试
flutter test test/widgets/[WIDGET_FILE_NAME]_semantics_test.dart

# 查看详细输出
flutter test test/widgets/[WIDGET_FILE_NAME]_semantics_test.dart --verbose

# 查看覆盖率
flutter test --coverage test/widgets/[WIDGET_FILE_NAME]_semantics_test.dart
```

**通过标准**：
- [ ] 所有测试通过（0 failed）
- [ ] 无 import 错误
- [ ] `meetsGuideline` 断言全部通过
- [ ] 语义标签验证准确

---

## 🛠️ 常见调整

### 1. 获取嵌套 Widget 的语义

**问题**：
```dart
final semantics = tester.getSemantics(find.byType(IconButton));
// ❌ 如果有多个 IconButton，会报错
```

**修复**：
```dart
// 方式1：用 Key 精准定位
final semantics = tester.getSemantics(
  find.byKey(Key('favorite_button')),
);

// 方式2：用 first/last/at 索引
final semantics = tester.getSemantics(
  find.byType(IconButton).first,
);

// 方式3：组合 finder
final semantics = tester.getSemantics(
  find.descendant(
    of: find.byType(ProductCard),
    matching: find.byIcon(Icons.favorite),
  ),
);
```

### 2. 验证元素被排除

```dart
testWidgets('decorative element excluded', (tester) async {
  await tester.pumpWidget(MyWidget());
  await tester.pumpAndSettle();
  
  final finder = find.byType(Divider);
  
  // ✅ 用 nullOk 避免抛异常
  final semanticsNode = tester.getSemantics(finder, nullOk: true);
  
  expect(semanticsNode, isNull); // ✅ 被 ExcludeSemantics 排除
});
```

### 3. 验证 MergeSemantics 效果

```dart
testWidgets('card merges child labels', (tester) async {
  await tester.pumpWidget(
    MaterialApp(
      home: MergeSemantics(
        child: Column(
          children: [
            Text('标题'),
            Text('副标题'),
          ],
        ),
      ),
    ),
  );
  await tester.pumpAndSettle();
  
  final semantics = tester.getSemantics(find.byType(Column));
  
  // ✅ 合并后的标签包含所有子文本
  expect(semantics.label, contains('标题'));
  expect(semantics.label, contains('副标题'));
});
```

### 4. 处理异步加载的语义

```dart
testWidgets('shows loading semantics initially', (tester) async {
  await tester.pumpWidget(MyWidget());
  
  // 第一帧：加载中
  final loadingSemantics = tester.getSemantics(
    find.byType(CircularProgressIndicator),
  );
  expect(loadingSemantics.label, '正在加载');
  
  // 等待数据加载
  await tester.pumpAndSettle();
  
  // 第二帧：加载完成
  final dataSemantics = tester.getSemantics(
    find.byType(ListView),
  );
  expect(dataSemantics.label, contains('商品列表'));
});
```

### 5. 验证自定义语义动作

```dart
testWidgets('custom semantic action is available', (tester) async {
  await tester.pumpWidget(
    Semantics(
      label: '商品卡片',
      customSemanticsActions: {
        CustomSemanticsAction(label: '加入购物车'): () {},
      },
      child: ProductCard(),
    ),
  );
  await tester.pumpAndSettle();
  
  final semantics = tester.getSemantics(find.byType(ProductCard));
  
  expect(semantics.customSemanticsActions, isNotEmpty);
  expect(
    semantics.customSemanticsActions.keys.first.label,
    '加入购物车',
  );
});
```

---

## 🔗 相关

- 工作流：[workflows/semantics-accessibility.md](../../workflows/semantics-accessibility.md)
- 补充语义标签：[add-semantics-labels.md](add-semantics-labels.md)
- Flutter 官方文档：
  - [无障碍测试](https://docs.flutter.dev/ui/accessibility/accessibility-testing)
  - [测试 Widget 语义](https://api.flutter.dev/flutter/flutter_test/WidgetTester/getSemantics.html)
