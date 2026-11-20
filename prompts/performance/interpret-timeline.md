# 提示词：解读 Timeline 性能数据

**用途**：让 AI 分析 Flutter DevTools Timeline/Performance 数据，识别性能瓶颈并给出针对性诊断

---

## 📋 适用场景

- 应用出现卡顿、掉帧、动画不流畅
- Timeline 显示部分帧耗时超过 16ms（60fps 标准）
- 需要定位 UI/Raster 线程瓶颈
- 不清楚 Timeline 数据如何解读

---

## 📥 输入要求

在 Cursor 中准备：

1. **Timeline 数据**（以下任一方式）：
   - DevTools Performance 面板的截图（含 Frame Chart 和 Summary）
   - 手动记录的关键指标（帧号、UI/Raster 耗时、Build/Layout/Paint 分布）
   - 导出的 Timeline JSON 文件摘要（可选）

2. **运行模式确认**：
   - ✅ 必须是 `flutter run --profile` 或 `--release`
   - ❌ Debug 模式数据无参考价值

3. **问题描述**：
   - 具体卡顿场景（如滚动列表、播放动画、切换页面）
   - 复现步骤（可选）

---

## 💬 提示词模板

```markdown
我用 Flutter DevTools Performance 采集了以下 Timeline 数据，需要分析性能瓶颈。

**运行模式**：[profile / release]

**问题描述**：[具体症状，如：列表滚动时出现明显卡顿，部分帧耗时超过 40ms]

**复现场景**：[可选，如：打开用户列表页 → 快速向下滚动 → 出现卡顿]

**Timeline 数据**：

[方式一：粘贴截图链接]
[截图链接：https://...]

[方式二：手动记录关键指标]
Frame #[帧号] (卡顿帧):
- Total time: [X]ms
- UI thread: [X]ms
  - Build: [X]ms
    - [具体 Widget/方法]: [X] 次调用
  - Layout: [X]ms
  - Paint: [X]ms
- Raster thread: [X]ms
  - [具体操作，如 Shader 编译、图层合成]

Frame #[帧号]~[帧号] (正常帧):
- Average total: [X]ms

[方式三：粘贴 JSON 摘要片段]
{
  "frame": 145,
  "elapsed": 52000,
  "ui": 48000,
  "raster": 4000,
  ...
}

---

**分析要求**：

1. **判断主要瓶颈**：
   - UI 线程问题 or Raster 线程问题？
   - 哪个阶段耗时最多（Build / Layout / Paint / GPU 渲染）？

2. **识别具体原因**（选择适用的）：
   - ⚠️ 过度 rebuild（Widget 被不必要地多次构建）
   - ⚠️ 深层嵌套（Widget 树层级过深）
   - ⚠️ 同步 I/O 或重计算（堵住主线程）
   - ⚠️ 列表未虚拟化（非 ListView.builder）
   - ⚠️ 图片同步解码（NetworkImage 未优化）
   - ⚠️ 缺少 RepaintBoundary（整屏重绘）
   - ⚠️ 过度图层合成（大量 Opacity、ClipRRect、Transform）
   - ⚠️ Shader 编译卡顿（首次渲染）
   - ⚠️ 其他（请明确说明）

3. **量化影响**：
   - 哪些 Widget 或操作占用耗时最多？（如 `UserListItem.build() x 25 次，累计 40ms`）
   - 占总耗时的百分比？

4. **提出针对性建议**（不要空泛建议）：
   - 明确需要修改的**文件路径**和**类名/方法名**
   - 给出具体优化方向（如"用 const 构造函数"、"添加 RepaintBoundary"、"改用 ListView.builder"）
   - 预期改善效果（如"Build 耗时预计从 40ms 降至 10ms"）

---

**输出格式**：

## 诊断结果

### 主要瓶颈：[UI 线程 / Raster 线程] - [具体阶段]

**问题**：[一句话总结，如"UserListItem.build() 在单帧内被调用 25 次，累计耗时 40ms"]

**根本原因**：
1. [原因一：如过度 rebuild]
2. [原因二：如同步图片解码]

### 具体优化方向

#### 1. [优化点一]（优先级：高/中/低）
- **位置**：[文件路径，如 `lib/screens/user_list_screen.dart`]
- **问题**：[具体描述]
- **方案**：
  - [具体改动 1]
  - [具体改动 2]

#### 2. [优化点二]（优先级：高/中/低）
- **位置**：[文件路径]
- **问题**：[具体描述]
- **方案**：
  - [具体改动]

### 预期改善
- [指标名称]：[优化前] → [优化后]（[百分比]改善）
- 目标帧耗时：[X]ms → <16ms（达到 60fps）
```

---

## ✅ 验证步骤

AI 输出后检查：

- [ ] 诊断明确指出**代码位置**（文件路径 + 类名/方法名）
- [ ] 给出**技术原因**（不是"性能不好"等空泛描述）
- [ ] 提供**可落地的改动**（具体到代码级别）
- [ ] 有**预期改善效果**的量化（如耗时从 40ms 降至 10ms）

**若 AI 输出过于空泛**，追问：

```markdown
请进一步明确：

1. 需要修改的具体文件路径和类名
2. 哪个 Widget 或方法导致卡顿（调用次数、单次耗时）
3. 给出 Before/After 代码对比示例
4. 预期改善的耗时数值（如 40ms → 10ms）
```

---

## 🛠️ 常见分析场景

### 场景 1：UI 线程 Build 阶段卡顿

**输入示例**：
```
Frame #145:
- Total time: 52ms
- UI thread: 48ms
  - Build: 40ms (UserListItem.build() x 25)
  - Layout: 6ms
  - Paint: 2ms
- Raster thread: 4ms
```

**AI 应识别**：
- 过度 rebuild（25 次调用）
- 建议用 const 构造 + RepaintBoundary
- 预期 Build 耗时降至 10ms

---

### 场景 2：Raster 线程卡顿（GPU 瓶颈）

**输入示例**：
```
Frame #88:
- Total time: 28ms
- UI thread: 10ms
- Raster thread: 18ms
  - Layer compositing: 15ms
  - Shader compilation: 3ms
```

**AI 应识别**：
- 过度图层合成（大量 Opacity、ClipRRect）
- 建议减少图层或添加 RepaintBoundary
- Shader 编译可用 SkSL warmup 预热

---

### 场景 3：首次加载慢

**输入示例**：
```
Frame #1:
- Total time: 1200ms
- UI thread: 950ms
  - Build: 920ms (深层嵌套，200+ Widget)
  - Layout: 25ms
  - Paint: 5ms
- Raster thread: 250ms
```

**AI 应识别**：
- Widget 树层级过深
- 建议拆分组件、懒加载 Tab 内容
- 同步数据加载应移到 FutureBuilder

---

## 🔗 相关

- 工作流：[workflows/performance-timeline.md](../../workflows/performance-timeline.md)
- 修复卡顿：[fix-jank-from-timeline.md](fix-jank-from-timeline.md)
- Flutter Performance 文档：[https://docs.flutter.dev/perf](https://docs.flutter.dev/perf)
