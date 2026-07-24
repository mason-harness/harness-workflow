# 本仓 Skill 撰写规范

## 只写本仓特有约束

> **更新日期**：2026-07-24

## 与全局 create-skill 的关系（避免重复）

通用 skill 撰写——目录结构、frontmatter 基础写法、Progressive Disclosure 分级加载、Skill Creation Checklist、创建流程——已由全局安装的 `create-skill` 覆盖（位于 `~/.claude/skills/create-skill/SKILL.md`，不属本仓库）。**本指南不重述通用写法，只记录本仓库 8 个现有 skill 实际遵循、且 create-skill 未覆盖的特有约束。**

新增或修改本仓 skill 时：通用结构以 `create-skill` 为准，本仓特有约束以本指南为准。两者不重叠，互补。

## 适用边界

本指南适合：
- 在本仓库新增一个 skill
- 重构或对齐现有 skill 的结构
- 评审一个 skill 是否符合本仓规约

不适合：
- 学习通用 skill 写法（用 `create-skill`）
- 查某 skill 的具体业务规则（读对应 `skills/*/SKILL.md` 原文）

---

## frontmatter 与 description 约束

| 项 | 本仓约定 | 证据 |
|---|---|---|
| `name` | 与目录同名的 kebab-case，8/8 一致 | 全部 SKILL.md frontmatter |
| `description` 语态 | 英文以 `Use when ...` 开头；中文以"当遇到……时使用"开头 | 7 个英文 + `harness-claude-setup:3` 中文 |
| `description` 结构 | 用 ` - ` 分两段：前段=触发条件，后段=产出/动作 | 全部 8 个 |
| `description` 必含 | 触发条件（8/8）；**边界声明**（6/8），如 `without replacing the OpenSpec lifecycle.` / `cannot be rationalized away by the model.` | `harness-openspec-apply:3`、`harness-hook-setup:3` 等 |
| `description` 长度 | 40–65 词（中文约 70 汉字） | 实测各 description |

**底线**：description 必须能回答"什么时候触发 + 这个 skill 做什么 + 不替代什么"。前两者是 create-skill 也要求的，第三者是本仓 overlay/流水线 skill 的特有约定。

---

## 章节骨架：选配构件，而非固定模板

**本仓不存在"8 项全有"的固定骨架。** 跨 8 个 skill，章节是一组**反复出现的构件，按 skill 类型选配**。

| 章节 | 出现 | 适用 skill 类型 |
|---|---|---|
| Overview / 核心原则 | 8/8 | 全部 |
| Workflow / 工作流程 | 8/8 | 全部 |
| Quick Reference（速查表） | 8/8 | 全部 |
| 规则章节（Critical/Core/Discipline/Evidence Gate Rules） | 8/8（名称不统一） | 全部 |
| When to Use / 使用时机 | 5/8 | setup 型、overlay 型 |
| Boundaries / 职责边界 | 4/8 | overlay 型、流水线型 |
| Common Mistakes | 3/8 | overlay 型 |
| STOP / BLOCKED Handling | 2/8 | overlay 型（apply/verify） |
| Subagent Delegation | 2/8 | overlay 型 |
| Minimal Draft Structure | 2/8 | overlay 型 |
| Response Template（固定输出模版） | 3/8 | 流水线型（slices 家族） |
| Handoffs / 交接 | 7/8 | 除 openspec-setup 外 |
| 参考资源 / References | 6/8 | 有 references 的 skill |

按 skill 类型选配：

- **setup 型**（harness-claude-setup / harness-openspec-setup）：核心原则 + 工作流程 + 文档职责模型 + 质量检查清单 + 输出要求。
- **overlay 型**（harness-openspec-apply / harness-openspec-verify）：Overview + When to Use + Responsibilities Boundary + Quick Reference + Workflow + 专属规则 + STOP/BLOCKED + Handoff + Common Mistakes + Minimal Draft Structure。
- **流水线型**（openspec-slices-plan / -register / -track）：Overview + Quick Reference + Critical/Core Rules + Boundaries + Workflow + Response Template + Handoffs + Failure Handling + References。

**底线**：按类型选配，不要为了模板完整保留空章节（与 `create-skill` 的 progressive disclosure 一致）。

---

## 本仓特有约束（create-skill 未覆盖）

### 约束 1：overlay skill 不替代生命周期

overlay 型 skill（apply/verify）必须显式声明自己**不是第二套生命周期**，而是 OpenSpec 主生命周期的纪律增强层。

证据：
- `harness-openspec-apply:3` description 结尾 `without replacing the OpenSpec lifecycle.`
- `harness-openspec-apply:10-11`："为 OpenSpec 的 apply 阶段提供 hardness overlay，而不是替代 `/opsx:apply`。"
- `harness-openspec-apply:50`："将本 skill 视为 apply discipline profile，而不是第二套流程"
- verify 对称（`harness-openspec-verify:3,10,50`）

**怎么写**：Overview 首句 + description 结尾 + 正文一处，三处一致地声明"为 X 阶段提供 Y overlay，不替代 `/opsx:X`"。

### 约束 2：配对 skill 用镜像骨架

apply ↔ verify 是配对 overlay，**刻意采用镜像骨架**——两者章节几乎一一对应（Overview / When to Use / Responsibilities Boundary / Quick Reference / Workflow / 专属规则 / STOP-BLOCKED / Subagent Delegation / Handoff / Common Mistakes / Minimal Draft Structure）。

镜像的目的：让二者间的 handoff 契约有**稳定的对称承载点**——apply 产出"已运行命令+结果摘要+未解决 blocker"，verify 消费"命令/输出/时间戳证据"。

证据：apply:29-253 与 verify:29-185 的逐章对照。

slices 家族（plan → register → track）同理：共享 Overview / Quick Reference / Critical Rules / Boundaries / Workflow / Response Template / Handoffs / Failure Handling / References 骨架。

**怎么写**：同族配对/流水线 skill，新成员要镜像同族骨架，不要自创一套。

### 约束 3：流水线 skill 用固定 Response Template + `Warnings - None`

流水线型 skill（slices-plan / -register / -track）必须：
- 用固定 Response Template（共享骨架：Result / Core Output / Handoff / Next Step / Warnings）。
- **无警告时也必须保留 `## Warnings` 并写 `- None`**——"无警告"是显式信号，不是缺失。

证据：
- `openspec-slices-plan:107`、"track:76"："最终回答必须使用以下固定结构与顺序；无警告时也必须保留 `## Warnings` 并写 `- None`。"
- `openspec-slices-plan:36`："最终回答必须使用本技能固定模版，不得改成自由摘要或执行日志"
- `openspec-slices-register:33`："最终回答只输出登记结果，不重做拆分、不汇报进度"

**怎么写**：固定模版不得改成自由摘要或执行日志；Warnings 节恒保留。

### 约束 4：用"不做"清单互指兄弟 skill，形成单一职责出口网

每个流水线/overlay skill 用**否定清单**把兄弟 skill 的工作显式排除，形成"单一决策出口"网。

证据：
- `openspec-slices-plan:35`："本技能是 Change 拆分的唯一决策出口；其他技能不得自行拆分"
- `openspec-slices-plan:47-53` 的"不做"清单拒绝 register 的事
- `openspec-slices-track:46-52`："不做拆分决策（交 plan）""不做登记落地（交 register）"
- `openspec-slices-register:32`："不重做拆分、不汇报进度"
- `harness-openspec-apply:21-25` 的"不要用于"互指另外三个 harness skill

**怎么写**：用 Boundaries / "不要用于" / "不做" 章节把应交给兄弟 skill 的工作显式列出，且每个职责有且只有一个出口 skill。

### 约束 5：自报 Hardness 层级 + MUST/SHOULD/MAY/STOP/BLOCKED 术语

| 术语 | 含义 | 证据 |
|---|---|---|
| MUST / MUST NOT | 硬约束，不因速度或催促绕过 | `harness-claude-setup:36` |
| SHOULD | 默认行为，可被更强证据覆盖 | `harness-claude-setup:37` |
| MAY | 可选增强 | `harness-claude-setup:38` |
| STOP | 门禁条件；暂停、报告冲突/缺证据/待决策 | `harness-claude-setup:39` |
| BLOCKED | 已尝试合理修复但无法继续；保留未完成并记录 blocker | `harness-openspec-setup/references/hardness-levels.md:9` |

**自报层级**：overlay/流水线 skill 在 Critical Rules 内自报所属 Hardness 层级（如 `openspec-slices-plan:43`"L1-L2"、`openspec-slices-track:42`"L1"）。

**怎么写**：规则用短句命令式逐条 bullet，先写必须做什么再写禁止什么，写可执行约束和完成标准，不写"保证高质量""充分测试"等口号（见 `harness-openspec-setup/references/rule-mapping.md:39-48`）。

### 约束 6：不重复载体（CLAUDE.md / repository/ / OpenSpec / README 职责分离）

skill 不得把稳定项目知识写进 `CLAUDE.md`，不得复制 `CLAUDE.md`/`repository/` 全文，不得把 skill 使用步骤写进 `CLAUDE.md`。

证据：
- `harness-claude-setup:30`："必须区分文档职责"
- `harness-claude-setup:53-54`："`CLAUDE.md` 只保留目标项目长期执行规则，不写'调用某个 skill'的操作步骤"
- `harness-openspec-setup:30-31`："质量原则投影为 artifact 规则，不复制 `CLAUDE.md` 或 `repository/` 全文"

**怎么写**：本仓 skill 撰写时，新增的"skill 怎么写"知识本身也守这条——不重复写进多个载体（本指南只在 docs/，不复制进 repository/ 或 CLAUDE.md）。

### 约束 7：references/ 增量增补，先 SKILL.md 再视情况加

- 草案阶段最小结构只建 `SKILL.md`。
- 只有在反复出现相同模板/契约时，才增补 `references/`（apply/verify 刻意不建 references，由 Minimal Draft Structure 声明）。
- 内容移入 references 是为 token efficiency（`harness-openspec-setup:64-65`）。
- SKILL.md 用**行内引用**（"详见 `references/xxx.md`"）+ **末尾清单**（References 章节）两模式引用。

证据：6/8 有 references，2/8 刻意不建；`harness-openspec-apply:248-253` 与 `harness-openspec-verify:180-185` 的 Minimal Draft Structure。

**怎么写**：先只产 SKILL.md，重复模板出现再拆 references；引用用"详见"行内 + 末尾清单。

### 约束 8：STOP 的软/硬两路径认知

skill 必须认知：自身规则（CLAUDE.md/config.yaml/SKILL.md）是**软 STOP**，模型可能忽略；**硬 STOP** 只有 Hook（PreToolUse）能做。

证据：
- `harness-hook-setup:46-52` 软/硬 STOP 对照表
- `harness-claude-setup:42`："`CLAUDE.md` 通过会话启动注入上下文实现软约束。要转化为硬门禁需配合 Claude Code Hooks（PreToolUse）。"
- `harness-openspec-setup/references/hardness-levels.md:20-31`

**怎么写**：skill 涉及高风险不可逆操作时，不声称自身规则能硬拦，交接给 `harness-hook-setup`（见 apply:21-25 把 Hook 配置交给 hook-setup）。

---

## 快速自检（新增/重构 skill 前）

- [ ] name 与目录同名 kebab-case
- [ ] description 含触发条件 + 产出 + 边界声明
- [ ] 章节按 skill 类型选配，无空章节
- [ ] overlay skill 三处声明"不替代生命周期"
- [ ] 配对/流水线 skill 镜像同族骨架
- [ ] 流水线 skill 用固定 Response Template，Warnings 恒保留（- None）
- [ ] 用"不做"清单互指兄弟 skill，单一职责出口
- [ ] 自报 Hardness 层级，规则用 MUST/SHOULD/MAY/STOP/BLOCKED 短句
- [ ] 不复制 CLAUDE.md/repository/ 全文，不写 skill 操作步骤进 CLAUDE.md
- [ ] 先只产 SKILL.md，重复模板再拆 references
- [ ] 高风险操作不声称自硬拦，交接 harness-hook-setup

---

## 关键证据索引（指向原文，不重述）

| 约束 | 原文位置 |
|---|---|
| overlay 不替代生命周期 | `skills/harness-openspec-apply/SKILL.md:3,10-11,50` |
| apply↔verify 镜像骨架 | `skills/harness-openspec-apply/SKILL.md` 与 `harness-openspec-verify/SKILL.md` 全文对照 |
| slices 家族流水线骨架 | `skills/openspec-slices-{plan,register,track}/SKILL.md` |
| 固定 Response Template + Warnings - None | `openspec-slices-plan:107`、`slices-track:76` |
| 不做清单互指 | `openspec-slices-plan:47-53`、`slices-track:46-52`、`harness-openspec-apply:21-25` |
| MUST/SHOULD/MAY/STOP/BLOCKED | `harness-claude-setup:36-40`、`harness-openspec-setup/references/hardness-levels.md:5-9` |
| 不重复载体 | `harness-claude-setup:30,53-54`、`harness-openspec-setup:30-31` |
| Minimal Draft Structure | `harness-openspec-apply:248-253`、`harness-openspec-verify:180-185` |
| 软/硬 STOP | `harness-hook-setup:46-52`、`harness-claude-setup:42` |
| LLM-readable 规则风格 | `harness-openspec-setup/references/rule-mapping.md:39-48` |

---

## 通用写法不在本指南（指向 create-skill）

以下交给全局 `create-skill`（`~/.claude/skills/create-skill/SKILL.md`），本指南不展开：
- skill 目录结构（`skill-name/SKILL.md` + `scripts/ references/ assets/`）
- frontmatter 基础字段（name + description，max 1024 字符）
- Progressive Disclosure 三级加载
- Skill Creation Checklist 与六步创建流程
