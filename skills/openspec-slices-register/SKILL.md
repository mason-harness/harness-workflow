---
name: openspec-slices-register
description: Use when a confirmed Slice Plan needs OpenSpec change registration, especially for multi-slice, single-repo vs cross-repo, or initiative-linked work - registers each slice via subagents using openspec-new-change or /opsx:new, completes proposal.md during registration, and avoids re-splitting, direct new-change loops, or progress tracking
---

# OpenSpec Change 登记执行

## Overview

将**已确认**的 Slice Plan 落地为一个或多个 OpenSpec changes。父流程只做校验、store / initiative 准备、结果汇总；**每个 slice 必须由 subagent 创建 change 并在当次写完整 `proposal.md`**。

## Use When

- 已有 `openspec-slices-plan` 产出的 Slice Plan，且 `user_confirmed: true`
- 需要把多切片登记成 changes
- 需要处理 `single-repo` / `cross-repo` / `initiative`
- 需要避免直接裸跑 `openspec new change` 多次循环
- 需要在登记阶段完成 `proposal.md`，而不是只落 stub

## Core Rules

- 只处理**已确认** Slice Plan；缺字段或未确认就 STOP
- 每个 slice 都必须通过 subagent 调 `openspec-new-change`；不可用时退到 `/opsx:new`
- 不允许只创建 change、只落 `Context / Dependencies / Scope` stub、或把 proposal 留给后续补写
- `proposal.md` 至少完成：`Goal`、`Context`、`Dependencies`、`Scope`、`Requirements`、`Assumptions`、`Non-Goals`
- 默认不新建独立交接文档；切片间影响若只是边界与顺序，写入 `Dependencies / Scope / Assumptions` 即可
- 当且仅当 Slice Plan 的 `handoff` 为对象时，必须在当前 `proposal.md` 增加 `Handoff` 章节，并物化 `handoff_to / artifacts/contracts / ready_signal / consumer_expectations`
- `cross-repo` 时先准备 `context-store` 和 `initiative`，并在登记完成后幂等同步 initiative `tasks.md` 中的切片索引
- initiative `tasks.md` 只维护切片索引、依赖、repo/change 映射与 handoff 摘要，不维护 task 级进度；实时状态仍以 CLI 为准
- 若 initiative `tasks.md` 存在人工编辑冲突，或当前格式无法安全更新，则 STOP 并要求人工确认
- `workspace change` 需要 link initiative 时 STOP
- 最终回答只输出登记结果，不重做拆分、不汇报进度

## Workflow

1. 校验 Slice Plan 与必填字段。
2. 判断 `single-repo` 或 `cross-repo`。
3. `cross-repo` 时检查/创建 `context-store` 与 `initiative`。
4. 为每个 slice 整理 `goal / context / dependencies / scope / handoff`。
5. 启动 subagent 创建或复用 change，并在同轮完成 `proposal.md`；严格按 Slice Plan 的 `handoff` 字段决定是否写 `Handoff` 章节。
6. `cross-repo` 时幂等同步 initiative `tasks.md`：写入切片索引、`depends_on`、repo/change 映射与 handoff 摘要，并明确 live status via CLI。
7. 校验 initiative link、initiative slice index、proposal 完整度、`Handoff` 与 Slice Plan.handoff 的一致性，以及整体边界是否仍与 Slice Plan 对齐。
8. 输出固定结果；下一步交 `openspec-continue` 或后续 apply / verify / archive。

## Quick Reference

| Case | Action |
|---|---|
| single-repo | subagent 创建 change → 完成 proposal |
| cross-repo | 先 store / initiative → 再逐 slice 创建 → 同步 initiative slice index |
| already exists | 视为成功；检查 link 与 proposal 是否已完整 |
| proposal 冲突 | STOP，要求人工确认是否覆盖 |
| command unavailable | `openspec-new-change` → `/opsx:new` → 都不可用则 STOP |

## Subagent Contract

subagent 输入至少包含：
- `sequence` / `name` / `goal`
- `context` / `dependencies` / `scope.in` / `scope.out`
- `handoff`：直接读取 Slice Plan 中的 `handoff` 字段；`null` 表示不写 `proposal.md` 的 `Handoff`，对象表示必须物化为 `proposal.md` 的 `Handoff`
- `initiative` / `store`（如适用）

subagent 输出必须保证：
- change 已创建或确认存在
- `proposal.md` 已完整
- `handoff` 为对象时，`proposal.md` 含非空 `Handoff` 章节，且字段语义与 Slice Plan 保持一致；`handoff: null` 时，不得为了“完整”额外创建 `Handoff`
- `cross-repo` 时返回足以同步 initiative `tasks.md` 的切片索引信息：`sequence`、change 名称、repo、`depends_on`、handoff 摘要
- 不含 `TODO` / `TBD` / 空章节 / stub-only 内容

## Response Template

注意：下面回答模版里的 `## Handoff` 指的是**本技能对后续流程的交接**，不是 `proposal.md` 内部的切片契约章节。只有存在真实跨切片契约时，才要求在 `proposal.md` 内新增 `## Handoff`。

```md
## Result
- skill: openspec-slices-register
- status: registered | partially-registered | stop
- boundary_check: registration only; no re-splitting, tracking, or implementation

## Core Output
- registration_path: single-repo | cross-repo
- initiative_link: <store/id summary or none>
- registered_changes:
  - change: s01-foo
    goal: <one line>
    depends_on: []
    creation_mode: subagent-openspec-new-change | subagent-opsx-new | already-existed
    status: created | already-existed | linked
- proposal_status:
  - change: s01-foo
    state: complete | already-complete | synced-from-plan | stop-on-conflict
- initiative_sync: synced | unchanged | stop-on-conflict | none
- initiative_task_rows:
  - sequence: s01
    change: s01-foo
    repo: <repo>
    depends_on: []
- sequencing_hint: <archive or parallel reminder>

## Handoff
- handoff_to: openspec-continue | apply/verify/archive flow
- handoff_input: registered changes + initiative slice index
- handoff_reason: proposal is already complete, and cross-repo tracking can read the synced initiative plan directly

## Next Step
- recommended_action: continue with the next artifact for the next ready change
- requires_user_confirmation: no

## Warnings
- <missing field, workspace limitation, proposal conflict, CLI failure, or None>
```

## Handoff Decision Rule

| Situation | What to do |
|---|---|
| 仅有先后顺序、scope 边界、前序归档要求 | 只写 `Dependencies / Scope / Assumptions`，不新建交接文档 |
| 下游只需知道“这个切片做完了” | 只在 `Dependencies` 标明 `Blocks`/`Depends on` |
| Slice Plan 的 `handoff` 为对象 | 在当前 `proposal.md` 增加 `Handoff` 章节，并直接物化 `handoff_to / artifacts/contracts / ready_signal / consumer_expectations` |
| Slice Plan 的 `handoff` 为 `null` | 不创建 `proposal.md` 的 `Handoff`；只保留 `Dependencies / Scope / Assumptions` |
| 已存在人工维护的独立交接文档且当前流程明确要求沿用 | 在 `Handoff` 章节引用该文档；不要静默新建第二份 |

## References

- `references/cli-contract.md`：CLI 语法、initiative 限制、kebab-case 规则
- `references/landing-patterns.md`：single-repo / cross-repo 示例、完整 proposal 写法、幂等与冲突处理
