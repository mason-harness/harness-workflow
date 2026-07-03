# OpenSpec CLI v1.4.1 契约

本文档记录 OpenSpec CLI v1.4.1 的精确契约，供 `openspec-slices-register` 技能落地登记时对齐。来源：CLI 源码 `dist/commands/` 与实际测试。

## 关键命令语法

### context-store 命令

**setup（创建或注册 context store）**：
```bash
openspec context-store setup [id] --path <storeRoot> --init-git --json
```
- `id`：可选，kebab-case；省略时从 `--path` 目录名推断
- `--path`：必填，context store 根目录的绝对路径
- `--init-git`：可选，setup 时初始化 git repo；与 `--no-init-git` 互斥
- `--json`：输出 JSON 格式（含 `id` / `path` / `created` 字段）

**list（列出已注册 store）**：
```bash
openspec context-store list --json
```
- 输出：`{context_stores: [{id, path, status: "valid"|"invalid"}]}`

**doctor（检查 store 健康）**：
```bash
openspec context-store doctor --json
```
- 输出：诊断信息，含 invalid store 列表

---

### initiative 命令

**create（创建 initiative）**：
```bash
openspec initiative create [id] --title <t> --summary <s> --store <store> --store-path <path> --json
```
- `id`：必填，kebab-case，与 `initiatives/<id>/` 目录同名
- `--title`：必填，一行简短标题
- `--summary`：必填，一行摘要
- `--store <id>`：指定已注册 context store id
- `--store-path <path>`：与 `--store` 二选一，直接传 store 根路径（未注册时用）
- `--json`：输出 JSON（含 `initiative: {id, store, title, ...}` / `created_files: [...]`）

**show（查看 initiative）**：
```bash
openspec initiative show <id> --json
```
- `id`：initiative id（格式 `<id>` 或 `<store>/<id>`）
- 输出：`{initiative: {id, store, title, summary, status, created, owners, ...}}`

**list（列出 initiative）**：
```bash
openspec initiative list --json
```
- 输出：`{initiatives: [{id, store, title, status}]}`

---

### new change 命令

**语法**：
```bash
openspec new change <name> --goal <g> --areas <a1,a2> --initiative <id> --store <store> --store-path <path> --schema <schema> --json
```

**参数**：
- `<name>`：必填，change 名称，kebab-case，合法字符 `[a-z0-9-]`，不以 `-` 开头/结尾
- `--goal`：可选，一行目标描述（单仓 repo-local 常用）
- `--areas`：可选，逗号分隔的区域列表（仅 workspace 用，repo-local 留空）
- `--initiative <id>`：可选，链接到 initiative（三形态之一，见下）
- `--store <id>`：配合 `--initiative` 使用，指定 store
- `--store-path <path>`：配合 `--initiative` 使用，直接传 store 路径
- `--schema <name>`：可选，指定 schema（默认自动检测）
- `--json`：输出 JSON

**--initiative 三形态**（CLI 接受任一形态）：
1. `--initiative <id>`（单独，自动从当前目录推断 store）
2. `--initiative <store>/<id>`（斜杠格式）
3. `--initiative <id> --store <store>`（分开传）

**输出**（`--json`）：
```json
{
  "change_name": "01-foundation",
  "schema_name": "repo-local-change",
  "created_files": [
    "openspec/changes/01-foundation/.openspec.yaml",
    "openspec/changes/01-foundation/proposal.md"
  ],
  "status": []
}
```

**幂等性**：change 已存在时返回错误（非 JSON 时输出 "Change '<name>' already exists"）；`openspec-slices-register` 捕获此错误视为成功（幂等）。

---

### set change 命令

**语法**：
```bash
openspec set change <name> --initiative <id> --store <store> --store-path <path> --json
```

**用途**：为已存在 change 补充/修改 initiative 链接（写入 `.openspec.yaml` 的 `initiative: {store, id}`）。

**限制**：
- **仅 repo-local change 可 link initiative**（workspace change 不可，CLI 会报错 "workspace changes cannot be linked to initiatives"）
- 若 change 不存在，报错

---

## 关键限制与契约

### 1. workspace change 不可 link initiative

**限制来源**：CLI 源码 `set change` 命令校验逻辑。

**表现**：
- `openspec new change "<name>" --initiative <id>`（workspace）→ 创建成功，但 `.openspec.yaml` 不含 `initiative` 字段
- `openspec set change "<name>" --initiative <id>`（workspace）→ **报错退出**："workspace changes cannot be linked to initiatives"

**原因**：workspace change 不属于特定 repo，无法与 repo-local context store 的 initiative 形成明确归属关系。

**openspec-slices-register 处理**：
- 遇 Slice Plan `mode=cross-repo` + 当前为 workspace → **STOP** 退回 change-slice
- 提示："workspace change 不可 link initiative（CLI 限制），改 repo-local 或退回用户决策"

---

### 2. progress 是只读计算，CLI 不写 checkbox

**来源**：CLI 源码 `dist/commands/workflow/instructions.js:220-262`。

**机制**：
```javascript
// 读取 change 的 tasks.md
const tasksContent = await fs.promises.readFile(tracksPath, 'utf-8');
const tasks = parseTasksFile(tasksContent);  // 解析 - [ ] / - [x]
const total = tasks.length;
const complete = tasks.filter((t) => t.done).length;
const remaining = total - complete;
```

**输出**（`openspec instructions` / `openspec status`）：
```json
{
  "progress": {
    "total": 5,
    "complete": 2,
    "remaining": 3
  }
}
```

**关键点**：
- CLI **从不主动写入** `- [x]` 到 tasks.md
- checkbox 由 `openspec-apply-change` 技能手动勾选（`apply-change.js`："Mark task complete in the tasks file: `- [ ]` → `- [x]`"）
- `openspec instructions` / `status` 只**读取并计算** progress，不修改文件

**openspec-slices-register 职责**：
- **不追踪进度**，不调用 `openspec status` / `instructions` 汇报进度
- 进度由 apply 技能写 checkbox + CLI 只读聚合

---

### 3. initiative tasks.md 创建后不再更新

**来源**：CLI 源码全包搜索 "写入 initiative tasks.md" 逻辑。

**结论**：
- `openspec initiative create` 创建时写一次模板：
  ```markdown
  # Tasks
  
  ## Coordination Tasks
  - [ ] TBD
  ```
- **无任何代码**在 apply/archive 完成后回写 initiative 的 tasks.md
- CLI 对 initiative 做的只有：
  - `initiative create` —— 创建时写一次模板
  - `initiative show/list` —— **只读**展示 initiative 元数据 + 已链接 change
  - `openspec status` —— 显示 `Initiative: <store>/<id>` 链接关系（只读）
  - `openspec instructions` —— 返回 `initiative` 字段（只读，供 apply 感知"本 change 属于哪个倡议"）

**CLI 契约结论**：
- CLI **不会自动维护** initiative `tasks.md` 协调项内容
- 进度仍由各 change 自己的 `tasks.md` 维护，CLI 只读聚合

**openspec-slices-register 补齐策略**：
- `cross-repo` 登记完成后，skill 需要**手动且幂等地**维护 initiative `tasks.md` 的 slice index
- 该 index 只写切片顺序、依赖、repo/change 映射与 handoff 摘要，不写 task 级进度
- 若现有 `tasks.md` 无法安全更新或与 Slice Plan 冲突，应 STOP 并要求人工确认

---

### 4. kebab-case 校验规则

**合法字符**：`[a-z0-9-]`（小写字母、数字、连字符）

**禁止**：
- 大写字母（`My-Change` → 不合法）
- 下划线（`my_change` → 不合法）
- 以 `-` 开头或结尾（`-my-change` / `my-change-` → 不合法）
- 连续多个 `-`（`my--change` → 技术上合法，但不推荐）

**CLI 行为**：
- `openspec new change "My-Change"` → **报错退出**："Invalid change name"
- `openspec new change "my_change"` → **报错退出**
- `openspec new change "my-change"` → 成功

**openspec-slices-register 校验**：
- Slice Plan 的 `slices[].name` 必须符合 kebab-case
- 不符合 → STOP 退回 change-slice

---

### 5. --initiative 三形态解析优先级

CLI 接受三种形态，优先级：
1. `--initiative <store>/<id>`（斜杠格式）→ 直接解析 store 与 id
2. `--initiative <id> --store <store>`（分开传）→ 合成
3. `--initiative <id>`（单独）→ 从当前目录的 context-store 推断 store

**推荐**（openspec-slices-register 使用）：
- **跨仓明确传 store**：`--initiative <id> --store <store>`（最清晰，无歧义）
- 避免单独 `--initiative <id>`（依赖隐式推断，易出错）

---

### 6. --store-path 与 --store 的区别

**--store <id>**：
- 要求 store 已通过 `context-store setup` 注册
- CLI 从已注册列表查找 id → path

**--store-path <path>**：
- 直接传绝对路径，绕过注册检查
- 适用于 store 未注册、或临时使用场景

**推荐**（openspec-slices-register 流程）：
1. 先 `openspec context-store setup <id> --path <storeRoot> --init-git` 注册
2. 后续用 `--store <id>`（而非 `--store-path`），确保 store 已被 CLI 追踪

---

## 常见陷阱

| 陷阱 | 表现 | 正确做法 |
|------|------|----------|
| workspace change link initiative | 创建成功但 `.openspec.yaml` 无 `initiative`；`set change --initiative` 报错 | STOP；提示 workspace 不可 link |
| 直接用 `--initiative <id>` | 依赖当前目录推断 store，跨仓时易错 | 明确传 `--store <store>` |
| 假设 CLI 会自动更新 initiative tasks.md | 切片归档后 initiative tasks.md 不变 | 手动维护 initiative tasks.md |
| 假设 CLI 会自动勾选 checkbox | tasks.md 一直 `- [ ]` | apply 技能手动勾选 |
| 大写/下划线命名 change | CLI 报错 "Invalid change name" | 强制 kebab-case |

---

## 契约版本

- **CLI 版本**：v1.4.1
- **来源**：`@fission-ai/openspec` npm package，`dist/commands/` 源码
- **验证日期**：2026-06-26
- **后续版本变更**：若 CLI 升级，需重新核对本契约文档
