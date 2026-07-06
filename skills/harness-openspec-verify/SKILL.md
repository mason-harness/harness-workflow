---
name: harness-openspec-verify
description: Use when validating an OpenSpec Change through /opsx:verify or equivalent verify flow and stronger completion evidence gates are needed - adds a hardness overlay for task-to-evidence checks, command/output/timestamp requirements, STOP/BLOCKED handling, and archive handoff without replacing the OpenSpec lifecycle.
---

# Harness OpenSpec Verify

## Overview

为 OpenSpec 的 verify 阶段提供 evidence gate 与 completion gate，而不是替代 `/opsx:verify`。
核心原则：**OpenSpec verify 判断实现是否匹配 artifacts；本 skill 判断是否有足够证据宣称“已完成”，以及是否允许进入 archive。**

## When to Use

在以下情况使用：
- 已进入 `/opsx:verify`、`openspec-verify-change` 或等价的验证阶段
- 需要核对 `tasks.md`、`verify.md`、命令结果、证据完整性
- 需要在 archive 前增加更强的完成门禁
- 发现团队容易出现“功能看起来对，但没有证据”“verify 只有结论没有命令输出”

不要用于：
- 创建或修复 `config.yaml` 规则（交给 `harness-openspec-setup`）
- 配置 archive 的硬拦截 Hook（交给 `harness-hook-setup`）
- 替代 OpenSpec 的 proposal/spec/design/tasks 生命周期
- 替代真实测试执行；本 skill 负责核对证据与完成判断，不代替实现阶段本身

## Responsibilities Boundary

| Layer | Responsibility |
|---|---|
| OpenSpec verify | 检查 Completeness / Correctness / Coherence，判断实现是否匹配 artifacts |
| harness-openspec-verify | 核对 task-to-evidence、命令/输出/时间戳、未完成项、CRITICAL gate、archive handoff |
| Hook | 在 archive 或其他高风险动作前做不可绕过拦截 |

## Quick Reference

| Stage | Required action |
|---|---|
| Verify context | 读取 proposal/specs/design/tasks/verify 相关文件 |
| Evidence audit | 对账 task 与证据，检查命令/输出/时间戳 |
| Completion gate | 证据缺失视为未完成；CRITICAL 未解决不得通过 |
| STOP/BLOCKED | 区分需人工确认与无法继续验证的状态 |
| Archive handoff | 输出是否可归档、未决风险、是否建议 Hook 硬拦截 |

## Workflow

1. **进入 OpenSpec verify 上下文**
   - 先由 `/opsx:verify` 或等价流程锁定目标 change
   - 读取 `proposal.md`、specs、`design.md`、`tasks.md`、`verify.md`（如存在）
   - 将本 skill 视为 verify evidence profile，而不是第二套 verify 生命周期

2. **执行 task-to-evidence 对账**
   - 列出所有测试/验证相关 task
   - 为每项 task 寻找对应证据
   - 证据至少包含：
     - 命令
     - 输出摘要或关键结果
     - 时间戳
     - 记录位置（通常在 `verify.md`）
   - 没有证据的 task 不能视为完成

3. **执行 completion gate**
   - `tasks.md` 若仍有未完成 checkbox，默认不得宣称验证完成
   - 证据缺失时标记为未完成或 `[EVIDENCE MISSING]`
   - 发现 CRITICAL 级别问题时，不得建议 archive
   - 不得以“已经验证过”“应该没问题”代替实际证据

4. **校对 OpenSpec 三维结果与证据结果**
   - OpenSpec verify 已给出 Completeness / Correctness / Coherence 结论时，再叠加 evidence gate
   - 若实现语义通过但证据不足，结论仍应为“不可归档”
   - 若证据齐全但存在 CRITICAL 设计/需求偏差，也不可归档

5. **输出 STOP / BLOCKED / PASS 建议**
   - **PASS**：任务完成、证据完整、无 CRITICAL，允许继续 archive
   - **STOP**：需人工确认、存在高风险例外或归档策略例外
   - **BLOCKED**：验证所需证据或状态无法补齐，当前不能继续归档

6. **交接给 archive / hook**
   - 可归档时：明确说明通过依据
   - 不可归档时：列出未完成 task、缺失证据、CRITICAL 项
   - 对需要不可绕过门禁的项目，提示 `harness-hook-setup` 为 archive 配 Hook

## Evidence Gate Rules

- 每个测试/验证 task 必须能映射到具体证据
- 证据必须包含命令、输出、时间戳
- 证据缺失视为未完成，无例外
- 不得以“已经验证过”代替实际证据
- 不得只看测试通过，而忽略任务完成度与 CRITICAL 项
- 发现 archive 前提不满足时，必须明确报告“不得归档”

## Evidence Audit

执行证据审计时，至少按以下字段对账：

| Task | Evidence location | Command | Output summary | Timestamp | Result |
|---|---|---|---|---|---|
| `<task>` | `verify.md#...` 或 `[EVIDENCE MISSING]` | `<command>` | `<关键结论行>` | `<time>` | PASS / WARNING / CRITICAL |

证据可信度按以下顺序判断：
- **高**：CI、Hook、测试报告文件等工具生成证据
- **中**：可复现命令 + 关键输出 + 时间戳的记录
- **低**：只写“已验证”“已检查”“应该通过”的自报告

防造假要求：
- 关键验证应重新运行；无法重跑时必须说明原因和风险
- 命令输出必须包含可验证结论行，例如 passed / failed / 0 error / error count
- 发现明显无法复现或疑似伪造的证据时，标记为 `[EVIDENCE FORGED]`，不得建议 archive

## Subagent Delegation

以下场景可以委托 Subagent 做只读证据核对：
- `verify.md` 很长，需要提炼证据完整性结论
- 需要跨 `tasks.md`、`verify.md`、artifacts、测试输出对账
- archive 前需要独立门禁扫描

Subagent 只负责校验并回传结论，不替代 Hook 做强拦截，也不替代真实测试执行。

## Archive Gate Output

archive 前必须给出明确结论：
- **PASS**：tasks 完成、证据完整、无 CRITICAL，可继续 archive
- **WARNING**：存在轻微缺口，原则上应先修复；如继续需说明风险
- **CRITICAL**：存在未完成 task、证据缺失、伪造证据或设计/需求偏差，不得 archive

CRITICAL 结论必须列出不得 archive 的直接依据，而不是只写“风险较高”。

## STOP / BLOCKED Handling

### STOP

触发条件：
- 用户想带着未完成 task 或 CRITICAL 例外归档
- 证据与任务状态冲突，需要人工裁决
- verify 结论将改变项目归档或发布决策

输出应包含：
- 触发原因
- 风险说明
- 需要的人工决策
- 当前不可归档的依据

### BLOCKED

触发条件：
- 无法找到必要证据，且短期无法补齐
- task 与证据映射关系不清，无法可靠判断完成状态
- 需要回到 apply 或 tasks/design 阶段修复才能继续验证

处理方式：
- 明确列出缺失证据
- 标记未完成项
- 不得输出“验证通过”

## Archive Handoff

归档前至少输出以下结论：
- 是否存在未完成 task
- 是否存在 `[EVIDENCE MISSING]`
- 是否存在 CRITICAL 问题
- 是否建议继续 `/opsx:archive`
- 是否建议使用 `harness-hook-setup` 为 archive 配硬门禁

推荐结论分级：
- **PASS**：可继续 archive
- **WARNING**：原则上应先修复，再 archive
- **CRITICAL**：不得 archive

## Common Mistakes

| Mistake | Fix |
|---|---|
| 把本 skill 当成新的 verify 生命周期 | 只把它当成 `/opsx:verify` 的证据与完成门禁层 |
| 只检查测试通过，不检查证据结构 | 必须对账命令、输出、时间戳 |
| 看到 verify.md 有结论就默认通过 | 没有证据细节仍视为不足 |
| tasks 已勾选就默认可归档 | 还要检查 CRITICAL 与 evidence completeness |
| 证据缺失时给出模糊建议 | 明确标记为未完成或不可归档 |
| 把 Hook 责任写进 verify 文本 | Hook 只由 `harness-hook-setup` 配置 |

## Minimal Draft Structure

草案阶段最小结构只需要：
- `skills/harness-openspec-verify/SKILL.md`

只有在反复使用统一证据模板、verify report 模板或 archive gate 清单时，再考虑增加 `references/`。
