# AGENTS.md

This file provides guidance to Qoder (qoder.com) when working with code in this repository.

## 项目概述

自动化构建集成 KernelSU + SUSFS 的 Android GKI 内核。纯 GitHub Actions 驱动，**不支持本地构建内核**。支持 5 个内核版本线：5.10 / 5.15 / 6.1 / 6.6 / 6.12，对应 Android 12~16。

## 开发命令

### Web 文档站（`web/` 目录）

```bash
cd web
npm ci                          # 安装依赖
npx webpack --mode development --watch  # 开发模式（热重载）
npx webpack --mode production   # 生产构建 → web/dist/
```

入口：`web/js/main.js`（导入 SCSS + 初始化各模块）→ 输出 `dist/app.bundle.js` + `dist/style.bundle.css`。

### 数据抓取脚本（`scripts/` 目录）

```bash
cd scripts
python fetch_all.py       # 全量抓取所有版本 sublevel 数据 → data/
python update_data.py     # 增量更新（仅抓取缺失月份），exit 0=有变更 / 2=无变更 / 1=错误
python update_lts.py      # 仅更新各版本 LTS sublevel
```

依赖 Python 3.10+，无第三方包。数据源：`android.googlesource.com` 的 Makefile（base64 编码）。

### 安全补丁脚本

```bash
# 必须从 kernel_platform/common 目录执行
bash /path/to/security_patch/apply_cve_2026_43499.sh <kernel-version> <sublevel> [patch-dir]
# 例: bash security_patch/apply_cve_2026_43499.sh 6.1 158
```

### ZRAM NEON 补丁

```bash
# 从内核源码 common/ 目录执行
bash /path/to/zram/apply_lz4_neon.sh
```

## 构建系统架构

### 工作流层次

```
main.yml（统一手动入口，调度所有版本）
├── get-manager.yml（下载 KSU Manager APK + SUSFS 模块）
├── kernel-a12-5-10.yml ─┐
├── kernel-a13-5-15.yml  │
├── kernel-a14-6-1.yml   ├── 各版本入口（workflow_dispatch + workflow_call）
├── kernel-a15-6-6.yml   │   ↓ 调用
├── kernel-a16-6-12.yml ─┘   build.yml（~1633 行可复用核心流水线）
├── collect-rejects（汇总 .rej 补丁冲突文件）
└── release（创建 GitHub Release，仅原仓库）

独立入口：
├── kernel-a14-6-1-138.yml（仅 6.1.138，默认 SukiSU(40796)）
├── kernel-custom.yml（任意 sub_level）
├── Auto_Trigger.yml（每 3 天检测 SukiSU-Ultra 更新，仅限 zzh20188/elysias123）
├── emergency-cleanup.yml（紧急清理 Release）
└── update-pages.yml（数据更新 + GitHub Pages 部署）
```

### 构建流水线（`build.yml`）

```
checkout → 清理磁盘 → 配置环境 → ccache → 下载工具链
→ 生成签名密钥 → 克隆 AnyKernel3/SUSFS/补丁仓库
→ repo init/sync 内核源码 → Stock Config 伪装
→ 应用 CVE 补丁 → 添加 KernelSU（6 种变体可选）
→ 复制到 common/drivers/kernelsu_local 参与 Bazel 编译
→ 配置 SukiSU 版本标识 → 应用 SUSFS 补丁
→ Droidspaces（可选）→ 配置内核选项（KSU/BBG/ZRAM/BBR/Re-Kernel）
→ 伪装内核版本和时间 → Bazel 编译（diff fragment 模式）
→ 准备 boot 镜像 → 打包 AnyKernel3 → 上传产物
```

### 数据管道（Web 仪表盘）

```
scripts/update_data.py → data/{androidXX}/{ver}.json
                              ↓ (update-pages.yml 每 3 天 / push 触发)
                         cp data/ → web/data/
                              ↓
                         webpack build → GitHub Pages 部署
```

JSON 结构：`{ android_version, kernel_version, lts, entries: [{date, kernel}] }`

## 核心机制

### KernelSU 变体选择
`ksu_variant` 输入控制源码来源；SukiSU(40796) 已设为默认（6.1/6.6/6.12 入口）：
- **SukiSU** — `SukiSU-Ultra` 仓库 `builtin` 分支，`setup.sh -s builtin`
- **SukiSU(40796)** — 固定 commit `70a4ba3a`，KSU_VERSION 硬编码 40796
- **SukiSU(40726)/(40548)** — 老版固定提交，通过 `config/SukiSU(*).config` 指定
- **ReSukiSU** — `ReSukiSU` 仓库
- **Official** — `tiann/KernelSU` 官方版
- **Next** — `KernelSU-Next`（`dev_susfs` 分支，当前已注释禁用）

### SUSFS 来源
- **SukiSU 变体**：优先 `ShirkNeko/susfs4ksu`（GitHub），回退 `simonpunk/susfs4ksu`（GitLab）
- **其他变体**：直接 `simonpunk/susfs4ksu`（GitLab）
- 老版固定变体额外从 SukiSU(*).config 读取 SUSFS 固定 commit
- 补丁使用 `patch -p1 || true` 容错模式；patch 失败后自动 fallback 创建 SUSFS Kconfig 声明

### 内核配置方式
通过 `diff "$DEFCONFIG.orig" "$DEFCONFIG" | grep '^>'` 提取修改生成 fragment，传给 Bazel `--defconfig_fragment`。原始 defconfig 备份在 `gki_defconfig.orig`。

### Bazel 编译要点
- `BUILD_SYSTEM_DLKM=0` + 移除 `MODULES_ORDER` + `KMI_SYMBOL_LIST_STRICT_MODE`
- 6.12 使用 `--lto=none`（thin LTO 不兼容），其他用 `--lto=thin`
- 自动向 `module_outs` 追加 tcp_htcp/tcp_bic/tcp_westwood 模块
- 使用 `nick-fields/retry` 重试（仅超时触发，最多 3 次，30 分钟超时）

## 关键目录

| 路径 | 用途 |
|------|------|
| `.github/workflows/build.yml` | 核心构建流水线（~1633 行） |
| `.github/workflows/main.yml` | 统一手动触发入口（648 行），调度所有版本构建 + Release |
| `config/config` | 自定义 commit 配置（`custom=true` 启用） |
| `config/SukiSU(*).config` | 老版 SukiSU 固定提交配置 |
| `config/stock_defconfig` | 伪装 `/proc/config.gz` 的参考配置（存在即自动应用） |
| `config/zram.config` | ZRAM 压缩算法配置片段 |
| `scripts/` | Python 数据抓取脚本（gki_fetch.py 为公共库） |
| `data/` | 各版本每月内核 sublevel 数据（JSON），供 Web 仪表盘使用 |
| `zram/` | 华为 LZ4KD ZRAM 算法源码 + NEON 加速补丁脚本 |
| `security_patch/` | CVE-2026-43499/CVE-2026-53163 GhostLock 安全补丁 |
| `web/` | 文档网站（Webpack + SCSS，部署到 GitHub Pages） |
| `hmbird_patch.c` | OPLUS HMBird 内核类型伪装补丁（OGKI → GKI） |

## 非直观陷阱

1. **Stock Config 伪装**：替换 `kernel/Makefile` 中 `$(obj)/config_data: $(KCONFIG_CONFIG) FORCE` 规则为 stock_defconfig 路径。如果 `.config` 规则不是预期格式会报错退出。

2. **SUSFS 补丁 sublevel 兼容**：不同 sublevel 的补丁上下文差异巨大。build.yml 内有大量 `sed`/`perl` 临时调整逻辑，覆盖 android12/13/14/15/16 的五种内核版本。新增 sublevel 时检查 `fs/proc/base.c` / `fs/namespace.c` / `fs/notify/fdinfo.c` / `fs/proc/task_mmu.c` 等文件的上下文是否匹配。

3. **Bazel 缓存路径**：`--disk_cache=/home/runner/.cache/bazel` 是 GitHub Actions 专用路径。

4. **KernelSU 源码编译**：KernelSU 的 `kernel/Makefile` 是外置模块格式（`make -C ... M=... modules`），对 Bazel 不适用。当前方案：`cp -r KernelSU/kernel → common/drivers/kernelsu_local`，然后 `obj-$(CONFIG_KSU) += kernelsu_local/` 追加到 `common/drivers/Makefile`。**头文件路径**：KernelSU 源码中 `#include "uapi/xxx.h"` 需要以 kernelsu_local 根为基准查找，必须在复制后往 `Kbuild` 第 1 行插入 `ccflags-y += -I$(src)`（`build.config` 中的 `KBUILD_CPPFLAGS` 在 Bazel 构建中不传递）。

5. **BBR 配置绕过**：不能用 defconfig fragment 加 `CONFIG_TCP_CONG_ADVANCED=y`（触发额外 TCP 模块编译 / module_outs 报错）。改用 POST_DEFCONFIG_CMDS 在 olddefconfig 之后写入 `.config`，绕过 Bazel fragment 校验。

6. **Bazel module_outs 沙箱**：`common/BUILD.bazel` 的 `module_outs` 必须在编译前列出所有 =m 模块。Bazel 沙箱重置文件修改，故不能用 sed 在 build 步骤中修改 BUILD.bazel。目前编译步骤中会先 sed 追加 tcp_* 模块到 module_outs（沙箱外操作，实际有效）。

7. **KSU_VERSION 固定**：SukiSU(40796) 通过 `sed -i '/^GITHUB_COMMITS/c\GITHUB_COMMITS := 3611' KernelSU/kernel/Kbuild` 固定，避免 git commit 数计算偏差。SukiSU(40726) 同理固定 GITHUB_COMMITS=3541。

8. **get-manager.yml 新仓库处理**：首次构建无历史 artifacts 时跳过（`exit 0`，非 `exit 1`）。管理器 APK 可在 SukiSU-Ultra Releases 手动下载。

9. **build_time 格式严格**：必须符合 `%a %b %d %T UTC %Y`（如 `Sun Dec 01 08:10:00 UTC 2024`），且星期与日期必须真实对应。格式错误或逻辑无效时直接 exit 1。

10. **Re-Kernel 驱动**：编译时需要修正头文件路径（`../android/binder_internal.h` → 相对路径），5.10 内核需补充 `#include <linux/seq_file.h>` 用于 `DEFINE_SHOW_ATTRIBUTE`。

11. **config/zram.config**：启用 ZRAM 时追加到 defconfig（`CONFIG_CRYPTO_LZ4K*` / `CONFIG_CRYPTO_842`）。zram/ 目录下的 LZ4KD 补丁仅支持 6.1 内核，版本不匹配时 `|| true` 忽略。

12. **补丁冲突收集**：构建结束后自动收集所有 `.rej` + 对应原文件，打包上传为 artifact（`*-Rejects`），main.yml 汇总为 `AIO-REJ.zip`，排查失败时直接下载。

13. **Auto_Trigger 限制**：仅仓库所有者为 `zzh20188` 或 `elysias123` 时运行自动检测/构建。使用 `sha` 分支存储上次构建的 commit hash 进行比较。

14. **SUSFS Kconfig fallback**：SUSFS patch 失败未声明 Kconfig 时，build.yml 自动在 `common/drivers/` 下创建 `Kconfig.susfs` 并 source 到主 Kconfig，包含所有 SUSFS 子选项。

15. **AnyKernel3 来源**：使用 `WildKernels/AnyKernel3`（fork 版，`gki-2.0` 分支），非 Osmosis 原版。

16. **AGENTS.md 被 .gitignore 排除**：本文件在 `.gitignore` 列表中，修改后需要 `git add -f AGENTS.md` 才能提交。

17. **CVE 补丁幂等性**：`apply_cve_2026_43499.sh` 通过 grep 检测已修复标记（`struct task_struct *waiter_task`），已修复则跳过主补丁但仍执行后续修正（CVE-2026-53163 proxy cleanup + remove_waiter NULL guard）。sublevel 阈值：6.1 < 177、6.6 < 144、6.12 < 95 才需要打补丁。

18. **update-pages.yml 部署条件**：Pages 部署仅限 `zzh20188/GKI_KernelSU_SUSFS` 原仓库。数据提交到 `dev` 分支后自动触发部署。
