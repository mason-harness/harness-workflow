# 证据指南

## 证据映射表

为每个重要的 `CLAUDE.md` 规则或知识库事实维护内部证据映射：

| 待写入声明 | 已检查证据 | 结果 |
|---|---|---|
| `npm test` 可用 | `package.json` scripts | 仅当 script 存在时写入 |
| 使用 pnpm | lockfile/workspace 配置 | 仅当 lockfile 确认时写入 |
| API 代码位于 `src/server` | 源码树/入口点 | 仅当路径存在时写入 |
| 自定义构建命令 | Makefile / justfile / task 配置 | 仅当可执行定义存在时写入 |
| 术语 X 为稳定领域概念 | 代码命名、接口字段、稳定团队规范、现有知识库 | 仅当概念在多个稳定来源中可验证时写入 |
| 模块 Y 负责 Z | 目录结构、入口关系、调用边界、稳定文档 | 仅当职责边界清晰可验证时写入 |

如果证据缺失：
- 仅当缺失本身有信息价值时才写“未配置”
- 否则忽略该声明
- 如果用户期望不存在的命令有效并要求记录为事实，STOP
- 如果用户要求把未验证假设写进 `repository/`，STOP

## 冲突优先级（从高到低）

1. 当前用户明确指定的项目范围和目标
2. 已验证的可执行项目文件：package scripts、Makefile、config、源码入口点
3. 现有 `CLAUDE.md` 与现有 `repository/` 中仍有证据支持的条目
4. 用户提供的稳定团队规则
5. README/wiki/docs 作为较弱证据（除非被可执行文件确认）

**STOP 条件**：冲突会改变长期规则或稳定知识；目标项目不明确；请求的命令不存在；来源矛盾且无解决路径。

当来源冲突时，仅保留有证据支持的事实。将未解决的事实标记为未知或询问；不要猜测。

## 过时命令检测

命令**必须**来自已验证来源：
- `package.json` scripts
- Makefile / justfile
- Task 配置（如 `.vscode/tasks.json`、`Taskfile.yml`）
- 已记录的项目工具

**禁止**保留旧 `CLAUDE.md` 或旧知识库中的命令，如果没有可执行来源仍支持它们。

如果用户要求记录不存在的命令且期望它有效，STOP。将其报告为不存在或过时，而不是添加到 `CLAUDE.md`。

**可以**提及重要缺失命令的“未配置”状态，但不要发明 scripts。

## 验证后写入

| 内容 | 事实来源 | 写法 |
|---|---|---|
| 命令 | `package.json` scripts、Makefile、任务配置 | 只写存在的命令；缺失命令写“未配置”，不要捏造 |
| 包管理器 | lockfile、workspace 配置、README 冲突 | 说明冲突并采用更强证据，不盲从单一文档 |
| 技术栈 | 依赖、配置、入口文件 | 写关键技术栈，避免过细版本号 |
| 架构边界 | 入口、路由、模块、服务层、数据层 | `CLAUDE.md` 写执行边界；详细结构写入 `repository/architecture.md` 或 `modules.md` |
| 团队规范 | 用户提供的稳定规则 | 提炼为 Claude 可执行的 Do / Don't |
| 术语与集成 | 代码、接口定义、稳定文档 | 写入 `repository/glossary.md`、`integrations.md`，不要塞回单文件 |
| 发布/数据变更 | 脚本和团队规则 | 标注需用户确认，不写私密凭据或未证实地址 |

## 知识库写入规则

- `repository/` 只写稳定、可复用、跨会话仍成立的项目知识
- 目录结构应优先固定为 `README.md / glossary.md / architecture.md / modules.md / integrations.md / decisions/`
- 若某类知识无法确认稳定性，不要先建文件凑结构
- 不要再新增 `project.md` 作为项目知识主载体；需要摘要时写入 `repository/README.md`
