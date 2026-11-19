# 提示词模板库

**可直接复用的 Flutter AI 协作提示词片段**

---

## 📂 目录

### Testing（测试）

- [generate-widget-test.md](testing/generate-widget-test.md) — 生成 Widget 测试用例
- [fix-flaky-widget-test.md](testing/fix-flaky-widget-test.md) — 修复不稳定测试
- [generate-golden-test.md](testing/generate-golden-test.md) — 生成 Golden（UI 快照）测试
- [fix-golden-diff.md](testing/fix-golden-diff.md) — 修复 Golden 测试快照差异
- [generate-patrol-test.md](testing/generate-patrol-test.md) — 生成 Patrol 端到端测试
- [fix-patrol-flaky.md](testing/fix-patrol-flaky.md) — 修复不稳定的 Patrol 测试
---

### State（状态管理）

- [refactor-to-riverpod.md](state/refactor-to-riverpod.md) — 将现有代码重构到 Riverpod
- [generate-riverpod-provider.md](state/generate-riverpod-provider.md) — 为新功能生成 Riverpod Provider

### Navigation（路由）

- [migrate-to-gorouter.md](navigation/migrate-to-gorouter.md) — 迁移到 go_router
- [add-typed-route-and-features.md](navigation/add-typed-route-and-features.md) — 添加类型安全路由与功能
---

### Flavor（多环境配置）

- [generate-flavor-config.md](flavor/generate-flavor-config.md) — 从零生成 Flavor 配置
- [debug-flavor-build-error.md](flavor/debug-flavor-build-error.md) — 排查 Flavor 构建错误
---

### Performance（性能）

- [interpret-timeline.md](performance/interpret-timeline.md) — 解读 DevTools Timeline 性能数据
- [fix-jank-from-timeline.md](performance/fix-jank-from-timeline.md) — 修复 Timeline 诊断出的卡顿问题
---

### Accessibility（无障碍）

- [add-semantics-labels.md](accessibility/add-semantics-labels.md) — 补充 Semantics 语义标签
- [generate-semantics-test.md](accessibility/generate-semantics-test.md) — 生成语义自动化测试
---

### Theming（主题）

- [audit-hardcoded-styles.md](theming/audit-hardcoded-styles.md) — 审计硬编码颜色、字体、间距
- [implement-theme-system.md](theming/implement-theme-system.md) — 落地 Material 3 主题体系与明暗色支持
---

### L10n（国际化）

- [extract-hardcoded-strings.md](l10n/extract-hardcoded-strings.md) — 审计项目中的硬编码字符串
- [setup-gen-l10n.md](l10n/setup-gen-l10n.md) — 配置官方 gen-l10n 工具链
---

### Networking（网络）

- [generate-dio-client.md](networking/generate-dio-client.md) — 生成 Dio API 客户端
- [unify-api-error-handling.md](networking/unify-api-error-handling.md) — 统一 API 错误处理
---

### Codegen（代码生成）

- [generate-freezed-model.md](codegen/generate-freezed-model.md) — 生成 freezed 数据模型
- [fix-build-runner-errors.md](codegen/fix-build-runner-errors.md) — 修复 build_runner 生成错误
---

### Database（数据库）

- [generate-drift-schema.md](database/generate-drift-schema.md) — 生成 Drift 表结构与 DAO
- [add-drift-migration.md](database/add-drift-migration.md) — 添加 Schema 迁移脚本
---

### DI（依赖注入）

- [setup-get-it-injectable.md](di/setup-get-it-injectable.md) — 搭建 get_it + injectable 架构
- [register-or-fix-injectable.md](di/register-or-fix-injectable.md) — 注册新服务或修复 injectable 错误
---

## 🎯 使用方法

1. **选择场景**：找到匹配你需求的提示词文件
2. **复制模板**：按模板替换 `[占位符]` 为实际内容
3. **附上上下文**：在 Cursor 中打开相关源文件
4. **验证输出**：运行 `flutter test` 或其他验证命令

---

## 🤝 贡献新提示词

提交 PR 时请包含：

1. **文件名**：清晰描述用途（如 `refactor-to-riverpod.md`）
2. **结构**：
   - 适用场景
   - 输入要求（上下文、参数）
   - 提示词模板（带占位符）
   - 验证步骤
   - 常见问题
3. **实测验证**：确保在真实项目中可用

---

**持续更新中 🚀**
