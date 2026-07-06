# CLAUDE.md

This file provides guidance to Claude Code when working with code in this repository.

## 项目概览

这是一个围绕 Harness 与 OpenSpec skills 组织的单项目仓库。根目录主要包含：
- `README.md`：仓库导览、技能清单、Quick Start
- `repository/`：稳定项目知识
- `skills/*/SKILL.md`：各 skill 的具体行为边界与使用时机

## Source of Truth

- `README.md` 负责人类导览与上手说明。
- `repository/` 负责稳定知识：架构、模块边界、接线关系、工作流总览。
- `skills/*/SKILL.md` 负责具体 skill 的触发条件、职责边界、失败处理与 handoff。
- OpenSpec CLI 的精确契约以 `skills/openspec-slices-register/references/cli-contract.md` 为准。

## 常用命令

已验证存在 `package.json`，当前唯一可执行来源是 `scripts.install-skills`（`npm run install-skills`）。

除安装/同步 skills 外，未发现 build、test、lint、deploy 等可执行来源，因此不要在本仓库声明这类本地命令。

在修改文档前，优先阅读：
- `README.md`
- `repository/README.md`
- 目标 skill 的 `skills/*/SKILL.md`
- 如涉及 OpenSpec CLI 契约，阅读 `skills/openspec-slices-register/references/cli-contract.md`

## 开发指南

- 先确认目标是根级工作文档、知识库，还是某个具体 skill 文档。
- 更新长期规则时，只写有证据支持的事实；证据优先级：当前仓库文件 > 已验证知识库 > README。
- 根级 `CLAUDE.md` 只写 Claude 在本仓库如何工作，不写完整项目百科。
- 稳定项目知识放进 `repository/`，不要继续堆在单个 workflow 文档里。
- 具体 skill 行为优先以对应 `skills/*/SKILL.md` 为准，不在根级文档中复制整段规则。

## Do

- 先读再写，尤其是现有 `README.md`、`repository/` 和相关 `skills/*/SKILL.md`。
- 只在文档职责清晰时增量编辑；需要重组时先保持信息可追溯。
- 用 `repository/architecture.md`、`repository/modules.md`、`repository/integrations.md` 承载稳定知识。
- 将流程总览与时机判断保留在 `repository/harness-workflow.md`。
- 提醒用户：如需把危险操作升级为不可绕过的硬门禁，可使用 `/harness-hook-setup`。

## Don't

- 不要发明本地构建、测试、lint 或部署命令。
- 不要把 README 的导览内容复制到 `CLAUDE.md` 或 `repository/`。
- 不要把 `skills/*/SKILL.md` 的完整 workflow、错误处理表逐段复制到知识库。
- 不要把一次性需求、今日任务或聊天上下文写入长期文档。
- 不要在没有证据时声明 OpenSpec CLI 能力、Hook 已配置，或某个流程“可直接执行”。

## Before Finishing（每次完成前必须检查）

- [ ] 只修改了与目标文档职责相关的文件。
- [ ] 新增或更新的事实都能回溯到 `README.md`、`repository/`、`skills/*/SKILL.md` 或已验证参考文件。
- [ ] 没有写入未验证的本地 build、test、lint、deploy 命令。
- [ ] 没有把稳定知识继续堆回单文件，也没有把 workflow 细节重复写进多个载体。
- [ ] `repository/` 中的知识文件分工清晰，`CLAUDE.md` 仍然只承载执行规则。

## Dangerous Operations（STOP）

以下情况必须停止并重新核实，而不是直接写入文档：
- 把不存在的命令写成事实。
- 在没有证据的情况下声明 OpenSpec CLI、Hook 或其他工具能力。
- 把稳定知识错误写入 `CLAUDE.md`，或把执行规则错误写入 `repository/`。
- 为了“简化结构”而删除或覆盖现有知识文件，却没有先迁出准确信息。
- 将 README、人类导览、OpenSpec 通用流程整段复制进长期知识库。

这些危险操作建议配置 Hook 硬门禁以实现模型无法绕过的拦截。使用 `/harness-hook-setup` 自动配置 Hook。

## 禁止的自我合理化（MUST NOT）

- 不得以“这类仓库通常会有”来脑补命令或能力。
- 不得以“README 提到过流程”代替“本仓库已验证存在该命令/配置”。
- 不得以“只是文档调整”跳过证据核对与职责边界检查。
- 不得以“内容正确就行”忽略文档应该落在哪个载体。
- 不得以“先写进去以后再修”把未验证事实放入长期文档。

## Notes for Claude Code

- 这是文档治理型仓库，不要默认按应用代码仓的方式寻找构建、测试或运行入口。
- 若需要更细的 skill 规则，直接读取目标 `skills/<name>/SKILL.md`，不要依赖根级摘要替代原文。