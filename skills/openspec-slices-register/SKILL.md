---
name: openspec-slices-register
description: Use when a confirmed Slice Plan needs OpenSpec change registration, especially for multi-slice or cross-repo work where a centralized workspace must be created first - registers each slice via subagents using openspec-new-change or /opsx:new inside the workspace (repo-local, no initiative), completes proposal.md during registration, persists the Slice Plan YAML for downstream tracking, and avoids re-splitting, direct new-change loops, --initiative, or progress tracking
---

# OpenSpec Change 登记执行

## Overview

将**已确认**的 Slice Plan 落地为一个或多个 OpenSpec changes。父流程负责校验、工作空间准备（cross-repo 时 `openspec init`）、结果汇总；**每个 slice 必须由 subagent 创建 change 并在当次写完整 `proposal.md`**。

**数据存储原则**：所有产物（`slice-plan.yaml` 与 changes）永远落在执行项目工作空间内——single-repo 用当前 repo 的 `openspec/`，cross-repo 用先 `openspec init` 创建的集中工作空间 repo——使其随 git 同步、跨设备可用。**不使用** context-store / initiative / `--initiative`。

## Use When

- 已有 `openspec-slices-plan` 产出的 Slice Plan，且 `user_confirmed: true`
- 需要把多切片登记成 changes
- 需要处理 single-repo（`workspace: null`）或 cross-repo（`workspace:` 为对象，需先建集中工作空间）
- 需要避免直接裸跑 `openspec new change` 多次循环
- 需要在登记阶段完成 `proposal.md`，而不是只落 stub

## Core Rules

- 只处理**已确认** Slice Plan；缺字段或未确认就 STOP
- **遇带 `initiative:` 字段的遗留 Slice Plan → STOP，要求退回 `openspec-slices-plan` 重新产出 `workspace:` 形态**——register 不重做 plan 决策（`initiative:`→`workspace:` 是 plan 的事）
- cross-repo（`workspace:` 为对象）且 `init_status: required` → 先 `openspec init --tools none --force <workspace.path>` 建集中工作空间；`already-initialized` → 仅校验 `openspec/config.yaml` 存在（不跑 `validate`）
- 每个 slice 都必须通过 subagent 调 `openspec-new-change`；不可用时退到 `/opsx:new`
- 创建 change 一律 **repo-local，不带 `--initiative`**（CLI 契约，见 `references/cli-contract.md`）
- 不允许只创建 change、只落 `Context / Dependencies / Scope` stub、或把 proposal 留给后续补写
- `proposal.md` 至少完成：`Goal`、`Context`、`Dependencies`、`Scope`、`Requirements`、`Assumptions`、`Non-Goals`
- 默认不新建独立交接文档；切片间影响若只是边界与顺序，写入 `Dependencies / Scope / Assumptions` 即可
- 当且仅当 Slice Plan 的 `handoff` 为对象时，必须在当前 `proposal.md` 增加 `Handoff` 章节，并物化 `handoff_to / artifacts/contracts / ready_signal / consumer_expectations`
- cross-repo 时把 `workspace.note`（切片代码落地哪些 repo）物化进 proposal 的 `Scope`/`Dependencies`
- 最终回答只输出登记结果，不重做拆分、不汇报进度

**MUST NOT（封堵回退旧机制）**:
- **不得**使用 `context-store setup` / `initiative create` / `set change --initiative`——数据须落工作空间 repo 内（可 git 同步）
- **不得**给 slice change 带 `--initiative`
- **不得**使用 `openspec workspace setup` 建集中工作空间——它把数据写到机器本地 `~/.local/share/openspec/`，不可 git 同步、不可跨设备（CLI 契约已验证）；改用 `openspec init`
- **不得**同步/维护 initiative `tasks.md`——批次计划真相统一由 `openspec/slice-plans/<change_name>.yaml` 承载

## Workflow

1. 校验 Slice Plan 与必填字段，**且必须 `user_confirmed: true`**（未确认不得进入登记或持久化，防绕过 `openspec-slices-plan` 的确认门禁）。遇 `initiative:` 字段 → STOP 退回 plan。
2. 判断 `workspace`：
   - `workspace: null` → single-repo，用当前 repo 的 `openspec/`
   - `workspace:` 为对象 → cross-repo：若 `init_status: required`，跑 `openspec init --tools none --force <workspace.path>` 建集中工作空间；`already-initialized` 则仅校验 `openspec/config.yaml` 存在
3. 为每个 slice 整理 `goal / context / dependencies / scope / handoff`（cross-repo 附 `workspace.note` 指明代码落地 repo）。
4. 启动 subagent 在工作空间内创建或复用 change，并在同轮完成 `proposal.md`；严格按 Slice Plan 的 `handoff` 字段决定是否写 `Handoff` 章节。`new change` **不带 `--initiative`**。
5. 持久化 Slice Plan YAML：把**确认的全量 Slice Plan YAML 原文**写入 `<workspace>/openspec/slice-plans/<change_name>.yaml`（`<workspace>` 为当前 repo 或集中工作空间 repo；CLI 不管理此子目录；仅当 step 1 已确认 `user_confirmed: true`）。幂等与冲突规则见 `references/landing-patterns.md`「Slice Plan YAML 持久化」。
6. 校验 proposal 完整度、`Handoff` 与 Slice Plan.handoff 的一致性、slice-plan.yaml 已持久化且与 Slice Plan 一致、cross-repo 时各 proposal 的 scope/dependencies 已含落地 repo 标注，以及整体边界是否仍与 Slice Plan 对齐。
7. 输出固定结果；下一步交 `openspec-continue` 或后续 apply / verify / archive。

## Quick Reference

| Case | Action |
|---|---|
| single-repo（`workspace: null`） | 直接在当前 repo 的 `openspec/` 下 subagent 创建 change → 完成 proposal |
| cross-repo（`workspace:` 对象, `init_status: required`） | 先 `openspec init --tools none --force <path>` 建集中工作空间 → 再在其内逐 slice 创建 repo-local change（不带 `--initiative`） |
| cross-repo（`init_status: already-initialized`） | 校验 `openspec/config.yaml` 存在 → 直接逐 slice 创建 |
| already exists | 视为成功；检查 proposal 是否已完整 |
| proposal 冲突 | STOP，要求人工确认是否覆盖 |
| 遗留 `initiative:` Slice Plan | STOP，退回 `openspec-slices-plan` 重新产出 `workspace:` 形态 |
| command unavailable | `openspec-new-change` → `/opsx:new` → 都不可用则 STOP |

## Subagent Contract

subagent 输入至少包含：
- `sequence` / `name` / `goal`
- `context` / `dependencies` / `scope.in` / `scope.out`
- `handoff`：直接读取 Slice Plan 中的 `handoff` 字段；`null` 表示不写 `proposal.md` 的 `Handoff`，对象表示必须物化为 `proposal.md` 的 `Handoff`
- `workspace`（cross-repo 适用）：`path`、`note`（代码落地 repo，物化进 proposal 的 scope/dependencies）
- 工作目录指针：subagent 必须在 `<workspace>`（当前 repo 或集中工作空间 repo）内运行 CLI

subagent 输出必须保证：
- change 已创建或确认存在（repo-local，不带 `--initiative`）
- `proposal.md` 已完整
- `handoff` 为对象时，`proposal.md` 含非空 `Handoff` 章节，且字段语义与 Slice Plan 保持一致；`handoff: null` 时，不得为了“完整”额外创建 `Handoff`
- cross-repo 时 proposal 的 `Scope`/`Dependencies` 含落地 repo 标注（来自 `workspace.note`）
- 不含 `TODO` / `TBD` / 空章节 / stub-only 内容

## Response Template

注意：下面回答模版里的 `## Handoff` 指的是**本技能对后续流程的交接**，不是 `proposal.md` 内部的切片契约章节。只有存在真实跨切片契约时，才要求在 `proposal.md` 内新增 `## Handoff`。

```md
## Result
- skill: openspec-slices-register
- status: registered | partially-registered | stop
- boundary_check: registration only; no re-splitting, tracking, or implementation

## Core Output
- workspace: null | {path: <workspace>, init_status: created | already-initialized}
- registered_changes:
  - change: export-feature-01-foundation   # {change-name}-{change-num}-{slice-change-name}
    goal: <one line>
    depends_on: []
    creation_mode: subagent-openspec-new-change | subagent-opsx-new | already-existed
    status: created | already-existed
- proposal_status:
  - change: export-feature-01-foundation
    state: complete | already-complete | synced-from-plan | stop-on-conflict
- slice_plan_persisted: <workspace>/openspec/slice-plans/<change_name>.yaml | none  # 未持久化或仅确认前快照为 none
- sequencing_hint: <archive or parallel reminder>

## Handoff
- handoff_to: openspec-continue | apply/verify/archive flow
- handoff_input: registered changes + persisted Slice Plan YAML
- handoff_reason: proposal is already complete, and tracking can read the persisted slice-plan.yaml directly

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
| cross-repo（`workspace.note` 非空） | 在 proposal 的 `Scope`/`Dependencies` 标注切片代码实际落地哪个 repo |

## References

- `references/cli-contract.md`：CLI 语法（`init` / `new change` / `list` / `status` / `validate` 等）、init/workspace-setup 限制、kebab-case 规则
- `references/landing-patterns.md`：single-repo / cross-repo 集中工作空间示例、完整 proposal 写法、幂等与冲突处理
