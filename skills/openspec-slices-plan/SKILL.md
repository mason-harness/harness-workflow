---
name: openspec-slices-plan
description: Use when a request is too large for one Change, needs slice strategy selection, or spans single-repo vs cross-repo boundaries - produces the approved Slice Plan by choosing vertical business slices for single-project work and technical or repo-boundary slices for multi-project or multi-team delivery before registration
---

# OpenSpec Change 拆分决策

## Overview

**唯一的 Change 拆分决策技能。** 先判一个需求是否需要拆分，再按项目形态选切片维度：单一项目优先垂直业务切片，多项目/多仓分离交付可按技术层或仓库边界切。产出经用户确认的 **Slice Plan**，再交 `openspec-slices-register` 登记；若为 `cross-repo`，登记阶段还必须把切片索引同步进 initiative `tasks.md`，供后续 `openspec-slices-track` 作为计划源读取。

**核心原则**：以“可独立推进、依赖清晰、owner 明确”为标准。先判项目形态，再定切法；拿不准时先合后拆。

**边界**：只产 Slice Plan 文本，零写文件——不执行 CLI 登记、不写 proposal/specs/design/tasks 全文、不写业务代码、不跟踪进度、不改 config.yaml/managed skills。`cross-repo` 下只声明“initiative 作为后续计划源”的契约，不负责实际同步 initiative 文档。

## Quick Reference

| Task | Key Actions |
|------|-------------|
| **从 explore 衔入** | 读取 explore 结晶结果或用户需求，确认范围与现有行为 |
| **判该不该拆** | 单一责任单元 + 文件 ≤5 + 代码 ≤150 行 → 单切片；否则拆分 |
| **判项目形态** | 单一项目/单仓 → 优先垂直业务切片；多项目/多仓分离 → 可按技术层或仓库边界切 |
| **生成切片与序号** | `sNN-` 前缀 + kebab-case，定义 goal / depends_on / scope |
| **确定模式** | 单仓 → `single-repo`；跨仓/多团队 → `cross-repo` + initiative |
| **请求确认** | 输出固定模版；未确认不得交给 register |
| **交接** | 仅把已确认 Slice Plan 交 `openspec-slices-register` |

## Critical Rules

**Read Before Decide**：读取 explore 结晶结果或用户需求、`openspec list --json`（避免重复 change）、相关现有行为。范围不清或来源冲突时 **STOP**。

**MUST（三层封堵 L2——本技能是唯一拆分决策者）**:
- 任何 `openspec new change` 之前必须经本技能拆分评估并产出经用户确认的 Slice Plan，无例外
- 不得以“需求不大”“已知道怎么拆”“用户催促开始”跳过拆分评估
- 本技能是 Change 拆分的唯一决策出口；其他技能不得自行拆分
- 最终回答必须使用本技能固定模版，不得改成自由摘要或执行日志

**STOP**:
- 未持有用户确认的 Slice Plan 时不得进入 Change 登记环节
- 不得自行代替 `openspec-slices-register` 执行 `openspec new change` / `set change` / `initiative create` / `context-store setup`
- 跨仓场景遇 workspace change（非 repo-local）→ STOP，workspace change 不得 link initiative

**Hardness**：本技能处于 L1-L2（Explore/Proposal 区间）——拆分决策是结构化建议，但触发条件与确认门禁是硬约束。

## Boundaries

**不做**:
- 不执行 `openspec new change` / `set change` / `initiative create` / `context-store setup`
- 不写 proposal.md / specs / design / tasks 完整内容
- 不写业务代码
- 不勾选或跟踪 task 进度
- 不修改 config.yaml / managed skills
- 未获用户确认时不得将 Slice Plan 交给 register

## Workflow

1. **接收上下文** — 从 `openspec-explore` 结晶结果或用户需求读取目标范围与现有行为；运行 `openspec list --json` 避免重复；范围不清或来源冲突时 STOP。
2. **判断是否需要拆分** — 单一责任单元 + 文件影响 ≤5 + 代码 ≤150 行 → 单切片；否则进入拆分。详见 `references/split-strategy.md`。
3. **判断项目形态并选维度** — 单一项目/单仓优先按用户可感知价值做垂直业务切片；若需求天然跨多个项目、仓库或职责面（如前端 / 后端 / 数据库分离交付），可按技术层或仓库边界切，但每片必须有清晰 owner 与依赖。
4. **生成切片与序号** — 每切片使用 `sNN-` 序号前缀 + kebab-case 命名；定义 `goal`、`areas`、`depends_on`。
5. **依赖序列** — 选择 `sequencing_rule`：`archive-N-before-N+1` / `parallel` / `merge-first-split-later`。
6. **判断单仓 vs 跨仓** — 跨仓/多团队协作 → `cross-repo` + initiative；单仓 → `single-repo`；workspace 冲突时 STOP。
7. **起草骨架** — 为每切片起草 `context`、`dependencies`、`scope`，并判断是否存在需要下游显式消费的 `handoff` 契约；`cross-repo` 时还要声明后续将由 initiative `tasks.md` 承接切片索引、依赖与 repo/change 映射；只定义边界，不落盘任何文件。
8. **使用固定模版输出 Slice Plan** — 按 `Response Template` 原样输出，保留 `Slice Plan` 作为唯一交接契约。
9. **请求确认并交接** — 用户确认后，交 `openspec-slices-register`；若为 `cross-repo`，登记结果还必须确认 initiative slice index 已同步；未确认时保持在 plan 阶段。

## Slice Plan Schema

本 schema 是 slice → register 的唯一交接契约。register 只可据此登记，不得重新决策；字段缺失即 STOP 退回本技能。若 `mode: cross-repo`，登记结果还必须把切片索引同步进 initiative `tasks.md`，供后续 `openspec-slices-track` 作为计划源读取。

`handoff` 字段规则：
- 仅有顺序依赖、scope 边界、归档前置条件时，填 `null`
- 需要下游显式消费接口、状态字段、事件、迁移顺序、feature flag 切换点、人工步骤时，填结构化对象
- `handoff` 的内容由 `openspec-slices-register` 物化为 `proposal.md` 中的 `## Handoff`

```yaml
Slice Plan (user_confirmed: true)
mode: single-repo | cross-repo
initiative:
  id: <kebab>
  store: <kebab>
  store_path: <storeRoot>
  title: <one line>
  summary: <one line>
sequencing_rule: archive-N-before-N+1 | parallel | merge-first-split-later
slices:
  - sequence: "s01"
    name: <kebab>
    goal: <one line>
    areas: [backend, db]
    depends_on: []
    context: <1-2 行故事线定位>
    dependencies: [<前序 sequence / 外部依赖>]
    scope: { in: [...], out: [...] }
    handoff: null | {
      handoff_to: <downstream slice or repo/team>
      artifacts/contracts: [<interface, state field, event, migration note, manual step>]
      ready_signal: [<what must be true before downstream starts>]
      consumer_expectations: [<what downstream may rely on and must not change>]
    }
```

完整 few-shot 示例见 `references/slice-plan-examples.md`。

## Response Template

最终回答必须使用以下固定结构与顺序；无警告时也必须保留 `## Warnings` 并写 `- None`。

```md
## Result
- skill: openspec-slices-plan
- status: pending-confirmation | confirmed | stop
- boundary_check: planning only; no registration, tracking, or implementation

## Core Output
- decision: split | do-not-split | merge-first-split-later
- rationale: <why>
- mode: single-repo | cross-repo
- initiative: <initiative summary or null>
- sequencing_rule: <rule>
- slices:
  - sequence: s01
    name: <kebab>
    goal: <one line>
    depends_on: []
    scope:
      in: [...]
      out: [...]
    handoff: null | {handoff_to, artifacts/contracts, ready_signal, consumer_expectations}
- confirmation_status: user_confirmed: false | true

```yaml
Slice Plan (user_confirmed: true|false)
...
```

## Handoff
- handoff_to: openspec-slices-register | none
- handoff_input: confirmed Slice Plan
- handoff_reason: registration starts only after confirmation; in cross-repo mode, registration must also sync the initiative slice index for downstream tracking

## Next Step
- recommended_action: confirm or adjust the Slice Plan
- requires_user_confirmation: yes
- tracking_plan_source: initiative (`cross-repo`) | memory or user-provided plan (`single-repo`)

## Warnings
- <missing scope, unclear boundary, workspace limitation, or None>
```

## Handoffs

- **接收自**:
  - `openspec-explore`
  - 用户直接请求拆分
  - `harness-openspec-setup`
- **交给**:
  - `openspec-slices-register`
  - 登记完成后继续 `openspec-continue` 或后续 apply / verify / archive 流程

## Failure Handling

| Failure | Action |
|---------|--------|
| 需求范围不清 | STOP；要求用户或 explore 澄清后再决策 |
| 来源冲突（需求 vs 现有 spec） | STOP 并报告冲突 |
| 单一项目却误按技术层过早横切 | 改为垂直业务切片；引用 references Wrong/Right 示例 |
| 多项目/多仓协作却硬套垂直业务切片 | 改按技术层或仓库边界切，并补清 owner / depends_on / initiative |
| 只写了 scope.out，却遗漏下游必须消费的契约 | 补 `handoff` 字段；不要把接口/状态/迁移前提藏在自然语言里 |
| 拿不准是否拆 | 选 `merge-first-split-later`，先合成单 Change |
| 跨仓但遇 workspace change | STOP；workspace 不可 link initiative，改 repo-local 或退回用户决策 |
| 切片过多（过度拆分） | 合并；坚持“可独立交付”标准 |
| 单 Change tasks 超 50 项（拆分不足） | 检查可否独立为子 Change |

## References

- **references/split-strategy.md**：拆分决策树、按项目形态选择切片维度的 Wrong/Right 示例、序号前缀目录、merge-first-split-later、Proposal storyline 策略、Initiative 机制与 workspace-change 限制
- **references/slice-plan-examples.md**：完整 Slice Plan few-shot 与固定回答模版示例
