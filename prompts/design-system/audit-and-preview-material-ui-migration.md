# 提示词:审计依赖与预览 material_ui / cupertino_ui 迁移

**用途**:在应用迁移前,审计第三方依赖兼容性,预览 `dart fix` 变更,给出是否立即迁移的建议

---

## 📋 适用场景

- 准备从框架内设计库迁移到 material_ui / cupertino_ui
- 需要评估项目依赖是否阻塞迁移
- 想先了解迁移影响范围再决定是否执行
- 在正式迁移前做风险评估

---

## 📥 输入要求

在 Cursor 中准备:

1. **打开项目根目录**,确保可访问 `pubspec.yaml` 和 `lib/` 目录
2. **确保依赖已安装**:`flutter pub get` 已运行
3. **工作区干净**:无未提交的变更(或已暂存当前工作)
4. **Flutter 版本**:确认 >= 3.47(建议 3.50+)

---

## 💬 提示词模板

```markdown
我准备将 Flutter 项目从框架内设计库迁移到独立的 material_ui / cupertino_ui 包。请帮我做迁移前审计和预览。

---

## 任务 1:审计第三方依赖兼容性

**目标**:判断项目依赖是否阻塞迁移。

### 步骤 1.1:列出直接依赖

从 `pubspec.yaml` 的 `dependencies` 中提取所有非 SDK 依赖(排除 `flutter`、`flutter_localizations` 等 SDK 包)。

**输出格式**:

| 依赖包 | 当前版本 |
|--------|----------|
| provider | ^6.1.0 |
| dio | ^5.0.0 |
| ...    | ...      |

### 步骤 1.2:检查依赖使用方式

对每个依赖,在项目代码中搜索其使用,判断风险等级:

- **高风险**:在 API 调用中传递了设计库核心类型(ColorScheme、TextTheme、ThemeData、FloatingActionButtonLocation、ButtonStyle 等)作为参数或接收返回值
- **中风险**:依赖包文档未明确说明支持 material_ui/cupertino_ui,但使用方式只涉及基本类型(String、Widget、Function、BuildContext)
- **低风险**:已确认支持新包 或 只用 Widgets 层,不涉及设计库

**代码搜索示例**:

```bash
# 搜索依赖包的使用
grep -r "import.*package:[依赖包名]" lib/
# 查看具体调用,判断是否有类型传递
```

### 步骤 1.3:生成兼容性报告

**输出格式**:

| 依赖包 | 版本 | 风险 | 说明 | 建议 |
|--------|------|------|------|------|
| provider | ^6.1.0 | ✅ 低 | 只用于状态管理,不涉及设计库类型 | 可迁移,无影响 |
| flutter_svg | ^2.0.0 | ✅ 低 | 只返回 Widget,无设计库耦合 | 可迁移,无影响 |
| custom_theme_plugin | ^1.2.0 | ❌ 高 | API 参数要求 `ColorScheme`(旧类型),尚未更新到 material_ui | **阻塞迁移**,需等待更新 |
| some_ui_kit | ^3.0.0 | ⚠️ 中 | 文档未说明支持新包,但使用中只传递 Widget 和 String | 可迁移,建议测试后用兼容桥保险 |

---

## 任务 2:预览 dart fix 迁移变更

**目标**:在不实际修改文件的情况下,查看 `dart fix` 会做哪些变更。

### 步骤 2.1:运行 dry-run(在终端执行)

如果你有终端访问权限,建议运行:

```bash
dart fix --dry-run --code=migrate_design_widgets
```

如果无法执行命令,请告知,我将基于项目文件模拟预览。

### 步骤 2.2:分析预览输出

分析 `dart fix` 预览的变更:

**检查项**:

1. **导入替换正确性**:
   - `package:flutter/material.dart` → `package:material_ui/material_ui.dart` ✅
   - `package:flutter/cupertino.dart` → `package:cupertino_ui/cupertino_ui.dart` ✅
   - **非设计库导入不应被改**:
     - `package:flutter/widgets.dart` 保持不变 ✅
     - `package:flutter/services.dart` 保持不变 ✅
     - `package:flutter/foundation.dart` 保持不变 ✅

2. **覆盖范围**:
   - `lib/` 目录下所有 `.dart` 文件
   - `test/` 目录下所有测试文件
   - 有无遗漏的文件?

3. **依赖声明**:
   - `pubspec.yaml` 中是否需要手动添加 `material_ui` / `cupertino_ui`?

### 步骤 2.3:生成预览报告

**输出格式**:

```
【dart fix 预览分析】

✅ 导入替换:共 [X] 处,路径正确
  - lib/main.dart:1  package:flutter/material.dart → package:material_ui/material_ui.dart
  - lib/pages/home.dart:2  package:flutter/cupertino.dart → package:cupertino_ui/cupertino_ui.dart
  - ...

✅ 范围覆盖:lib/ 和 test/ 均已覆盖

⚠️ 依赖声明:需手动在 pubspec.yaml 添加 material_ui 和 cupertino_ui
  (dart fix 未自动修改 pubspec.yaml)

❌ 误改发现:[若有]
  - lib/services/platform_service.dart:5  package:flutter/services.dart 被误改为 material_ui
    → 需人工恢复

【预计影响】:
- 修改文件数:[X] 个
- 修改行数:[Y] 行
- 预计迁移耗时:[Z] 分钟(审查 + 应用 + 测试)
```

---

## 任务 3:生成迁移建议

基于任务 1 和任务 2 的结果,给出明确建议。

**输出格式**:

```
【迁移建议】

🔍 依赖兼容性:[✅ 通过 / ⚠️ 需观察 / ❌ 存在阻塞]

[若存在阻塞]
❌ 建议:**暂缓迁移**

**阻塞依赖**:
- custom_theme_plugin ^1.2.0:API 参数要求旧 ColorScheme
  
**后续行动**:
1. 在 GitHub 提 issue 或联系包作者:https://github.com/author/custom_theme_plugin/issues
2. 关注 pub.dev 版本更新:https://pub.dev/packages/custom_theme_plugin
3. 考虑临时 fork 自行迁移(若紧急)

[若无阻塞]
✅ 建议:**可立即迁移**

**风险提示**:
- [若有中风险依赖] 需在迁移后测试 [包名] 的功能,必要时添加兼容桥

**迁移步骤**:
1. 手动添加依赖:`flutter pub add material_ui cupertino_ui`
2. 应用 dart fix:`dart fix --apply --code=migrate_design_widgets`
3. 解决 flutter_localizations 模糊导入(见工作流 Step 5)
4. 更新 localization delegates(见工作流 Step 6)
5. 运行测试和构建验证(见工作流 Step 8)

**预计收益**:
- 跟进独立包周更,获得最新修复和组件
- 消除 Flutter 3.50+ 的 deprecated 警告
- 为未来的风格中立核心做准备
```

---

## 任务 4:生成迁移前检查清单

**输出**:

```
【迁移前准备清单】

环境检查:
- [ ] Flutter 版本 >= 3.47(当前:[版本])
- [ ] 工作区干净(git status clean)
- [ ] 所有依赖已安装(flutter pub get)

依赖审计:
- [ ] 已识别所有高风险依赖
- [ ] 阻塞性依赖已评估解决方案
- [ ] 中风险依赖已制定测试计划

预览变更:
- [ ] dart fix 预览已审查
- [ ] 无非设计库导入被误改
- [ ] 变更范围符合预期

备份与回滚:
- [ ] 当前工作已提交或暂存
- [ ] 知道如何回滚(git reset --hard)

知识准备:
- [ ] 已阅读工作流文档
- [ ] 了解兼容桥的能力与限制
- [ ] 知道 gen-l10n 的配置变化
```

---

**注意事项**:

1. **不要在此步骤实际修改代码**,只做审计和预览
2. 若发现阻塞依赖,停止迁移,制定依赖更新计划
3. 预览通过后,再执行工作流 Step 2-9 正式迁移
```

---

## ✅ 验证步骤

提示词执行后:

1. **审查依赖报告**:
   - [ ] 是否有高风险依赖?
   - [ ] 阻塞性依赖的解决方案是否可行?
   - [ ] 中风险依赖的测试计划是否充分?

2. **审查预览报告**:
   - [ ] 导入替换路径正确?
   - [ ] 有无非设计库导入被误改?
   - [ ] 变更范围是否覆盖 lib/ 和 test/?

3. **审查迁移建议**:
   - [ ] 建议是否明确(立即迁移 / 暂缓)?
   - [ ] 风险提示是否充分?
   - [ ] 后续步骤是否可操作?

4. **做出决策**:
   - **若建议立即迁移**:继续工作流 Step 2
   - **若建议暂缓**:跟踪阻塞依赖,制定更新计划

---

## 🛠️ 常见调整

### 1. 只关注特定依赖

如果已知某些依赖安全,可缩小审计范围:

```markdown
只审计以下依赖的兼容性,其他视为低风险:
- custom_theme_plugin
- legacy_ui_kit
- internal_design_system
```

### 2. 跳过 dart fix 预览(若无法执行命令)

```markdown
我无法在当前环境运行 `dart fix` 命令,请基于项目文件模拟预览:
- 列出所有包含 `package:flutter/material.dart` 或 `package:flutter/cupertino.dart` 导入的文件
- 说明这些导入将如何被替换
- 判断是否有非设计库导入可能被误改
```

### 3. 包含特定依赖的详细分析

```markdown
请特别关注 `[包名]` 的兼容性,提供详细分析:
1. 在项目中的所有使用位置
2. API 调用中传递的参数类型
3. 返回值类型
4. 是否有类型签名风险
5. 建议的处理方案(兼容桥 / 等待更新 / fork)
```

---

## 📊 报告示例片段

```markdown
# Flutter material_ui / cupertino_ui 迁移前审计报告

生成日期:2026-11-27
项目:my_flutter_app
Flutter 版本:3.50.0

---

## 一、依赖兼容性审计

### 直接依赖列表(共 15 个)

| 依赖包 | 版本 | 风险 | 说明 | 建议 |
|--------|------|------|------|------|
| provider | ^6.1.0 | ✅ 低 | 状态管理,不涉及设计库类型 | 可迁移 |
| dio | ^5.0.0 | ✅ 低 | 网络请求,无设计库耦合 | 可迁移 |
| flutter_svg | ^2.0.0 | ✅ 低 | SVG 渲染,只返回 Widget | 可迁移 |
| go_router | ^14.0.0 | ✅ 低 | 路由管理,不涉及设计库 | 可迁移 |
| intl | ^0.19.0 | ✅ 低 | 国际化工具,独立于设计库 | 可迁移 |
| shared_preferences | ^2.0.0 | ✅ 低 | 本地存储,无设计库耦合 | 可迁移 |
| custom_bottom_sheet | ^2.1.0 | ⚠️ 中 | 文档未说明支持新包,使用中只传 Widget | 迁移后需测试 |
| legacy_theme_manager | ^1.5.0 | ❌ 高 | **API 要求 `ThemeData` 参数(旧类型)** | **阻塞迁移** |

### 高风险依赖详情

**legacy_theme_manager ^1.5.0**

- **使用位置**:`lib/core/theme/theme_switcher.dart:45`
- **问题代码**:
```dart
ThemeManager.setTheme(
  Theme.of(context), // 返回新 ThemeData(material_ui)
  // ↑ 但 setTheme 参数要求旧 ThemeData(flutter/material.dart)
  // 类型不匹配,编译错误
);
```

- **包状态**:pub.dev 显示最后更新 2025-03-10,无 material_ui 支持说明
- **社区反馈**:GitHub issues 中有 2 个相关请求尚未解决
  - https://github.com/author/legacy_theme_manager/issues/45
  - https://github.com/author/legacy_theme_manager/issues/52

- **建议方案**:
  1. **等待官方更新**(推荐):在 issue #45 留言表达需求,关注版本更新
  2. **临时 fork**:fork 仓库,自行迁移后用 `dependency_overrides`,待官方更新后移除
  3. **寻找替代**:评估是否可用其他主题管理方案替代

**结论**:❌ **当前不建议迁移**,等待 legacy_theme_manager 更新

---

## 二、dart fix 预览分析

### 变更统计

- **修改文件数**:23 个(.dart 文件)
- **导入替换**:28 处
  - material.dart → material_ui.dart:20 处
  - cupertino.dart → cupertino_ui.dart:8 处
- **pubspec.yaml**:需手动添加依赖

### 详细变更列表

**lib/ 目录(18 个文件)**

| 文件 | 变更内容 |
|------|----------|
| lib/main.dart | 导入 material.dart → material_ui.dart |
| lib/app.dart | 导入 material.dart → material_ui.dart |
| lib/pages/home_page.dart | 导入 material.dart → material_ui.dart |
| lib/pages/settings_page.dart | 导入 material.dart + cupertino.dart → 两者都改为新包 |
| ...  | ... |

**test/ 目录(5 个文件)**

| 文件 | 变更内容 |
|------|----------|
| test/widget_test.dart | 导入 material.dart → material_ui.dart |
| test/pages/home_test.dart | 导入 material.dart → material_ui.dart |
| ...  | ... |

### 检查结果

✅ **导入替换正确性**:所有设计库导入路径正确

✅ **非设计库导入保持不变**:
- `package:flutter/widgets.dart`:未被改动 ✅
- `package:flutter/services.dart`:未被改动 ✅
- `package:flutter/foundation.dart`:未被改动 ✅

✅ **覆盖范围充分**:lib/ 和 test/ 目录均已覆盖

⚠️ **依赖声明**:需手动执行 `flutter pub add material_ui cupertino_ui`

❌ **误改**:无

---

## 三、迁移建议

🔍 **依赖兼容性**:❌ **存在阻塞**

### 结论:建议**暂缓迁移**

**阻塞原因**:
- `legacy_theme_manager ^1.5.0` 在 API 签名中使用旧 `ThemeData` 类型
- 兼容桥无法解决此类类型不匹配问题
- 若强行迁移,会导致编译错误

### 后续行动计划

1. **短期(本周)**:
   - 在 `legacy_theme_manager` GitHub 仓库 issue #45 留言,说明迁移需求
   - 评估是否可用其他主题管理方案临时替代

2. **中期(2-4 周)**:
   - 关注 `legacy_theme_manager` pub.dev 版本更新
   - 若 4 周内无响应,考虑 fork 自行迁移

3. **长期(规划)**:
   - 若包长期不维护,制定完全移除该依赖的技术方案
   - 在团队技术债清单中记录此项

### 若移除阻塞后可迁移

**预计收益**:
- 跟进 material_ui 周更,获得最新修复
- 消除 Flutter 3.50+ 的 28 处 deprecated 警告
- 为未来独立包生态做准备

**预计成本**:
- 迁移执行:1-2 小时(应用 dart fix + 解决导入 + 配置本地化)
- 测试验证:2-3 小时(单元测试 + Widget 测试 + 手测关键流程)
- 中风险依赖测试:0.5 小时(`custom_bottom_sheet` 功能验证)

**总计**:约 0.5 个工作日

---

## 四、迁移前准备清单

环境检查:
- [x] Flutter 版本 >= 3.47(当前:3.50.0) ✅
- [x] 工作区干净(git status clean) ✅
- [x] 所有依赖已安装(flutter pub get) ✅

依赖审计:
- [x] 已识别 1 个高风险依赖(legacy_theme_manager)
- [ ] ❌ 阻塞性依赖尚未解决(等待更新)
- [x] 中风险依赖已制定测试计划(custom_bottom_sheet)

预览变更:
- [x] dart fix 预览已审查
- [x] 无非设计库导入被误改 ✅
- [x] 变更范围符合预期(23 文件,28 处导入)

备份与回滚:
- [x] 当前工作已提交 ✅
- [x] 知道如何回滚(git reset --hard)

知识准备:
- [x] 已阅读工作流文档
- [x] 了解兼容桥的能力与限制
- [x] 知道 gen-l10n 的配置变化

**状态**:⏸️ 暂缓迁移,等待 legacy_theme_manager 更新

---

**审计完成时间**:2026-11-27 10:30
**下次复查**:2026-12-11(2 周后检查依赖更新)
```

---

## 🔗 相关

- 工作流:[workflows/material-cupertino-ui-packages.md](../../workflows/material-cupertino-ui-packages.md)
- 应用迁移提示词:[apply-material-ui-migration.md](apply-material-ui-migration.md)
- Flutter 官方文档:[Migrate to material_ui and cupertino_ui](https://docs.flutter.dev/release/breaking-changes/material-ui-and-cupertino-ui)
