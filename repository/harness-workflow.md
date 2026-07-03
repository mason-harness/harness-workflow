# Harness + OpenSpec 工作流总览

本文档只保留当前仓库的工作流总览、推荐接线方式与典型时机判断。

更稳定的结构化知识已经拆分到：
- [architecture.md](architecture.md) — 仓库定位与 OpenSpec / Harness / Hook 三层职责
- [modules.md](modules.md) — 各 skill 家族的模块边界
- [integrations.md](integrations.md) — 与 `/opsx:*`、OpenSpec CLI、Hook 的接线关系

## 核心原则

- OpenSpec 负责 Change lifecycle 主线。
- Harness 负责纪律、门禁、证据与工作文档治理。
- Hook 只负责高风险、不可绕过的硬 STOP。

一句话总结：

> Harness 先搭规则与知识边界，OpenSpec 再跑生命周期；切片 skills 解决规模问题，apply / verify harness 解决执行纪律与证据门禁。

## 推荐流程

### 1. 项目准备期

适用场景：
- `CLAUDE.md` 不准或缺失
- `repository/` 知识散乱
- OpenSpec 规则缺失
- 高风险操作需要硬门禁

推荐顺序：

```text
harness-claude-setup
harness-openspec-setup
harness-hook-setup   # 有高风险操作时
```

### 2. 普通单 Change 开发

```text
/opsx:propose
/opsx:apply
harness-openspec-apply
/opsx:verify
harness-openspec-verify
/opsx:archive
```

说明：
- `/opsx:*` 负责主生命周期。
- `harness-openspec-apply` 与 `harness-openspec-verify` 只做增强，不替代主流程。

### 3. 大需求 / 多切片开发

```text
/opsx:explore
openspec-slices-plan
openspec-slices-register
openspec-slices-track

# 然后每个 slice：
/opsx:apply
harness-openspec-apply
/opsx:verify
harness-openspec-verify
/opsx:archive
```

说明：
- 当一个 Change 装不下需求时，先切片，再逐个 slice 进入普通流程。
- `openspec-slices-track` 负责跨 session 的整体进度判断。

## 典型时机判断

### 先用 Harness 基础技能的情况

- 规则源不可信或文档过时
- 项目知识未沉淀到根级知识库
- OpenSpec 规则缺失或过弱
- 高风险操作还没有 Hook

### 先用切片技能的情况

- 一个 Change 无法独立交付
- 一个 Change 无法独立验证
- 一个 Change 无法独立回滚
- 顺序依赖明显，适合拆成多个 slice

### 进入 `harness-openspec-apply` 的情况

- 已进入实现阶段
- 想防止 scope 蔓延、未验证勾选、task 设计缺口被吞掉

### 进入 `harness-openspec-verify` 的情况

- 已进入验收阶段
- 想防止“没有证据却宣称完成”
- 想在 archive 前增加更强的软门禁

## 默认策略

### 项目初始化默认策略

```text
harness-claude-setup
harness-openspec-setup
# 如有高风险操作：
harness-hook-setup
```

### 普通 Change 默认策略

```text
/opsx:propose
/opsx:apply
harness-openspec-apply
/opsx:verify
harness-openspec-verify
/opsx:archive
```

### 大需求默认策略

```text
openspec-slices-plan
openspec-slices-register
openspec-slices-track
# 然后每个 slice 按普通 Change 流程执行
```

## 相关文档

- [../README.md](../README.md) — 仓库总览与技能清单
- [README.md](README.md) — 知识库入口
- [architecture.md](architecture.md) — 架构与分层职责
- [modules.md](modules.md) — 技能模块边界
- [integrations.md](integrations.md) — 接线关系与边界