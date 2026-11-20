# 提示词：修复 Timeline 诊断出的性能问题

**用途**：基于已确认的 Timeline 诊断结果，让 AI 生成针对性优化代码并落地改动

---

## 📋 适用场景

- 已用 AI 或人工分析 Timeline，明确了性能瓶颈
- 需要生成具体优化代码（如减少 rebuild、添加 RepaintBoundary、异步化）
- 需要对比优化前后的代码改动

---

## 📥 输入要求

在 Cursor 中准备：

1. **Timeline 诊断结果**：
   - AI 已分析的瓶颈位置（文件路径、类名、方法名）
   - 具体优化方向（如"用 const 构造"、"添加 RepaintBoundary"）

2. **目标源文件**：
   - 打开需要优化的 Widget 文件（如 `lib/screens/user_list_screen.dart`）
   - 附上相关依赖（如 ViewModel、Service）

3. **优化约束**（可选）：
   - 是否需要保持现有 API 不变
   - 是否可以引入新依赖（如 `cached_network_image`）

---

## 💬 提示词模板

```markdown
基于以下 Timeline 诊断结果，请生成优化代码。

**诊断摘要**：
[粘贴 AI 的诊断结果，或手动总结，例如：]

瓶颈：UI 线程 Build 阶段，UserListItem.build() 被调用 25 次，累计耗时 40ms
原因：
1. 过度 rebuild（父 Widget setState 触发整列表重建）
2. 同步图片解码（NetworkImage 未优化）

优化方向：
1. 用 const 构造减少 rebuild
2. 添加 RepaintBoundary 隔离重绘
3. 改用 CachedNetworkImageProvider 异步解码

---

**目标文件**：`[文件路径，如 lib/screens/user_list_screen.dart]`

**优化要求**：

1. **必须改动**：
   - [改动点 1，如：UserListItem 改用 const 构造函数]
   - [改动点 2，如：ListView.builder 的 itemBuilder 包裹 RepaintBoundary]
   - [改动点 3，如：NetworkImage 改用 CachedNetworkImageProvider + cacheWidth]

2. **保持不变**：
   - [约束 1，如：保持现有 API 签名不变]
   - [约束 2，如：不改变 UI 外观和交互逻辑]

3. **代码规范**：
   - 生成**完整的修改后代码**（不要省略 import 和其他方法）
   - 在关键改动处添加注释说明优化原因（如 `// ✅ 优化：减少 rebuild`）
   - 若需要新依赖，明确说明如何添加（如 `flutter pub add xxx`）

4. **输出格式**：
   - 修改后的完整文件代码
   - Before/After 对比表（说明关键改动）
   - 若有新依赖，列出安装命令

---

**期望输出**：

## 优化后代码

### 文件：`[文件路径]`

```dart
[完整的修改后代码，包含所有 import、类定义、方法]
```

### 关键改动说明

| Before | After | 原因 |
|--------|-------|------|
| `UserListItem(user: ...)` | `const UserListItem(...)` + `RepaintBoundary` | 减少 rebuild 范围 |
| `NetworkImage(url)` | `CachedNetworkImageProvider(..., maxWidth: 100)` | 异步解码 + 预缩放 |
| [其他改动] | [改动后] | [原因] |

### 新增依赖（如有）

```bash
flutter pub add cached_network_image
```

### 验证步骤

1. 应用代码更改
2. 运行 `flutter run --profile`
3. 复现相同场景，观察 Timeline 帧耗时
4. 预期改善：Frame 耗时从 52ms 降至 <16ms
```

---

## ✅ 验证步骤

AI 生成代码后：

1. **应用代码**：
   ```bash
   # 若有新依赖，先安装
   flutter pub add [package_name]
   
   # 复制 AI 生成的代码到文件
   ```

2. **编译检查**：
   ```bash
   # 确保无编译错误
   flutter analyze
   ```

3. **重测 Timeline**：
   ```bash
   # 重新运行 profile 模式
   flutter run --profile
   
   # 打开 DevTools Performance，复现相同场景
   # 对比优化前后的帧耗时
   ```

4. **验证清单**：
   - [ ] 代码编译通过，无语法错误
   - [ ] 功能正常，UI 外观无异常
   - [ ] Timeline 显示帧耗时明显降低
   - [ ] 无新增 bug（回归测试通过）

---

## 🛠️ 常见优化模式

### 模式 1：减少 rebuild（const 构造 + RepaintBoundary）

**Before**:
```dart
ListView.builder(
  itemBuilder: (context, index) {
    return UserListItem(user: users[index]);
  },
)
```

**After**:
```dart
ListView.builder(
  itemBuilder: (context, index) {
    // ✅ 优化：RepaintBoundary 隔离重绘
    return RepaintBoundary(
      child: UserListItem(
        user: users[index],
        key: ValueKey(users[index].id), // ✅ 优化：添加 Key
      ),
    );
  },
)

class UserListItem extends StatelessWidget {
  final User user;
  // ✅ 优化：const 构造减少 rebuild
  const UserListItem({Key? key, required this.user}) : super(key: key);
  
  @override
  Widget build(BuildContext context) { ... }
}
```

---

### 模式 2：异步图片解码

**Before**:
```dart
CircleAvatar(
  backgroundImage: NetworkImage(user.avatarUrl),
)
```

**After**:
```dart
// ✅ 优化：异步解码 + 缓存 + 预缩放
CircleAvatar(
  backgroundImage: CachedNetworkImageProvider(
    user.avatarUrl,
    cacheKey: user.id,
    maxWidth: 100, // 预缩放，减少解码耗时
  ),
)
```

**新增依赖**：
```bash
flutter pub add cached_network_image
```

---

### 模式 3：状态下沉，避免顶层 setState

**Before**:
```dart
class MyScreen extends StatefulWidget {
  @override
  _MyScreenState createState() => _MyScreenState();
}

class _MyScreenState extends State<MyScreen> {
  int counter = 0;

  @override
  Widget build(BuildContext context) {
    return Column(
      children: [
        ExpensiveWidget(), // ❌ 每次 setState 都会 rebuild
        ElevatedButton(
          onPressed: () => setState(() => counter++),
          child: Text('$counter'),
        ),
      ],
    );
  }
}
```

**After**:
```dart
class MyScreen extends StatelessWidget {
  @override
  Widget build(BuildContext context) {
    return Column(
      children: [
        ExpensiveWidget(), // ✅ 不再受 counter 影响
        _CounterButton(), // ✅ 状态下沉到局部
      ],
    );
  }
}

class _CounterButton extends StatefulWidget {
  @override
  _CounterButtonState createState() => _CounterButtonState();
}

class _CounterButtonState extends State<_CounterButton> {
  int counter = 0;

  @override
  Widget build(BuildContext context) {
    return ElevatedButton(
      onPressed: () => setState(() => counter++),
      child: Text('$counter'),
    );
  }
}
```

---

### 模式 4：ListView 虚拟化

**Before**:
```dart
// ❌ 一次性构建所有 Item
ListView(
  children: users.map((u) => UserListItem(user: u)).toList(),
)
```

**After**:
```dart
// ✅ 按需构建可见 Item
ListView.builder(
  itemCount: users.length,
  itemBuilder: (context, index) {
    return UserListItem(user: users[index]);
  },
)
```

---

### 模式 5：异步化重计算（Isolate）

**Before**:
```dart
// ❌ 同步重计算堵住 UI 线程
final result = heavyComputation(data); // 耗时 200ms
setState(() => this.result = result);
```

**After**:
```dart
// ✅ 移到后台 Isolate
final result = await compute(heavyComputation, data);
setState(() => this.result = result);
```

---

## 🛠️ 若优化后仍卡顿

使用以下提示词让 AI 深入分析：

```markdown
应用优化后重测 Timeline，仍有卡顿：

**优化前**：Frame #145 - 52ms (UI: 48ms)
**优化后**：Frame #201 - 28ms (UI: 14ms, Raster: 14ms)

改善：-46% 总耗时，但仍超过 16ms 标准。

**新的 Timeline 数据**：
[粘贴优化后的 Timeline 数据]

请分析：
1. 是否还有其他瓶颈？（如 Raster 线程成为新瓶颈）
2. 需要进一步优化哪些地方？
3. 是否需要架构层面的改动（如引入状态管理、拆分页面）？
```

---

## 🔗 相关

- 工作流：[workflows/performance-timeline.md](../../workflows/performance-timeline.md)
- 解读 Timeline：[interpret-timeline.md](interpret-timeline.md)
- Flutter Performance 最佳实践：[https://docs.flutter.dev/perf/best-practices](https://docs.flutter.dev/perf/best-practices)
