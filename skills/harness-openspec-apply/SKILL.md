---
name: harness-openspec-apply
description: Use when implementing an OpenSpec Change through /opsx:apply or equivalent apply flow and stronger execution discipline is needed - adds a hardness overlay for task preflight, one-checkbox-at-a-time execution, test-before-checkoff, blocker handling, and verify handoff without replacing the OpenSpec lifecycle.
---

# Harness OpenSpec Apply

## Overview

为 OpenSpec 的 apply 阶段提供 hardness overlay，而不是替代 `/opsx:apply`。
核心原则：**OpenSpec 决定做哪个 Change、按哪些 tasks 做；本 skill 决定在实现阶段必须如何遵守执行纪律。**

## When to Use

在以下情况使用：
- 已进入 `/opsx:apply`、`openspec instructions apply` 或等价的 Change 实施阶段
- 需要把 `tasks.md` 执行纪律、测试纪律、STOP/BLOCKED 处理显式化
- 发现 apply 阶段容易出现“先改完再说”“未验证先勾选”“顺手做了 tasks 外工作”
- 需要为后续 `verify.md` / archive 准备可核对的实现与验证结果

不要用于：
- 创建或修复 `openspec/config.yaml`（交给 `harness-openspec-setup`）
- 配置硬门禁 Hook（交给 `harness-hook-setup`）
- 修改 `CLAUDE.md` 或 `repository/`（交给 `harness-claude-setup`）
- 替代 OpenSpec 的 change 选择、proposal/design/tasks 生命周期

## Responsibilities Boundary

| Layer | Responsibility |
|---|---|
| OpenSpec apply | 选择 change、读取 proposal/specs/design/tasks、推进 change lifecycle |
| harness-openspec-apply | 约束 apply 阶段如何执行 task、何时验证、何时 STOP/BLOCKED、如何为 verify 交接 |
| Hook | 拦截不可逆或高风险操作 |

## Quick Reference

| Stage | Required action |
|---|---|
| Preflight | 检查 tasks 是否可执行、可验证、已拆分实现/测试 |
| Execution | 一次只推进一个 checkbox；只做 tasks 已列工作 |
| Validation | 未验证不得勾选；测试失败不得勾选 |
| Blocker | 超出 Change 边界、缺命令、task 设计缺口时 STOP/BLOCKED |
| Handoff | 记录已运行命令、结果摘要、未解决 blocker，交给 verify |

## Workflow

1. **进入 OpenSpec apply 上下文**
   - 先由 `/opsx:apply` 或等价流程选定目标 change
   - 读取 `proposal.md`、相关 specs、`design.md`、`tasks.md`
   - 将本 skill 视为 apply discipline profile，而不是第二套流程

2. **执行 preflight**
   - 检查 `tasks.md` 是否存在 pending checkbox
   - 检查实现 task 与测试/验证 task 是否分离
   - 检查测试 task 是否写明：命令、范围、期望结果、失败处理、证据位置
   - 检查是否存在混合 task（如“实现 X 并测试”）
   - 检查命令是否可验证存在；不存在则 STOP，不把不存在命令当成完成标准

3. **按单个责任单元执行**
   - 一次只推进一个 checkbox 或一个最小责任单元
   - 只执行 `tasks.md` 已列工作，不补做未列任务
   - 如果发现缺测试、缺任务、缺证据要求，这是 task 设计缺口，不要自行扩大范围

4. **验证后再勾选**
   - 实现 task：先确认对应结果可观察，再勾选
   - 测试/验证 task：必须先运行命令并得到结果，再勾选
   - 不得以“改动小”“逻辑上正确”“基本完成”代替验证

5. **遇到异常时 STOP 或 BLOCKED**
   - **STOP**：需要用户或上游设计决策时暂停
   - **BLOCKED**：已做最小修复尝试，但仍不能继续时保留未完成状态并记录 blocker
   - 常见触发：
     - 命令不存在
     - 任务设计缺口
     - 需要修改 Change 边界外内容
     - 设计/spec divergence
     - 测试失败且继续尝试已无明显进展

6. **交接给 verify**
   - 汇总本轮已完成 task
   - 汇总已运行命令与结果摘要
   - 标记未完成 task 与 blocker
   - 提醒后续使用 `harness-openspec-verify` 或等价 verify gate 核对证据与完成判定

## Discipline Rules

- 只按 `tasks.md` 已列 checkbox 执行
- 一次只推进一个 checkbox；不要并行推进多个未收敛任务
- 测试失败不得勾选完成，无例外
- 命令未运行不得勾选完成，无例外
- 不得以“改动小不需要测试”跳过验证
- 发现缺测试或缺验证要求时，暂停并反馈 task 设计缺口
- 超出当前 Change 边界的修复需求，必须 STOP 或 BLOCKED

## STOP / BLOCKED Handling

### STOP

触发条件：
- `tasks.md` 混合实现与测试，无法安全执行
- 测试命令不存在或完成标准不可验证
- 需求/设计与代码现实冲突，需要人工决策
- 修复将显著超出当前 Change 范围

输出应包含：
- 触发原因
- 风险说明
- 需要的决策
- 当前未完成 task

### BLOCKED

触发条件：
- 已按最小修复路径尝试，但失败特征不再收敛
- 继续修复需要新的设计、依赖或跨边界改动
- 无足够证据判断下一步最小修复

处理方式：
- 保持 checkbox 未完成
- 记录 blocker、失败命令、错误摘要
- 交给 verify/report，不伪装为已完成

## Handoff to Verify and Archive

apply 完成后至少交接以下信息：
- 已完成的 task 列表
- 已运行命令列表
- 每条命令的结果摘要
- 尚未完成的 task
- blocker / assumption / unresolved divergence
- 是否建议进入 `harness-openspec-verify`
- 如涉及高风险 archive 门禁，提示 `harness-hook-setup`

## Common Mistakes

| Mistake | Fix |
|---|---|
| 把本 skill 当成新的 apply 生命周期 | 只把它当成 `/opsx:apply` 的纪律增强层 |
| 发现 tasks 缺测试就直接补做并勾选 | 先停止并反馈 tasks 设计缺口 |
| 一次推进多个 checkbox | 保持单责任单元推进，先收敛再继续 |
| 先勾选再补验证 | 先验证，再勾选 |
| 测试失败后继续勾选“实现已完成” | 保持未完成并记录 blocker |
| 把 verify 证据要求留到最后临时回忆 | 在 apply 结束时就整理命令和结果摘要 |

## Minimal Draft Structure

草案阶段最小结构只需要：
- `skills/harness-openspec-apply/SKILL.md`

只有在反复出现相同 evidence 摘要模板或 blocker 模板时，再考虑增加 `references/` 或 `assets/`。
