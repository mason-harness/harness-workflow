# openspec-slices-track

用于跨 session 追踪多切片 Slice Plan 的整体进度。

## 适用场景

- 已经通过 `openspec-slices-plan` + `openspec-slices-register` 创建了一组切片 changes
- 想知道当前应该继续哪个切片
- 想判断哪些切片已经 ready，哪些仍 blocked
- 新 session 启动后需要快速恢复整体推进上下文

## 核心输入

- auto-memory 中保存的 Slice Plan 依赖关系
- `openspec list --json` 返回的当前 active changes 状态，以及对缺失切片逐个补查得到的 archived 状态

## 核心输出

- 整体切片进度图
- archived / in-progress / ready / blocked 分类
- 下一步推进建议

## 配套技能

- `openspec-slices-plan`：拆分切片
- `openspec-slices-register`：登记切片
- `openspec-slices-track`：追踪切片推进
