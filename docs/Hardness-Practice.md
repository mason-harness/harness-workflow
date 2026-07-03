# Hardness 实践指南

## 基于 OpenSpec + Claude Code 的硬约束工程

> **更新日期**：2026-07-03  

## 什么是 Hardness

**Hardness** = 对 AI Agent 输出的**约束强度**。它不是“最好这样做”，而是“不满足就不能继续”。

> 类比：LLM 是蛮力十足但方向感差的马，Hardness 就是那副把力气引到正确方向的马具。

### 设计目标

- 承认 AI 漂移不可避免
- 把错误从“事后发现”前移到“阶段门禁”中暴露
- 让漂移保持在可观察、可回滚、可追溯的范围内
- 把人类监督从“逐行盯代码”提升为“在关键门禁点做决策”

**Hardness 不是追求完全无人值守，而是追求可控自动化。**

## 快速开始：5 分钟最小配置

只需配置 2 个文件，即可先解决最常见的两类问题：**跳过验证** 和 **未完成就归档**。

### 1. `openspec/config.yaml`

```yaml
schema: spec-driven
rules:
  apply:
    - "测试失败不得勾选完成"
    - "命令未运行不得勾选完成"
  archive:
    - "tasks 未完成必须告警并 STOP"
    - "verify 有 CRITICAL 项不得归档"
```

### 2. `CLAUDE.md`

```markdown
## Before Finishing（每次完成前必须检查）
- [ ] 运行相关测试，全部通过
- [ ] 确认 tasks.md 全部勾选

## 危险操作（STOP）
- 涉及数据删除 → STOP，必须人工确认
```

### 预期效果

- 减少“测试未跑就宣称完成”
- 减少“tasks 未完成但直接归档”
- 让最关键的执行纪律先落地，再逐步增强

---

## 第一部分：核心概念

## 控制论视角：Hardness 在系统中处于什么位置

Hardness 的核心定位不是替代人类，而是把系统拆成**不同速度、不同权限**的三层回路：

| 回路 | 主要职责 | 响应速度 | 决策权限 |
|---|---|---|---|
| **LLM Agent** | 生成、修改、执行 | 快 | 低 |
| **Rules** | 检查结构、约束流程、要求证据 | 中 | 中 |
| **Human** | 授权高风险操作、处理冲突、最终验收 | 慢 | 高 |

### 人在回路中（Human-in-the-Loop）

Hardness 默认假设：

- 确定性执行可以自动化
- 高风险、不可逆、语义冲突的决策必须留给人类

因此它更适合：

- 交互式开发
- Agent 辅助编码
- 需求到实现的规范驱动流程

而不适合：

- 无人值守的夜间自动化
- 无法响应 STOP 的纯 CI/CD 流程

### STOP 既是门禁，也是诊断信号

STOP 不只是“暂停”，也是流程设计是否健康的报警器。

| STOP 场景 | 表层作用 | 深层信号 |
|---|---|---|
| 高频高风险授权 | 防止误操作 | tasks 设计可能过粗 |
| 规则冲突 | 避免模型卡死 | rules 之间可能互相矛盾 |
| 测试反复失败 | 防止假完成 | 可能应回退到 Design / Tasks 阶段 |

## Hardness 的四层模型

### 第 1 层：Soft Guidance（软引导）

**适用阶段**：Explore、需求澄清  
**典型用语**：建议、优先、可以考虑  
**Hardness 值**：极低

```yaml
explore:
  guidance: |
    可以提出 2-3 个技术方案并比较优劣。
    不确定的内容请标注 [NEEDS CLARIFICATION]。
```

### 第 2 层：Structured Requirement（结构化要求）

**适用阶段**：Proposal、Specs、Design  
**典型用语**：输出必须包含、按以下结构  
**Hardness 值**：中

```yaml
rules:
  proposal:
    - "Proposal 必须包含 Context、Scope、Non-goals、Depends On、Blocks"
    - "Non-goals 至少列出 3 项明确排除的能力"
  specs:
    - "每个 Spec 必须包含至少一个 Acceptance Scenario"
    - "Acceptance Scenario 必须使用 Given-When-Then 格式"
```

### 第 3 层：Hard Constraint（硬约束）

**适用阶段**：Tasks、Apply、Verify  
**典型用语**：必须、不得、只允许、不满足则  
**Hardness 值**：高

```yaml
rules:
  tasks:
    - "每个 checkbox 必须是独立可执行、可验证、可勾选的责任单元"
    - "测试/验证 checkbox 必须写明命令、范围、期望结果、失败处理"
  apply:
    - "只执行 tasks.md 已列 checkbox"
    - "测试失败不得勾选完成"
```

### 第 4 层：Gate / Stop Condition（门禁）

**适用阶段**：Archive、发布、删除  
**典型用语**：STOP、暂停、必须确认  
**Hardness 值**：最高

```yaml
rules:
  archive:
    - "tasks 未完成必须告警并 STOP"
    - "verify 有 CRITICAL 项不得归档"
  security:
    - "涉及删除文件/目录的操作，STOP，必须人工确认"
```

## 证据系统

### 为什么需要证据

“必须测试通过”不等于“已经测试通过”。Hardness 用**客观证据**对抗模型的语言能力。

| LLM 的擅长 | Hardness 的对策 |
|---|---|
| 语言上把任务说得像完成了 | 要求可观察证据 |
| 用逻辑推断“应该正确” | 要求实际运行结果 |
| 用摘要替代验证过程 | 要求证据位置、命令、时间戳 |

### 合格证据的最小要素

一个合格的证据至少包含：

1. **命令**：完整、可复现
2. **输出**：关键结果或 summary
3. **时间戳**：何时执行
4. **位置**：记录在哪里

### 证据记录位置

| 证据类型 | 记录位置 | 格式要求 |
|---|---|---|
| 测试结果 | `openspec/changes/<id>/verify.md` 的 Evidence 章节 | 命令 + 输出摘要 + 时间戳 |
| 类型检查 | `verify.md` | 命令 + `0 error` 或错误数 |
| 手动验证 | `verify.md` | `[MANUAL]` + 步骤 + 结果 |

### 证据模板

```markdown
## Evidence

### 单元测试
- **命令**: `npm test -- user.service.test.ts`
- **执行时间**: 2026-06-23 14:30:25
- **结果**: 全部通过（12 passed, 0 failed）
- **输出摘要**:
  ```
  PASS  src/services/user.service.test.ts
    UserService
      ✓ should create user successfully (25ms)
      ✓ should validate email format (12ms)
  Tests: 12 passed, 12 total
  ```

### 类型检查
- **命令**: `npm run typecheck`
- **执行时间**: 2026-06-23 14:32:10
- **结果**: 0 error

### 手动验证
- **类型**: [MANUAL]
- **验证内容**: 验证登录页面 UI 一致性
- **步骤**:
  1. 启动开发服务器 `npm run dev`
  2. 访问 http://localhost:3000/login
  3. 检查表单对齐、按钮样式、错误提示位置
- **结果**: 通过（UI 符合设计稿）
```

### 证据缺失规则

```yaml
rules:
  verify:
    - "证据缺失时，该 task 视为未完成"
    - "发现证据缺失时，必须在 verify.md 标注 [EVIDENCE MISSING]"
    - "不得以‘已经验证过’代替实际证据"
```

### 证据可信度与防造假

LLM 可以生成“像是真的”输出，因此证据还需要考虑**来源可信度**。

| 证据类型 | 产生方式 | 可信度 | 适用场景 |
|---|---|---|---|
| **工具生成证据** | CI、hook、测试报告文件 | 高 | 关键测试、构建、类型检查 |
| **LLM 记录证据（可复现）** | 记录命令与关键输出，并允许 Verify 重跑 | 中 | 单元测试、lint |
| **LLM 自报告证据** | 只写“已验证” | 低 | 不可接受 |

建议增加以下校验：

```yaml
rules:
  verify:
    - "必须重新运行关键测试/验证命令，不得仅依赖 Apply 阶段记录"
    - "命令输出必须包含可验证的结论行"
    - "发现明显伪造或无法复现的证据时，标注 [EVIDENCE FORGED]"
```

### 何时委托给 Subagent

复杂证据核对适合委托给 **Subagent**，尤其是：

- `verify.md` 很长
- 需要跨多个文件对账
- 读取大量测试输出只为得出一个结论

典型适用场景：

1. 检查每个 task 是否具备命令、输出、时间戳
2. 核对 tasks / verify / artifacts 是否一致
3. Archive 前做门禁扫描

**注意**：Subagent 适合“校验并回传结论”，不适合替代 Hook 做强拦截。

## STOP 行为规范

### STOP 的两种实现路径

| 类型 | 实现原语 | 触发方式 | 可绕过？ | 适用场景 |
|---|---|---|---|---|
| **软 STOP** | Rule（`CLAUDE.md` / `config.yaml`） | 模型读取规则后自行暂停 | 是 | 需求边界、证据缺失、tasks 未完成 |
| **硬 STOP** | Hook（PreToolUse） | 工具调用前被拦截 | 否 | 删除、发布、危险命令 |

### 什么时候该用硬 STOP

高风险且不可逆的操作，不应只依赖软 STOP，例如：

- 数据删除
- 生产部署
- `terraform apply`
- 大范围覆盖或回滚

如果要把这些升级成不可绕过的门禁，应使用 Hook。

### STOP 的输出格式

```markdown
## ⛔ STOP - 需要人工确认

### 触发原因
<具体说明触发了哪条规则>

### 风险说明
<说明该操作的风险和影响范围>

### 需要的决策
- [ ] 确认风险可接受，继续执行
- [ ] 调整方案，避免该操作
- [ ] 暂时跳过，稍后处理

### 如果确认继续
请明确回复：“确认继续，我理解风险”
```

### STOP 后的恢复路径

```yaml
rules:
  stop_behavior:
    - "触发 STOP 时必须输出：触发原因、风险说明、需要的用户决策"
    - "STOP 后不得继续执行任何改动操作（读取操作允许）"
    - "用户确认后必须在 verify.md 记录：STOP 原因、用户确认内容、执行时间"
```

### STOP 场景处理矩阵

| 用户响应 | Agent 行动 | 记录要求 |
|---|---|---|
| **确认继续** | 执行操作 | `[STOP CONFIRMED]` |
| **拒绝继续** | 标注阻塞并询问替代方案 | `[STOP REJECTED]` |
| **要求回滚** | 回到最近 checkpoint | `[STOP ROLLBACK]` |

## 责任单元判断标准

### 什么是责任单元

**责任单元**：一个可以独立执行、独立验证、独立勾选的最小工作单元。

### 判断标准

| 标准 | 说明 | 检验方法 |
|---|---|---|
| **可观察完成** | 有明确验证命令或可观察结果 | 能回答“如何验证它完成了？” |
| **文件影响范围** | 影响文件数尽量 ≤ 5 | 预估需要修改的文件 |
| **代码量边界** | 新增/修改量尽量 ≤ 150 行 | 预估改动范围 |
| **失败独立性** | 失败时不会拖垮其他 checkbox | 跳过该 task 后其余 task 仍可推进 |

**优先级顺序**：可观察完成 > 文件影响范围 > 失败独立性 > 代码量边界。

### 粒度对比

| 类型 | 示例 | 判断 | 原因 |
|---|---|---|---|
| **太粗** | “实现用户模块” | ❌ | 无法独立验证 |
| **太细** | “添加 import UserService” | ❌ | 变成编辑流水账 |
| **合格** | “实现 UserService.login 方法并通过类型检查” | ✅ | 独立能力 + 明确标准 |
| **合格** | “编写 login 方法的单元测试（覆盖成功/失败/边界）” | ✅ | 独立任务 + 可验证 |

### 常见错误示例

#### ❌ 混合 Task

```text
[ ] 实现用户登录并编写测试
```

应拆为：

```text
[ ] 实现 UserService.login 方法，类型检查通过
[ ] 编写 login 方法的单元测试：`npm test -- user.service.test.ts`，期望 12 passed
```

#### ❌ 过度细化

```text
[ ] 添加 import UserService
[ ] 定义 login 方法签名
[ ] 实现密码验证逻辑
```

应改为：

```text
[ ] 实现 UserService.login 方法（包含密码验证、token 生成、错误处理），类型检查通过
```

### 测试任务的特殊要求

测试/验证 checkbox 必须写明：

1. **命令**
2. **范围**
3. **期望结果**
4. **失败处理**
5. **证据位置**

示例：

```text
[ ] 单元测试 - UserService.login：`npm test -- user.service.test.ts`，
    期望 12 passed（覆盖成功/失败/边界情况），
    失败时定位错误并修复后重跑，
    结果记录在 verify.md
```

### 快速自检

设计 task 时，至少问自己四个问题：

1. 能否用一条命令或一个明确观察点验证完成？
2. 影响文件是否过多？
3. 失败后会不会把多个任务一起卡住？
4. 这是不是能力单元，而不是代码编辑步骤？

---

## 第二部分：配置指南

## 在 OpenSpec 中配置 Hardness

`openspec/config.yaml` 是 Hardness 的主要入口，但要注意：它**不是 Claude Code 的独立强制原语**。

### 实现机制：`config.yaml` 本质上是 Rule 的扩展

它的 `rules` 通常由 OpenSpec skill 读取，并注入模型上下文，因此本质上仍然属于**软约束**：

```text
config.yaml rules
    └─ OpenSpec 技能读取
        └─ 注入上下文
            └─ 模型自觉遵守
```

这意味着：

- `config.yaml` 很适合表达阶段规则
- 但它不能替代 Hook 做硬拦截
- 高风险操作仍应升级到工具层门禁

### 配置三机制

1. **Default Workflow Schema**：定义 artifact 依赖图
2. **Project Context Injection**：注入全局背景信息
3. **Per-Artifact Rules**：为每个 artifact 追加特定约束

### 完整配置示例

```yaml
# openspec/config.yaml
schema: spec-driven
context: |
  项目：用户管理系统（微服务架构）
  技术栈：Node.js + TypeScript + PostgreSQL
  测试框架：Jest

rules:
  proposal:
    - "Proposal 必须包含 Context、Scope、Non-goals、Depends On、Blocks"
    - "Non-goals 必须列出至少 3 项明确不做的能力"
    - "Scope 必须用用户故事格式：'作为 [角色]，我想要 [能力]，以便 [价值]'"

  specs:
    - "每个能力变更必须对应至少一个 Spec 文件"
    - "每个 Spec 必须包含至少一个 Acceptance Scenario"
    - "Acceptance Scenario 必须使用 Given-When-Then 格式"
    - "涉及 API 变更的 Spec 必须包含请求/响应示例"

  design:
    - "Design 必须说明推荐方案和至少 1 个替代方案"
    - "必须说明模块边界和数据/API 归属"
    - "涉及外部依赖时必须说明版本和兼容性"

  tasks:
    - "tasks.md 必须按阶段分组：Implementation、Contracts and Docs、Tests and Verification"
    - "每个 checkbox 必须是独立可执行、可验证、可勾选的责任单元"
    - "不得把‘实现 X 并测试’写成一个 checkbox"
    - "测试/验证 checkbox 必须写明：命令、范围、期望结果、失败处理、证据位置"

  apply:
    - "只执行 tasks.md 已列 checkbox"
    - "每完成一个 checkbox 前必须验证对应结果"
    - "测试失败不得勾选完成"
    - "无法通过时保持 checkbox 未完成并记录 blocker"

  verify:
    - "必须核对 tasks.md 是否全部完成"
    - "必须核对每个测试/验证 task 是否有结果证据"
    - "必须核对实现是否匹配 proposal/specs/design"
    - "证据缺失则视为未完成"

  archive:
    - "tasks 未完成必须告警并 STOP"
    - "verify 有 CRITICAL 项不得归档"
    - "归档后必须更新 openspec/changelist.md"
```

## 在 Claude Code 中配置 Hardness

### `CLAUDE.md`：长期规则宪法

`CLAUDE.md` 适合存放项目长期有效的规则，例如：

- 架构边界
- 常用命令
- 长期 Do / Don’t
- Before Finishing 检查清单
- 危险操作与 STOP 条件

### `CLAUDE.md` 应该放什么

- 项目级长期规则
- 已验证的命令与约束
- 长期稳定的危险操作策略

### `CLAUDE.md` 不应该放什么

- 某个 feature 的临时需求
- 今日任务清单
- 一次性 TODO
- 未验证命令

### 示例

```markdown
# CLAUDE.md - 项目长期规则

## 项目概述
用户管理系统，微服务架构，Node.js + TypeScript + PostgreSQL。

## 常用命令
- 测试：`npm test -- <pattern>`
- Lint：`npm run lint`
- Build：`npm run build`

## 编码规则（MUST）
- 所有数据库操作必须使用 Repository 模式
- 所有 API 必须包含 OpenAPI 注解
- 敏感信息必须加密存储，不得明文

## 编码规则（MUST NOT）
- 不得直接修改数据库 schema 而不经过 migration
- 不得在 Controller 中写业务逻辑
- 不得使用 `any` 类型

## Before Finishing（每次完成前必须检查）
- [ ] 运行 `npm run typecheck`，0 error
- [ ] 运行 `npm run lint`，0 error
- [ ] 运行相关测试，全部通过
- [ ] 确认只修改了预期文件（`git status`）

## 危险操作（STOP）
- 涉及 `DROP TABLE` 或 `DELETE` 无 WHERE 条件 → STOP，必须人工确认
- 涉及修改 `src/config/` 下文件 → STOP，必须人工 review
```

### 如何把软规则升级为硬门禁

仅靠 `CLAUDE.md` 仍然是软规则。要做不可绕过的门禁，需配合 **Claude Code Hooks**。

```python
# 阻止危险的 terraform apply

def terraform_policy(input_data: ToolUseEvent):
    if not input_data.tool_is_bash:
        return
    command = input_data.command.strip()
    if re.match(r'^terraform\s+apply', command):
        yield PolicyDecision(
            action=PolicyAction.DENY,
            reason="terraform apply 被禁止。请使用 `terraform plan` 预览变更。"
        )
```

```json
{
  "hooks": {
    "PreToolUse": [
      {
        "hooks": [
          {
            "matcher": "*",
            "command": "devleaps-policy-client"
          }
        ]
      }
    ]
  }
}
```

## 按阶段配置 Hardness

### Explore（低硬度）
- 允许假设与比较方案
- 不确定内容标注 `[NEEDS CLARIFICATION]`
- 不产出最终结论或长期规则

### Proposal（中硬度）
- 控制需求边界
- 必须包含 Context、Scope、Non-goals、Depends On、Blocks
- 过大需求先切片

### Specs（中高硬度）
- 固化可观察行为
- 每个 Spec 至少一个 Acceptance Scenario
- 使用 Given-When-Then

### Design（中硬度）
- 控制方案质量
- 说明推荐方案与替代方案
- 明确模块边界与数据/API 归属

### Tasks（高硬度）
- 控制执行计划
- checkbox 必须是责任单元
- 开发与测试拆开

### Apply（高硬度）
- 只按 tasks 执行
- 完成前先验证
- 失败保持未完成并记录 blocker

### Verify（很高硬度）
- 对账 tasks、证据、实现一致性
- 证据缺失视为未完成

### Archive（最高硬度）
- 未完成不得归档
- 有 CRITICAL 不得归档
- 所有异常先 STOP 再决策

---

## 第三部分：进阶实践

## 规则编写规范

### 用短句，不用长段落

❌ 不好：

> 实现阶段应该在保证质量的前提下根据任务逐步推进，并结合测试情况判断是否可以完成。

✅ 好：

> - 只按 tasks.md 已列 checkbox 执行
> - 每完成一个 checkbox 前必须验证结果
> - 测试失败不得勾选完成

### 用“失败时怎么办”定义硬度

❌ 弱：

> 必须测试。

✅ 强：

> 测试失败时：定位原因、修复、重跑；仍失败则保持未完成并记录 blocker。

### 用“证据”替代“相信”

❌ 弱：

> 确认功能正常。

✅ 强：

> 运行 `<command>`，期望 `0 error`；在 `verify.md` 的 Evidence 章节记录命令和结果。

### 区分 MUST / SHOULD / MAY / MUST NOT / STOP

| 类型 | 含义 | 示例 |
|---|---|---|
| **MUST** | 不满足就不能继续 | 测试失败不得勾选完成 |
| **SHOULD** | 默认应该做，可说明例外 | 复杂状态机应画 Mermaid 图 |
| **MAY** | 可选增强 | 简单变更可省略替代方案 |
| **MUST NOT** | 明确禁止 | 不得把测试拆成独立 Change |
| **STOP** | 暂停并请求决策 | 目标目录不清时停止并询问 |

### Hard Rule 的简式公式

一条高质量硬规则通常同时包含四件事：

1. **对象**：约束谁 / 哪个阶段
2. **动作**：必须做什么或不能做什么
3. **失败处理**：不满足时怎么办
4. **证据**：如何证明已经满足

例如：

> Verify 阶段必须核对每个测试 task 的命令、输出与时间戳；任一缺失则视为未完成，不得归档。

## 防止模型自我合理化

### 常见合理化路径

| 合理化语句 | Hardness 封堵 |
|---|---|
| “这个改动很小，不需要测试。” | 所有改动必须按 tasks 执行 |
| “我已经看过代码，应该没问题。” | 必须运行命令并保留证据 |
| “虽然没跑，但逻辑上是对的。” | 未运行且无跳过理由，不得勾选 |
| “这个 task 基本完成，可以先勾选。” | 完成标准是证据，不是“差不多” |

### 在 `CLAUDE.md` 中明确封堵

```markdown
## 禁止的自我合理化（MUST NOT）
- 不得以“改动小”为由跳过测试验证
- 不得以“逻辑上正确”代替实际运行验证
- 不得以“基本完成”为由勾选未验证的 checkbox
- 不得以“代码看起来没问题”代替实际测试
```

### 在 `config.yaml` 中设置硬约束

```yaml
rules:
  apply:
    - "测试失败时，不得勾选完成，无例外"
    - "命令未运行时，不得勾选完成，无例外"
    - "证据缺失时，视为未完成，无例外"
```

## 失效诊断

### 为什么 Hardness 会失效

1. 规则太软
2. 规则互相冲突
3. 缺少证据要求
4. 没有封堵合理化路径
5. 高风险操作仍停留在软 STOP

### 症状 → 诊断 → 修复表

| 症状 | 可能原因 | 修复建议 |
|---|---|---|
| 模型跳过测试 | Apply 规则太弱 | 加强失败处理路径 |
| 测试未运行就勾选 | 缺少证据要求 | 增加“证据缺失视为未完成” |
| tasks 未完成就归档 | Archive 规则太弱 | 增加 STOP 条件 |
| 证据只有“已验证” | 证据格式未定义 | 补充证据模板 |
| 危险命令被继续执行 | 只写了软 STOP | 升级为 Hook |

### 诊断清单

```markdown
## Hardness 失效诊断清单

### 基础检查
- [ ] `CLAUDE.md` 存在且包含长期规则
- [ ] `openspec/config.yaml` 存在且包含 rules 配置
- [ ] rules 按阶段分组

### 规则质量检查
- [ ] 使用短句 bullet，不用长段落
- [ ] 硬规则包含失败处理路径
- [ ] 关键规则绑定可观察证据
- [ ] 区分了 MUST / SHOULD / MAY / MUST NOT / STOP

### 证据系统检查
- [ ] verify.md 有 Evidence 章节
- [ ] 测试 task 写明命令和期望结果
- [ ] verify 规则要求“证据缺失视为未完成”

### 封堵检查
- [ ] `CLAUDE.md` 有“禁止的自我合理化”章节
- [ ] 封堵了“改动小不需要测试”
- [ ] 封堵了“逻辑上正确”代替实际验证

### 工具层门禁检查
- [ ] 高风险不可逆操作已评估是否需要 Hook
```

## 渐进式采纳路径

### 阶段 1：最小可行 Hardness

目标：先解决**跳过验证**和**未完成归档**。

### 阶段 2：补充验证门禁

目标：让每个验证步骤都有证据。

新增重点：

- `verify` 要求证据
- `tasks` 明确测试命令、范围、期望结果
- 证据缺失视为未完成

### 阶段 3：完整约束体系

目标：建立 Proposal / Specs / Design / Tasks / Apply / Verify / Archive 全流程护栏。

### 采纳建议

| 团队情况 | 建议起点 |
|---|---|
| 刚开始用 OpenSpec | 阶段 1 |
| 已用 OpenSpec，但测试常跳过 | 阶段 1 + 阶段 2 |
| 流程成熟，想提高质量与一致性 | 阶段 3 |

---

## 第四部分：边界与禁忌

## 适用边界与场景选择

### 推荐使用的场景

1. 规范驱动的工程项目
2. 已有测试框架的成熟项目
3. 多人协作或长期维护项目
4. 可响应 STOP 的交互式开发场景

### 不适合或应降低硬度的场景

1. **探索性原型 / PoC**：配置成本可能高于收益
2. **创意驱动的前端设计**：更多依赖 `[MANUAL]` 验证
3. **一次性脚本**：直接用 `CLAUDE.md` 的 Before Finishing 通常更划算
4. **完全无人值守自动化**：STOP 无人响应，应该交给传统 CI/CD 门禁

### 场景选择决策树

```text
是否需要长期维护？
  ├─ 否 → 只用 CLAUDE.md 的 Before Finishing
  └─ 是
      ├─ 是否有明确测试命令？
      │   ├─ 否 → 先用 L1-L2，降低 Apply/Verify 硬度
      │   └─ 是
      │       ├─ 是否需要证据审计？
      │       │   ├─ 否 → 阶段 1-2
      │       │   └─ 是 → 阶段 3
```

## Hardness 的禁忌

1. **不要在探索阶段过度 Hardness**：会压制发散与比较
2. **不要把所有规则都写成 MUST**：要保留层级
3. **不要用未验证事实写硬规则**：Hardness 会放大错误
4. **不要让 Hard Rules 互相冲突**：否则模型无法执行
5. **不要把 Hardness 当成验证本身**：规则不能替代运行与证据
6. **不要把 tasks 写成流水账**：Hardness 不是碎片化编辑

---

## 常见问题

### Q1：Hardness 会降低开发效率吗？

短期会增加一些约束成本，但通常能减少返工、沟通误差和“假完成”。

### Q2：如何平衡 Hardness 和灵活性？

按阶段分层。Explore 低硬度，Apply / Verify 高硬度；简单变更从轻，关键变更从严。

### Q3：规则如何真正被执行？

可以理解为两层机制：

1. **Prompt 层**：`CLAUDE.md` 与 `config.yaml`，软约束
2. **工具层**：Hook，硬门禁

复杂校验还可以交给 **Subagent**，但它负责“读完并回传结论”，不负责硬拦截。

### Q4：什么时候必须上 Hook？

当操作同时满足以下任意条件时，应优先考虑 Hook：

- 不可逆
- 影响外部系统
- 风险高于语义判断价值
- 不能接受模型偶发忽略规则

---

## 总结

Hardness 的核心不是“语气更强”，而是：

> **规则更具体、边界更清楚、失败更可处理、证据更可检查、关键操作更不可绕过。**

一句话概括：

> **用 Soft Guidance 促进思考，用 Structured Requirement 固化意图，用 Hard Constraint 控制执行，用 Evidence 和 STOP 决定是否完成。**
