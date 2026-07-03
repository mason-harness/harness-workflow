# Modules

## Harness 系列

### `harness-claude-setup`

职责：维护单项目 `CLAUDE.md` 与根目录 `repository/`，分离执行规则与稳定项目知识，防止多项目混淆、业务需求混入和未验证命令进入长期文档。

边界：
- 负责文档治理与知识库结构
- 不负责 OpenSpec artifact 配置
- 不负责 Hook 配置
- 不负责业务实现

### `harness-openspec-setup`

职责：维护 `openspec/config.yaml` 项目级规则，把 `CLAUDE.md` 与 `repository/` 中的稳定规则投影为 OpenSpec artifact 约束与 Hardness 分级。

边界：
- 负责 OpenSpec 配置与规则映射
- 不负责需求拆分、业务代码、进度追踪
- 不替代 apply / verify 阶段执行

### `harness-hook-setup`

职责：把 `CLAUDE.md` 与 OpenSpec 中标注为硬门禁的危险操作落地为 Claude Code Hooks（PreToolUse）。

边界：
- 负责 Hook 配置、拦截脚本、覆盖检查
- 不修改业务规则来源本身
- 不替代 skill 文本流程

### `harness-openspec-apply`

职责：作为 `/opsx:apply` 的纪律增强层，约束 task preflight、一次只推进一个 checkbox、先验证再勾选、异常时 STOP / BLOCKED。

边界：
- 负责 apply 阶段的执行纪律
- 不创建或修复 OpenSpec 配置
- 不替代 change 选择或 proposal/design/tasks 生命周期

### `harness-openspec-verify`

职责：作为 `/opsx:verify` 的证据门禁与完成门禁层，对账 tasks、命令、输出、时间戳，并决定是否允许继续 archive。

边界：
- 负责验证证据与 archive handoff
- 不替代真实测试执行
- 不负责编写 Hook 或重写 OpenSpec 生命周期

## OpenSpec 切片系列

### `openspec-slices-plan`

职责：在需求过大时进行切片决策，按可交付、可验证、可回滚的边界拆分 change。

### `openspec-slices-register`

职责：把已确认的 Slice Plan 登记为 OpenSpec changes，并在需要时依据 CLI 契约维护关联信息。

### `openspec-slices-track`

职责：对多切片计划做跨 session 进度追踪，根据依赖关系和实时状态给出下一步建议。

## 模块关系

- `harness-claude-setup` 是根级工作文档与知识库入口治理模块。
- `harness-openspec-setup` 消费 `CLAUDE.md` 与 `repository/`，把规则下沉到 OpenSpec 配置。
- `harness-hook-setup` 消费 `CLAUDE.md` / OpenSpec 中需要硬门禁的规则，配置不可绕过拦截。
- `harness-openspec-apply` 与 `harness-openspec-verify` 分别附着在 `/opsx:apply` 与 `/opsx:verify` 周围，不形成第二套生命周期。
- `openspec-slices-plan/register/track` 解决“大需求如何拆、如何登记、如何持续追踪”的规模问题。

## 不在这里展开的内容

以下内容应继续以原始来源为准，而不是在本文件中复制：
- 各 skill 的完整 workflow
- STOP / BLOCKED 的逐条细则
- 失败处理表、输出模版、精确 CLI 语法
- README 的技能导览与 Quick Start