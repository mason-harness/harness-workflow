# Architecture

## 仓库定位

这是一个围绕 Claude Code skills 组织的单项目仓库，核心目标是通过 Harness 系列与 OpenSpec 系列技能，为 Claude 驱动的开发流程提供规则、门禁、证据与切片能力。

根目录主要由三部分组成：
- `README.md`：仓库导览与技能清单
- `repository/`：稳定项目知识
- `skills/`：各个 skill 的正式定义与参考资料

## 三层职责模型

### 1. OpenSpec 层

负责 Change lifecycle 与相关 artifacts，覆盖 proposal、design、tasks、apply、verify、archive，以及多切片场景下的计划、登记与追踪。

对应能力主要由以下技能接入：
- `openspec-slices-plan`
- `openspec-slices-register`
- `openspec-slices-track`
- `/opsx:*` 生命周期命令在知识库中只作为接线对象，不作为这里的主文档来源

### 2. Harness 层

负责在 OpenSpec 主流程之外提供纪律增强层，包括：
- `CLAUDE.md` 与根级知识库治理
- OpenSpec 配置规则投影
- apply 阶段执行纪律
- verify 阶段证据门禁

对应 skills：
- `harness-claude-setup`
- `harness-openspec-setup`
- `harness-openspec-apply`
- `harness-openspec-verify`

### 3. Hook 层

负责把高风险软规则升级成不可绕过的硬门禁。

对应 skill：
- `harness-hook-setup`

## 关系原则

- OpenSpec 负责主生命周期，Harness 负责增强，不应形成第二套并行流程。
- Hook 只用于高风险、不可逆、必须不可绕过的动作，不替代日常规则说明。
- `repository/` 负责稳定知识，具体行为细节仍应回到对应 `skills/*/SKILL.md`。

## 稳定结构边界

- `skills/harness-*`：围绕规则、门禁、证据与文档治理
- `skills/openspec-*`：围绕切片规划、登记、追踪等 OpenSpec 生命周期补充能力
- `skills/*/references/`：仅在具体 skill 需要精确契约、模板或参考资料时存在

## 证据来源

本文件中的事实主要来自：
- `README.md`
- `repository/harness-workflow.md`
- `skills/harness-claude-setup/SKILL.md`
- `skills/harness-openspec-setup/SKILL.md`
- `skills/harness-hook-setup/SKILL.md`
- `skills/harness-openspec-apply/SKILL.md`
- `skills/harness-openspec-verify/SKILL.md`
- `skills/openspec-slices-track/SKILL.md`