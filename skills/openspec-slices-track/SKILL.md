---
name: openspec-slices-track
description: Use when a multi-slice OpenSpec plan already exists and someone needs a cross-session progress board - compares saved slice dependencies with live OpenSpec status from active and archived changes, classifies archived/in-progress/ready/blocked items, and returns the next recommended slice in fixed response sections without re-splitting or registering changes
---

# OpenSpec 多切片进度追踪

## Overview

**多切片 Slice Plan 的进度管理器。** 跨 session 追踪所有切片状态，基于依赖关系识别哪些可以启动、哪些被阻塞，主动建议下一步推进目标。

**核心价值**：把“多个切片分散在 active change 列表与已归档状态中”的局面整理成“整体进度地图 + 推进建议”，让用户跨 session 无缝衔接工作。

**边界**：只做进度追踪与建议，不执行登记、不做拆分决策、不执行 apply/verify/archive。

## Quick Reference

| Task | Key Actions |
|------|-------------|
| **读取 Slice Plan** | `cross-repo` 优先从 initiative `tasks.md` 读取依赖关系与 `sequencing_rule`；否则回退到 auto-memory 或用户提供的 plan |
| **查询实时状态** | 先列出 active changes，再逐个补查缺失切片的 archived 状态 |
| **对比分析** | 基于合并后的状态分类为 archived / in-progress / ready / blocked |
| **生成进度图** | 使用固定 ASCII progress board |
| **建议下一步** | 优先继续进行中的切片，否则启动 ready 切片 |
| **固定输出** | 只输出追踪模版，不登记、不重做拆分 |

## Critical Rules

**Read Before Track**：必须先读取计划源，再获取 active changes，并对缺失切片逐个补查 archived 状态；`cross-repo` 优先读取 initiative `tasks.md`，`single-repo` 或历史场景回退到 Slice Plan memory 或用户提供的 plan；缺任一必需输入时 STOP。

**MUST**:
- 依赖判断必须基于计划源中的 `depends_on` 字段，不得猜测
- 只有合并后的 CLI 状态为 `archived` 才算满足依赖
- 建议推进时优先级：进行中 > ready > blocked，不得建议 blocked 切片直接启动
- 最终回答必须使用本技能固定模版，不得输出登记命令或重新拆分建议

**STOP**:
- `cross-repo` 缺 initiative plan source，且无可接受的显式降级来源 → STOP
- `single-repo` 或历史场景下，memory 中无 Slice Plan 且用户未提供 → STOP
- active 或 archived 任一状态源获取失败且无法确认归档状态 → STOP

**Hardness**：L1（信息聚合与建议），输出只供追踪与决策参考。

## Boundaries

**不做**:
- 不执行 `openspec new change` / `openspec apply` / `openspec archive`
- 不做拆分决策（交 `openspec-slices-plan`）
- 不做登记落地（交 `openspec-slices-register`）
- 不修改 Slice Plan（只读）
- 不自动启动下一个切片（只建议，由用户决策）

## Workflow

1. **读取计划源** — `cross-repo` 优先从 initiative `tasks.md` 读取当前切片索引；解析 `slices`、`depends_on`、repo/change 映射、`sequencing_rule`；`single-repo` 或历史场景回退到 auto-memory 或用户提供的 Slice Plan；缺失则 STOP。
2. **查询实时状态** — 先运行 `openspec list --json` 获取 active changes；再对计划源中未出现在 active 列表的切片逐个运行 `openspec status --change <name> --json` 或等效 CLI 查询，补齐 archived 状态；最后合并为统一状态表，至少覆盖 `name / status / completedTasks / totalTasks / lastModified`。
3. **依赖分析** — 对每个 slice 基于计划源中的 `depends_on` 与合并后的 CLI 状态分类为 archived / in-progress / ready / blocked。
4. **生成进度图** — 使用固定 ASCII progress board，逐行展示切片状态。
5. **建议下一步** — 优先继续进行中的切片；如无进行中，则建议启动 ready 切片；blocked 只报告原因。
6. **输出固定模版** — 按 `Response Template` 原样输出，包含 progress board、summary、recommendation、consistency check。

## CLI Command Map

| Action | Command |
|--------|---------|
| 查询 active changes | `openspec list --json` |
| 查询 archived changes | 对未出现在 active 列表的切片运行 `openspec status --change <name> --json` 或等效 CLI 查询，补齐 `archived` 状态 |
| 查询单个 change 详情 | `openspec status --change <name> --json` |

## Plan Source Schema

`cross-repo` 优先使用 initiative `tasks.md` 作为计划与依赖真相源，承载切片列表、`depends_on`、repo/change 映射与 `sequencing_rule`；`single-repo`、历史场景或显式降级时使用 Slice Plan memory。计划源不承担实时状态；实时状态一律以 CLI 为准，并且必须同时覆盖 active 与 archived changes。完整示例见 `references/memory-schema.md`。

## Response Template

最终回答必须使用以下固定结构与顺序；无警告时也必须保留 `## Warnings` 并写 `- None`。

```md
## Result
- skill: openspec-slices-track
- status: tracked | stop
- boundary_check: tracking only; no registration, re-splitting, or implementation

## Core Output
- plan_source: <initiative tasks.md | memory file | user-provided plan>
- live_status_source: cli
- progress_board: |
    <ASCII board>
- summary:
  - archived: 0/0
  - in_progress: 0/0
  - ready: 0/0
  - blocked: 0/0
- recommendation: <continue X | start Y | all archived>
- blocked_items:
  - name: s03-foo
    waiting_on: [s01, s02]
- consistency_check:
  - <initiative/CLI mismatch, memory/CLI mismatch, initiative/memory mismatch, or none>

## Handoff
- handoff_to: openspec-continue | openspec-apply | openspec-archive | none
- handoff_input: recommended next slice
- handoff_reason: follow-up action depends on current progress state

## Next Step
- recommended_action: continue in-progress slice or start the first ready slice
- requires_user_confirmation: yes

## Warnings
- <missing plan source, CLI mismatch, source mismatch, or None>
```

## Handoffs

- **接收自**:
  - 用户主动调用
  - CLAUDE.md 指示“启动时检查进度”
  - 其他 skill 完成后建议“查看整体进度”
- **交给**:
  - `openspec-continue`
  - `openspec-apply`
  - `openspec-archive`

## Failure Handling

| Failure | Action |
|---------|--------|
| `cross-repo` 无 initiative plan source | STOP；提示先完成 `openspec-slices-register` 的 initiative slice index 同步，或显式提供可降级的 plan source |
| `single-repo`/历史场景下 memory 中无 Slice Plan | STOP；提示先运行 `openspec-slices-plan` |
| active changes 为空且 archived changes 也为空 | STOP；提示先运行 `openspec-slices-register` |
| CLI 任一状态源返回错误 | 报告错误详情；检查是否在正确项目目录，并说明 archived 状态可能未被纳入 |
| 依赖关系不一致 | 警告计划源与实际 changes 不匹配；列出缺失/多余项 |

## References

- **references/progress-tracking.md**：状态映射规则、依赖满足判定、优先级排序、ASCII 模版与固定回答示例
- **references/memory-schema.md**：Slice Plan memory 格式约定、字段说明、示例
