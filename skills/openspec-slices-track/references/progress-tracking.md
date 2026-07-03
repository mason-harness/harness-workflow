# 进度追踪规则

## 状态映射

基于 OpenSpec CLI 的 active change 列表与逐个补查得到的 archived 状态合并结果，将每个切片映射为以下四类状态：

- `archived` → **已归档**
- `in-progress` → **进行中**
- `ready` → **可启动**
- `blocked` → **被阻塞**

## 判定规则

### 1. archived

满足全部条件：
- 该 change 不在 `openspec list --json` 的 active 列表中
- 且对该 change 运行 `openspec status --change <name> --json` 或等效 CLI 查询后可确认其状态为 `archived`

显示格式：
```text
✅ s01-project-bootstrap      [archived]
```

### 2. in-progress

满足任一条件：
- `totalTasks > 0`
- 或 `completedTasks > 0`
- 且不是 `archived`

显示格式：
```text
🔄 s02-chapter-content-core   [5/12 tasks, 41%]
```

百分比计算：
- `progress = floor(completedTasks / totalTasks * 100)`
- 若 `totalTasks = 0`，不显示百分比

### 3. ready

满足全部条件：
- 当前切片不是 `archived`
- 当前切片不是 `in-progress`
- `depends_on` 中所有前序切片都已 `archived`

显示格式：
```text
⏳ s03-rule-checking-engine   [ready, 依赖已满足]
```

### 4. blocked

满足全部条件：
- 当前切片不是 `archived`
- 当前切片不是 `in-progress`
- `depends_on` 中至少有一个前序切片未 `archived`

显示格式：
```text
🚫 s04-state-machine          [blocked, 等待 s02/s03]
```

## 依赖满足判定

依赖判断只认合并状态中的 `archived`。

- 前序切片 `in-progress` → 仍视为未满足
- 前序切片 `no-tasks` → 仍视为未满足
- 前序切片不存在 → 视为不一致，需警告

## 优先级排序

输出建议时按以下顺序：

1. **继续进行中的切片**
   - 避免上下文切换成本
   - 如果有多个进行中，按 `lastModified` 最新优先

2. **启动 ready 的切片**
   - 依赖已满足，可以立即推进
   - 如果有多个 ready，按 Slice Plan 顺序优先

3. **提示 blocked 的切片**
   - 只报告阻塞原因，不建议直接启动

4. **全部 archived**
   - 输出完成提示，不再建议下一步

## ASCII 模板

```text
┌─────────────────────────────────────────────────┐
│     Novel Generator 切片进度（7 个切片）          │
├─────────────────────────────────────────────────┤
│ ✅ s01-project-bootstrap      [archived]        │
│ 🔄 s02-chapter-content-core   [5/12 tasks]      │
│ ⏳ s03-rule-checking-engine   [ready]           │
│ 🚫 s04-state-machine          [blocked by s02]  │
│ 🚫 s05-volume-management      [blocked by s02]  │
│ 🚫 s06-destiny-weaving        [blocked by s05]  │
│ 🚫 s07-archive-and-export     [blocked by s04]  │
├─────────────────────────────────────────────────┤
│ 📊 整体进度: 1/7 归档, 1/7 进行中               │
└─────────────────────────────────────────────────┘
```

## 不一致检测

如计划源与合并后的 CLI 状态结果不一致，需要提示：

- initiative 中有切片，但 CLI 中缺失
- memory 中有切片，但 CLI 中缺失
- CLI 中有同族切片，但计划源中缺失
- `depends_on` 指向不存在的切片

输出格式建议：

```text
⚠️ 计划源与当前 changes 不完全一致：
- initiative 中声明了 s06-destiny-weaving，但 CLI 未找到
- CLI 中存在 s08-extra-change，但不在当前计划源中
```

## 固定回答模版示例

```md
## Result
- skill: openspec-slices-track
- status: tracked
- boundary_check: tracking only; no registration, re-splitting, or implementation

## Core Output
- plan_source: initiative/novel-generator/tasks.md
- live_status_source: cli
- progress_board: |
    ┌─────────────────────────────────────────────────┐
    │     Novel Generator 切片进度（7 个切片）          │
    ├─────────────────────────────────────────────────┤
    │ ✅ s01-project-bootstrap      [archived]        │
    │ 🔄 s02-chapter-content-core   [5/12 tasks]      │
    │ ⏳ s03-rule-checking-engine   [ready]           │
    │ 🚫 s04-state-machine          [blocked by s02]  │
    └─────────────────────────────────────────────────┘
- summary:
  - archived: 1/7
  - in_progress: 1/7
  - ready: 1/7
  - blocked: 4/7
- recommendation: continue s02-chapter-content-core
- blocked_items:
  - name: s04-state-machine
    waiting_on: [s02, s03]
- consistency_check:
  - None

## Handoff
- handoff_to: openspec-continue
- handoff_input: s02-chapter-content-core
- handoff_reason: 当前已有进行中的切片，优先减少上下文切换

## Next Step
- recommended_action: continue the in-progress slice before starting any ready slice
- requires_user_confirmation: yes

## Warnings
- None
```
