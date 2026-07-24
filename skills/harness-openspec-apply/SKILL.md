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
- 如果确认继续，需要记录的证据或恢复动作

统一格式：
```markdown
## STOP - 需要人工确认

### 触发原因
<说明触发了哪条 apply 纪律或 Change 边界>

### 风险说明
<说明继续执行可能造成的偏差、越界或假完成风险>

### 需要的决策
- [ ] 调整 tasks / design 后再继续
- [ ] 确认风险可接受，继续当前路径
- [ ] 标记 BLOCKED，交回 verify / 上游设计处理

### 当前未完成 task
<列出 checkbox 或责任单元>
```

STOP 后恢复记录：

| 用户响应 | Apply 行动 | Verify 交接记录 |
|---|---|---|
| 确认继续 | 继续执行最小责任单元 | `[STOP CONFIRMED]` + 原因 + 用户决策 + 执行时间 |
| 拒绝继续 | 保持 task 未完成 | `[STOP REJECTED]` + blocker + 需要的调整 |
| 要求回滚 | 回到最近安全状态 | `[STOP ROLLBACK]` + 回滚范围 + 后续验证要求 |

### BLOCKED

触发条件：
- 已按最小修复路径尝试，但失败特征不再收敛
- 继续修复需要新的设计、依赖或跨边界改动
- 无足够证据判断下一步最小修复

处理方式：
- 保持 checkbox 未完成
- 记录 blocker、失败命令、错误摘要
- 交给 verify/report，不伪装为已完成

BLOCKED 记录最少包含：
- 失败命令
- 输出摘要或关键错误
- 已尝试的最小修复
- 仍未完成的 checkbox
- 继续推进所需的新设计、依赖、证据或人工决策

## Progressive Repair Loop

测试或验证失败时，只允许在 Change 边界内执行渐进修复：

1. 记录失败命令和输出摘要
2. 识别最小可能原因
3. 做最小修复，不扩大 tasks 范围
4. 先重跑最窄验证，再考虑更宽检查
5. 只要失败特征仍在变化且有收敛迹象，可以继续一轮
6. 当失败特征不再变化、修复需要越界、或证据不足以判断根因时，停止为 BLOCKED
7. 保持相关 checkbox 未完成，并把命令、输出、blocker 交给 verify

## Subagent Delegation

### 适用与不适配

在单个 checkbox / 责任单元内，可把 **dev（实现）→ test（验证）** 拆成两个 subagent 串行执行（subagents-in-sequence），用于隔离读写污染、保护主上下文。

**用 Agent tool（subagents in sequence），不用 Workflow tool**：
- Workflow 由 script 决定下一步运行什么，而 apply 纪律要求 **main agent 逐轮判断收敛**。
- Workflow 不支持 mid-run 用户输入，与 apply 的 **STOP / BLOCKED 人工裁决**冲突。
- Workflow 完成才停，无法在 dev→test 之间插入 checkoff 门。

因此 dev→test 串行编排只能用 subagents-in-sequence。

### 限定边界

- **只限单 checkbox 内的 dev→test 串行**，不是多 checkbox 并行（违反“一次只推进一个 checkbox”，见 Discipline Rules）。
- dev 与 test 必须先由 preflight 确认已分离为独立 task（见 Workflow §2 preflight）；混合 task（如“实现 X 并测试”）不委派，先 STOP。
- 不跨 Change 边界委派；越界即 STOP/BLOCKED。

**为什么“分离”不能被“委派”替代**：自动把一个 task 拆成 dev→test 两个 subagent 串行，只改变“谁干这两段”，没改变“完成判定的语义”。混合 task 会让勾选粒度不可分（test 失败时既不能勾、又不能说“实现那半完成”）、证据映射模糊、失败独立性丧失。preflight 要的 impl/test 分离是 **design 级结构要求**（管 checkbox 边界与证据落位）；subagent 串行是 **execution 级编排**（管谁干）。两者不同层，不互斥——委派不能替 design 兜边界。

### 委派契约（dev / test 不对称，不照搬 verify 的“只回结论”）

**dev subagent（实现）**——回传**证据载荷**，不是“done”：
- 改动文件列表
- 执行命令
- 可验证结论行（如 `typecheck: 0 error`）
- 时间戳
- 失败特征（若未通过）

**test subagent（验证）**——**独立运行命令**，回传中-高可信证据：
- 契合 verify 防造假要求（自报告 = 低可信；可复现命令 + 结论行 + 时间戳 = 中可信；工具生成 = 高可信）。
- 不复用 dev subagent 的“我说改对了”，独立跑命令取结果。

**main agent 持有（不外包给 subagent）**：
- 收敛判断（是否进入下一 checkbox）
- checkoff
- STOP / BLOCKED 裁决
- handoff to verify
- task lifecycle（subagent 不持 task lifecycle）

### Progressive Repair Loop 与委派的交叉

测试失败触发修复环时（见 Progressive Repair Loop）：
- 修复环**跨 subagent**：每轮 test subagent 只回传失败特征摘要，**main 持跨轮序列**判断“失败特征是否仍在变化、是否收敛”。
- 越界 / 不收敛由 **main 裁决** STOP/BLOCKED，**不让 dev subagent 自行 STOP**。
- subagent 只回依据，不持收敛判；这与“决策不外包给 subagent”一致。

### Delegation 反模式（MUST NOT）

- ❌ 用 Workflow tool 编排 dev→test（抹掉 main 收敛 + 人工 STOP）。
- ❌ 让 dev subagent 回传“done”而无证据载荷（等于自报告低可信）。
- ❌ 让 test subagent 复用 dev 的“改对了”声明而不独立跑命令。
- ❌ 让 subagent 持 task lifecycle / checkoff / STOP 裁决。
- ❌ 并行多 checkbox（违反单责任单元推进）。

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
