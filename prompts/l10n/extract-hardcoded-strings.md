# 提示词：审计硬编码字符串

**用途**：扫描 Flutter 项目中所有面向用户的硬编码字符串，生成待迁移到 ARB 的清单

---

## 📋 适用场景

- 已有项目需国际化，评估工作量
- 重构前审计哪些文案需要翻译
- 新增语言支持前检查遗漏的硬编码

---

## 📥 输入要求

在 Cursor 中准备：

1. **打开 `lib/` 目录**（或指定需审计的子目录）
2. **（可选）排除列表**：明确不需要翻译的文件或目录（如 `lib/generated/`, `lib/test_utils/`）

---

## 💬 提示词模板

```markdown
扫描 `lib/` 目录下所有 Dart 文件，列出所有面向最终用户的硬编码字符串，生成待迁移到 ARB 文件的清单。

---

## 需要提取的字符串（包含）

- **Text Widget**：`Text('Hello')`, `Text.rich()` 中的字符串
- **输入组件**：`TextField` / `TextFormField` 的 `hintText`, `labelText`, `helperText`, `errorText`
- **AppBar**：`title` 中的字符串
- **SnackBar / Dialog**：`content`, `title`, `message` 中的文本
- **Button 标签**：`ElevatedButton`, `TextButton`, `IconButton` 的 `child` 文本
- **Tooltip**：`tooltip` 属性
- **表单验证消息**：`validator` 返回的错误文本
- **Semantics label**：`Semantics(label: '...')`
- **ListTile**：`title`, `subtitle` 中的字符串

---

## 不应提取的字符串（排除）

- **日志输出**：`print()`, `debugPrint()`, `log()`, `logger.info()` 等
- **分析事件名**：`analytics.logEvent('button_click')` 中的事件名
- **路由路径**：`Navigator.pushNamed(context, '/home')` 中的路由字符串
- **资产路径**：`'assets/images/logo.png'`, `'fonts/Roboto.ttf'`
- **API endpoint**：`'https://api.example.com/users'`
- **常量标识符**：枚举值、配置 key（如 `'API_KEY'`, `'en_US'`）
- **测试断言字符串**：单元测试中的 `expect(result, 'expected value')`
- **JSON key**：`json['user_name']` 中的 key
- **正则表达式**：`RegExp(r'^\d+$')`
- **代码生成标记**：`// ignore_for_file`, `@pragma('vm:entry-point')`

---

## 输出格式

生成 Markdown 表格，包含以下列：

| 文件路径 | 行号 | 原始字符串 | 建议 ARB Key | 优先级 | 备注 |
|---------|------|------------|--------------|--------|------|
| lib/pages/home_page.dart | 23 | "Welcome to My App" | homeWelcome | 高 | 首屏标题 |
| lib/widgets/login_button.dart | 15 | "Log In" | loginButton | 高 | 主按钮 |
| lib/pages/settings_page.dart | 45 | "Language" | settingsLanguage | 中 | 设置项 |
| lib/utils/validators.dart | 12 | "Email is invalid" | errorInvalidEmail | 中 | 表单验证 |

---

## 优先级判断标准

- **高**：首屏、主流程、核心功能的文案（用户必看）
- **中**：设置页、次要功能、表单提示（用户常见）
- **低**：错误提示、边缘场景、极少出现的文案

---

## ARB Key 命名规范

- 使用 **camelCase**：`homeWelcome`, `loginButton`
- 包含**上下文前缀**：`errorNetworkTimeout`（而非仅 `timeout`）
- 保持简洁：不超过 40 字符
- 避免缩写：`userName` 优于 `usrNm`

---

## 额外要求

1. **至少列出 20 条字符串**（小项目可少于此，但需说明总数）
2. **排除测试文件**：不扫描 `test/` 目录
3. **标注占位符**：如果字符串中包含变量（如 `"Hello $userName"`），在备注列标注「需占位符」
4. **标注复数形式**：如果涉及数量（如 `"$count items"`），在备注列标注「需 plural」

---

完成后，输出完整表格，并统计：
- 总共找到多少条硬编码字符串
- 高/中/低优先级各多少条
- 是否有遗漏需人工复查的场景
```

---

## ✅ 验证步骤

收到 AI 生成的表格后：

1. **检查表格完整性**：
   - [ ] 每行包含文件路径、行号、原始字符串、ARB Key、优先级
   - [ ] ARB Key 符合 camelCase 命名规范
   - [ ] 优先级判断合理（首屏文案应为「高」）

2. **抽查文件**：
   - 打开表格中列出的 2-3 个文件
   - 确认原始字符串确实存在于对应行号
   - 验证没有混入日志、路由 path 等不应提取的内容

3. **手动补充遗漏**：
   - 在关键页面（如首页、登录页）搜索 `Text(` 和 `hintText:`
   - 确认没有遗漏重要文案

4. **验证占位符标注**：
   - 检查带 `$` 或 `{}` 的字符串是否标注了「需占位符」
   - 包含数字的字符串是否标注了「需 plural」

---

## 🛠️ 常见调整

### 问题 1：AI 混入了日志输出

**修正提示词**：
```markdown
重新扫描，明确排除所有 `print()`, `debugPrint()`, `log()`, `logger.xxx()` 中的字符串。
只提取显示在 UI 上给最终用户看到的文本。
```

### 问题 2：遗漏了 Dialog 或 SnackBar 的文案

**修正提示词**：
```markdown
补充扫描以下场景的字符串：
- `showDialog()` 中的 `AlertDialog(title:, content:)`
- `ScaffoldMessenger.showSnackBar()` 中的 `SnackBar(content:)`
- `showModalBottomSheet()` 中的文本内容
```

### 问题 3：ARB Key 命名不规范

**修正提示词**：
```markdown
重新生成 ARB Key，遵循以下规则：
- 使用 camelCase（首字母小写）
- 包含功能前缀：如 `home`, `login`, `settings`, `error`
- 示例：`homeWelcome`, `loginButtonLabel`, `errorNetworkFailed`
```

---

## 💡 提示

- **首次扫描可能不完整**：建议分模块扫描（先 `lib/pages/`, 再 `lib/widgets/`），AI 输出质量更高
- **大型项目**：限制扫描范围（如「仅扫描 `lib/pages/home/` 目录」），避免 AI 输出过长截断
- **保存清单**：将生成的表格保存为 Markdown 或 CSV，后续步骤中可直接引用

---

## 🔗 相关

- 工作流：[workflows/l10n-internationalization.md](../../workflows/l10n-internationalization.md)
- 配置工具链：[setup-gen-l10n.md](setup-gen-l10n.md)
