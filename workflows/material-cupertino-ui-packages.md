# material_ui / cupertino_ui 独立设计组件包迁移工作流

**将 Flutter 项目从框架内设计库迁移到独立发布的 package:material_ui 和 package:cupertino_ui**

> 作者:Loong (SSLoong) | 日期:2026-11-27

---

## 📋 背景

Flutter 3.50(2026-11-11 随 Dart 3.14 发布)正式弃用框架内设计库,改用独立发布的 `package:material_ui` 和 `package:cupertino_ui`。这是官方的 **breaking change**,旨在实现设计系统与核心框架解耦:

- **Flutter 3.44** 起框架内 `package:flutter/material.dart` 和 `package:flutter/cupertino.dart` 的贡献冻结
- **Flutter 3.47** 起可选用独立包(版本 1.0.0 对齐冻结代码)
- **Flutter 3.50** 起新项目默认使用独立包,框架内设计库正式标记 deprecated

**解耦带来的核心价值**:
- **独立发版节奏**:组件更新、bug 修复不再等待季度 SDK 发布(计划周更)
- **社区贡献通道**:开发者可向 `flutter/packages` 直接贡献
- **风格中立核心**:为未来的风格无关 widget catalog 铺路

**与主题迁移的区别**:
- 本工作流处理 **包导入与依赖**,不涉及主题系统重构
- Material 3 主题体系(ThemeData/ColorScheme)仍需参考 [workflows/theme-consistency.md](theme-consistency.md)
- 两者可独立进行,也可结合使用

**与 dart migrate 的区别**:
- 本工作流用 `dart fix --code=migrate_design_widgets`,专门迁移设计库导入
- 不要与 `dart migrate` SDK 升级工具混淆(那是用于 Dart 2.x → 3.x 的 null safety 迁移)

---

## 🎯 目标

- ✅ 理解何时应立即迁移、何时可暂缓
- ✅ 评估项目依赖对独立包的兼容性
- ✅ 用 `dart fix` 自动迁移导入路径与依赖声明
- ✅ 解决 `flutter_localizations` 的模糊导入问题
- ✅ 正确配置本地化(localization)delegates
- ✅ 按需添加兼容桥接,记录技术债
- ✅ 在 analyze/test/构建中验证无回归

---

## 🔧 适用场景

| 场景 | 建议 | 说明 |
|------|------|------|
| 新项目(Flutter 3.50+) | ⭐⭐⭐⭐⭐ | 官方默认,无历史包袱,立即迁移 |
| 自有应用,依赖可控 | ⭐⭐⭐⭐⭐ | 可快速跟进独立包周更,获得最新修复 |
| 公司内部依赖包,统一迁移 | ⭐⭐⭐⭐ | 可分步迁移底层依赖再迁移应用 |
| 大量第三方包在公共 API 中暴露旧类型 | ⭐⭐ | 兼容桥无法解决类型签名不匹配,等依赖更新 |
| 正在做其他重大迁移(如 null safety) | ⭐⭐ | 建议先完成其他迁移,避免变更叠加 |
| 未使用 Material/Cupertino 组件的纯自定义应用 | ⭐ | 实际上不受影响,但仍建议迁移以消除 deprecated 警告 |

**决策要点**:
- **立即迁移**:希望跟进独立包周更、新项目、依赖包完全自主
- **暂缓迁移**:依赖包在 API 签名中大量使用旧类型(如参数/返回值是框架内 `ColorScheme`、`TextTheme`、`FloatingActionButtonLocation`),需等待依赖先迁移
- **框架内库已 deprecated ≠ 今天不迁就无法编译**:旧导入在 3.50 仍可编译,只是会触发 deprecated 警告

---

## 🚀 完整流程

### Step 0: 确认 Flutter 版本与迁移必要性

**人工操作**:检查项目 Flutter SDK 版本。

```bash
flutter --version

# 检查当前导入
grep -r "package:flutter/material.dart" lib/
grep -r "package:flutter/cupertino.dart" lib/
```

**决策树**:

```
Flutter 版本 < 3.47？
  └─ 建议:先升级 SDK 到 3.50+,再迁移独立包

Flutter 版本 >= 3.47 且 < 3.50？
  └─ 可选:可迁移,但不强制(框架内库尚未 deprecated)

Flutter 版本 >= 3.50？
  └─ 建议:尽快迁移(框架内库已 deprecated,触发 analyze 警告)

是否有 `package:flutter/material.dart` 或 `package:flutter/cupertino.dart` 导入？
  └─ 否 → 无需迁移(纯 Widgets 应用)
  └─ 是 → 继续评估依赖
```

**检查点**:
- [ ] Flutter SDK >= 3.47(建议 3.50+)
- [ ] 项目中存在需要迁移的导入
- [ ] 工作区干净(无未提交变更)

---

### Step 1: 评估第三方依赖兼容性

**关键概念:公共 API 类型不匹配**

兼容桥(`MaterialUiCompatibilityBridge`)可以让未迁移的子组件通过 context 读取主题和本地化,**但无法解决类型签名问题**:

```dart
// ❌ 兼容桥无法解决此类问题:
// 第三方包参数要求旧类型 ColorScheme(来自 package:flutter/material.dart)
somePackageWidget(
  colorScheme: Theme.of(context).colorScheme, // 新类型,来自 material_ui
  // ↑ 编译错误:类型不匹配
);
```

**AI 提示词**(可复制):

```markdown
请审计项目依赖包,判断是否可立即迁移到 material_ui / cupertino_ui。

**审计对象**:`pubspec.yaml` 中的直接依赖(不含 SDK 依赖)

**审计维度**:

1. **检查依赖文档**:访问 pub.dev,查看包是否已声明支持 material_ui / cupertino_ui
2. **检查使用方式**:在项目代码中搜索依赖包的使用,判断是否在 API 参数/返回值中传递了设计库类型
   - 高风险:参数类型是 `ColorScheme`、`TextTheme`、`ThemeData`、`FloatingActionButtonLocation` 等设计库核心类型
   - 低风险:只传递基本类型(String、Widget、Function)或读取 Theme.of(context)
3. **分类汇总**:
   - ✅ 兼容:已支持新包 或 只用基本类型
   - ⚠️ 需观察:尚无新包支持说明,但使用方式低风险
   - ❌ 阻塞:已确认在 API 签名中使用旧类型,且尚未迁移

**输出格式**:

| 依赖包 | 版本 | 兼容性 | 风险说明 |
|--------|------|--------|----------|
| provider | ^6.1.0 | ✅ | 只传递基本类型,不涉及设计库 |
| flutter_svg | ^2.0.0 | ✅ | 只用 Widget,无类型耦合 |
| custom_theme_plugin | ^1.2.0 | ❌ | API 参数要求 `ColorScheme`,尚未更新 |

**结论**:
- 若有 ❌ 阻塞项:建议暂缓迁移,等待依赖更新或联系作者
- 若全 ✅ / ⚠️:可立即迁移,用兼容桥处理低风险依赖
```

**检查点**:
- [ ] 依赖审计报告已生成
- [ ] 阻塞性依赖已识别(若有)
- [ ] 迁移决策已明确(立即迁移 / 暂缓)

**若有阻塞依赖,暂停迁移**,关注依赖包更新动态。

---

### Step 2: 添加独立包依赖(可选)

**说明**:`dart fix` 在某些情况下会自动添加依赖,但为保险起见,建议先手动添加。

```bash
# 添加 material_ui(若项目使用 MaterialApp 或 Material 组件)
flutter pub add material_ui

# 添加 cupertino_ui(若项目使用 CupertinoApp 或 Cupertino 组件)
flutter pub add cupertino_ui
```

**检查点**:
- [ ] `pubspec.yaml` 的 `dependencies` 中包含 `material_ui` 和/或 `cupertino_ui`
- [ ] `flutter pub get` 无报错

---

### Step 3: 预览自动迁移(dry-run)

**关键**:先 dry-run 查看变更,再决定是否应用。

```bash
# dry-run:不实际修改文件,只输出 diff
dart fix --dry-run --code=migrate_design_widgets > migration_preview.txt

# 人工审查 migration_preview.txt
cat migration_preview.txt
```

**AI 提示词**(可复制):

```markdown
请审查 `migration_preview.txt` 中 dart fix 预览的变更,判断是否安全应用。

**审查要点**:

1. **导入替换正确性**:
   - `package:flutter/material.dart` → `package:material_ui/material_ui.dart`
   - `package:flutter/cupertino.dart` → `package:cupertino_ui/cupertino_ui.dart`
   - 检查是否有非设计库的 flutter 导入被误改(如 `package:flutter/widgets.dart` 应保持不变)

2. **依赖声明**:
   - `pubspec.yaml` 中 `material_ui` / `cupertino_ui` 是否已添加(或需手动补充)

3. **范围覆盖**:
   - 所有 `.dart` 文件中的旧导入是否都被检测到
   - 有无遗漏的文件(如 `test/` 目录)

**输出格式**:

```
【预览审查结果】

✅ 导入替换:共 X 处,路径正确
⚠️ 依赖声明:需手动在 pubspec.yaml 添加 cupertino_ui
✅ 范围覆盖:lib/ 和 test/ 均已覆盖
❌ 误改:发现 1 处误将 package:flutter/services.dart 改为 material_ui

【建议】:
- 先手动添加 cupertino_ui 到 pubspec.yaml
- 应用 dart fix 后,人工恢复 lib/utils/platform_service.dart:5 的 services 导入
```
```

**检查点**:
- [ ] 预览变更已人工审查
- [ ] 无明显误改(如非设计库导入被改)
- [ ] 确认可以应用

---

### Step 4: 应用自动迁移

**人工操作**:

```bash
# 应用迁移
dart fix --apply --code=migrate_design_widgets

# 检查 pubspec.yaml 是否需要手动补充依赖
cat pubspec.yaml | grep -A 5 "dependencies:"

# 若 dart fix 未自动添加,手动补充(Step 2)

# 再跑一次 dart fix 处理其他规则(如 import 排序)
dart fix --apply

# 查看变更
git diff
```

**关键变更示例**:

**修改前**(`lib/main.dart`):
```dart
import 'package:flutter/material.dart';
import 'package:flutter/cupertino.dart';

void main() {
  runApp(MaterialApp(home: HomeScreen()));
}
```

**修改后**:
```dart
import 'package:material_ui/material_ui.dart';
import 'package:cupertino_ui/cupertino_ui.dart';

void main() {
  runApp(MaterialApp(home: HomeScreen()));
}
```

**检查点**:
- [ ] 所有 `package:flutter/material.dart` 已改为 `package:material_ui/material_ui.dart`
- [ ] 所有 `package:flutter/cupertino.dart` 已改为 `package:cupertino_ui/cupertino_ui.dart`
- [ ] `pubspec.yaml` 包含必要依赖
- [ ] `git diff` 变更符合预期

---

### Step 5: 解决模糊导入(ambiguous_import)

**问题**:`dart fix` 不会自动删除 `package:flutter_localizations/flutter_localizations.dart` 导入。与新包同时导入会导致 `GlobalMaterialLocalizations` 和 `GlobalCupertinoLocalizations` 的 `ambiguous_import` 错误。

**解决方案 1**:不再需要 flutter_localizations,直接删除该导入

```dart
// ❌ 删除此行(若不需要 GlobalWidgetsLocalizations)
// import 'package:flutter_localizations/flutter_localizations.dart';

import 'package:material_ui/material_ui.dart';
```

**解决方案 2**:仍需 `GlobalWidgetsLocalizations`,用 `show` 限制导入

```dart
// ✅ 只导入 GlobalWidgetsLocalizations
import 'package:flutter_localizations/flutter_localizations.dart'
    show GlobalWidgetsLocalizations;
import 'package:material_ui/material_ui.dart';
```

**AI 提示词**(可复制):

```markdown
项目迁移到 material_ui 后,flutter analyze 报告 ambiguous_import 错误。

**错误信息**:
```
error - lib/main.dart:3:8 - The name 'GlobalMaterialLocalizations' is defined in 
the libraries 'package:flutter_localizations/flutter_localizations.dart' and 
'package:material_ui/material_ui.dart' - ambiguous_import
```

**要求**:

1. 检查 lib/ 和 test/ 中所有同时导入 flutter_localizations 和 material_ui/cupertino_ui 的文件
2. 判断每个文件是否仍需 GlobalWidgetsLocalizations:
   - 若不需要:删除 flutter_localizations 导入
   - 若需要:改为 `import 'package:flutter_localizations/flutter_localizations.dart' show GlobalWidgetsLocalizations;`
3. 生成修复后的完整文件列表

**输出格式**:

| 文件 | 操作 | 修复后导入 |
|------|------|------------|
| lib/main.dart | 删除 | 只保留 material_ui |
| lib/app.dart | 限制 | show GlobalWidgetsLocalizations |
```

**检查点**:
- [ ] `flutter analyze` 无 ambiguous_import 错误
- [ ] 仍需的 GlobalWidgetsLocalizations 功能正常

---

### Step 6: 更新本地化(Localization)配置

**关键变更**:Material 和 Cupertino 的本地化 delegates 和译文现在在独立包中。

#### 场景 A:不使用 gen-l10n,手动配置 delegates

**修改前**(`lib/main.dart`):
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
// 不再需要导入 flutter_localizations

MaterialApp(
  // ✅ 简化:自动包含 Cupertino 和 Widgets delegates
  localizationsDelegates: GlobalMaterialLocalizations.delegates,
  supportedLocales: const [Locale('en'), Locale('zh')],
  // ...
);
```

**CupertinoApp 注意事项**:

```dart
// ✅ 纯 Cupertino 应用(不用 Material 组件)
CupertinoApp(
  localizationsDelegates: GlobalCupertinoLocalizations.delegates,
  // ...
);

// ✅ CupertinoApp 中也用了 Material 组件(如 Card、TextField)
CupertinoApp(
  // 必须用 GlobalMaterialLocalizations.delegates,否则 Material 组件会报错
  // "No MaterialLocalizations found"
  localizationsDelegates: GlobalMaterialLocalizations.delegates,
  // ...
);
```

#### 场景 B:使用 gen-l10n

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

**检查点**:
- [ ] 使用项目锁定的 SDK 运行 `flutter gen-l10n` 和 `flutter analyze`，确认生成文件、导入路径及入口 delegates 正确。
- [ ] 用项目已支持的一种非英文语言（如 `zh`）运行应用：核对一条 App 自有译文，再打开独立包的日期选择器，确认可打开且控件文本也使用目标语言，而不是回退到英文。纯 Cupertino 应用检查 `CupertinoDatePicker` 的日期格式与文案；混用 Material 的入口也要测 Material 日期选择器。
- [ ] 记录 Flutter 版本、语言和实测结果；没跑过界面就写“未验证”，不要按生成成功或英文页面正常判定通过。

---

### Step 7: 添加兼容桥(按需,记录技术债)

**仅当**项目依赖包尚未迁移,且这些包的子组件需要从 context 读取主题或本地化时,才添加兼容桥。

**兼容桥能解决**:
- ✅ 子组件读取 `Theme.of(context)` / `MaterialLocalizations.of(context)`
- ✅ 向下传播新包的主题数据到旧包的 widget 树

**兼容桥不能解决**:
- ❌ API 签名中的类型不匹配(如参数要求旧 `ColorScheme`)
- ❌ 依赖包在内部导入了旧设计库的私有 API

**AI 提示词**(可复制):

```markdown
项目已迁移到 material_ui,但使用的第三方包 `[包名]` 尚未迁移。请评估是否需要兼容桥。

**使用情况**:
- 包版本:`[版本号]`
- 使用方式:
```dart
// 示例:展示如何使用该包
SomePackageWidget(
  config: SomeConfig(),
  child: Text('示例'),
);
```

**要求**:

1. 分析该包是否会从 context 读取 Theme 或 Localizations
2. 判断是否需要兼容桥:
   - 需要:包的子组件调用 Theme.of(context) 或 MaterialLocalizations.of(context)
   - 不需要:包只用基本 widget,不依赖设计库 context
3. 若需要,生成兼容桥代码,并标注技术债

**输出格式**:

```
【评估结果】
包名:`some_package`
是否需要兼容桥:✅ 是 / ❌ 否
原因:[简要说明]

【兼容桥代码】(若需要)

// lib/main.dart
import 'package:material_ui/material_ui.dart';

MaterialApp(
  builder: (context, child) {
    // ignore: deprecated_member_use
    return MaterialUiCompatibilityBridge(child: child!);
  },
  // ...
);

【技术债记录】
// TODO(tech-debt): 移除 MaterialUiCompatibilityBridge
// 等待 some_package 升级到 material_ui 后删除
// 跟踪:https://github.com/author/some_package/issues/XXX
```
```

**示例代码**:

```dart
import 'package:material_ui/material_ui.dart';

void main() {
  runApp(const MyApp());
}

class MyApp extends StatelessWidget {
  const MyApp({super.key});

  @override
  Widget build(BuildContext context) {
    return MaterialApp(
      theme: ThemeData(
        colorScheme: ColorScheme.fromSeed(
          seedColor: const Color(0xFF6750A4),
        ),
      ),
      builder: (BuildContext context, Widget? child) {
        // ignore: deprecated_member_use
        // TODO(tech-debt): 移除兼容桥,等待 legacy_ui_package 更新
        return MaterialUiCompatibilityBridge(child: child!);
      },
      home: const HomeScreen(),
    );
  }
}
```

**CupertinoApp 示例**:

```dart
import 'package:cupertino_ui/cupertino_ui.dart';

CupertinoApp(
  builder: (context, child) {
    // ignore: deprecated_member_use
    return CupertinoUiCompatibilityBridge(child: child!);
  },
  // ...
);
```

**检查点**:
- [ ] 只在必要时添加兼容桥(非默认添加)
- [ ] 兼容桥加了 `// ignore: deprecated_member_use`
- [ ] 技术债已记录在代码注释或项目文档中
- [ ] 已跟踪依赖包的更新状态(GitHub issue / pub.dev 版本)

---

### Step 8: 验证与测试(必须!)

**人工操作**:

```bash
# 1. 静态分析
flutter analyze --fatal-infos

# 预期:无错误,无 deprecated_member_use 警告(除非使用兼容桥)

# 2. 单元测试与 Widget 测试
flutter test

# 预期:所有测试通过,无本地化或主题相关失败

# 3. 构建应用(各平台)
flutter build apk --debug        # Android
flutter build ios --debug        # iOS(需 macOS)
flutter build web                 # Web(若支持)

# 预期:构建成功,无类型错误或链接错误

# 4. 运行应用并手动测试
flutter run --debug

# 验证清单:
# - [ ] 应用启动无报错
# - [ ] 主题色、文字样式显示正常
# - [ ] 切换语言(若支持多语言)功能正常
# - [ ] Material 组件(Button、Card、AppBar)渲染正确
# - [ ] Cupertino 组件(若使用)渲染正确
# - [ ] 第三方包组件(若有)显示和交互正常

# 5. 热重载测试
# 在 `flutter run` 状态下修改代码(如改 Text 文字),保存
# 预期:热重载成功,无报错,UI 立即更新
```

**常见问题排查**:

| 现象 | 可能原因 | 解决方案 |
|------|----------|----------|
| "No MaterialLocalizations found" | CupertinoApp 中用了 Material 组件但未提供 MaterialLocalizations delegate | 改用 `GlobalMaterialLocalizations.delegates` |
| "ambiguous_import" 错误 | 同时导入 flutter_localizations 和 material_ui/cupertino_ui | 删除 flutter_localizations 或用 `show` 限制 |
| 类型不匹配(Type 'ColorScheme' is not a subtype of type 'ColorScheme') | 依赖包 API 参数要求旧类型 | 等待依赖更新或联系作者,暂用兼容桥(若可行) |
| 编译错误 "未定义符号 MaterialApp" | material_ui 依赖未添加或未 pub get | 运行 `flutter pub add material_ui && flutter pub get` |
| 非英语 locale 下本地化缺失 | gen-l10n 生成的 delegates 不正确(旧工具链) | 参考 Step 6 场景 B,手动组合 delegates 或升级 Flutter |

**检查点**:
- [ ] `flutter analyze --fatal-infos` 通过
- [ ] `flutter test` 全部通过
- [ ] 至少一个平台构建成功(Android/iOS/Web)
- [ ] 应用运行正常,关键页面手测无回归
- [ ] 多语言(若有)切换正常

---

### Step 9: 提交变更

**人工操作**:

```bash
# 查看变更
git status
git diff

# 提交
git add .
git commit -m "迁移到 material_ui / cupertino_ui 独立设计组件包

- 应用 dart fix --code=migrate_design_widgets 迁移导入
- 更新 localization delegates 配置
- 解决 flutter_localizations 模糊导入
- [若有] 添加兼容桥处理未迁移依赖包

参考:https://docs.flutter.dev/release/breaking-changes/material-ui-and-cupertino-ui
Flutter 版本:3.50.0"

# 推送分支
git push -u origin [分支名]
```

**检查点**:
- [ ] commit 信息清晰,说明变更内容
- [ ] 分支已推送到远程
- [ ] CI/CD 构建通过(若有)

---

## ✅ 验证清单

完成后确认:

- [ ] 所有 `package:flutter/material.dart` 已改为 `package:material_ui/material_ui.dart`
- [ ] 所有 `package:flutter/cupertino.dart` 已改为 `package:cupertino_ui/cupertino_ui.dart`
- [ ] `pubspec.yaml` 包含 `material_ui` 和/或 `cupertino_ui` 依赖
- [ ] `flutter_localizations` 的 ambiguous_import 已解决
- [ ] Localization delegates 配置正确(手动或 gen-l10n 生成)
- [ ] `flutter analyze --fatal-infos` 无错误和警告(除非有文档记录的技术债)
- [ ] `flutter test` 全部通过
- [ ] 应用在至少一个平台构建并运行正常
- [ ] 关键页面手测无回归(主题、本地化、第三方包组件)
- [ ] 兼容桥(若使用)已记录技术债,跟踪依赖更新

---

## 🤔 FAQ

### Q1: 迁移后 analyze 仍提示 deprecated_member_use?

**A**: 检查是否有以下情况:

1. **使用了兼容桥**:这是正常的,兼容桥本身就是 deprecated。确保加了 `// ignore: deprecated_member_use`。
2. **测试文件未迁移**:检查 `test/` 目录是否有文件仍导入旧包。
3. **生成代码未更新**:如使用 `build_runner` 或 `flutter gen-l10n`,重新生成:`flutter pub run build_runner build --delete-conflicting-outputs` / `flutter gen-l10n`。
4. **第三方包引入**:依赖包内部仍用旧导入,无法直接修复,需等待依赖更新。

---

### Q2: 兼容桥可以长期使用吗?

**A**: ❌ 不建议。兼容桥是**临时迁移工具**,刻意标记为 deprecated。

**原因**:
- 有性能开销(额外 InheritedWidget 层)
- 无法解决类型签名不匹配
- 随着生态迁移,会成为技术债

**建议**:
- 记录技术债,标注移除条件(如 "等待 package_x v2.0 发布")
- 定期检查依赖更新,第一时间移除桥

---

### Q3: 迁移后第三方包报错 "Type 'XXX' is not a subtype of type 'XXX'"?

**A**: 这是**类型签名不匹配**,兼容桥无法解决。

**示例**:
```dart
// 第三方包 API
void setTheme(flutter_material.ColorScheme scheme); // 要求旧类型

// 你的代码
setTheme(Theme.of(context).colorScheme); // 传入新类型
// ↑ 编译错误:两个 ColorScheme 是不同库的,Dart 视为不同类型
```

**解决方案**:
1. **等待依赖更新**:联系包作者或提交 PR,建议升级到 material_ui。
2. **降级依赖**(临时):若包有旧版本兼容旧设计库,暂时降级(不推荐长期)。
3. **Fork 修复**(短期):Fork 依赖包,自行迁移后用 `dependency_overrides`,待官方更新后移除。

---

### Q4: 为什么 CupertinoApp 中用 Material 组件会报错?

**A**: `CupertinoApp` 默认不提供 `MaterialLocalizations` delegate(与 `MaterialApp` 不同)。

**正确配置**:
```dart
CupertinoApp(
  // ✅ 若应用中用了 Material 组件,必须提供 MaterialLocalizations
  localizationsDelegates: GlobalMaterialLocalizations.delegates,
  // ...
);
```

`GlobalMaterialLocalizations.delegates` 包含 Material、Cupertino 和 Widgets 三个 delegates,可覆盖所有场景。

---

### Q5: gen-l10n 生成的代码仍导入 flutter_localizations 怎么办?

**A**: 先确认命令用的是项目锁定的 SDK，也确认 `pubspec.yaml` 声明了实际使用的独立包。如果重新生成后仍是旧导入（例如 Flutter 3.50.0 和 3.50.1），按 Step 6 场景 B 在应用入口手动组合 delegates，不必为此升级 SDK。未来如果升级了项目 SDK，先检查重新生成的结果，再决定是否移除手动配置。

---

### Q6: 迁移对 Web 平台有影响吗?

**A**: 无特殊影响。material_ui 和 cupertino_ui 在 Web 上与旧包行为一致。

**注意**:
- 包大小可能略有变化(独立包与框架内打包策略不同)
- 若使用 tree-shaking,确保 `flutter build web` 仍能正确剔除未使用代码

---

### Q7: 能否只迁移 Material,不迁移 Cupertino(反之亦然)?

**A**: ✅ 可以。只迁移项目实际使用的设计库。

**示例**:纯 Material 应用
- 只添加 `material_ui` 依赖
- 只迁移 `package:flutter/material.dart` 导入
- 不需要 `cupertino_ui`

若将来需要 Cupertino 组件,再添加 `cupertino_ui` 并迁移相关导入。

---

### Q8: 迁移后如何跟进独立包的周更?

**A**: 

1. **查看变更日志**:
   - material_ui: https://pub.dev/packages/material_ui/changelog
   - cupertino_ui: https://pub.dev/packages/cupertino_ui/changelog

2. **更新依赖**:
```bash
flutter pub upgrade material_ui cupertino_ui
# 或在 pubspec.yaml 中用 ^1.0.0 自动获取 minor/patch 更新
```

3. **测试验证**:
   - 运行 `flutter test` 确保无回归
   - 关注 breaking changes(major 版本更新)

4. **订阅通知**:
   - GitHub Watch `flutter/packages` 仓库
   - 关注 Flutter Release Notes

---

### Q9: 包作者如何迁移并发布?

**A**: 

1. **迁移代码**:按本工作流 Step 2-6 操作。
2. **版本号**:这是 **breaking change**,必须 bump major 版本(如 1.x.x → 2.0.0)。
3. **CHANGELOG**:说明依赖方需同步迁移到 material_ui/cupertino_ui。
4. **发布**:`flutter pub publish`

**注意**:未迁移的消费者会遇到类型不匹配错误,务必在文档中说明迁移步骤。

---

### Q10: 迁移是否影响 Flutter SDK 升级?

**A**: ❌ 不影响。迁移后仍可正常升级 Flutter SDK。

独立包与 Flutter SDK 解耦后:
- SDK 升级不再强制更新设计库
- 设计库更新不依赖 SDK 发布周期
- 两者可独立升级,更灵活

---

## 💡 最佳实践

### 1. 迁移前做好依赖审计

不要盲目迁移。先用 AI 或手动检查关键依赖是否已支持新包,避免迁移后陷入类型不匹配困境。

---

### 2. 分步迁移:先依赖包,再应用

**若维护多个相关包**:
- 先迁移底层库(不依赖其他内部包的)
- 再迁移中间层
- 最后迁移应用层

这样避免内部依赖间的类型不匹配。

---

### 3. 兼容桥是临时方案,不是架构

```dart
// ❌ 不要把兼容桥当成长期架构
// 兼容桥有性能开销,且本身是 deprecated

// ✅ 正确做法:
// 1. 记录技术债
// 2. 跟踪依赖更新
// 3. 第一时间移除桥
```

---

### 4. 预览再应用,避免误改

```bash
# ✅ 正确:先 dry-run
dart fix --dry-run --code=migrate_design_widgets > preview.txt
# 人工审查 preview.txt
dart fix --apply --code=migrate_design_widgets

# ❌ 错误:直接应用,未审查
dart fix --apply --code=migrate_design_widgets
```

---

### 5. 测试用例也要迁移

```dart
// ✅ 测试文件也需迁移导入
// test/widget_test.dart
import 'package:material_ui/material_ui.dart'; // 而非 flutter/material.dart
import 'package:flutter_test/flutter_test.dart';

testWidgets('Counter increments smoke test', (tester) async {
  await tester.pumpWidget(MaterialApp(home: MyHomePage()));
  // ...
});
```

---

### 6. CI/CD 配置检查

确保 CI 环境 Flutter 版本 >= 3.47(建议 3.50+),否则构建会因找不到 material_ui 包而失败。

```yaml
# .github/workflows/flutter.yml
- uses: subosito/flutter-action@v2
  with:
    flutter-version: '3.50.0' # 或 'stable'
```

---

### 7. 文档化技术债

```dart
// ✅ 明确记录为什么用兼容桥,何时移除
MaterialApp(
  builder: (context, child) {
    // ignore: deprecated_member_use
    // TODO(tech-debt): 移除兼容桥
    // 原因:依赖 some_package v1.x 尚未支持 material_ui
    // 移除条件:升级到 some_package v2.0+
    // 跟踪:https://github.com/author/some_package/issues/123
    return MaterialUiCompatibilityBridge(child: child!);
  },
  // ...
);
```

---

### 8. 利用 AI 但人工把关

**AI 适合做**:
- 依赖审计
- 预览变更分析
- 生成兼容桥代码

**人工必须做**:
- 最终决策是否迁移
- 审查 dart fix 输出的 diff
- 手测关键页面和多语言
- 验证构建和测试通过

---

### 9. 跟踪生态迁移进度

关键依赖包的 GitHub issues/releases,第一时间知道何时可以移除兼容桥:

- 在包的 GitHub 仓库搜索 "material_ui" 或 "cupertino_ui" 相关 issue
- 订阅 pub.dev 包更新通知
- 参与社区讨论(如 Flutter Discord/Reddit)

---

### 10. 新项目直接使用独立包

Flutter 3.50+ 新建项目默认已用 material_ui。若手动创建项目或使用旧模板,确保 `pubspec.yaml` 和导入路径使用新包。

---

## 🔗 相关资源

- **工作流**:
  - [workflows/theme-consistency.md](theme-consistency.md) — Material 3 主题体系与明暗色
  - [workflows/l10n-internationalization.md](l10n-internationalization.md) — 官方 gen-l10n 国际化
- **提示词模板**:
  - [prompts/design-system/audit-and-preview-material-ui-migration.md](../prompts/design-system/audit-and-preview-material-ui-migration.md) — 审计依赖与预览迁移
  - [prompts/design-system/apply-material-ui-migration.md](../prompts/design-system/apply-material-ui-migration.md) — 安全应用迁移与验证
- **Flutter 官方文档**:
  - [Breaking change: Migrate to material_ui and cupertino_ui](https://docs.flutter.dev/release/breaking-changes/material-ui-and-cupertino-ui)
  - [What's new in Flutter 3.50](https://flutter.dev/blog/whats-new-in-flutter-3-50)
  - [package:material_ui](https://pub.dev/packages/material_ui)
  - [package:cupertino_ui](https://pub.dev/packages/cupertino_ui)
- **相关 Issues & PRs**:
  - [Issue 191072 - gen-l10n support](https://github.com/flutter/flutter/issues/191072)
  - [PR 194500 - gen-l10n standalone UI packages support](https://github.com/flutter/flutter/pull/194500)
  - [Flutter 官方文档：本地化代码生成到源码目录](https://docs.flutter.dev/release/breaking-changes/flutter-generate-i10n-source)
  - [Issue 191448 - Breaking change tracking](https://github.com/flutter/flutter/issues/191448)

---

**拥抱解耦,跟进独立包周更 🚀**
