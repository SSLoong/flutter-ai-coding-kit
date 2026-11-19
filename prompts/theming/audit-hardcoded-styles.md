# 提示词：审计硬编码样式

**用途**：扫描 Flutter 项目中硬编码的颜色、字体、间距，生成重构清单

---

## 📋 适用场景

- 准备迁移到 Material 3 主题体系
- 代码中存在大量 `Color(0x...)` 或 `Colors.xxx` 硬编码
- 字体样式和间距不统一
- 需要建立主题规范前的摸底调查

---

## 📥 输入要求

在 Cursor 中准备：

1. **打开项目根目录**，确保 `lib/` 目录可见
2. **（可选）指定审计范围**：如只审计 `lib/features/home/` 或某个模块
3. **（可选）现有主题文件**：如已有 `theme.dart`，一并打开作为参考

---

## 💬 提示词模板

```markdown
请对这个 Flutter 项目进行主题样式审计，找出所有需要重构到主题系统的硬编码。

**审计范围**：`lib/` 目录下所有 Dart 文件（或指定：`lib/features/[模块名]/`）

---

### 1. 硬编码颜色

扫描以下模式：

- `Color(0xFFxxxxxx)`, `Color(0xAARRGGBB)`
- `Color.fromARGB(...)`, `Color.fromRGBO(...)`
- `Colors.blue`, `Colors.red` 等具体颜色常量
- `CupertinoColors.xxx`

**输出格式**：

```
【硬编码颜色】

文件：lib/widgets/product_card.dart
行号：23
代码：Container(color: Color(0xFF1E88E5))
建议：改用 Theme.of(context).colorScheme.primary

文件：lib/screens/home_screen.dart
行号：45
代码：Text('标题', style: TextStyle(color: Colors.black))
建议：改用 Theme.of(context).colorScheme.onSurface

...（列出所有发现的问题）
```

---

### 2. 硬编码字体样式

扫描以下模式：

- `TextStyle(fontSize: ...)` 未使用 Theme
- `fontWeight: FontWeight.bold` 独立使用
- `fontFamily: 'xxx'` 散落在代码中

**输出格式**：

```
【硬编码字体】

文件：lib/widgets/title_widget.dart
行号：12
代码：Text('大标题', style: TextStyle(fontSize: 24, fontWeight: FontWeight.bold))
建议：改用 Theme.of(context).textTheme.headlineMedium

...
```

---

### 3. 硬编码间距与尺寸

扫描以下模式：

- `EdgeInsets.all(16)`, `EdgeInsets.symmetric(...)` 中的魔法数字
- `SizedBox(width: 24)`, `SizedBox(height: 16)`
- `padding: const EdgeInsets.only(left: 12, right: 12)`

**输出格式**：

```
【硬编码间距】

文件：lib/widgets/card_list.dart
行号：34
代码：padding: EdgeInsets.all(16)
建议：定义 AppSpacing.md = 16.0，改用 EdgeInsets.all(AppSpacing.md)

...
```

---

### 4. 优先级标注

对每个问题标注优先级：

- **高优先级**：全局共用组件（如通用按钮、AppBar、底部导航栏）
- **中优先级**：业务页面主体内容
- **低优先级**：临时调试代码、注释掉的代码

---

### 5. 汇总统计

在报告末尾提供：

```
【汇总】
- 硬编码颜色：共 X 处（高优先级 Y 处）
- 硬编码字体：共 X 处（高优先级 Y 处）
- 硬编码间距：共 X 处（高优先级 Y 处）

【建议】
1. 优先重构 [具体文件或模块]
2. 建立统一的 AppTheme 和 AppSpacing 常量类
3. 使用 ColorScheme.fromSeed 生成主题配色
```

---

**注意**：
- 只列出问题，不要现在就修改代码
- 忽略第三方包内部的硬编码（如 `package:flutter/` 下）
- 忽略已正确使用 `Theme.of(context)` 的代码
```

---

## ✅ 验证步骤

审计报告生成后：

1. **人工复查**：
   - 打开报告中列出的文件，确认问题是否真实存在
   - 检查是否有遗漏的关键文件（如弹窗、对话框组件）

2. **优先级排序**：
   - 根据业务重要性调整优先级
   - 制定分步重构计划（如先改首页，再改详情页）

3. **存档报告**：
   - 将报告保存为 `docs/theme-audit-report.md`
   - 作为后续重构的检查清单

---

## 🛠️ 常见调整

### 1. 排除特定文件或目录

如果某些文件不需要审计（如 `lib/generated/` 或测试文件），在提示词中说明：

```markdown
**排除范围**：
- lib/generated/ （自动生成代码）
- test/ （测试文件）
- lib/legacy/ （待废弃模块）
```

### 2. 识别已有的主题常量

如果项目中已有 `AppColors` 或 `Styles` 类，但未正确使用，可要求 AI 特别标注：

```markdown
项目中已有 `lib/constants/app_colors.dart`，但部分代码未使用。
请在报告中区分：
- ✅ 已使用 AppColors.xxx
- ❌ 仍使用硬编码
```

### 3. 扫描特定类型的问题

如果只关心颜色，不关心间距，可以简化提示词：

```markdown
只审计硬编码颜色，忽略字体和间距。
```

---

## 📊 报告示例片段

```markdown
# Flutter 项目主题审计报告

生成日期：2026-11-13
审计范围：lib/ 目录所有文件

---

## 一、硬编码颜色（共 23 处）

### 高优先级（8 处）

**1. lib/widgets/common/custom_button.dart:45**
```dart
Container(
  color: Color(0xFF6200EE), // ❌ 硬编码主色
  child: Text('按钮'),
)
```
→ 建议：`color: Theme.of(context).colorScheme.primary`

**2. lib/screens/home_screen.dart:78**
```dart
AppBar(
  backgroundColor: Colors.white, // ❌ 不支持深色模式
)
```
→ 建议：`backgroundColor: Theme.of(context).colorScheme.surface`

...

---

## 二、硬编码字体（共 15 处）

### 高优先级（5 处）

**1. lib/widgets/product_card.dart:23**
```dart
Text(
  product.name,
  style: TextStyle(fontSize: 18, fontWeight: FontWeight.bold), // ❌
)
```
→ 建议：`style: Theme.of(context).textTheme.titleLarge`

...

---

## 三、汇总与建议

- **硬编码颜色**：23 处（高优先级 8 处）
- **硬编码字体**：15 处（高优先级 5 处）
- **硬编码间距**：31 处（高优先级 12 处）

**重构建议**：
1. 优先处理 `lib/widgets/common/` 下的通用组件（影响全局）
2. 建立 `lib/core/theme/app_theme.dart` 统一管理主题
3. 定义 `AppSpacing` 常量类管理间距
4. 使用 `ColorScheme.fromSeed` 生成 Material 3 配色

**预计工作量**：
- 高优先级重构：约 2-3 小时
- 中优先级重构：约 4-5 小时
- 低优先级重构：约 1-2 小时

```

---

## 🔗 相关

- 工作流：[workflows/theme-consistency.md](../../workflows/theme-consistency.md)
- 落地主题系统：[implement-theme-system.md](implement-theme-system.md)
- Flutter ColorScheme 文档：[https://api.flutter.dev/flutter/material/ColorScheme-class.html](https://api.flutter.dev/flutter/material/ColorScheme-class.html)
