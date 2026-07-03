# Claude Code 扩展集合

这是一套围绕 Claude Code 构建的技能仓库，用于增强 AI 辅助开发中的规则治理、证据门禁、OpenSpec 工作流衔接与多切片推进能力。

## 仓库定位

这个仓库是一个单项目的 skills 集合，核心目标是把以下几类能力组织成可复用的工作流工具：
- **Harness**：规则治理、硬门禁、执行纪律、证据门禁
- **OpenSpec slices**：大需求拆分、登记、追踪
- **Repository knowledge**：把长期稳定知识从单文件说明中拆分出来，形成可维护入口

## 从哪里开始看

如果你刚接手这个仓库，建议按这个顺序阅读：

1. [`CLAUDE.md`](CLAUDE.md) — Claude 在本仓库中的执行规则
2. [`repository/README.md`](repository/README.md) — 根级知识库入口
3. [`repository/architecture.md`](repository/architecture.md) — 仓库定位与 OpenSpec / Harness / Hook 三层职责
4. [`repository/modules.md`](repository/modules.md) — 各 skill 家族的模块边界
5. [`repository/integrations.md`](repository/integrations.md) — 与 `/opsx:*`、OpenSpec CLI、Hook 的接线关系
6. [`repository/harness-workflow.md`](repository/harness-workflow.md) — 工作流总览与典型时机判断

## 技能列表

### Harness 系列

#### `harness-claude-setup`
维护根级 `CLAUDE.md` 与 `repository/`，分离执行规则与稳定项目知识。

#### `harness-openspec-setup`
维护 OpenSpec 项目级规则，把 `CLAUDE.md` 与 `repository/` 中的稳定约束投影为 artifact 规则。

#### `harness-openspec-apply`
作为 apply 阶段的纪律增强层，强调 task preflight、一次只推进一个责任单元、先验证再勾选。

#### `harness-openspec-verify`
作为 verify 阶段的证据门禁与完成门禁层，对账任务、命令、输出与时间戳。

#### `harness-hook-setup`
把危险操作规则转化为 Claude Code Hooks（PreToolUse），用于实现不可绕过的硬 STOP。

### OpenSpec 切片系列

#### `openspec-slices-plan`
在需求过大时做切片决策，判断是否需要拆分成多个可验证 Change。

#### `openspec-slices-register`
把已确认的 Slice Plan 登记为 OpenSpec changes，并在需要时依据 CLI 契约维护关联信息。

#### `openspec-slices-track`
跨 session 追踪多切片进度，识别 archived / in-progress / ready / blocked 状态并建议下一步。

## 文档职责边界

| 载体 | 主要职责 |
|---|---|
| `README.md` | 仓库导览、技能地图、阅读入口 |
| `CLAUDE.md` | Claude 在本仓库如何工作 |
| `repository/` | 稳定项目知识、模块边界、接线关系、工作流总览 |
| `skills/*/SKILL.md` | 各 skill 的详细规则、边界、失败处理、handoff |

## 工作流入口

这个仓库不再把完整规则、示例和长期知识都堆在 README 中。

如果你要看：
- **整体工作流**：看 [`repository/harness-workflow.md`](repository/harness-workflow.md)
- **三层职责模型**：看 [`repository/architecture.md`](repository/architecture.md)
- **技能边界**：看 [`repository/modules.md`](repository/modules.md)
- **接线关系**：看 [`repository/integrations.md`](repository/integrations.md)
- **具体 skill 细则**：直接看对应的 `skills/<name>/SKILL.md`

## OpenSpec 与 Hook 说明

- OpenSpec 在这里被视为 Change lifecycle 的主线。
- Harness skills 负责在关键阶段增加纪律、证据和治理能力。
- Hook 只用于高风险、不可逆、必须不可绕过的场景。

如需精确 OpenSpec CLI 契约，请查看：
- [`skills/openspec-slices-register/references/cli-contract.md`](skills/openspec-slices-register/references/cli-contract.md)

如需把危险操作升级为硬门禁，使用：
- `harness-hook-setup`

## 通过 npm 安装或同步 skills

仓库提供了一个显式安装脚本，用来把 `skills/` 下的内容复制到目标 `skills` 目录。

默认行为：
- 来源目录：当前仓库根目录下的 `skills/`
- 复制语义：覆盖同路径已有文件，但不会删除目标目录中仓库之外的额外文件
- 保留完整 skill 子树：包括 `SKILL.md`、`references/`、`testing/` 等嵌套内容
- 运行环境：Node.js `>=18`

### 目标目录模式

1. **默认模式**：复制到当前工作目录下的 `.claude/skills/`
2. **全局模式**：复制到 `~/.claude/skills/`
3. **自定义模式**：复制到 `--target` 指定目录

### 在仓库中执行

```bash
npm run install-skills
npm run install-skills -- --global
npm run install-skills -- --target /path/to/skills-dir
```

### 作为 npm 包执行

当前 `package.json` 中的包名是 `claude-harness-skills`，CLI 入口是 `install-claude-skills`。

```bash
npx claude-harness-skills
npx claude-harness-skills --global
npx claude-harness-skills --target /path/to/skills-dir
```

如需查看参数帮助：

```bash
npx claude-harness-skills --help
```

### 安装结果示例

- 默认模式会写入：`./.claude/skills/<skill-name>/...`
- 全局模式会写入：`~/.claude/skills/<skill-name>/...`
- 自定义模式会写入：`<target>/<skill-name>/...`

## 目录结构

```text
.
├── CLAUDE.md
├── README.md
├── package.json
├── scripts/
│   └── install-skills.js
├── repository/
│   ├── README.md
│   ├── architecture.md
│   ├── modules.md
│   ├── integrations.md
│   └── harness-workflow.md
├── docs/
│   └── ... 专题文档与参考资料
└── skills/
    ├── harness-claude-setup/
    ├── harness-hook-setup/
    ├── harness-openspec-apply/
    ├── harness-openspec-setup/
    ├── harness-openspec-verify/
    ├── openspec-slices-plan/
    ├── openspec-slices-register/
    └── openspec-slices-track/
```

## 补充资料

`docs/` 下还保留了一些历史专题文档与参考资料，适合作为补充阅读，例如：
- `Hardness-Practice.md`
- `OpenSpec Change 拆分策略文档.md`
- `ClaudeCode原语能力边界.md`
- `OpenSpec 参考文档.md`

这些文档仅作为补充参考材料保留，不承担仓库主入口或正式规则载体职责。

## 最后更新

2026-07-03
