# harness-claude-setup 技能优化记录

## 优化日期
2026-06-24

## 优化前问题

### 关键问题
1. ❌ **无 TDD 证据** - 未进行基线测试
2. ❌ **语言不一致** - 中英文混杂
3. ⚠️ **描述未第三人称** - 不符合 CSO 规范
4. ❌ **Token 效率低** - 304 行 / ~2400 词，远超 500 词建议
5. ❌ **未使用 references/** - 所有内容都内联

### 质量评分
- 内容质量：9/10（非常全面）
- TDD 合规：0/10（无测试证据）
- Token 效率：3/10（过长）
- CSO：6/10（描述需优化）
- 结构：7/10（好但可用 references/）

## 优化措施

### 1. 基线测试（RED 阶段）

测试场景 | 无技能时的行为 | 期望行为
---|---|---
业务需求混入 | ✅ 正确拒绝 | 拒绝并建议替代文档
多项目歧义 | ❌ 直接更新根目录 | STOP 并询问目标项目
不存在的命令 | ✅ 验证并拒绝 | STOP 并报告不一致

**关键发现**：多项目歧义检测是薄弱点，需要在技能中强化。

### 2. 结构重组

**优化前**：304 行单文件
```
harness-claude-setup/
  └── SKILL.md (所有内容)
```

**优化后**：压缩到 391 词 + references/
```
harness-claude-setup/
  ├── SKILL.md (核心流程，~391 词)
  └── references/
      ├── evidence-guidelines.md (证据映射、冲突优先级)
      ├── structure-templates.md (CLAUDE.md 模板)
      └── common-mistakes.md (错误表、红旗警告)
```

### 3. 语言统一

- 全部改为中文（与现有内容主体语言一致）
- 保留必要的英文术语（MUST/SHOULD/MAY/STOP）

### 4. 描述优化

**优化前**（英文，不含症状）：
```yaml
description: Use when initializing, improving, restructuring, localizing, or synchronizing one project's CLAUDE.md - maintains accurate Claude Code working guidance from verified project facts while avoiding business specs, stale commands, and multi-project drift.
```

**优化后**（中文，症状导向）：
```yaml
description: 当遇到 CLAUDE.md 过时命令、多项目目录歧义、未验证声明或需要初始化项目文档时使用 - 从可执行来源维护单项目工作指南，防止多项目混淆和业务需求混入
```

关键改进：
- 添加触发症状："过时命令"、"多项目目录歧义"、"未验证声明"
- 更符合 CSO 搜索优化原则

### 5. 内容压缩

压缩技术 | 应用
---|---
移动详细表格到 references/ | ✅ 证据映射表、常见错误表
保留核心流程 | ✅ 5 步工作流程精简版
交叉引用 | ✅ "详见 references/xxx.md"
删除冗余 | ✅ 移除重复的章节说明

## 验证结果（GREEN 阶段）

### 测试场景重跑

测试场景 | 优化后行为 | 结果
---|---|---
多项目歧义 | ✅ STOP 并明确询问目标项目 | **通过**
不存在的命令 | ✅ STOP 并报告冲突，拒绝写入 | **通过**
时间压力（"赶时间"） | ✅ 坚持 MUST 级硬约束，不绕过验证 | **通过**
业务需求混入 | ✅ 识别并重定向到 spec/issue | **通过**

### 合理化封堵

测试的合理化借口 | 技能是否抵御
---|---
"用户说有这个命令" | ✅ 要求验证可执行来源
"赶时间，快点写" | ✅ MUST 约束不因速度绕过
"根目录有 CLAUDE.md，更新它吧" | ✅ 多项目时必须 STOP 询问
"这是标准命令" | ✅ 标准≠存在，必须验证

## 优化后质量评分

- 内容质量：9/10（保持高质量）
- TDD 合规：10/10（完整的 RED-GREEN-REFACTOR）
- Token 效率：9/10（391 词，符合 <500 词建议）
- CSO：9/10（症状导向描述）
- 结构：10/10（清晰的 references/ 分离）

**总分：9.4/10**

## 关键改进点

1. **多项目歧义检测强化** - 工作流程第 1 步明确要求 STOP
2. **MUST 级硬约束** - 不因速度或催促绕过验证
3. **红旗警告系统化** - 移至 references/common-mistakes.md，易于维护
4. **证据驱动流程** - 详细的证据映射表在 references/evidence-guidelines.md

## 遵循的 TDD 原则

- ✅ 先运行基线测试（RED）
- ✅ 记录实际失败模式
- ✅ 针对性改进技能（GREEN）
- ✅ 重跑验证通过（GREEN 确认）
- ✅ 识别合理化路径并封堵（REFACTOR）

## 下一步建议

1. **持续监控** - 在实际使用中观察是否有新的合理化路径
2. **压力测试** - 组合多个压力（时间 + 权威 + 沉没成本）
3. **边界案例** - 测试更复杂的 monorepo 结构（嵌套包、工作区）
4. **用户反馈** - 收集 STOP 询问的频率，判断是否过于严格

## 技能使用指南

**何时使用**：
```bash
/harness-claude-setup
```

当遇到以下情况时调用：
- CLAUDE.md 包含 `npm test` 但 package.json 中无此 script
- Monorepo 中不清楚更新哪个 CLAUDE.md
- 用户想把 feature spec 写入 CLAUDE.md
- 需要初始化新项目的 CLAUDE.md

**何时不用**：
- 只是阅读或解释 CLAUDE.md
- 处理 OpenSpec 或 README
- 一次性 TODO 或聊天上下文
