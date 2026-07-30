# Slice 工作流端到端示例

> **更新日期**：2026-07-30
> 本文为补充参考资料，展示从需求输入到最后一个切片归档的完整生命周期。正式规则以各 `skills/*/SKILL.md` 与 `repository/` 为准。

## 目的

三个 slice 技能各自有完整示例，但缺少跨技能、跨 session 的端到端串联演示。本文用一个中性场景串联 `openspec-slices-plan → openspec-slices-register → openspec-slices-track`，展示：

- 切片决策如何从 plan 流向 register
- register 如何物化 plan 的边界（changes + proposal + slice-plan.yaml）
- track 如何跨 session 恢复进度
- 相对路径 `workspace.path` 如何在三技能间一致传递

示例名均为中性示意（`export-feature`），不代表特定项目类型或技术栈。

---

## 场景：用户导出功能（单仓多切片）

**需求**：用户可导出数据为 CSV，含进度条、错误提示、取消操作。
**项目形态**：单一项目、单仓。

---

### Step 1：openspec-slices-plan

**输入**：用户需求 + `openspec list --json`（确认无既有同名 change）。

**判定**：3 个用户可感知价值点（基础导出 / 进度反馈 / 错误韧性）→ **垂直业务切片，顺序依赖**。

**输出**（Slice Plan 片段，`user_confirmed: false`）：

```yaml
Slice Plan (user_confirmed: false)
mode: single-workspace
workspace: null              # 单仓，用当前 repo 的 openspec/
change_name: export-feature
sequencing_rule: archive-N-before-N+1
slices:
  - sequence: "01"
    name: foundation
    goal: CSV export golden path
    depends_on: []
    scope:
      in: [CSV 格式导出, 同步查询, 下载触发]
      out: [进度条（交 02）, 取消操作（交 02）]
    handoff: null
  - sequence: "02"
    name: progress-feedback
    goal: Add progress bar and cancel
    depends_on: ["01"]
    scope:
      in: [进度条, 取消操作]
      out: [错误处理（交 03）]
    handoff:
      handoff_to: export-feature-03-error-resilience
      artifacts/contracts: [导出状态新增 progressPercent/processedRows/cancelledAt]
      ready_signal: [02 已归档，状态字段可读]
      consumer_expectations: [03 可基于 cancelled|failed|complete 判断重试]
  - sequence: "03"
    name: error-resilience
    goal: Add error handling and retry
    depends_on: ["01", "02"]
    scope:
      in: [错误提示, 重试逻辑]
      out: []
    handoff: null
```

**plan 边界**：只声明，零写文件，不跑 `openspec init`。用户确认 → `user_confirmed: true`。

---

### Step 2：openspec-slices-register

**输入**：确认的 Slice Plan（`user_confirmed: true`）。

**执行**：
1. 校验 `user_confirmed: true`、必填字段完整；`workspace: null` → 用当前 repo 的 `openspec/`。
2. 逐切片启动 subagent 创建 repo-local change（**不带 `--initiative`**）：
   - `openspec new change "export-feature-01-foundation" --goal "CSV export golden path"`
   - `openspec new change "export-feature-02-progress-feedback" --goal "Add progress bar and cancel"`
   - `openspec new change "export-feature-03-error-resilience" --goal "Add error handling and retry"`
3. subagent 当场写完整 `proposal.md`（Goal/Context/Dependencies/Scope/Requirements/Assumptions/Non-Goals；02 额外写 `## Handoff`，物化 plan 的 handoff 对象）。
4. 持久化 `openspec/slice-plans/export-feature.yaml`（确认 Slice Plan 原文）。

**输出**：

```md
## Core Output
- workspace: null
- registered_changes:
  - export-feature-01-foundation (created)
  - export-feature-02-progress-feedback (depends on 01, created)
  - export-feature-03-error-resilience (depends on 01,02, created)
- proposal_status: 三者均 complete
- slice_plan_persisted: openspec/slice-plans/export-feature.yaml
- sequencing_hint: Archive 01 before starting 02
```

**register 边界**：只登记 + 物化 + 持久化 yaml，不实现、不追踪、不归档。

---

### Step 3：openspec-slices-track（session A，首次追踪）

**输入**：无显式输入（读取计划源 + CLI 状态）。

**执行**：
1. 扫描 `openspec/slice-plans/*.yaml` → 唯一 `export-feature.yaml` → 直接读取。
2. 运行 `openspec list --json` → 三个 active changes，均 `no-tasks`（尚未生成 tasks.md）。
3. 依赖分析：01 无依赖 → ready；02/03 依赖 01 → blocked。

**输出**：

```
Progress Board
  01 foundation        [ready    ] depends: -
  02 progress-feedback [blocked  ] depends: 01
  03 error-resilience  [blocked  ] depends: 01,02

Recommendation: 启动 export-feature-01-foundation（唯一 ready）
```

---

### Step 4：推进切片 01

由 apply/verify 阶段执行（非 slice 技能）：

```
/opsx:apply export-feature-01-foundation  → harness-openspec-apply（task preflight）
/opsx:propose ... /opsx:tasks ...         → 生成 specs/tasks
/opsx:verify export-feature-01-foundation → harness-openspec-verify（证据门禁）
/opsx:archive export-feature-01-foundation  → 归档
```

归档后，`export-feature-01-foundation` 移入 `openspec/changes/archive/`，不再出现在 `openspec list --json` 的 active 列表。

---

### Step 5：openspec-slices-track（session B，新 session 恢复）

**输入**：无（新 session，无上下文记忆）。

**执行**：
1. 扫描 `openspec/slice-plans/*.yaml` → 唯一 `export-feature.yaml` → 读取依赖关系。
2. `openspec list --json` → active: `export-feature-02`、`export-feature-03`（01 已不在）。
3. 01 不在 active 列表 → 运行 `openspec status --change export-feature-01-foundation --json` 补查 → 确认 archived。
4. 依赖分析：01 archived → 02 依赖已满足 → ready；03 仍依赖 02 → blocked。

**输出**：

```
Progress Board
  01 foundation        [archived ] depends: -
  02 progress-feedback [ready    ] depends: 01(✓ archived)
  03 error-resilience  [blocked  ] depends: 01(✓),02

Recommendation: 启动 export-feature-02-progress-feedback
```

**track 的价值**：新 session 无需任何上下文记忆，仅凭持久化的 `slice-plan.yaml` + CLI 状态即可重建进度，识别"01 已归档、02 现已就绪"。

---

### Step 6：推进切片 02 → 03

重复 Step 4 流程，归档 02；再次调用 track（Step 5 流程）→ 03 ready；推进并归档 03。

---

### Step 7：openspec-slices-track（最终）

**执行**：
1. 读取 `export-feature.yaml`。
2. `openspec list --json` → active 列表为空。
3. 逐个 `status --change` 补查 → 01/02/03 均 archived。

**输出**：

```
Progress Board
  01 foundation        [archived ] depends: -
  02 progress-feedback [archived ] depends: 01(✓)
  03 error-resilience  [archived ] depends: 01(✓),02(✓)

Recommendation: 无（全部已归档，批次完成）
```

---

## cross-repo 变体：相对路径的传递

若上述场景改为跨仓（auth-service / api-gateway / web-frontend），workspace 声明变为：

```yaml
workspace:
  kind: integration-repo
  path: ../oauth-migration          # 相对于目标项目根目录
  path_base: target-project-root   # 默认值
  init_status: required
  note: 切片代码分别落地 auth-service/api-gateway/web-frontend
```

> 相对路径的设计目的：让计划随项目一起迁移/克隆时无需改路径，**数据可移植**。绝对路径会在换机器、换目录时失效，使计划与目标项目脱钩。

三技能对 `path` 的处理一致：
- **plan**：只声明相对路径（不解析、不创建）。
- **register**：解析 `../oauth-migration` 为绝对路径 → `openspec init --tools none --force <resolved_path>` → 在工作空间内登记 changes → 持久化 `<resolved_path>/openspec/slice-plans/oauth-migration.yaml`。
- **track**：读取 yaml 的 `workspace.path`，按 `path_base` 解析为绝对路径 → 扫描 `<resolved_path>/openspec/slice-plans/` → 聚焦 → 补查 CLI 状态。

全程相对路径只在 yaml 中声明一次，解析逻辑由 register/track 各自承担，无需用户在多个技能间重复提供绝对路径。

---

## 要点总结

| 阶段 | 技能 | 输入 | 输出 | 边界 |
|------|------|------|------|------|
| 决策 | plan | 需求 + list | Slice Plan（user_confirmed） | 零写文件 |
| 登记 | register | 确认 Slice Plan | changes + proposal.md + slice-plan.yaml | 不实现、不追踪 |
| 追踪 | track | 无（读 yaml + CLI） | progress board + recommendation | 只读，不改状态 |

- 计划真相（依赖/序号/scope/handoff）只活在 `slice-plan.yaml` 一处，由 register 写、track 读。
- 实时状态永远以 CLI 为准，slice-plan.yaml 不承担状态。
- track 跨 session 恢复靠"持久化 yaml + CLI 状态"，不靠对话记忆。
