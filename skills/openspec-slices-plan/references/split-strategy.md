# Change 拆分策略

本文档提供 Change 拆分的决策树、按项目形态选择切片维度的原则、Wrong/Right 示例、序号前缀管理、progressive 拆分策略、Proposal storyline 机制、Initiative 跨仓协作与 workspace-change 限制。

## 拆分决策树

```
需求输入
  ↓
范围清晰？ ──No→ STOP，先澄清边界
  ↓ Yes
单一责任单元 + 文件≤5 + 代码≤150行？
  ↓ Yes                    ↓ No
单切片 Slice Plan        可独立推进？
  (mode=single-repo)        ↓ No              ↓ Yes
                          合并 → 单 Change    判断项目形态
                                                ↓
                           单一项目/单仓？ ──Yes→ 垂直业务切片
                                ↓ No                    ↓
                     多项目/多仓/多职责面       按用户价值端到端切
                                ↓                       ↓
                     技术层/仓库边界切片        depends_on + sequencing
                                ↓                       ↓
                         跨仓/多团队？ ──Yes→ mode=cross-repo + Initiative
                                ↓ No                    ↓
                          mode=single-repo       跨仓: 检查 workspace ──Yes→ STOP
                                                        ↓ No
                                                  合法: 可 link initiative
```

**拿不准时**：选 `sequencing_rule: merge-first-split-later`，先合成单 Change，边界清晰后再拆。

## 按项目形态选择切片维度

### 单一项目 / 单仓：优先垂直业务切片

**定义**：按用户可感知的价值端到端切，每个切片归档后主干可编译可部署、系统仍处合法状态。

**适用原因**：同一仓内若过早按 frontend/backend/db 横切，常会得到“半成品切片”——某片完成后无法独立验证，主干虽能合并但能力不完整。

### 多项目 / 多仓 / 多职责面：允许技术层或仓库边界切片

**定义**：当需求天然分布在多个项目、仓库或团队职责面时，可按 frontend/backend/db 等技术层，或直接按 repo/service 边界切片；每片必须有清晰 owner、交付边界与 `depends_on`。

**适用原因**：此类场景的真实协调边界本就存在于仓库或团队之间，强行按单一业务故事竖切，反而会把一个切片拆散到多个 owner、多个 repo 中，降低登记与跟踪清晰度。

### Wrong vs Right 示例 1：单一项目的用户导出功能

**需求**：用户可导出数据为 CSV/JSON，含进度条、错误提示、取消操作。

#### ❌ Wrong: 在单一项目里按技术层横切

```
01-frontend-ui        # 导出按钮 + 弹窗 UI（后端接口未对接，点击无反应）
02-backend-api        # /api/export 接口（frontend 未调用，孤立）
03-database-query     # 优化导出 SQL（无人调用）
04-error-handling     # 错误边界（依赖前三者全完成）
```

**问题**：
- 01 归档后，按钮存在但点击无反应（用户困惑）
- 02 归档后，接口存在但无人调用（孤立代码）
- 03 归档后，优化存在但未生效（死代码）
- 04 最后才能补，前三个切片无错误处理（系统脆弱）

#### ✅ Right: 在单一项目里做垂直业务切片

```
01-foundation         # CSV 导出主路径（按钮 → API → 查询 → 下载），golden path，无进度/取消/错误
02-progress-feedback  # 进度条 + 取消操作（依赖 01 的主路径，增强体验）
03-error-resilience   # 错误提示 + 重试（依赖 01/02，最后增强健壮性）
```

**优点**：
- 01 归档后：CSV 导出基础功能可用，主干合法（虽无进度条但能用）
- 02 归档后：进度条生效，体验改善，主干仍合法
- 03 归档后：错误处理生效，系统更健壮
- 每步归档后系统都处于"可发布"状态

### Wrong vs Right 示例 2：多项目分离交付的认证升级

**场景**：`web-frontend`、`api-gateway`、`auth-service` 分属不同仓库/职责面。

#### ❌ Wrong: 强行按单一业务故事竖切

```
01-login-story        # 同时改 web-frontend / api-gateway / auth-service
02-session-story      # 再同时改三个仓库
03-permission-story   # 继续跨三个仓库
```

**问题**：
- 每个切片都跨多个 owner，责任边界不清
- 任一切片都需要多仓同时推进，登记、归档、回滚都更重
- `depends_on` 只能描述故事顺序，无法表达真实的仓库/服务依赖

#### ✅ Right: 按技术层或仓库边界切片

```
01-auth-service-provider   # auth-service 提供 OAuth provider
02-gateway-client          # api-gateway 对接 OAuth client
03-frontend-login          # web-frontend 更新登录流程
```

**优点**：
- 每片 owner、repo、测试边界明确
- `depends_on` 能准确表达服务链依赖
- 更适合 cross-repo initiative、归档顺序与跨团队协作

**context / dependencies / scope 示例**（01-foundation）：

```yaml
context: |
  用户导出功能的基础主路径（golden path）。属于"数据导出 Epic"，是后续进度条与错误处理的基座。
dependencies:
  - 无（foundation 切片，无前序依赖）
scope:
  in:
    - CSV 格式导出
    - 同步查询（小数据集，≤1000 行）
    - 下载触发
  out:
    - 进度条（交 02-progress-feedback）
    - 取消操作（交 02）
    - 错误提示/重试（交 03-error-resilience）
    - JSON 格式（未来 Epic）
    - 异步导出/大数据集分页（未来 Epic）
```

## 序号前缀管理

单仓多切片场景，用**两位序号前缀** + kebab-case 名称管理依赖顺序：

```
openspec/changes/
  01-foundation/
    .openspec.yaml
    proposal.md
    ...
  02-progress-feedback/
    .openspec.yaml      # 可在此注明 "Depends: 01-foundation must be archived first"
    proposal.md
    ...
  03-error-resilience/
    .openspec.yaml
    proposal.md
    ...
```

**命名规则**：
- 序号：`01`/`02`/...，两位数字，按依赖顺序递增
- 名称：kebab-case，简明扼要（≤30 字符）
- 合成：`openspec new change "01-foundation"` / `"02-progress-feedback"`

**依赖标注**：每个切片的 `depends_on` 字段列前序序号（如 `["01"]`），由 `openspec-slices-register` 物化进 proposal.md 的 Dependencies 章节。

## Progressive 拆分策略：先合后拆

**merge-first-split-later**: 拿不准边界时，先合成单 Change 实现，边界清晰后再拆。

**适用场景**：
- 新领域探索，切片边界不明
- 复杂依赖，拆分可能引入循环
- 小型需求（≤300 行），拆分管理成本超开发成本

**操作**：
- 初期：产出单切片 Slice Plan（`slices=[1]`, `sequencing_rule: merge-first-split-later`）
- 实现中：若发现可拆，在 proposal.md 标注"TODO: 边界清晰后拆为 XX/YY 切片"
- 事后：用 `openspec-slices-plan` 重新评估，产出多切片 Slice Plan

**判定过度拆分**：
- 切片数 >5 且每切片 <50 行 → 考虑合并
- 切片间存在频繁交叉调用（非清晰依赖链）→ 合并
- 管理成本（序号维护、handoff 协调）超开发成本 → 合并

**判定拆分不足**：
- 单 Change tasks >50 项 → 检查可否独立为子 Change
- 单 Change 文件 >15 个 → 检查可否按模块拆
- 单 Change 跨越多个用户可感知价值点 → 检查可否按价值拆

## Proposal storyline 策略

每个 proposal.md 通过 **Context / Dependencies / Scope / Handoff** 四问串联故事线：

1. **Context（我在哪）**：1–2 行故事线定位，回答"本 Change 属于哪个 Epic / 在整体需求中的位置"
2. **Dependencies（谁依赖我/我依赖谁）**：列出前序切片 sequence / 外部依赖 / 后续切片（Blocks）
3. **Scope（坚决不做啥）**：in（本次做）/ out（本次坚决不做，交给谁）
4. **Handoff（下游能依赖什么）**：仅在存在真实跨切片契约时填写，说明交给谁、交付什么、ready signal 与 consumer expectations

**示例**（02-progress-feedback 的 proposal.md 片段）：

```markdown
## Context
用户导出功能的进度反馈增强。属于"数据导出 Epic"，依赖 01-foundation 已归档的 CSV 导出主路径，为后续 03-error-resilience 提供进度状态基础。

## Dependencies
- **Depends on**: 01-foundation（CSV 导出主路径必须已归档并可用）
- **Blocks**: 03-error-resilience（错误处理依赖本切片的进度状态）
- **External**: 无

## Scope
### In（本次做）
- 导出进度条（百分比 + 行数）
- 取消操作（中断查询）
- 进度状态持久化（供 03 错误重试读取）

### Out（本次坚决不做）
- 错误提示/重试逻辑（交 03-error-resilience）
- JSON 格式导出（未来 Epic）
- 异步导出/大数据集分页（未来 Epic）
```

## Initiative 跨仓协作机制

**适用场景**：跨多个仓库 / 多团队协作的需求，需中心化协调。

**数据模型**：
- **Context store**: 中心化存储，一个 git repo，含 `initiatives/<id>/` 目录
- **Initiative**: `{store, id}`，存于 `<storeRoot>/initiatives/<id>/`，含 `initiative.yaml` + 5 md（requirements/design/decisions/questions/tasks）
- **Change↔Initiative 链接**: 写 change 的 `.openspec.yaml` 的 `initiative: {store, id}`

**登记流程**（由 `openspec-slices-register` 执行）：
1. `openspec context-store setup <store-id> --path <storeRoot> --init-git`（若 store 不存在）
2. `openspec initiative create <id> --title <t> --summary <s> --store <store>`
3. 各仓库内：`openspec new change "<name>" --initiative <store>/<id> --goal <g>`

**关键限制**：
- **仅 repo-local 可链接 initiative**：workspace change 不得 link（CLI 契约限制，`set change --initiative` 对 workspace 报错）
- 遇 workspace change 需跨仓场景 → **STOP**，要求改 repo-local 或退回用户决策

## 过度/不足拆分判定清单

### 过度拆分（应合并）
- [ ] 切片数 >5 且每切片代码 <50 行
- [ ] 切片间存在频繁交叉调用（非清晰依赖链）
- [ ] 序号前缀管理成本 + handoff 协调时间超开发时间
- [ ] 单切片无法独立验证/测试（过细粒度）

### 拆分不足（应拆分）
- [ ] 单 Change tasks >50 项
- [ ] 单 Change 文件 >15 个，跨越多个模块
- [ ] 单 Change 跨越多个用户可感知价值点（如"导出 + 分享 + 权限"三合一）
- [ ] 归档后无法独立发布（缺少可独立交付价值）

### Right-sized Change 检查（无需拆）
- [ ] 单一责任单元（一个用户可感知价值点）
- [ ] 文件影响 ≤5
- [ ] 代码 ≤150 行
- [ ] tasks ≤15 项
- [ ] 归档后主干可编译可部署、系统处合法状态
