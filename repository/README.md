# Repository Knowledge Base

这个目录承载当前仓库的稳定项目知识，不承载 Claude 执行规则，也不复制 README 的人类导览内容。

## 阅读顺序

1. `architecture.md`：先理解仓库的分层职责与整体定位
2. `modules.md`：再看各 skill 家族的模块边界
3. `integrations.md`：最后看与 `/opsx:*`、OpenSpec CLI、Hook 的接线关系
4. `harness-workflow.md`：需要整体工作流和时机判断时再读

## 文件职责

以下文件是当前整理后的主入口：
- `architecture.md`：仓库定位、OpenSpec / Harness / Hook 三层职责、skills 分类方式
- `modules.md`：各 skill 家族的职责边界与相互关系
- `integrations.md`：本仓库与 `/opsx:*` 生命周期、OpenSpec CLI 契约、Hook 的接线边界
- `harness-workflow.md`：工作流总览、推荐流程、典型时机判断

## 现有其他文档

`repository/` 目录中还保留了一些较早的专题文档与参考资料。除非它们被重新整理进上述主入口，否则应将它们视为补充材料，而不是当前知识库结构的一级入口。

## 不应写入这里的内容

- `CLAUDE.md` 级别的执行规则、Do / Don't、Before Finishing
- README 的仓库介绍、价值主张、Quick Start
- 各 `skills/*/SKILL.md` 的完整规则正文
- 一次性需求、临时计划、会话上下文
- 未验证的本地命令或工具能力