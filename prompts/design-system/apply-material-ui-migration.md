# 提示词:安全应用 material_ui / cupertino_ui 迁移并验证

**用途**:在审计和预览通过后,安全执行迁移,处理模糊导入、本地化配置、兼容桥,并用 analyze/test/构建验证无回归

---

## 📋 适用场景

- 已完成审计和预览(参考 `audit-and-preview-material-ui-migration.md`)
- 确认无阻塞性依赖或已解决
- 准备正式应用 `dart fix` 迁移
- 需要 AI 辅助处理迁移后的配置和验证

---

## 📥 输入要求

在 Cursor 中准备:

1. **审计报告已完成**:确认可立即迁移(无阻塞依赖)
2. **工作区干净**:无未提交变更或已备份
3. **依赖已添加**(若未自动添加):
   ```bash
   flutter pub add material_ui cupertino_ui
   ```
4. **打开关键文件**:
   - `lib/main.dart` 或 MaterialApp/CupertinoApp 所在文件
   - `pubspec.yaml`
   - `l10n.yaml`(若使用 gen-l10n)
   - 审计中识别的中/高风险依赖使用处

---

## 💬 提示词模板

```markdown
我已完成迁移前审计,确认可以迁移到 material_ui / cupertino_ui。请帮我安全应用迁移并验证。

**项目信息**:
- Flutter 版本:[如 3.50.0]
- 使用的应用入口:[MaterialApp / CupertinoApp / 两者都有]
- 是否使用 gen-l10n:[是 / 否]
- 中/高风险依赖(若有):[列出包名及版本]

---

## 任务 1:应用 dart fix 迁移

### 步骤 1.1:应用自动迁移(需人工执行命令)

请在终端执行以下命令:

```bash
# 应用迁移
dart fix --apply --code=migrate_design_widgets

# 再跑一次处理其他规则(如 import 排序)
dart fix --apply

# 查看变更
git diff
```

### 步骤 1.2:审查 git diff 输出

**审查要点**:

1. **导入替换**:
   - ✅ `package:flutter/material.dart` → `package:material_ui/material_ui.dart`
   - ✅ `package:flutter/cupertino.dart` → `package:cupertino_ui/cupertino_ui.dart`
   - ❌ 检查是否有非设计库导入被误改(如 services.dart、widgets.dart)

2. **pubspec.yaml**:
   - 确认 `material_ui` 和/或 `cupertino_ui` 已在 dependencies 中

3. **变更范围**:
   - lib/ 和 test/ 目录均已覆盖

**若发现误改**:

```markdown
请告知我需要恢复的文件和行号,我将手动执行:
```bash
# 示例:恢复单个文件的特定变更
git checkout HEAD -- lib/services/platform_service.dart
```
```

**若 dart fix 未修改 pubspec.yaml**:

```markdown
请提醒我手动添加依赖:
```bash
flutter pub add material_ui cupertino_ui
```
```

### 步骤 1.3:确认迁移应用成功

**检查点**:
- [ ] `git diff` 变更符合预期
- [ ] pubspec.yaml 包含必要依赖
- [ ] 无误改的非设计库导入

---

## 任务 2:解决 flutter_localizations 模糊导入

### 步骤 2.1:查找模糊导入问题

运行静态分析:

```bash
flutter analyze
```

查找 `ambiguous_import` 错误,通常是:

```
error - lib/xxx.dart:X:X - The name 'GlobalMaterialLocalizations' is defined in 
the libraries 'package:flutter_localizations/flutter_localizations.dart' and 
'package:material_ui/material_ui.dart' - ambiguous_import
```

### 步骤 2.2:修复模糊导入

对每个报错的文件,判断处理方式:

**方式 A**:不再需要 flutter_localizations,直接删除该导入

**修改前**:
```dart
import 'package:flutter_localizations/flutter_localizations.dart';
import 'package:material_ui/material_ui.dart';

// 若文件中不使用 GlobalWidgetsLocalizations
```

**修改后**:
```dart
// ✅ 删除 flutter_localizations 导入
import 'package:material_ui/material_ui.dart';
```

**方式 B**:仍需 `GlobalWidgetsLocalizations`,用 `show` 限制

**修改前**:
```dart
import 'package:flutter_localizations/flutter_localizations.dart';
import 'package:material_ui/material_ui.dart';

// 文件中使用了 GlobalWidgetsLocalizations
```

**修改后**:
```dart
import 'package:flutter_localizations/flutter_localizations.dart'
    show GlobalWidgetsLocalizations;
import 'package:material_ui/material_ui.dart';
```

### 步骤 2.3:生成修复后的文件列表

请分析所有报 `ambiguous_import` 的文件,判断每个文件应该用方式 A 还是方式 B,生成修复后的完整文件。

**输出格式**:

```
【模糊导入修复】

文件:lib/main.dart
方式:A(删除 flutter_localizations)
修复后导入部分:
```dart
import 'package:material_ui/material_ui.dart';
// flutter_localizations 已删除
```

文件:lib/app.dart
方式:B(限制为 GlobalWidgetsLocalizations)
修复后导入部分:
```dart
import 'package:flutter_localizations/flutter_localizations.dart'
    show GlobalWidgetsLocalizations;
import 'package:material_ui/material_ui.dart';
```
```

**验证**:修复后再运行 `flutter analyze`,确认无 `ambiguous_import` 错误。

---

## 任务 3:更新本地化配置

### 场景 A:手动配置 localization delegates(不使用 gen-l10n)

**步骤 3A.1**:找到 MaterialApp 或 CupertinoApp 配置

**步骤 3A.2**:更新 localizationsDelegates

**修改前**(MaterialApp):
```dart
import 'package:flutter/material.dart';
import 'package:flutter_localizations/flutter_localizations.dart';

MaterialApp(
  localizationsDelegates: const [
    GlobalMaterialLocalizations.delegate,
    GlobalCupertinoLocalizations.delegate,
    GlobalWidgetsLocalizations.delegate,
  ],
  supportedLocales: const [Locale('en'), Locale('zh')],
  // ...
);
```

**修改后**:
```dart
import 'package:material_ui/material_ui.dart';
// 不再需要 flutter_localizations

MaterialApp(
  // ✅ 简化:自动包含 Cupertino 和 Widgets delegates
  localizationsDelegates: GlobalMaterialLocalizations.delegates,
  supportedLocales: const [Locale('en'), Locale('zh')],
  // ...
);
```

**若是 CupertinoApp**:

判断应用中是否使用了 Material 组件(如 Card、TextField、Scaffold):

- **纯 Cupertino**(不用 Material 组件):
  ```dart
  CupertinoApp(
    localizationsDelegates: GlobalCupertinoLocalizations.delegates,
    // ...
  );
  ```

- **混合使用**(CupertinoApp 中有 Material 组件):
  ```dart
  CupertinoApp(
    // 必须用 GlobalMaterialLocalizations.delegates
    // 否则 Material 组件会报 "No MaterialLocalizations found"
    localizationsDelegates: GlobalMaterialLocalizations.delegates,
    // ...
  );
  ```

请生成修复后的完整 MaterialApp/CupertinoApp 配置代码。

---

### 场景 B:使用 gen-l10n

以项目锁定的 Flutter SDK 运行 `flutter gen-l10n`，不要只看版本号。Flutter **3.50.0 和 3.50.1** 仍可能生成旧的 `flutter_localizations` 导入和 delegates。其他版本也以实际生成的文件为准。

在 `l10n.yaml` 中查找 `output-dir`；未设置时用 `arb-dir`。文件名以 `output-localization-file` 为准，默认是 `app_localizations.dart`。从生成到 `lib/` 下的实际源码路径导入。

- **已生成与入口匹配的独立包 delegates**：可以直接用 `AppLocalizations.localizationsDelegates`，并保留项目已有的自定义 delegates。
- **仍生成旧 delegates，或生成的列表不匹配入口**：不要手改生成文件，也不要直接使用旧列表。在应用入口组合 `AppLocalizations.delegate` 和独立包 delegates。只要生成文件还导入 `flutter_localizations`，就保留 `pubspec.yaml` 中的 SDK 依赖，即使入口不再导入它。

示例：应用入口在 `lib/main.dart`，本地化文件生成到 `lib/l10n/app_localizations.dart`。

```dart
import 'package:material_ui/material_ui.dart';

import 'l10n/app_localizations.dart';

MaterialApp(
  localizationsDelegates: const [
    AppLocalizations.delegate,
    ...GlobalMaterialLocalizations.delegates,
  ],
  supportedLocales: AppLocalizations.supportedLocales,
  // ...
);
```

纯 Cupertino 入口且不使用 Material 组件时，从 `cupertino_ui` 导入并改用 `GlobalCupertinoLocalizations.delegates`；Cupertino 入口混用新 Material 组件时，仍用 `material_ui` 的 `GlobalMaterialLocalizations.delegates`。如果项目有额外的 delegates，保留它们。

输出实际 Flutter 版本、生成文件路径、应用入口采用的 delegates 和验证结果。

- [ ] 使用项目锁定的 SDK 运行 `flutter gen-l10n` 和 `flutter analyze`，确认生成文件、导入路径及入口 delegates 正确。
- [ ] 用项目已支持的一种非英文语言（如 `zh`）运行应用：核对一条 App 自有译文，再打开独立包的日期选择器，确认可打开且控件文本也使用目标语言，而不是回退到英文。纯 Cupertino 应用检查 `CupertinoDatePicker` 的日期格式与文案；混用 Material 的入口也要测 Material 日期选择器。
- [ ] 记录 Flutter 版本、语言和实测结果；没跑过界面就写“未验证”，不要按生成成功或英文页面正常判定通过。

---

## 任务 4:处理中/高风险依赖(若有)

**若审计中识别了中风险依赖**,需评估是否添加兼容桥。

### 步骤 4.1:评估是否需要兼容桥

对每个中风险依赖:

1. **检查使用位置**:在项目中搜索该包的使用
2. **判断依赖需求**:该包的子组件是否调用 `Theme.of(context)` 或 `MaterialLocalizations.of(context)`?
   - 若是:需要兼容桥
   - 若否:不需要,只需测试验证

### 步骤 4.2:生成兼容桥代码(若需要)

**MaterialApp 示例**:

```dart
import 'package:material_ui/material_ui.dart';

MaterialApp(
  theme: ThemeData(
    colorScheme: ColorScheme.fromSeed(
      seedColor: const Color(0xFF6750A4),
    ),
  ),
  builder: (BuildContext context, Widget? child) {
    // ignore: deprecated_member_use
    // TODO(tech-debt): 移除兼容桥
    // 原因:依赖 [包名] v[版本] 尚未支持 material_ui
    // 移除条件:升级到 [包名] v[目标版本]+
    // 跟踪:[GitHub issue 链接]
    return MaterialUiCompatibilityBridge(child: child!);
  },
  home: const HomeScreen(),
);
```

**CupertinoApp 示例**:

```dart
import 'package:cupertino_ui/cupertino_ui.dart';

CupertinoApp(
  builder: (context, child) {
    // ignore: deprecated_member_use
    // TODO(tech-debt): 移除兼容桥
    // 原因:[说明]
    return CupertinoUiCompatibilityBridge(child: child!);
  },
  // ...
);
```

### 步骤 4.3:记录技术债

请生成技术债记录,包含:
- 添加兼容桥的原因(哪个依赖未迁移)
- 移除条件(依赖升级到哪个版本)
- 跟踪链接(GitHub issue 或 pub.dev)

**输出格式**:

```
【技术债记录】

项目:my_flutter_app
记录日期:2026-11-27

技术债项:MaterialUiCompatibilityBridge 兼容桥

原因:依赖 `custom_ui_kit` v3.2.0 尚未支持 material_ui,其子组件读取 Theme.of(context)

影响:
- 额外 InheritedWidget 层,轻微性能开销
- 无法传递新类型参数给该依赖的 API(若有)

移除条件:
- `custom_ui_kit` 升级到 v4.0.0+(已确认 roadmap 包含 material_ui 支持)

跟踪:
- pub.dev:https://pub.dev/packages/custom_ui_kit
- GitHub issue:https://github.com/author/custom_ui_kit/issues/78

复查周期:每 2 周检查一次版本更新

下次复查:2026-12-11
```

---

## 任务 5:验证迁移结果(关键!)

### 步骤 5.1:静态分析

```bash
flutter analyze --fatal-infos
```

**预期**:无错误,无警告(除非使用兼容桥的 deprecated_member_use,已用 ignore 标注)

**若有错误**:

根据错误类型分类处理:

| 错误类型 | 可能原因 | 解决方案 |
|----------|----------|----------|
| ambiguous_import | flutter_localizations 模糊导入未解决 | 返回任务 2 |
| undefined_class | material_ui/cupertino_ui 依赖未添加 | 运行 flutter pub get |
| type_mismatch | 依赖 API 类型签名不匹配 | 检查是否有遗漏的高风险依赖 |

请分析报错信息,给出具体修复方案。

---

### 步骤 5.2:运行测试

```bash
# 单元测试 + Widget 测试
flutter test

# 若有特定测试套件,也运行
# flutter test test/integration_test/
```

**预期**:所有测试通过

**若测试失败**:

分析失败原因:

1. **本地化相关失败**:"No MaterialLocalizations found"
   - 原因:测试中 MaterialApp 配置缺少 localizationsDelegates
   - 解决:在测试的 MaterialApp 中添加 `GlobalMaterialLocalizations.delegates`

2. **Golden 测试失败**:UI 快照不匹配
   - 原因:material_ui 组件渲染可能与旧版略有差异
   - 解决:人工审查 diff,若无问题,更新 golden 文件

3. **主题相关失败**:Theme.of(context) 返回意外值
   - 原因:兼容桥未覆盖该测试
   - 解决:在测试中也添加兼容桥(若必要)

请分析失败测试的具体错误信息,给出修复代码。

---

### 步骤 5.3:构建验证

```bash
# Android
flutter build apk --debug

# iOS(若在 macOS)
flutter build ios --debug

# Web(若支持)
flutter build web
```

**预期**:构建成功,无类型错误或链接错误

**若构建失败**:

分析构建日志,常见问题:

- Dart 编译错误:说明仍有未解决的类型问题
- 依赖冲突:检查 pubspec.yaml 版本约束
- 平台特定错误:检查原生代码(Android/iOS)是否有设计库耦合

请提供构建日志中的关键错误信息,我将分析并给出修复方案。

---

### 步骤 5.4:手动功能测试

运行应用并验证:

```bash
flutter run --debug
```

**测试清单**:

应用启动:
- [ ] 启动无报错
- [ ] 首屏正常显示

主题与样式:
- [ ] 主题色、文字样式正确
- [ ] Material 组件(Button、Card、AppBar 等)渲染正常
- [ ] Cupertino 组件(若使用)渲染正常

本地化:
- [ ] 默认语言显示正确
- [ ] 切换语言功能正常(若支持多语言)
- [ ] 非英文环境下，App 自有译文和独立包日期选择器均正确（见任务 3）

第三方包功能:
- [ ] 中/高风险依赖包的组件显示和交互正常
- [ ] 无类型相关 crash

热重载:
- [ ] 修改代码保存后热重载成功
- [ ] UI 立即更新,无报错

**若发现问题**:请详细描述:
- 问题现象(报错信息 / UI 异常 / crash)
- 触发步骤
- 预期行为 vs 实际行为

我将分析并提供修复方案。

---

## 任务 6:生成迁移完成报告

请生成最终报告,包含:

```
【迁移完成报告】

项目:my_flutter_app
迁移日期:2026-11-27
Flutter 版本:3.50.0

---

## 一、迁移执行概况

✅ 自动迁移:`dart fix --code=migrate_design_widgets` 已应用
✅ 导入替换:共 [X] 处,路径正确
✅ 依赖添加:`material_ui` 和 `cupertino_ui` 已添加到 pubspec.yaml
✅ 模糊导入:已解决 [Y] 个文件的 ambiguous_import 问题
✅ 本地化配置:已更新为 GlobalMaterialLocalizations.delegates
[若有] ⚠️ 兼容桥:已添加(临时,等待依赖更新)

---

## 二、验证结果

✅ 静态分析:`flutter analyze --fatal-infos` 通过
✅ 单元测试:`flutter test` 全部通过 ([X] 个测试)
✅ 构建验证:Android debug apk 构建成功
✅ 手动测试:关键页面和功能无回归

---

## 三、变更文件清单

**lib/ 目录([X] 个文件)**:
- lib/main.dart:导入 material.dart → material_ui.dart,更新 localizationsDelegates
- lib/app.dart:导入 material.dart → material_ui.dart
- lib/pages/home_page.dart:导入 material.dart → material_ui.dart
- ...

**test/ 目录([Y] 个文件)**:
- test/widget_test.dart:导入 material.dart → material_ui.dart
- ...

**配置文件**:
- pubspec.yaml:添加 material_ui 和 cupertino_ui 依赖
- [若有] l10n.yaml:记录本次实际调整和理由

---

## 四、技术债记录(若有)

[若无技术债]
✅ 无技术债,迁移完全完成

[若有技术债]
⚠️ 技术债项 1:MaterialUiCompatibilityBridge 兼容桥

- 原因:依赖 `[包名]` v[版本] 尚未支持 material_ui
- 位置:lib/main.dart MaterialApp.builder
- 移除条件:升级到 `[包名]` v[目标版本]+
- 跟踪:[链接]
- 复查周期:每 2 周
- 下次复查:2026-12-11

---

## 五、后续建议

1. **提交变更**:
   ```bash
   git add .
   git commit -m "迁移到 material_ui / cupertino_ui 独立设计组件包"
   git push origin [分支名]
   ```

2. **监控 CI/CD**:确保远程构建也通过

3. **跟进技术债**(若有):
   - 定期检查依赖更新
   - 在日历中设置复查提醒

4. **团队同步**:
   - 通知团队成员本地 pull 最新代码
   - 确保团队 Flutter 版本 >= 3.47

5. **跟进独立包更新**:
   - 订阅 material_ui / cupertino_ui changelog
   - 定期运行 `flutter pub upgrade` 获取最新修复

---

## 六、回滚方案(若需要)

若迁移后发现严重问题,可回滚:

```bash
# 回滚到迁移前
git reset --hard HEAD~1

# 或回滚到特定 commit
git reset --hard [commit-sha]

# 恢复依赖
flutter pub get
```

---

迁移完成 ✅
```

---

**最终确认**:

请人工确认以下事项:
- [ ] 所有任务(1-6)已完成
- [ ] `flutter analyze --fatal-infos` 通过
- [ ] `flutter test` 通过
- [ ] 至少一个平台构建成功
- [ ] 应用运行正常,手测无回归
- [ ] 技术债(若有)已记录
```

---

## ✅ 验证步骤

提示词执行后:

1. **审查修复代码**:
   - [ ] 模糊导入修复正确?
   - [ ] 本地化配置符合应用类型(MaterialApp/CupertinoApp)?
   - [ ] 兼容桥(若有)只在必要时添加?

2. **执行验证命令**:
   ```bash
   flutter analyze --fatal-infos
   flutter test
   flutter build apk --debug  # 或其他平台
   flutter run --debug
   ```

3. **手测关键功能**:
   - [ ] 应用启动和主要流程
   - [ ] 多语言切换(若支持)
   - [ ] 第三方包组件

4. **审查技术债记录**(若有):
   - [ ] 原因说明清晰?
   - [ ] 移除条件可操作?
   - [ ] 跟踪链接可访问?

5. **提交变更**:
   ```bash
   git add .
   git commit -m "迁移到 material_ui / cupertino_ui 独立设计组件包

- 应用 dart fix --code=migrate_design_widgets
- 更新 localization delegates 配置
- 解决 flutter_localizations 模糊导入
- [若有] 添加兼容桥处理未迁移依赖

参考:https://docs.flutter.dev/release/breaking-changes/material-ui-and-cupertino-ui
Flutter 版本:3.50.0"
   ```

---

## 🛠️ 常见调整

### 1. 只迁移 Material,不迁移 Cupertino

```markdown
项目只使用 MaterialApp 和 Material 组件,不需要 Cupertino。

请:
- 只处理 `package:flutter/material.dart` 导入
- 只添加 `material_ui` 依赖
- 忽略所有 Cupertino 相关配置
```

### 2. 跳过某些验证步骤(若时间紧急)

```markdown
由于时间限制,请先完成迁移和静态分析验证:
- 任务 1-3:迁移和配置
- 任务 5.1:flutter analyze
- 任务 5.4:快速手测启动和首屏

任务 5.2(flutter test)和 5.3(构建)将在 CI 中验证。
```

### 3. 处理特定测试失败

```markdown
`flutter test` 中有 3 个测试失败,错误信息:

```
test/pages/home_test.dart:45:5
Expected: <true>
Actual: <false>
Error: No MaterialLocalizations found
```

请分析此测试失败原因并提供修复代码。
```

---

## 🔗 相关

- 工作流:[workflows/material-cupertino-ui-packages.md](../../workflows/material-cupertino-ui-packages.md)
- 审计提示词:[audit-and-preview-material-ui-migration.md](audit-and-preview-material-ui-migration.md)
- Flutter 官方文档:[Migrate to material_ui and cupertino_ui](https://docs.flutter.dev/release/breaking-changes/material-ui-and-cupertino-ui)
