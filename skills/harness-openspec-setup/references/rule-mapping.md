# OpenSpec Rule Mapping Reference

将通用质量目标写成 `openspec/config.yaml` 的 artifact 分组规则：

## Artifact Rules by Phase

| Artifact | Hardness | 必须约束 |
|---|---|---|
| **proposal** | L2（中） | 背景、用户价值、范围、非目标、依赖、假设、验证方式、现有行为搜索、right-sized Change 检查、当无法定义为单个可验证 Change 时必须先拆分、来源/假设标注、对应 Change List entry |
| **specs** | L2（中高） | 可观察行为、Given/When/Then、成功/失败/边界/权限场景；无实现任务 |
| **design** | L2（中） | 方案、替代方案、模块边界、**必须包含独立测试设计**（测试层级、关键场景、失败/边界场景、证据预期、完成前证明点）、迁移/回滚；检测实现/spec divergence |
| **tasks** | L3（高） | 阶段分组；实现、契约/文档、**测试/验证拆成独立 checkbox 或独立任务清单**；测试项写命令/范围/期望/失败处理/证据位置；每个 checkbox 是责任单元（独立可执行、可验证、可勾选） |
| **apply** | L3（高） | 只按 `tasks.md` 顺序执行已列任务；测试失败不得勾选完成；测试失败时定位、修复并重跑；发现 task 设计缺口时暂停反馈；STOP on 缺命令、不存在命令、未列必要测试、spec divergence |
| **verify** | L3（很高） | 核对 artifact completeness、实现/spec 对齐、命令证据、测试结果、未解决假设；证据缺失视为未完成；不只检查测试通过 |

## Anti-Rationalization Rules

Apply/Verify 规则必须显式封堵自我合理化：

### Apply Phase
- “测试失败不得勾选完成，无例外”
- “命令未运行不得勾选完成，无例外”
- “不得以‘改动小不需要测试’跳过测试”

### Verify Phase
- “证据必须包含命令、输出、时间戳”
- “证据缺失视为未完成，无例外”
- “不得以‘已经验证过’代替实际证据”

### Archive Phase
- “tasks 未完成必须告警并 STOP”
- “verify 有 CRITICAL 项不得归档”

### Proposal Phase (Change Sizing Trigger)
- “创建任何 Change 前必须先完成 right-sized Change 评估，无例外”
- “不得以‘需求不大’‘已知道怎么做’‘用户催促开始’跳过 Change 尺度判断”
- “当需求跨仓、跨团队、影响面过大、回滚边界不清，或无法定义为单个可验证 Change 时，必须先拆分再继续”

## LLM-Readable Rule Style

写入 `config.yaml rules` 时：
- 使用短句、命令式、逐条 bullet；每条只表达一个要求
- 先检查对应组默认规则，优先复用与去重，避免重复添加同义规则
- 先写必须做什么，再写禁止什么
- 写可执行约束和完成标准；不要写“保证高质量”“充分测试”等口号
- 对 `design` 明确独立测试设计，对 `tasks/apply/verify` 明确 checkbox 粒度、命令、期望结果、失败处理和证据要求
- 单个 `proposal/specs/design/tasks/apply/verify` 规则组最好不要超过 10 条；超过时优先合并、删重、压缩弱规则
- 保持 `context` 简短；详细编码规则留在 `CLAUDE.md`，稳定项目知识留在 `repository/`

## Progressive Test Repair Loop

When a test/verification task fails during apply:

1. Capture failing command and output
2. Identify smallest likely cause
3. Apply minimal fix within Change boundary
4. Re-run narrowest test first, then broader check
5. Repeat while failures change and progress evident
6. **Stop as BLOCKED when**:
   - Repeated attempts no longer change failure signature
   - Fix requires scope outside approved Change
   - Evidence insufficient to identify root cause
7. Keep verification checkbox incomplete and record command/output/blocker

Encode this loop in `config.yaml rules.apply` or `rules.verify`.
