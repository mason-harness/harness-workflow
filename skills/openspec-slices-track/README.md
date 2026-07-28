# openspec-slices-track

用于跨 session 追踪多切片 Slice Plan 的整体进度。

## 适用场景

- 已经通过 `openspec-slices-plan` + `openspec-slices-register` 创建了一组切片 changes
- 想知道当前应该继续哪个切片
- 想判断哪些切片已经 ready，哪些仍 blocked
- 新 session 启动后需要快速恢复整体推进上下文

## 核心输入

- 工作空间内 `<workspace>/openspec/slice-plans/<change_name>.yaml`（首选计划源，含依赖与 sequencing_rule）；无 openspec 目录时回退 auto-memory 或用户提供的 Slice Plan
- `openspec list --json` 返回的当前 active changes 状态，以及对缺失切片逐个补查得到的 archived 状态（`openspec status --change <name> --json`）

## 核心输出

- 整体切片进度图
- archived / in-progress / ready / blocked 分类
- 下一步推进建议

## 配套技能

- `openspec-slices-plan`：拆分切片
- `openspec-slices-register`：登记切片并持久化 slice-plan.yaml
- `openspec-slices-track`：追踪切片推进
