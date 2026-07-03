# OpenSpec Setup Common Mistakes Reference

| Mistake | Fix |
|---|---|
| 将通用质量原则写成长篇宣言 | 投影成 Proposal/Spec/Design/Tasks/Apply/Verify 规则 |
| 把 `config.yaml rules` 写成抽象长段落 | 改成短句 bullet，每条一个要求 |
| 将测试留给 apply 临时决定 | 在 `design` 阶段写独立测试设计，在 `tasks` 阶段写独立测试任务 |
| 将“实现并测试”写成一个 checkbox | 在同一 Change 的 `tasks.md` 内拆分实现与测试 checkbox/任务清单 |
| 把 task 拆成代码行、import、函数内部步骤 | 按责任单元拆分，保留实现细节给 apply 阶段判断 |
| 把测试拆成独立 Change 或写进项目级 tasklist | 测试/验证属于该 Change 的 `tasks.md` |
| apply 自动执行未列测试并改变完成标准 | apply 只执行 `tasks.md`；缺测试是 tasks 设计问题 |
| 初始化时不检查对应组默认规则，直接追加新规则 | 先读取该组已有默认规则，复用并去重后再补充，避免重复堆叠 |
| 单个规则组无限增补，导致 proposal/specs/design/tasks/apply/verify 过长 | 单个规则组最好不要超过 10 条；优先合并同义项、删除弱表达 |
| 把大需求写成巨大 Change | 创建 Change 前必须先做 right-sized Change 评估；超出单个可验证 Change 边界时先拆分 |
| 未做 Change 尺度判断就直接开始 | 必须先判断是否为单个可验证 Change；若跨仓、跨团队、影响面过大或回滚边界不清，先拆分再继续 |
| 让轻量需求也强制走复杂拆分流程 | 保留拆分护栏，但对轻量需求直接进入普通 Change 流程 |
| 删除 `openspec/AGENTS.md` | 保留；需要时用 `openspec update` 再生成 |
| 把 brownfield unknown 写成规则 | 只有 Fact 写成规则；Unknown/Validation Gap 进入报告或 assumptions |
| Apply/Verify 写成软建议而非硬约束 | 使用 MUST/STOP；Apply/Verify 必须是 high-hardness |
| 保留不存在命令在 validation gates | 报告 stale command 并移除或询问 |
| 无证据报告验证成功 | 列出 skipped checks 和 reasons |
| 测试失败后不重试或无限乱试 | 使用 progressive repair loop：最小修复、窄测试、判断进展、达到 BLOCKED 时停止 |
| 静默解决 `CLAUDE.md`/知识库/OpenSpec 冲突 | STOP 并报告冲突，使用 conflict priority 仲裁 |
| apply 规则未封堵“改动小不需要测试” | 补充规则：“测试失败不得勾选完成，无例外”“命令未运行不得勾选完成，无例外” |
| verify 规则未要求证据 | 补充规则：“证据必须包含命令、输出、时间戳”“证据缺失视为未完成，无例外” |
| archive 规则未设门禁 | 补充规则：“tasks 未完成必须告警并 STOP”“verify 有 CRITICAL 项不得归档” |

## Examples: Wrong vs Right

### ❌ 不复制 `CLAUDE.md` 或知识库

**错误**：把完整编码规则复制到 `context`，或把知识库全文复制进 OpenSpec

**正确**：`repository/` 写稳定项目上下文；`config.yaml rules` 写 artifact 生成护栏；详细编码/测试命令保留在 `CLAUDE.md`

### ❌ 不合并实现和测试

**错误**:
```md
- [ ] 实现用户创建接口并补充测试
```

**正确**:
```md
## Implementation
- [ ] 实现用户创建接口
- [ ] 接入用户创建错误处理

## Test Design
- [ ] 明确用户创建成功/失败/边界场景与证据要求

## Tests and Verification
- [ ] 补充用户创建成功路径单元测试：<命令/范围/期望/证据位置>
- [ ] 运行 typecheck：<命令>，期望 0 error
```

### ❌ 不把 task 拆成微操作

**错误**：创建文件、添加 import、写 if、添加 return 各自一个 checkbox

**正确**：按可验证责任单元拆分，例如“实现用户创建主流程”“添加重复邮箱和非法输入错误处理”
