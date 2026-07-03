# Slice Plan Memory 格式

## 目标

为 `openspec-slices-track` 提供跨 session 的依赖关系来源。

`openspec list --json` 只能告诉我们“当前有哪些 active change、每个 change 的 task 状态”，
但无法告诉我们“这些切片之间的依赖关系是什么”，也不会直接枚举已归档切片。

在 `cross-repo` 场景下，首选来源应是 initiative `tasks.md`；memory 主要用于 `single-repo`、历史兼容或显式降级场景，因此只需持久化 Slice Plan 的最小必要信息。

## 推荐 memory 文件格式

```markdown
---
name: novel-generator-slice-plan
description: Novel Generator 多切片 Slice Plan，供 openspec-slices-track 跨 session 读取依赖关系
metadata:
  type: project
---

Novel Generator 切片方案（7 个切片）。

**Why:** `openspec list --json` 只能返回 active change 状态，不能表达切片依赖关系，也不会直接枚举已归档切片；需要单独持久化 sequencing_rule 与 depends_on 供跨 session 推进使用。
**How to apply:** 新 session 中如要判断下一步该推进哪个切片，先确定计划源；`cross-repo` 优先读取 initiative `tasks.md`，`single-repo` 或显式降级场景再读取本 memory，然后结合 `openspec list --json` 与对缺失切片逐个补查的状态结果，判断 archived / in-progress / ready / blocked。

**Sequencing Rule:** archive-N-before-N+1

**Slices:**
- s01-project-bootstrap: []
- s02-chapter-content-core: [s01-project-bootstrap]
- s03-rule-checking-engine: [s01-project-bootstrap, s02-chapter-content-core]
- s04-state-machine-and-verdicts: [s02-chapter-content-core, s03-rule-checking-engine]
- s05-volume-management: [s02-chapter-content-core]
- s06-destiny-weaving: [s05-volume-management]
- s07-archive-and-export: [s04-state-machine-and-verdicts, s05-volume-management, s06-destiny-weaving]
```

## 最小必需字段

- `Sequencing Rule`
- `Slices`
- 每个切片的完整 change 名称
- 每个切片对应的 `depends_on`

## 可选增强字段

可以按需增加：

- `Current recommendation`: 当前推荐先做哪个
- `Archived milestones`: 某切片何时归档
- `Parallel groups`: 哪些切片可以并行

但这些都不是 `openspec-slices-track` 的最低要求。

## 读取策略

`openspec-slices-track` 的计划源读取顺序应为：

1. `cross-repo`：优先读取 initiative `tasks.md`
2. `single-repo`、历史场景或显式降级：读取 memory

读取 memory 时只需要解析：

1. 当前是不是某个 Slice Plan
2. 切片名字列表
3. 每个切片依赖谁
4. sequencing rule 是什么

不要把 memory 当作实时状态来源，也不要把 initiative 文本状态当作实时状态来源。

实时状态始终以 `openspec list --json` 与 `openspec status --change <name> --json` 为准。

## 更新时机

建议在以下时机更新该 memory：

1. `openspec-slices-register` 完成初次登记后（仅当没有 initiative 作为首选计划源时）
2. 切片被新增、合并、重命名后
3. sequencing rule 被修改后

不建议在每次 task 完成后更新，因为 task 级进度不应写入 memory。

## 边界

这个 memory 只存：
- 切片关系
- 推进顺序
- 非显而易见的流程约束

不要存：
- 每个切片当前 task 完成百分比
- 临时实现细节
- 具体代码路径
- 一次性 session 对话内容
