# 提示词模板库

**可直接复用的 Flutter AI 协作提示词片段**

---

## 📂 目录

### Testing（测试）
- [generate-widget-test.md](testing/generate-widget-test.md) — 生成 Widget 测试用例
- [fix-flaky-widget-test.md](testing/fix-flaky-widget-test.md) — 修复不稳定测试

### L10n（国际化）
- [extract-hardcoded-strings.md](l10n/extract-hardcoded-strings.md) — 审计项目中的硬编码字符串
- [setup-gen-l10n.md](l10n/setup-gen-l10n.md) — 配置官方 gen-l10n 工具链

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
