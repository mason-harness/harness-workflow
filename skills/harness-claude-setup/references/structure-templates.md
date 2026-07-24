# `CLAUDE.md` 与 `repository/` 结构模板

按项目实际情况增删章节；不要为了模板完整保留空泛章节。

## `CLAUDE.md` 模板

```markdown
# CLAUDE.md

This file provides guidance to Claude Code when working with code in this repository.

## 项目概览
## 常用命令
### 开发
### 构建
### 代码质量
### 测试
## 架构
### 入口点
### 路由 / 模块
### 状态管理
### 数据获取 / 服务层
### 样式
## 开发指南
## 项目约定
## Do
## Don't
## Before Finishing（必需章节）
## Dangerous Operations（必需章节）
## 禁止的自我合理化（推荐章节）
## Context Management（推荐章节）
## Notes for Claude Code
```

## 根目录知识库模板

将稳定项目知识放在根目录 `repository/`，作为对外暴露的标准知识目录：

```text
repository/
  README.md
  glossary.md
  architecture.md
  modules.md
  integrations.md
  decisions/
```

### 推荐职责

| 路径 | 推荐内容 | 避免内容 |
|---|---|---|
| `repository/README.md` | 知识库入口、文件说明、阅读顺序 | 复制全文到其他文件 |
| `repository/glossary.md` | 领域术语、缩写、关键名词 | 一次性需求 |
| `repository/architecture.md` | 架构边界、主流程、系统关系 | 命令清单 |
| `repository/modules.md` | 目录/模块职责、边界、复用点 | 逐文件流水账 |
| `repository/integrations.md` | 外部系统、接口契约、依赖约束 | 私密凭据 |
| `repository/decisions/` | 稳定设计决策、长期约束 | 临时讨论记录 |

## 必需章节

### Before Finishing（每次完成前必须检查）

这是 Hardness 的关键门禁，每个项目必须包含可执行、可验证的完成检查清单：

```markdown
## Before Finishing（每次完成前必须检查）

- [ ] 运行 `npm run typecheck`，期望 0 error
- [ ] 运行 `npm run lint`，期望 0 error
- [ ] 运行相关测试，全部通过
- [ ] 确认 tasks.md 全部勾选
- [ ] 确认只修改了预期文件（`git status`）
```

### Dangerous Operations（STOP 门禁）

标注高风险操作的 STOP 条件：

```markdown
## Dangerous Operations（STOP）

- 涉及 `DROP TABLE` 或 `DELETE` 无 WHERE 条件 → STOP，必须人工确认
- 涉及修改 `src/config/` 下文件 → STOP，必须人工 review
- 涉及数据删除或生产部署 → STOP，必须人工确认
```

### 禁止的自我合理化（推荐章节）

封堵模型常见的自我合理化路径：

```markdown
## 禁止的自我合理化（MUST NOT）

- 不得以“改动小”为由跳过测试验证
- 不得以“逻辑上正确”代替实际运行验证
- 不得以“基本完成”为由勾选未验证的 checkbox
- 不得以“代码看起来没问题”代替实际测试
```

### Context Management（推荐章节）

> 本节条目是**可独立生效的执行纪律**：复制到任意项目 `CLAUDE.md` 后无需引用其他文件即能约束会话行为。完整方法论（四层模型、委派契约、Apply 期读写减负）见本仓 `docs/Context-Hygiene-Practice.md`，但**不依赖该路径生效**——以下条目本身已是最小可执行集合。

把上下文窗口当作有限可耗尽资源主动管理，而不是等撑爆再被动处理：

```markdown
## Context Management（推荐章节）

- 探索/对账会读大量文件只为得一个结论 → 委派 subagent，主上下文只收结论+证据位置（文件:行 / 命令+摘要+时间戳），不收原文
- 窗口变重且本任务未完 → 先把进度落 tasks.md/verify.md，再 `/compact`，不丢状态继续
- 一个 Change 已归档、要切下一主题 → 状态已落盘后 `/clear` 开新会话，避免旧上下文污染新 Change
- 压缩/清空前必须确认 tasks 与 verify 证据已落盘；缺失则视为未完成，先补证据再压缩
- 委派的 subagent 只回依据，不持 task lifecycle / checkoff / STOP 裁决（决策留在主上下文）
```

## 技术栈特定章节建议

- **前端**：可补路由、状态、请求、样式
- **后端**：可补入口、路由/控制器、模块、服务层、数据层、配置
- **全栈框架**：可补客户端/服务端边界、数据加载和构建
- **库/组件包**：可补导出入口、类型声明、构建产物、示例或文档位置

## 写法提醒

- `CLAUDE.md` 写 Claude 如何执行，不写完整项目百科
- `repository/` 写稳定事实，不写“保持高质量”这类流程口号
- 需要迁移内容时，优先把术语、架构、模块、集成说明从单文件挪到 `repository/`
- 不要再新增 `project.md` 作为项目知识主载体
