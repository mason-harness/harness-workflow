# OpenSpec Hardness Levels Reference

## Terminology

- **MUST / MUST NOT**: 硬约束，不因速度或用户催促绕过
- **SHOULD**: 默认行为，可被更强证据覆盖
- **MAY**: 可选增强
- **STOP**: 门禁条件；暂停、报告冲突/缺证据/待决策，不继续猜测
- **BLOCKED**: 已尝试合理修复但无法继续；保留未完成状态并记录 blocker

## Four-Level Hardness Model

| Level | Hardness | OpenSpec Phase | Typical Language |
|---|---|---|---|
| L1: Soft Guidance | 极低 | Explore | 建议、优先、可以考虑 |
| L2: Structured Requirement | 中 | Proposal, Specs, Design | 输出必须包含、按以下结构 |
| L3: Hard Constraint | 高 | Tasks, Apply | 必须、不得、只允许、不满足则 |
| L4: Gate / Stop Condition | 最高 | Verify, Archive | STOP、暂停、必须确认 |

## Rule Implementation Mechanism

`config.yaml` 的 `rules` 字段不是独立原语——它通过 OpenSpec 技能读取后注入上下文，本质是 **Rule 的扩展实现**：

```
config.yaml rules
    └─ OpenSpec 技能读取（Skill）
        └─ 注入上下文（等效 Rule）
            └─ 模型自觉遵守（软约束）
```

**关键认知**: config.yaml 规则是"通过 Skill 间接实现的 Rule"，而非独立的强制机制。要实现硬门禁（模型无法绕过的 STOP），需配合 Hook（PreToolUse）。

## Phase-Specific Hardness

- **Explore/Proposal**: L1-L2 - MAY allow assumptions, but assumptions MUST be labeled
- **Specs/Design**: L2 - Structured requirements
- **Tasks/Apply**: L3 - Hard constraints, no bypass
- **Verify/Archive**: L3-L4 - Gates with STOP conditions

## Evidence Requirements

Verify 阶段必须核对证据，合格证据包含：
1. **命令**：完整的执行命令（可复现）
2. **输出**：关键输出内容（成功标志或错误信息）
3. **时间戳**：执行时间
4. **位置**：证据记录在 `verify.md` 的 Evidence 章节

## Task Checkbox Criteria (责任单元判断标准)

Tasks 阶段的 checkbox 必须符合：
- **可观察完成**：有明确的验证命令或可观察结果
- **文件影响范围**：影响文件数 ≤ 5 个（优先指标）
- **代码量边界**：新增/修改代码行数 ≤ 150 行
- **失败独立性**：失败时不会连锁影响其他 checkbox

优先级顺序：可观察完成 > 文件影响范围 > 失败独立性 > 代码量边界
