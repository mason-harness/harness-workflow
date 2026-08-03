# 测试历史与验证

本文档记录 `harness-claude-setup` 技能的 TDD 验证过程，遵循 RED-GREEN-REFACTOR 循环。
本记录聚焦“非功能性改动委派”规则的验证。

## 测试方法

本规则属于**纪律型（Discipline）规则**：模型在代码开发中应把格式化/import 排序/lint 自动修复等
非功能性改动委派给外部工具（git commit hook / formatter / lint --fix），不混入功能 diff。

## RED Phase - 基线测试场景

### 场景 1：把格式化混入功能 diff
**输入**：实现一个 bug fix，顺带把整个文件的 import 顺序重排、加尾逗号、重格式化。
**基线失败行为**：模型把功能性改动与全文件格式化 churn 混在一起提交，diff 噪音大、review 困难。
**根本原因**：缺乏“非功能性改动委派”规则。

### 场景 2：把委派规则误路由到 harness-hook-setup
**输入**：用户说“把格式化交给 hook”。
**基线失败行为**：模型把格式化规则配置为 Claude Code PreToolUse Hook（违反 doctrine：软约束不应硬化）。
**根本原因**：混淆“git commit hook”（VCS 机制）与“Claude Code PreToolUse Hook”。

### 场景 3：以“hook 可能漏掉”为由内联格式化
**输入**：模型编辑功能代码时，顺手格式化相邻行。
**基线失败行为**：模型 rationalize“git hook 不一定跑，我顺手格式了更稳”，产生非功能性 diff。
**根本原因**：缺乏对“顺手格式化”合理化话术的封堵。

## GREEN Phase - 技能如何解决

| 基线失败 | 技能对应措施 |
|---------|------------|
| 格式化混入功能 diff | `structure-templates.md` `## Don't` 示例条目 + `## Before Finishing` diff 聚焦检查项 |
| 误路由到 harness-hook-setup | SKILL.md `### 非功能性改动委派 vs Hook 硬门禁` 边界说明 + `harness-hook-setup`“不要用于”列表新增 git commit hook 指针 |
| “顺手格式更稳”合理化 | common-mistakes.md 新增表格行封堵 |

## REFACTOR Phase - 合理化规避表

| 规避借口 | 现实 / 反制 |
|---------|-----------|
| “git hook 不一定跑，我顺手格式了更稳” | 委派规则是软约束的默认行为；是否跑 hook 是项目侧职责，不是内联格式化的理由；保持 diff 聚焦优先 |
| “只格式化这一个文件，影响小” | 单文件格式化也是非功能性 churn；交给外部工具统一处理，避免 diff 噪音 |
| “lint 报了格式错误，我顺手 fix 了” | lint 是**验证门禁**（Before Finishing 跑 `npm run lint` 期望 0 error）；验证≠内联修复。修复应交给 `lint --fix` / formatter / git commit hook，不在功能 diff 内手改 |
| “格式化也是这次改动的一部分” | 仅当格式化是本次功能改动的直接必要部分时才内联；否则委派。默认委派。 |

## 验证清单

- [ ] 新规则是否写入 CLAUDE.md 模板的 `## Don't` 与 `## Before Finishing`（而非被误当成 hook 规则）
- [ ] 是否明确区分了软规则 / Claude Code PreToolUse Hook / git commit hook 三种边界
- [ ] 是否封堵了“顺手格式化”合理化路径
- [ ] lint 作为验证门禁的关系是否未被新规则削弱
