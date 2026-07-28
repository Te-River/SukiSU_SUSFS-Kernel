---
name: gh-workflow-validator
description: Validate GitHub Actions workflow YAML syntax and check for common SukiSU build config mistakes
user_invocable: true
disable_model_invocation: false
---
# GitHub Actions Workflow 验证器

对 `.github/workflows/` 下的 YAML 文件进行语法和逻辑检查。

## 检查项

1. **YAML 语法** — 使用 `python3 -c "import yaml; yaml.safe_load(open(...))"` 验证
2. **workflow_call 输入一致性** — 检查 caller（`kernel-a*.yml`）传参是否与 callee（`build.yml`）的 `on.workflow_call.inputs` 定义一致
3. **ksu_variant 选项** — 检查变体名称是否在 `config/` 下有对应的 `SukiSU(*).config` 文件（如果使用了带括号的变体名）
4. **env 引用** — 检查 `${{ env.* }}` 是否在使用前已被定义
5. **GITHUB_ENV 写入** — 检查 `>> $GITHUB_ENV` 的变量名在后续步骤中是否一致使用
6. **disk_cache 路径** — 必须为 `/home/runner/.cache/bazel`（GitHub Actions 专用）
7. **build_time 格式** — 必须符合 `%a %b %d %T UTC %Y`，且星期与日期真实对应
