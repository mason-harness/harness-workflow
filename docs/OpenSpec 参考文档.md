# OpenSpec 参考文档

## 1. OpenSpec CLI 脚本说明

### 1.1 OpenSpec 的基本工作单元

OpenSpec 以 **Change** 作为基本工作单元，用来承载一次可独立推进、独立验证的变更。

一个典型 Change 通常围绕这些 artifacts 展开：

- `proposal.md`：变更目标、背景、范围、依赖
- `specs/`：这次变更涉及的 Delta 规范
- `design.md`：技术设计与关键决策
- `tasks.md`：实施任务清单

### 1.2 常见生命周期命令

OpenSpec 的常见生命周期通常可以概括为：

```text
explore / propose
→ new / continue / ff
→ apply
→ verify
→ archive
```

它表达的是一条从规划到实施、再到验收与归档的 Change 生命周期。

### 1.3 已验证的 CLI 命令契约

#### 新建 Change

```bash
openspec new change <name> --goal <g> --areas <a1,a2> --initiative <id> --store <store> --store-path <path> --schema <schema> --json
```

关键点：

- `<name>` 必填，且必须是 kebab-case
- `--initiative` 支持多种传法，但跨仓场景更适合显式带 `--store <store>`
- `--json` 输出包含 `change_name`、`schema_name`、`created_files` 等结构化字段
- 如果 change 已存在，应按幂等场景处理，不能假装重新创建过

#### 为已有 Change 追加 Initiative 链接

```bash
openspec set change <name> --initiative <id> --store <store> --store-path <path> --json
```

关键限制：

- 仅 repo-local change 可 link initiative
- workspace change 不能 link initiative

#### Context Store 相关命令

```bash
openspec context-store setup [id] --path <storeRoot> --init-git --json
openspec context-store list --json
openspec context-store doctor --json
```

#### Initiative 相关命令

```bash
openspec initiative create [id] --title <t> --summary <s> --store <store> --store-path <path> --json
openspec initiative show <id> --json
openspec initiative list --json
```

### 1.4 关键 CLI 限制

#### `tasks.md` 的进度是只读计算结果

CLI 会：

- 读取 `tasks.md` 中的 `- [ ]` / `- [x]`
- 计算 `progress.total` / `progress.complete` / `progress.remaining`

CLI 不会：

- 自动改写 checkbox
- 自动把 `- [ ]` 改成 `- [x]`

这意味着 `tasks.md` 的完成状态必须由实施过程显式维护。

#### initiative `tasks.md` 不会被 CLI 持续自动维护

已验证的约束是：

- `initiative create` 创建时只写一次模板
- 后续 apply / archive 不会自动持续回写 initiative `tasks.md`

因此，initiative `tasks.md` 更适合承担：

- 切片顺序
- 依赖关系
- repo/change 映射
- handoff 摘要

而不适合承担 task 级实时进度。

#### 命名规则

Change 名称必须符合 kebab-case：

- 允许：小写字母、数字、连字符
- 不允许：大写字母、下划线、首尾连字符

常见命名示例：

- `s01-foundation`
- `s02-normal-flow`
- `s03-retry-guard`

---

## 2. OpenSpec skills 说明

### 2.1 Skills 的角色

OpenSpec 本身更偏向规范驱动与生命周期组织；而 **skills** 用来把某类流程、规则或任务执行方式包装成可以复用的能力单元。

可以把它理解为：

- OpenSpec 定义“这次 Change 应该有哪些 artifacts、遵循什么流程”
- skills 定义“在某类场景下，AI 应该如何行动、如何推进、如何交接”

### 2.2 Skills 的典型用途

OpenSpec 语境里的 skills 常见用途包括：

- 在规划阶段补充结构化分析
- 在实施阶段增加执行纪律
- 在验收阶段增加证据门禁
- 在需求过大时辅助切片
- 在多仓协作时辅助登记与追踪

### 2.3 Skills 与 Commands 的区别

在补充资料给出的理解里，OpenSpec 常通过两种形式把规则交给 AI 工具使用：

- **Skills**：通常用 `SKILL.md` 这类指令文件表达，偏通用、偏可复用
- **Commands**：通常以某个 AI 工具的斜杠命令形式暴露，偏工具侧入口

可以粗略理解为：

- Skills 更像“能力定义”
- Commands 更像“触发入口”

### 2.4 Skills 的边界

Skills 的价值在于提供：

- 触发条件
- 行为边界
- 输入输出约定
- 失败处理方式
- handoff 规则

但 Skills 不等于 OpenSpec 本体。OpenSpec 仍然负责生命周期与 artifacts 结构，Skills 负责把某些动作执行得更稳定、更一致。

---

## 3. OpenSpec 配置说明

### 3.1 配置的作用

OpenSpec 配置用于定义项目级的默认行为与规则边界，通常承担：

- schema 选择
- 项目级 context
- 各阶段 artifact rules
- apply / verify / archive 的约束

### 3.2 常见配置层级

补充资料中给出的常见层级可以概括为：

- 全局配置
- 项目配置
- Schema 定义
- 变更元数据

可以把它理解为从“全局默认”到“单个 Change 特例”的逐层收紧。

### 3.3 常见规则组

一个较完整的 OpenSpec 项目配置，通常会覆盖这些规则组：

- `proposal`
- `specs`
- `design`
- `tasks`
- `apply`
- `verify`
- `archive`

这些规则组常见职责分别是：

- **proposal**：定义目标、范围、非目标、假设
- **specs**：定义行为与边界场景
- **design**：定义方案、模块边界、测试设计
- **tasks**：定义可执行任务清单
- **apply**：约束实施过程
- **verify**：约束验收证据
- **archive**：约束归档前条件

### 3.4 配置设计的重点

比较重要的配置思路包括：

- `design` 最好包含独立测试设计，而不是只在实现描述里顺带提测试
- `tasks` 最好包含独立测试任务，而不是把“实现 + 测试”混成一个 checkbox
- `apply` 应只围绕 `tasks.md` 中已声明的任务推进
- `verify` 应要求命令、输出、时间戳或其他完成证据

### 3.5 Schema 与配置的关系

Schema 更偏向：

- artifacts 定义
- 模板结构
- artifact 级 instruction

项目配置更偏向：

- 项目级默认规则
- 全局约束
- 生命周期阶段 gate

两者通常是配合关系，而不是互相替代。

---

## 4. 高级应用：主动调用 Skills 与 Agents

### 4.1 先分清 OpenSpec 与 AI 工具的职责

在主动调用这个话题里，最重要的一点是：

- **AI 工具** 负责真正执行调用、编辑代码、与用户交互
- **OpenSpec** 负责通过 artifacts、配置与规则，定义“在什么条件下应当做什么”

也就是说，OpenSpec 更多是在**引导和约束调用时机**，而不是亲自执行调用。

### 4.2 常见四种方式

#### 方式一：通过 Schema `instruction` 引导调用

适合：希望在某个 artifact 生成阶段，提示 AI 主动调用指定 Skill 或 Agent。

典型思路：

- 在 `schema.yaml` 的 artifact `instruction` 中写明要求
- 例如要求在撰写 `design.md` 前先调用某个架构评审 Agent
- 或在拆解 `tasks.md` 时要求引用某个审查 Skill

#### 方式二：通过 `config.yaml` 的 `rules` 添加全局调用规则

适合：希望在 proposal / specs / apply / verify 等多个阶段统一追加约束。

典型做法：

- 在对应规则组里写入默认要求
- 例如在 `apply` 阶段要求每个任务完成后都做额外验证
- 或在 `proposal` / `specs` 阶段要求附带某类审查结果

#### 方式三：通过插件提供专用 Agent

适合：某个 AI 工具支持插件，并能在工具层暴露专门的 OpenSpec Agent。

这种方式的核心价值通常在于：

- 让规划工作由专门 Agent 处理
- 把 OpenSpec 文档编辑和业务代码编辑职责区分开
- 降低规划阶段过早编码的风险

#### 方式四：由 Agent 反向驱动 OpenSpec CLI

适合：希望由 Agent 主动驱动 `propose → validate → apply → archive` 一整套工作流。

在这种模式里：

- Agent 通过插件或工具接口调用 OpenSpec CLI
- OpenSpec 负责规范结构与生命周期
- Agent 负责执行顺序、决策与自动化推进

### 4.3 关键前提：任务粒度决定控制力

主动调用是否稳定，关键往往不只是 instruction 写得有多强，而是任务粒度是否足够细。

原因在于：

- 任务过粗时，AI 更容易跳步，一次性跨过多个阶段
- 任务拆成独立 checkbox 后，每一步都更容易被单独验证
- 如果再叠加验证 Agent，主动调用会更可预测

因此，在高级应用里，**原子任务拆分**通常比单纯强调“必须调用某个 Agent”更有效。

---

## 5. `initiative` 与 `context-store` 高级应用

### 5.1 基本定位

- **Context Store**：跨仓共享上下文的载体
- **Initiative**：放在 Context Store 里的跨仓协调对象

如果说 Change 是 repo-local 的推进单元，那么 initiative 更像 cross-repo 的协调单元。

### 5.2 什么时候需要它们

当需求已经不是单仓库、单 owner、单 Change 可以承载时，就应考虑引入：

- 需求横跨多个仓库
- 不同切片有不同 owner
- 需要明确 `depends_on`
- 需要跨 session、跨 repo 共享同一份协调视图

### 5.3 典型使用流程

一个常见的 cross-repo 组织方式可以概括为：

```text
先形成 Slice Plan
→ 建立或确认 context-store
→ 建立 initiative
→ 将各个 change 与 initiative 关联
→ 维护 initiative 侧的切片索引
```

常见 CLI 动作包括：

```bash
openspec context-store setup [id] --path <storeRoot> --init-git --json
openspec initiative create [id] --title <t> --summary <s> --store <store> --store-path <path> --json
openspec new change <name> --initiative <id> --store <store> --json
```

### 5.4 initiative `tasks.md` 适合保存什么

initiative `tasks.md` 更适合用作**协调索引**，保存：

- 切片顺序
- 依赖关系
- repo/change 映射
- handoff 摘要

不适合把它当作：

- 自动进度面板
- task 级实时状态来源
- apply / archive 后自动回写的系统表

实时状态仍然更应以各个 change 自己的 `tasks.md` 与 CLI 聚合结果为准。

### 5.5 重要限制

#### 限制一：workspace change 不能 link initiative

如果当前 change 是 workspace 形态，却要 link initiative，应当停止并调整流程，而不是继续登记。

#### 限制二：initiative `tasks.md` 不是自动维护的

如果要把切片索引写进去，需要显式维护，不能假设 CLI 会自动长期同步。

#### 限制三：跨仓场景应尽量显式传 `--store`

CLI 虽然支持多种 `--initiative` 传法，但在跨仓场景中，更推荐显式传：

```bash
--initiative <id> --store <store>
```

这样比依赖当前目录隐式推断更稳妥。