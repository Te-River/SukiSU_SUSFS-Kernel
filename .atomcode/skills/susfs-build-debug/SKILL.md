---
name: susfs-build-debug
description: Debug SUSFS/KernelSU kernel build issues — patch context, KSU compilation, BBR config, Kconfig fallback
---
# SUSFS 构建调试 Skill

帮助诊断 SukiSU_SUSFS 内核构建中的常见问题。

## 补丁兼容性问题
- 检查 `fs/proc/base.c`、`fs/namespace.c`、`fs/notify/fdinfo.c`、`fs/proc/task_mmu.c` 等文件的 SUSFS patch 上下文是否匹配当前 sublevel
- 检查 build.yml 中对应 android 版本 + kernel 版本的 `sed`/`perl` 临时调整逻辑是否需要更新

## KernelSU 编译检查
- 确认 `common/drivers/kernelsu_local/` 目录存在且非空
- 确认 `common/drivers/Makefile` 包含 `obj-$(CONFIG_KSU) += kernelsu_local/`
- 确认 `CONFIG_KSU=y` 在 defconfig 中已启用

## BBR 配置
- BBR 不走 defconfig fragment，而是通过 POST_DEFCONFIG_CMDS 写入 `.config`
- 检查 `common/build.config.gki.aarch64` 中的 `check_defconfig` 是否已被替换

## SUSFS Kconfig fallback
- 如果 patch 失败，build.yml 会自动在 `common/drivers/Kconfig.susfs` 中创建 Kconfig 声明
- 检查 `common/drivers/Kconfig` 中是否有 `source "drivers/Kconfig.susfs"`

## 常见错误
- `undefined symbol` → KernelSU 模块未编译进 vmlinux
- `.rej` 文件 → SUSFS 补丁上下文不匹配
- `module_outs` 错误 → Bazel 沙箱问题，需要 sed 追加模块列表
