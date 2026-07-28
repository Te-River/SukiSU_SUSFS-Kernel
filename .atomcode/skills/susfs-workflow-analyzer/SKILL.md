---
name: susfs-workflow-analyzer
description: Analyze SUSFS build failures — collect .rej files, check build logs, suggest fixes
subagent_type: reviewer
model: auto
tools: [Read, Grep, Glob, Bash]
---
# SUSFS 构建失败分析 Subagent

当 SukiSU_SUSFS 内核构建失败时，自动分析原因。

## 分析流程

1. **收集补丁冲突** — 查找所有 `.rej` 文件及其对应的原始文件
   ```bash
   find . -name '*.rej' 2>/dev/null
   ```

2. **检查链接错误** — 搜索构建日志中的 undefined symbol
   ```bash
   grep -r 'undefined reference' build.log 2>/dev/null
   ```

3. **检查 KernelSU 编译状态**
   - 确认 `common/drivers/kernelsu_local/` 存在
   - 确认 `common/drivers/Makefile` 包含 kernelsu_local 引用

4. **检查 SUSFS Kconfig**
   - 检查 `common/drivers/Kconfig.susfs` 是否存在（patch fallback 产物）

5. **生成修复建议**
   - 对 `.rej` 文件列出上下文差异行号
   - 建议对应 android 版本 + kernel 版本 + sublevel 的兼容调整

## 输出格式
- 冲突文件列表及行号
- 根因归类（patch 冲突 / 链接错误 / 配置遗漏）
- 具体的修复命令或 sed/perl 调整
