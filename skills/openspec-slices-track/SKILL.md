---
name: openspec-slices-track
description: Use when a multi-slice OpenSpec plan already exists and someone needs a cross-session progress board - compares the persisted Slice Plan YAML (the single plan source, in the workspace openspec/slice-plans/) with live OpenSpec status from active and archived changes, classifies archived/in-progress/ready/blocked items, and returns the next recommended slice in fixed response sections without re-splitting or registering changes
---

# OpenSpec 多切片进度追踪

## Overview

**多切片 Slice Plan 的进度管理器。** 跨 session 追踪所有切片状态，基于依赖关系识别哪些可以启动、哪些被阻塞，主动建议下一步推进目标。

**核心价值**：把"多个切片分散在 active change 列表与已归档状态中"的局面整理成"整体进度地图 + 推进建议"，让用户跨 session 无缝衔接工作。

**边界**：只做进度追踪与建议，不执行登记、不做拆分决策、不执行 apply/verify/archive。

## Quick Reference

| Task | Key Actions |
|------|-------------|
| **读取 Slice Plan** | 首选 `<workspace>/openspec/slice-plans/<change_name>.yaml`（两模式统一计划源）；无 yaml 时回退 auto-memory 或用户提供的 plan |
| **查询实时状态** | 先 `openspec list --json` 列 active changes；再逐个补查缺失切片的 archived 状态（`openspec status --change <name> --json`） |
| **对比分析** | 基于合并后的状态分类为 archived / in-progress / ready / blocked |
| **生成进度图** | 使用固定 ASCII progress board |
| **建议下一步** | 优先继续进行中的切片，否则启动 ready 切片 |
| **固定输出** | 只输出追踪模版，不登记、不重做拆分 |

## Critical Rules

**Read Before Track**：必须先读取计划源，再获取 active changes，并对缺失切片逐个补查 archived 状态；计划源优先级为：`<workspace>/openspec/slice-plans/<change_name>.yaml`（两模式首选，由 `openspec-slices-register` 持久化）→ auto-memory 或用户提供的 Slice Plan（无 openspec 目录/历史兼容）；缺任一必需输入时 STOP。

**MUST**:
- 依赖判断必须基于计划源中的 `depends_on` 字段，不得猜测
- 只有合并后的 CLI 状态为 `archived` 才算满足依赖
- 建议推进时优先级：进行中 > ready > blocked，不得建议 blocked 切片直接启动
- 最终回答必须使用本技能固定模版，不得输出登记命令或重新拆分建议

**MUST NOT（封堵回退旧机制）**:
- **不得**把 initiative `tasks.md` 当作计划源——slice 技能不再使用 initiative（register 不再同步它）
- **不得**用 `openspec show <change>` 查状态（实跑验证返回 "Unknown item"）；改用 `openspec status --change <name> --json`

**STOP**:
- memory 中无 Slice Plan 且用户未提供，且 `openspec/slice-plans/` 无 yaml → STOP；提示先运行 `openspec-slices-register`
- `openspec/slice-plans/` 存在 ≥2 个候选 yaml 且 CLI 活跃 change 集无法唯一解析到单族（`{change_name}` 前缀）→ STOP，要求用户指定当前批次
- active 或 archived 任一状态源获取失败且无法确认归档状态 → STOP

**Hardness**：L1（信息聚合与建议），输出只供追踪与决策参考。

## Boundaries

**不做**:
- 不执行 `openspec init` / `openspec new change` / `openspec apply` / `openspec archive`
- 不做拆分决策（交 `openspec-slices-plan`）
- 不做登记落地（交 `openspec-slices-register`）
- 不修改 Slice Plan（只读）
- 不自动启动下一个切片（只建议，由用户决策）

## Workflow

1. **读取计划源并聚焦当前批次** — 优先扫 `<workspace>/openspec/slice-plans/*.yaml`（`<workspace>` = 当前 repo 或集中工作空间 repo）：
   - 唯一 yaml → 直接读取为计划源，解析 `change_name` / `slices` / `depends_on` / `sequencing_rule` / 全量 `scope` / `goal` / `context` / `handoff` / `workspace`。
   - 多个 yaml → 用 step 2 已取的 `openspec list --json` 活跃 change 列表，按 `{change_name}-{seq}-{name}` 前缀（`openspec-slices-plan` 命名规则）自动匹配当前活跃族；唯一解析则聚焦该 yaml；≥2 族有活跃 change 且无法唯一解析 → STOP 让用户选。
   - 无 yaml → 回退 auto-memory 或用户提供的 Slice Plan。
   - cross-repo 读 yaml 后可取 `workspace.note` 获知切片代码落地哪个 repo（只读参考，不影响状态判定）。
2. **查询实时状态** — 先运行 `openspec list --json` 获取 active changes；再对计划源中未出现在 active 列表的切片逐个运行 `openspec status --change <name> --json`，补齐 archived 状态；最后合并为统一状态表，至少覆盖 `name / status / completedTasks / totalTasks / lastModified`。
3. **依赖分析** — 对每个 slice 基于计划源中的 `depends_on` 与合并后的 CLI 状态分类为 archived / in-progress / ready / blocked。
4. **生成进度图** — 使用固定 ASCII progress board，逐行展示切片状态。
5. **建议下一步** — 优先继续进行中的切片；如无进行中，则建议启动 ready 切片；blocked 只报告原因。
6. **输出固定模版** — 按 `Response Template` 原样输出，包含 progress board、summary、recommendation、consistency check。

## CLI Command Map

| Action | Command | 关键输出字段 |
|--------|---------|-------------|
| 查询 active changes | `openspec list --json` | `changes[].{name,completedTasks,totalTasks,lastModified,status}` |
| 补查 archived / 单 change 状态 | `openspec status --change <name> --json` | `changeName`、`planningHome.root`（工作空间根）、`changeRoot`、`artifactPaths`（artifact 路径与是否已生成） |
| 校验（可选健康检查） | `openspec validate [item-name] --changes --json` | 校验结果 |
| **禁止** | ~~`openspec show <change-name>`~~ | 实跑返回 "Unknown item"，不可靠 |

> `openspec list --json` 的 `status` 可为 `no-tasks`（尚未生成 tasks.md）；判断 archived 须以"不在 active 列表 + `status --change` 确认归档"为准。CLI 契约详见 `openspec-slices-register/references/cli-contract.md`。

## Plan Source Schema

`<workspace>/openspec/slice-plans/<change_name>.yaml`（由 `openspec-slices-register` 持久化的全量确认 Slice Plan）是两模式**首选**计划与依赖真相源，承载 `change_name`、`mode`、`workspace`、`slices`（含全量 `scope` / `goal` / `context` / `handoff`）、`depends_on`、`sequencing_rule`。计划源不承担实时状态；实时状态一律以 CLI 为准，并且必须同时覆盖 active 与 archived changes。无 openspec 目录或历史兼容场景回退 Slice Plan memory（仅含最小必需 + 可选增强字段）。完整示例见 `references/memory-schema.md`。

## Response Template

最终回答必须使用以下固定结构与顺序；无警告时也必须保留 `## Warnings` 并写 `- None`。

```md
## Result
- skill: openspec-slices-track
- status: tracked | stop
- boundary_check: tracking only; no registration, re-splitting, or implementation

## Core Output
- plan_source: <workspace>/openspec/slice-plans/<change_name>.yaml | memory file | user-provided plan
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
  - name: sample-feature-03-foo      # {change-name}-{change-num}-{slice-change-name}
    waiting_on: [01, 02]
- consistency_check:
  - <plan/CLI mismatch, or none>

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
  - CLAUDE.md 指示"启动时检查进度"
  - 其他 skill 完成后建议"查看整体进度"
- **交给**:
  - `openspec-continue`
  - `openspec-apply`
  - `openspec-archive`

## Failure Handling

| Failure | Action |
|---------|--------|
| 无 plan source（yaml 不存在 + memory 无 + 用户未提供） | STOP；提示先运行 `openspec-slices-register` 持久化 slice-plan.yaml |
| active changes 为空且 archived changes 也为空 | STOP；提示先运行 `openspec-slices-register` |
| CLI 任一状态源返回错误 | 报告错误详情；检查是否在正确工作空间目录，并说明 archived 状态可能未被纳入 |
| 依赖关系不一致 | 警告计划源与实际 changes 不匹配；列出缺失/多余项 |

## References

- **references/progress-tracking.md**：状态映射规则、依赖满足判定、优先级排序、ASCII 模版与固定回答示例
- **references/memory-schema.md**：Slice Plan memory 格式约定（无 openspec 目录时的回退源）、字段说明、示例
