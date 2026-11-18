# 提示词：生成 Golden 测试

**用途**：为 Flutter Widget 生成 UI 快照（Golden）测试，验证多尺寸和主题变体

---

## 📋 适用场景

- 纯展示型 Widget 需要视觉回归测试
- 多主题/暗色模式适配验证
- 响应式布局（手机/平板/桌面）验证
- 设计规范一致性检查

---

## 📥 输入要求

在 Cursor 中准备：

1. **目标 Widget 源码**（如 `lib/widgets/info_banner.dart`）
2. **数据模型定义**（若 Widget 依赖外部数据）
3. **主题配置**（如自定义 ThemeData）
4. **（可选）设计稿截图**（作为对比参考）

---

## 💬 提示词模板

```markdown
我需要为以下 Flutter Widget 生成 Golden 测试文件。

**目标 Widget**：`lib/widgets/[WIDGET_FILE_NAME].dart`

**要求**：

1. **测试文件路径**：`test/widgets/[WIDGET_FILE_NAME]_test.dart`

2. **快照场景覆盖**（必需）：
   - ✅ 手机尺寸（375x667 @2x）+ 亮色主题
   - ✅ 手机尺寸（375x667 @2x）+ 暗色主题
   - ✅ 平板尺寸（768x1024 @2x）+ 亮色主题
   - ✅ 平板尺寸（768x1024 @2x）+ 暗色主题
   - [可选] 其他状态变体：加载中、错误、空数据

3. **技术规范**：
   - 使用 `testWidgets()` + `expectLater(..., matchesGoldenFile(...))`
   - 用 `tester.view.physicalSize` 和 `tester.view.devicePixelRatio` 设置屏幕尺寸
   - 在 `addTearDown()` 中恢复默认尺寸（`tester.view.resetPhysicalSize()`）
   - 快照文件命名：`goldens/[组件名]_[尺寸]_[主题].png`

4. **数据处理**：
   - 使用固定测试数据（避免 `DateTime.now()`, `Random()` 等动态值）
   - 若有外部依赖，用 `mocktail` mock

5. **代码结构**：
   - 用 `group()` 分组测试
   - 在 `setUp()` 初始化测试数据
   - 每个 `testWidgets` 包含完整的 MaterialApp 包裹
   - 等待渲染完成：`await tester.pumpAndSettle()`

6. **完整性**：
   - 包含所有必要的 import（含 `dart:ui` for FlutterView）
   - 可直接运行 `flutter test --update-goldens test/widgets/[WIDGET_FILE_NAME]_test.dart`

生成完整测试代码，不要省略任何部分。
```

---

## ✅ 验证步骤

生成后执行：

```bash
# 首次生成 Golden 基线
flutter test --update-goldens test/widgets/[WIDGET_FILE_NAME]_test.dart

# 检查生成的快照文件
ls -lh test/goldens/

# 目视检查快照是否符合预期（用图片查看器打开）
open test/goldens/[WIDGET_FILE_NAME]_*.png  # macOS
xdg-open test/goldens/[WIDGET_FILE_NAME]_*.png  # Linux

# 后续回归测试（不更新基线）
flutter test test/widgets/[WIDGET_FILE_NAME]_test.dart
```

**检查清单**：
- [ ] 测试全部通过（0 failed）
- [ ] 生成了对应的 `.png` 快照文件
- [ ] 快照图像目视检查符合预期（文本、颜色、布局）
- [ ] 至少覆盖 4 个快照（2 尺寸 × 2 主题）
- [ ] 快照文件已准备提交到 Git

---

## 🛠️ 常见调整

生成后可能需要手动微调：

1. **添加更多尺寸变体**：
   ```dart
   testWidgets('desktop light mode', (tester) async {
     tester.view.physicalSize = const Size(1920, 1080);
     tester.view.devicePixelRatio = 1.0;
     // ...
   });
   ```

2. **自定义主题配置**：
   ```dart
   final customTheme = ThemeData.light().copyWith(
     primaryColor: Colors.blue,
     textTheme: const TextTheme(...),
   );
   
   await tester.pumpWidget(
     MaterialApp(
       theme: customTheme,
       home: Scaffold(body: MyWidget()),
     ),
   );
   ```

3. **验证特定状态**：
   ```dart
   testWidgets('loading state', (tester) async {
     await tester.pumpWidget(
       MaterialApp(
         home: Scaffold(
           body: MyWidget(isLoading: true),
         ),
       ),
     );
     await tester.pump(); // 仅一帧，不等待动画
     
     await expectLater(
       find.byType(MyWidget),
       matchesGoldenFile('goldens/my_widget_loading.png'),
     );
   });
   ```

4. **Mock 图片资源**（避免网络加载）：
   ```dart
   setUpAll(() {
     TestWidgetsFlutterBinding.ensureInitialized();
     // 注册 mock 图片
   });
   ```

---

## 🚨 注意事项

1. **数据必须稳定**
   - ❌ 错误：`Text('${DateTime.now()}')`
   - ✅ 正确：`Text('2026-11-06')`

2. **恢复屏幕尺寸**
   - 必须在每个 `testWidgets` 结尾调用：
     ```dart
     addTearDown(() {
       tester.view.resetPhysicalSize();
       tester.view.resetDevicePixelRatio();
     });
     ```

3. **等待渲染完成**
   - 静态 UI：`await tester.pumpAndSettle()`
   - 特定帧：`await tester.pump(Duration(milliseconds: 100))`

4. **快照文件命名清晰**
   - 推荐格式：`组件名_尺寸_主题_状态.png`
   - 示例：`info_banner_phone_dark_error.png`

---

## 🔗 相关

- 工作流：[workflows/golden-test-practice.md](../../workflows/golden-test-practice.md)
- 修复 Golden 差异：[fix-golden-diff.md](fix-golden-diff.md)
- Widget 测试生成：[generate-widget-test.md](generate-widget-test.md)
