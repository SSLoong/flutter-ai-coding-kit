# 提示词：补充 Semantics 语义标签

**用途**：为 Flutter Widget 补充无障碍语义标注（Semantics、semanticLabel）

---

## 📋 适用场景

- 图标按钮无文字描述
- 图片缺少替代文本
- 自定义 Widget 无语义信息
- 可点击目标尺寸过小（<48x48）
- 装饰性元素干扰屏幕阅读器

---

## 📥 输入要求

在 Cursor 中准备：

1. **目标 Widget 源码**（如 `lib/widgets/product_card.dart`）
2. **（可选）屏幕截图或设计稿**（帮助理解视觉意图）

---

## 💬 提示词模板

```markdown
请为以下 Flutter Widget 补充无障碍语义标注。

**目标 Widget**：`lib/widgets/[WIDGET_FILE_NAME].dart`

**审计检查项**：

1. **图标按钮语义**
   - IconButton、FloatingActionButton 是否有 `semanticLabel` 描述功能？
   - 纯图标交互是否用 Semantics 包裹并标注 `label` 和 `button: true`？

2. **图片替代文本**
   - Image 是否有 `semanticLabel` 描述内容？
   - 装饰性图片是否用 ExcludeSemantics 排除？

3. **点击目标尺寸**
   - 可交互元素是否 ≥48x48 像素？
   - 小于 48x48 的元素是否用 Semantics + SizedBox 扩大语义区域？

4. **装饰性元素**
   - Divider、纯视觉 Container、装饰图标是否用 ExcludeSemantics 排除？

5. **复杂组件合并**
   - 卡片、列表项等复杂组件是否需要 MergeSemantics 合并朗读？

**修复要求**：

1. **图标按钮**：优先用 `semanticLabel` 参数
   ```dart
   IconButton(
     icon: Icon(Icons.favorite),
     semanticLabel: '添加到收藏', // ✅ 描述功能，不是"收藏图标"
     onPressed: () {},
   )
   ```

2. **图片**：用 `semanticLabel` 参数或 Semantics 包裹
   ```dart
   // 方式1：直接用参数
   Image.network(
     'https://example.com/product.jpg',
     semanticLabel: '商品正面图：红色运动鞋',
   )
   
   // 方式2：用 Semantics 包裹（更灵活）
   Semantics(
     label: '商品正面图：红色运动鞋',
     image: true,
     child: Image.network('https://example.com/product.jpg'),
   )
   ```

3. **自定义交互**：用 Semantics 包裹并标注
   ```dart
   Semantics(
     label: '打开菜单',
     button: true,
     onTap: () => _openMenu(),
     child: GestureDetector(
       onTap: _openMenu,
       child: CustomMenuIcon(),
     ),
   )
   ```

4. **扩大点击区域**：对于小于 48x48 的元素
   ```dart
   Semantics(
     label: '删除商品',
     button: true,
     child: SizedBox(
       width: 48, // ✅ 语义目标扩大到最小尺寸
       height: 48,
       child: Center(
         child: GestureDetector(
           onTap: _delete,
           child: Icon(Icons.delete, size: 24), // 视觉24，语义48
         ),
       ),
     ),
   )
   ```

5. **排除装饰元素**：
   ```dart
   ExcludeSemantics(
     child: Divider(), // ✅ 分割线不需要朗读
   )
   
   ExcludeSemantics(
     child: Icon(Icons.arrow_forward), // ✅ 纯装饰箭头
   )
   ```

6. **合并复杂组件**：
   ```dart
   MergeSemantics(
     child: Card(
       child: Column(
         children: [
           Text('商品名称'),
           Text('¥99.00'),
           Icon(Icons.shopping_cart),
         ],
       ),
     ),
   ) // ✅ 屏幕阅读器一次朗读"商品名称 ¥99.00 购物车"
   ```

7. **动态内容通知**：
   ```dart
   // 在状态变化时通知屏幕阅读器
   void _onDataLoaded() {
     SemanticsService.announce(
       '商品数据已加载',
       TextDirection.ltr,
     );
   }
   ```

**输出要求**：

- 生成修复后的完整 Widget 代码
- 保持原有结构和样式
- 只添加必要的 Semantics 标注
- 标注要具体描述功能（不要用"按钮""图标"等泛词）
- 注释说明为什么添加/排除语义
```

---

## ✅ 验证步骤

修复后执行：

### 1. 真机屏幕阅读器测试

**Android (TalkBack)**:
```bash
# 1. 在设置 > 无障碍 > TalkBack 中启用
# 2. 运行应用：flutter run
# 3. 用手指滑动屏幕，听 TalkBack 朗读
# 4. 双击激活按钮，检查交互是否正常
```

**iOS (VoiceOver)**:
```bash
# 1. 在设置 > 辅助功能 > VoiceOver 中启用
# 2. 运行应用：flutter run
# 3. 用手指滑动屏幕，听 VoiceOver 朗读
# 4. 双击激活按钮，检查交互是否正常
```

**检查清单**：
- [ ] 图标按钮朗读功能描述（不是"按钮"）
- [ ] 图片朗读内容描述（不是文件名）
- [ ] 装饰元素不被朗读
- [ ] 朗读顺序符合逻辑
- [ ] 可交互元素易于激活

### 2. 自动化测试

```dart
testWidgets('has correct semantic labels', (tester) async {
  await tester.pumpWidget(
    MaterialApp(home: ProductCard()),
  );
  await tester.pumpAndSettle();
  
  // 验证语义标签
  final shareFinder = find.byIcon(Icons.share);
  final semantics = tester.getSemantics(shareFinder);
  expect(semantics.label, '分享商品');
  
  // 验证点击目标尺寸
  await expectLater(tester, meetsGuideline(androidTapTargetGuideline));
});
```

---

## 🛠️ 常见调整

### 1. 标签要具体

**❌ 不好**：
```dart
IconButton(
  icon: Icon(Icons.favorite),
  semanticLabel: '按钮', // 太泛
  onPressed: () {},
)
```

**✅ 好**：
```dart
IconButton(
  icon: Icon(Icons.favorite),
  semanticLabel: '添加到收藏', // 描述功能
  onPressed: () {},
)
```

### 2. 区分装饰性和信息性图片

**装饰性**（排除）：
```dart
ExcludeSemantics(
  child: Image.asset('assets/background_pattern.png'),
)
```

**信息性**（描述）：
```dart
Image.network(
  'https://example.com/product.jpg',
  semanticLabel: '商品正面图：红色运动鞋',
)
```

### 3. 合理使用 MergeSemantics

**❌ 过度合并**（丢失信息）：
```dart
MergeSemantics(
  child: Row(
    children: [
      Text('商品A'),
      ElevatedButton(onPressed: () {}, child: Text('购买')),
      Text('商品B'),
      ElevatedButton(onPressed: () {}, child: Text('购买')),
    ],
  ),
) // ❌ 屏幕阅读器无法区分两个购买按钮
```

**✅ 适度合并**（保留结构）：
```dart
Row(
  children: [
    MergeSemantics(
      child: Column(
        children: [Text('商品A'), Text('¥99')],
      ),
    ),
    ElevatedButton(onPressed: () {}, child: Text('购买')),
    MergeSemantics(
      child: Column(
        children: [Text('商品B'), Text('¥88')],
      ),
    ),
    ElevatedButton(onPressed: () {}, child: Text('购买')),
  ],
) // ✅ 每个商品独立朗读
```

### 4. 处理动态内容

```dart
// 加载状态
if (_isLoading) {
  Semantics(
    label: '正在加载商品数据',
    child: CircularProgressIndicator(),
  )
}

// 加载完成通知
void _onDataLoaded() {
  SemanticsService.announce(
    '已加载 ${products.length} 个商品',
    TextDirection.ltr,
  );
}
```

---

## 🔗 相关

- 工作流：[workflows/semantics-accessibility.md](../../workflows/semantics-accessibility.md)
- 生成语义测试：[generate-semantics-test.md](generate-semantics-test.md)
- Flutter 官方文档：
  - [Semantics API](https://api.flutter.dev/flutter/widgets/Semantics-class.html)
  - [无障碍指南](https://docs.flutter.dev/ui/accessibility)
