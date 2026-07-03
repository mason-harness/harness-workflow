# Integrations

## 与 `/opsx:*` 生命周期的接线

本仓库将 OpenSpec 视为 Change lifecycle 的主线，而 Harness skills 作为附着在关键阶段的增强层。

典型接线方式：
- `/opsx:apply` 之后接 `harness-openspec-apply`
- `/opsx:verify` 之后接 `harness-openspec-verify`
- 需求过大时，先进入 `openspec-slices-plan` / `openspec-slices-register` / `openspec-slices-track`

这里记录的是接线关系，不重复 OpenSpec artifacts 的完整规则，也不替代对应 skill 的详细说明。

## 与 OpenSpec CLI 的边界

OpenSpec CLI 相关精确契约应以：
- `skills/openspec-slices-register/references/cli-contract.md`

为准。该契约文件记录了 `context-store`、`initiative`、`new change`、`set change` 等命令的精确语法、限制与行为边界。

在本仓库的长期知识中，只保留以下稳定结论：
- CLI 契约需要以明确参考文件为准，不能凭印象写入工作文档。
- `openspec-slices-register` 依赖这份契约来约束 change 登记行为。
- 任何具体命令语法更新时，应优先更新契约参考文件，而不是在多个知识文档中散落副本。

## 与 Hook 的接线

当 `CLAUDE.md` 或 OpenSpec 规则中存在“必须硬门禁”的危险操作时，由 `harness-hook-setup` 负责将软 STOP 升级为 Claude Code Hooks。

稳定边界：
- `CLAUDE.md`：标记危险操作与 STOP 条件
- OpenSpec 配置：标记 verify / archive 等阶段的 gate
- `harness-hook-setup`：把这些 gate 落地为 PreToolUse Hook

因此，知识库中只记录“谁负责什么”，不复制 Hook 脚本实现细节。

## 与知识载体的接线

- README：导览和人类上手
- `CLAUDE.md`：Claude 在本仓库的执行规则
- `repository/`：稳定知识
- `skills/*/SKILL.md`：各 skill 的行为细节

更新任一载体时，应先判断信息属于哪一层，再决定写入位置。

## 不应写在这里的内容

- 完整的 OpenSpec apply / verify 纪律
- Hook 脚本代码与正则细节
- README 风格的教程
- 未验证的本地命令或系统配置状态