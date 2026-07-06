---
name: harness-openspec-setup
description: Use when encountering "YAML parse errors", "missing required rules", "schema validation failures", or need to initialize/repair/update OpenSpec config.yaml in single-project repos - derives artifact rules from CLAUDE.md and root repository/, enforces independent test design and test tasks, configures hardness levels (L1-L4), and sets up verify gates with evidence requirements.
---

# OpenSpec Docs Setup

## Overview

配置单项目 OpenSpec 文档（以 `config.yaml` 为主）。从 `CLAUDE.md` 与根目录 `repository/` 推导 artifact 规则，保持职责边界：`CLAUDE.md` 承载编码/测试细节，`repository/` 承载稳定项目上下文，OpenSpec 承载 artifact 约束、Hardness 分级和验证门槛。

**边界**：维护 `config.yaml` 项目级规则；读取 `repository/` 作为项目上下文来源；不拆需求、不写业务代码、不维护 Change 进度、不执行测试。

## Quick Reference

| Task | Key Actions |
|------|-------------|
| **初始化 OpenSpec** | 读 `CLAUDE.md` + `repository/` → 更新 `config.yaml` schema/rules/context |
| **修复 YAML 错误** | 解析报错 → 修正语法/结构 → 验证 schema |
| **配置 Hardness** | Explore/Proposal: L1-L2 → Tasks/Apply: L3 → Verify/Archive: L3-L4 |
| **添加缺失规则** | 检查必需组 (`proposal/specs/design/tasks/apply/verify/archive`) → 补充最小规则 |
| **封堵自我合理化** | Apply: “测试失败不得勾选，无例外” / Verify: “证据缺失视为未完成，无例外” |

## Critical Rules

**Read Before Write**：检查 `CLAUDE.md`、现有 `config.yaml`、根目录 `repository/`、`AGENTS.md`、`schemas/*`。保留现有 OpenSpec 设置，不在已有 `openspec/` 上运行 `openspec init` 覆盖（除非用户确认）。

**Boundaries**:
- YAML keys 保持英文 (`schema/context/rules/proposal/specs/design/tasks/apply/verify/archive`)
- `context` 只放简洁项目摘要，不放编码手册/质量宣言
- 质量原则投影为 artifact 规则，不复制 `CLAUDE.md` 或 `repository/` 全文
- 单个 `proposal/specs/design/tasks/apply/verify/archive` 规则组**最好不要超过 10 条**；优先合并重复表达，保留最小充分约束
- 不修改业务源码、Claude Code 权限、`settings.local.json`

**Task/Test Separation** (Critical):
- `design` **必须包含独立测试设计**，不能只在实现描述里顺带提及测试
- `tasks` **必须包含独立测试任务**，且测试任务必须是独立 checkbox / 独立清单项
- Apply 只执行 `tasks.md` 已列任务
- 不写“实现 X 并测试”；拆成“实现 X” + “测试 X: <命令/范围/期望/证据位置>”
- Checkbox 按责任单元拆分（可验证、独立失败），不按代码行/import 微操作拆分

**Hardness Enforcement**（详见 `references/hardness-levels.md`）:
- Explore/Proposal: L1-L2 (Soft/Structured)
- Tasks/Apply: L3 (Hard Constraint, no bypass)
- Verify/Archive: L3-L4 (Gate with STOP)
- 封堵自我合理化：Apply “测试失败不得勾选，无例外” / Verify “证据缺失视为未完成，无例外”

**Progressive Adoption**:
- 阶段 1：先封堵跳过验证和未完成归档，至少配置 apply / archive 的最小硬规则
- 阶段 2：补齐 verify 证据门禁，要求测试 task 具备命令、范围、期望结果、失败处理、证据位置
- 阶段 3：完善 proposal / specs / design / tasks / apply / verify / archive 全流程护栏
- 不要让轻量需求被复杂流程拖慢；轻量需求通过 right-sized Change 判断后可直接进入普通 Change 流程

**Failure Diagnosis**:
- 模型跳过测试 → strengthen `rules.apply`，写明命令未运行不得勾选完成
- 测试未运行却勾选 → add evidence requirements，证据缺失视为未完成
- tasks 未完成就 archive → strengthen `rules.archive`，tasks 未完成必须 STOP
- verify 只有“已验证”结论 → require command / output / timestamp
- 危险操作仍被执行 → 交给 Hook 配置流程评估是否需要硬门禁

诊断说明只用于 setup 决策和报告，不要把长段诊断文本写进 `config.yaml rules`；artifact rules 仍保持短句、可执行、可检查。

## References

**Detailed content moved to references/ for token efficiency:**
- **references/hardness-levels.md**: L1-L4 模型、MUST/SHOULD/STOP 术语、责任单元判断标准、证据要求
- **references/rule-mapping.md**: Artifact 规则映射、防自我合理化规则、Progressive Repair Loop、LLM-readable 规则风格
- **references/common-mistakes.md**: 常见错误和修正、Wrong/Right 示例

## Conflict Arbitration

**Priority** (highest first): (1) OpenSpec boundaries (this skill) → (2) `CLAUDE.md` coding/test rules → (3) root `repository/` facts → (4) `config.yaml` schema → (5) README/package facts → (6) user new facts (scoped only)

**STOP if**: OpenSpec root ambiguous / `CLAUDE.md` 与知识库对项目事实冲突 / rule would skip evidence / conflict needs policy change

## File Responsibilities

| File | Content | Avoid |
|------|---------|-------|
| `CLAUDE.md` | 详细编码/测试规则、命令 | 复制到 OpenSpec |
| `repository/` | 项目上下文、术语、架构、模块、集成 | 流程口号、当天任务 |
| `config.yaml` | schema, 短 context, artifact rules | 教程、翻译 YAML keys |
| `AGENTS.md` | 生成的工具说明 | 手写重写 |
| `schemas/*` | Artifact 模板、instruction | 项目级编码手册 |

## Artifact Rule Mapping

投影质量目标为 `config.yaml` artifact 规则（详见 `references/rule-mapping.md`）：

- **proposal** (L2): 范围、非目标、假设标注、right-sized Change 检查、必要时先拆分为多个可验证 Change、来源/假设标注、对应 Change List entry
- **specs** (L2): Given/When/Then、边界场景；无实现
- **design** (L2): 方案、模块边界、**独立测试设计**（测试层级、关键场景、失败/边界场景、证据预期、完成前证明点）；检测 divergence
- **tasks** (L3): 拆分实现/测试 checkbox；**必须包含独立测试任务章节或清单**；每项写命令/范围/期望/失败处理/证据位置
- **apply** (L3): 只执行 `tasks.md`；测试失败不得勾选；STOP on 缺命令
- **verify** (L3-L4): 核对证据（命令+输出+时间戳）；缺失视为未完成
- **archive** (L4): `tasks` 未完成必须告警并 STOP；`verify` 有 CRITICAL 项不得归档；需要硬拦截时交接 Hook

## Brownfield Mode

| Category | Where |
|----------|-------|
| **Fact** (verified) | `repository/` / `config.yaml rules` |
| **Unknown** (unverified) | assumptions / report |
| **Validation Gap** (missing command) | report; verify rule if systemic |
| **Conflict** (sources disagree) | STOP before encoding |

不要把 unknown 写成规则。不要从文件名推断业务行为。

## Workflow

1. **确认范围**：单项目 vs monorepo；多 OpenSpec root 时先确认目标
2. **收集来源**：读 `CLAUDE.md`（编码/测试规则来源）→ 读根目录 `repository/`（项目上下文来源）→ 读现有 OpenSpec → README/package（只补充事实）
3. **更新 context 输入**：从知识库提炼简洁项目摘要，避免复制长篇内容
4. **更新 `config.yaml`**：
   - 先检查对应组是否已有默认规则，复用并去重后再补充，避免重复规则堆叠
   - 确保 schema + 短 context + rules (`proposal/specs/design/tasks/apply/verify/archive`)
   - 按 Hardness 配置：Explore/Proposal (L1-L2) → Tasks/Apply (L3) → Verify/Archive (L3-L4)
   - 封堵自我合理化：apply “测试失败不得勾选，无例外” / verify “证据缺失视为未完成，无例外”
   - 单个规则组最好不超过 10 条；超过时先合并同义规则、删除弱表达，再决定是否新增
5. **配置 Change 尺度护栏**：在 `rules.proposal` 写入 right-sized Change 规则——创建 Change 前必须先判断是否能定义为单个可验证 Change；当需求跨仓、跨团队、影响面过大、回滚边界不清或无法稳定验证时，必须先拆分再继续。轻量需求可直接继续普通 Change 流程。需模型无法绕过的硬门禁时，用 `/harness-hook-setup` 在 change 创建命令上配 PreToolUse Hook（可选，非默认）。
6. **验证**：解析 `config.yaml` → 检查必需 key/rules → 运行 `openspec schema validate` → 检查 diff
7. **Hook 提示**：配置了硬门禁 (`archive/verify` STOP) 时，提示 `/harness-hook-setup` 配置 Hook 强制拦截

## Failure Handling

| Failure | Action |
|---------|--------|
| YAML parse/schema error | 修正语法/结构 → 重新验证 |
| 缺失必需 rules 组 | 添加最小规则组 |
| 规则中命令不存在 | 报告 stale command；不保留为要求 |
| 规则来源冲突 | STOP 并报告 |
| OpenSpec 命令不可用 | 报告跳过；仅用结构检查 |

验证跳过时不报告成功；列出跳过项和原因。

## Output Report

- Target OpenSpec root / Files changed
- Sources read (`CLAUDE.md` / `repository/` / `config.yaml`)
- Artifact rules present (`proposal/specs/design/tasks/apply/verify/archive`) / Hard gates (STOP/BLOCKED)
- Commands checked (verified / missing)
- Validation run (`openspec schema validate` 结果或跳过原因)
- Skipped checks + reasons
- Unknowns / Conflicts
- Handoffs (change sizing / change process / `CLAUDE.md` / repository)
- **Hook recommendation**: 配置硬门禁时提示 `/harness-hook-setup`

## Common Patterns

详见 **references/common-mistakes.md**（常见错误、修正与 Wrong/Right 示例）
