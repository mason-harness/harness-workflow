---
name: harness-hook-setup
description: Use when setting up PreToolUse Hooks, model bypasses safety rules, enforcing CLAUDE.md Dangerous Operations with hard gates, or validating Hook coverage - reads project rules from CLAUDE.md/config.yaml, generates Hook configurations to settings.json with testable enforcement scripts (SQL/config/deletion guards) that return DENY and cannot be rationalized away by the model.
---

# 配置 Claude Code Hooks

## Overview

将 `CLAUDE.md` 和 `openspec/config.yaml` 中标注需要硬门禁的规则转化为 Claude Code Hooks（PreToolUse），实现模型无法绕过的硬约束。

核心原则：**只配置必要的 Hook，不修改源规则；先缩小拦截面再写正则；生成的 Hook 必须可测试、可维护，并且默认不打扰正常开发。**

**第一原则：限制拦截范围优先于增加拦截数量。** 任何 Hook 在落地前都要先回答 4 个问题：拦截哪个工具、拦截哪个目录/仓库、拦截哪一类高风险输入、哪些正常开发路径必须放行。

**关键术语**：PreToolUse Hook, settings.json, DENY action, dangerous operations, hard gates, enforcement scripts

## When to Use

使用本 skill 处理以下请求：

- 配置、生成、更新 Claude Code Hooks（PreToolUse）
- 将 CLAUDE.md 的 Dangerous Operations 转化为硬拦截
- 将 config.yaml 的 STOP 条件转化为硬门禁
- 检查现有 Hook 配置是否覆盖了所有危险操作
- 修复或删除失效的 Hook 配置
- 模型绕过了 safety rules 需要 hard enforcement

不要用于：

- 修改 CLAUDE.md 或 config.yaml 本身（使用 `harness-claude-setup` 或 `harness-openspec-setup`）
- 配置 keybindings、model、theme 等非 Hook 设置（使用 `harness-hook-setup`）
- 执行 OpenSpec 工作流（使用 `openspec-slices-track`）

## Critical Guidelines

1. **必须先读后写。** 读取 `CLAUDE.md`、`openspec/config.yaml`、现有 `settings.json` / `settings.local.json`，再生成 Hook 配置。
2. **必须只配置一个目标项目。** 如果仓库有多个项目且 Hook 范围不清楚，先确认目标目录；全局 Hook（`~/.claude/settings.json`）和项目 Hook（`.claude/settings.json`）的影响范围不同。
3. **必须区分软/硬 STOP。** 只有 Dangerous Operations 和明确标注需要硬门禁的规则才配置 Hook；普通规则保持软约束。
4. **必须生成可测试的脚本。** Hook 拦截脚本必须可以独立测试；提供测试用例和验证命令。
5. **必须先定义放行面，再定义拦截面。** 先写作用域、工具类型、例外路径，再写危险模式；不要先写一个大正则再补丁式排除误拦截。
6. **必须使用完整 Hook 契约。** 文档和示例都要明确 `matcher`、`hooks`、`type: "command"`、stdin event 结构、通过/拦截时的输出与退出码。

## Hardness and Hook Model

### STOP 的两种实现路径

| 类型 | 实现原语 | 触发方式 | 可绕过？ | 适用场景 |
|------|----------|----------|----------|----------|
| **软 STOP** | Rule（CLAUDE.md / config.yaml） | 模型读取规则后主动判断并暂停 | 是（模型可能忽略） | 需求检查、证据缺失、tasks 未完成 |
| **硬 STOP** | Hook（PreToolUse） | 工具调用前被拦截，返回 DENY | 否（模型无法绕过） | 数据删除、生产部署、危险命令 |

### STOP Boundary

STOP 既是门禁，也是诊断信号；不是所有 STOP 都应该升级为 Hook。

适合升级为 Hook 的规则必须同时满足：
- 高风险或不可逆，例如数据删除、生产部署、大范围覆盖、权限变更
- 可通过工具输入客观检测，例如命令模式、文件路径、目标目录
- 误拦截面可控，能先限定 matcher、cwd、目标路径和例外路径
- 项目不能接受模型偶发忽略规则，需要不可绕过的硬拦截

不适合升级为 Hook 的场景：
- 需求冲突、证据缺失、tasks 设计缺口等需要语义判断的问题
- Apply / Verify 阶段的软 STOP 或 BLOCKED 判断
- 主要依赖人工裁决、上下文解释或流程诊断的规则

这些情况应保留在规则、报告或人工决策层；Hook 只承载可客观检测且高风险的硬门禁。

### Hook 配置位置

| 文件 | 作用域 | 何时使用 |
|------|--------|----------|
| `~/.claude/settings.json` | 全局（所有项目） | 跨项目通用的危险操作（如 `DROP TABLE`） |
| `<project>/.claude/settings.json` | 项目级 | 项目特定的约束（如特定目录保护） |
| `<project>/.claude/settings.local.json` | 项目级（不提交到 git） | 开发者个人的临时 Hook |

**默认策略**：
- 通用危险操作（SQL 删除、rm -rf）→ 全局 settings.json
- 项目特定约束（特定目录、归档门禁）→ 项目 settings.json
- 冲突时优先项目级配置

## Hook Pattern Library

完整的 Hook 拦截脚本示例参见 `references/hook-patterns.md`，包含：
- SQL Guard - 拦截 DROP TABLE、DELETE 无 WHERE、TRUNCATE
- Config Guard - 保护配置文件和目录
- Delete Guard - 拦截危险删除操作
- Archive Guard - OpenSpec 归档门禁
- Deployment Guard - 生产部署保护

### 快速示例：SQL Guard

**来源**：CLAUDE.md Dangerous Operations 中标注 `DROP TABLE` 或 `DELETE` 无 WHERE 条件 → STOP

**Hook 配置**：
```json
{
  "hooks": {
    "PreToolUse": [
      {
        "matcher": "Bash",
        "hooks": [
          { "type": "command", "command": "python3 .claude/hooks/sql-guard.py" }
        ]
      }
    ]
  }
}
```

**拦截脚本核心逻辑** (完整版见 `references/hook-patterns.md`)：
```python
#!/usr/bin/env python3
import sys, json, re

event = json.loads(sys.stdin.read())
if event.get("tool_name") == "Bash":
    command = event.get("tool_input", {}).get("command", "")
    patterns = [r'\bDROP\s+TABLE\b', r'\bDELETE\s+FROM\s+\w+\s*;', r'\bTRUNCATE\s+TABLE\b']
    for pattern in patterns:
        if re.search(pattern, command, re.IGNORECASE):
            print(f"危险 SQL 操作被拦截：{pattern}", file=sys.stderr)
            sys.exit(2)
```

**关键要点**：
- 从 stdin 读取 tool use event JSON
- 工具名在 `tool_name` 字段，参数嵌在 `tool_input` 下（如 `tool_input.command`、`tool_input.file_path`）
- 匹配危险模式后，将原因写入 **stderr** 并 `sys.exit(2)`（exit code 2 阻断工具调用，stderr 作为原因回传给模型）
- 通过检查时静默返回（exit 0，不输出任何内容）
- 必须添加执行权限：`chmod +x .claude/hooks/*.py`

## Workflow

### 1. 确认范围和目标

- 默认配置项目级 Hook（`.claude/settings.json`）
- 如果用户明确要求"全局 Hook"或"所有项目"，配置 `~/.claude/settings.json`
- 多项目仓库时，先确认目标项目目录

### 2. 收集 Hook 来源

**必须读取**：
1. 目标项目的 `CLAUDE.md`（如果存在）
   - 重点读取：Dangerous Operations 章节
   - 识别模式：`→ STOP`、`必须人工确认`、`必须 review`

2. 目标项目的 `openspec/config.yaml`（如果存在）
   - 重点读取：`rules.archive`、`rules.verify` 中需要硬门禁的条目
   - 识别模式：`必须告警并 STOP`、`不得归档`、`CRITICAL`

3. 现有 Hook 配置
   - 全局：`~/.claude/settings.json`
   - 项目：`<project>/.claude/settings.json`
   - 本地：`<project>/.claude/settings.local.json`

### 2.5. 确认 Hook 的作用域边界

**这是关键步骤，必须明确每个 Hook 在哪些上下文中生效**：

#### 识别规则的作用域

分析每条规则的作用域限定：

| 规则表述 | 作用域 | Hook 需要检查 cwd？ | 示例 |
|---------|--------|-------------------|------|
| "不要在本仓库..." | 特定仓库 | ✅ 是 | "不要在 novel-generator 仓库保存 .novel/" |
| "不要在 X 项目中..." | 特定项目 | ✅ 是 | "不要在 backend 项目修改 config/" |
| "不要 DROP TABLE..." | 全局 | ❌ 否 | 数据库删除操作全局禁止 |
| "[仅在 X 仓库] 不要..." | 特定仓库（明确标注） | ✅ 是 | CLAUDE.md 明确标注作用域 |

#### 工作目录检查模板

对于**特定仓库/项目**的规则，Hook 脚本必须先检查工作目录。

**进一步要求**：如果规则只保护某些目录，还要在 cwd 检查之后补一层“目标路径限定”和“例外路径放行”；仅检查仓库名仍然可能过度拦截整个仓库。

```python
def main():
    event = json.loads(sys.stdin.read())
    
    # ✅ 第一步：检查是否在目标仓库/项目
    cwd = os.getcwd()
    is_target_context = os.path.basename(cwd) == "<target-repo>" or \
                        "<target-repo>" in cwd
    
    if not is_target_context:
        # 不在目标上下文，放行所有操作
        return
    
    # 第二步：检查工具类型
    if event.get("tool_name") != "<expected-tool>":
        return
    # ... 后续检测逻辑
```

#### 常见错误

❌ **错误做法**：读到"不要在本仓库保存 X"，直接检测 X 相关操作，不管在哪个目录
```python
# ❌ 错误：缺少工作目录检查
if ".novel/" in file_path:
    sys.exit(2)  # 会在所有目录误拦截
```

✅ **正确做法**：先检查是否在目标仓库，再检测操作
```python
# ✅ 正确：先检查上下文
cwd = os.getcwd()
if "novel-generator" not in cwd:
    return  # 不在目标仓库，放行
    
if ".novel/" in file_path:
    sys.exit(2)  # 只在 novel-generator 仓库拦截
```

#### 混合场景处理

有些规则可能是"在 A 仓库禁止，在 B 目录允许"，或者"只保护仓库中的少数高风险路径"。这种情况必须显式写出白名单/例外路径：

```python
cwd = os.getcwd()

# 只在特定仓库检查
if "generator-repo" not in cwd:
    return

# 仓库内的例外目录
ALLOWED_PATHS = ["fixtures/", "examples/", "tests/"]
if any(prefix in file_path for prefix in ALLOWED_PATHS):
    return  # 例外目录放行

# 其他路径拦截
if dangerous_pattern_match:
    sys.exit(2)
```

### 3. 识别需要 Hook 的场景

按以下优先级分类：

| 优先级 | 场景类型 | 典型关键词 | Hook Pattern |
|--------|---------|-----------|--------------|
| P0 | 数据删除 | DROP TABLE、DELETE、TRUNCATE、rm -rf | sql-guard、delete-guard |
| P0 | 生产部署 | terraform apply、kubectl delete、deploy --prod | deployment-guard |
| P1 | 配置变更 | src/config/、.env.production | config-guard |
| P1 | 归档门禁 | tasks 未完成、verify CRITICAL | archive-guard |
| P2 | 权限变更 | chmod、chown、密码、token | permission-guard |

**识别方法**：
```python
def classify_dangerous_operation(text: str, context: str = None) -> tuple[str, bool]:
    """从 CLAUDE.md Dangerous Operations 提取 Hook 类型和作用域
    
    Returns:
        (hook_type, needs_cwd_check)
        - hook_type: 'sql-guard', 'config-guard', etc.
        - needs_cwd_check: True 表示需要检查工作目录
    """
    text_lower = text.lower()
    
    # 检查是否有作用域限定
    has_scope_limit = any(kw in text_lower for kw in [
        '本仓库', '在本项目', '仅在', 'only in', '在 x 项目'
    ])
    
    if any(kw in text_lower for kw in ['drop table', 'delete', 'truncate']):
        return ('sql-guard', has_scope_limit)
    
    if any(kw in text_lower for kw in ['config/', '.env', '配置']):
        return ('config-guard', has_scope_limit)
    
    if any(kw in text_lower for kw in ['rm -rf', '删除文件', '删除目录']):
        return ('delete-guard', has_scope_limit)
    
    if any(kw in text_lower for kw in ['归档', 'archive', 'tasks 未完成']):
        return ('archive-guard', has_scope_limit)
    
    return ('custom', has_scope_limit)
```

### 4. 生成 Hook 配置

**配置结构**：
```json
{
  "hooks": {
    "PreToolUse": [
      {
        "matcher": "Bash",
        "hooks": [
          {
            "type": "command",
            "command": "python3 .claude/hooks/sql-guard.py"
          }
        ]
      },
      {
        "matcher": "Write|Edit",
        "hooks": [
          {
            "type": "command",
            "command": "python3 .claude/hooks/config-guard.py"
          }
        ]
      }
    ]
  }
}
```

**完整写法契约**：
- `hooks.PreToolUse` 是数组；每个元素对应一组 matcher
- `matcher` 只匹配必要工具，不要为了省事写 `*`
- `hooks` 是数组；每个 hook 条目都必须包含 `type: "command"` 与 `command`
- `command` 应调用可独立测试的脚本；不要把大段 shell 逻辑内联到 settings.json
- Hook 脚本从 **stdin** 读取 event JSON；事件字段使用 `tool_name` 与 `tool_input.*`
- **ALLOW 契约**：exit 0 且无输出
- **DENY 契约**：stderr 输出拦截原因与继续方式，并 `sys.exit(2)`

**生成规则**：
1. 同类型 Hook 合并为一个 matcher（如 `Bash` 相关的 sql-guard、delete-guard、archive-guard 共用一个入口脚本）
2. 不同工具类型（Bash vs Write/Edit）分开配置
3. 已有 Hook 配置时，追加而非覆盖（除非用户明确要求替换）
4. 先按最小拦截面生成；只有在确实需要时才增加 matcher 或扩大模式

**Matcher 选择**：
- Bash 命令拦截 → `"Bash"`
- 文件写入拦截 → `"Write|Edit"`
- 单一文件创建拦截 → 优先 `"Write"`，不要默认连带 `Edit`
- 所有工具 → `"*"`（默认禁用；只有用户明确接受广域拦截且无法缩小工具面时才考虑）

### 5. 生成拦截脚本

**脚本模板**（完整版见 `references/hook-patterns.md`）：

#### 5.1 有作用域限定的脚本模板

对于"不要在本仓库..."、"仅在 X 项目"等有作用域限定的规则：

```python
#!/usr/bin/env python3
import sys
import json
import re
import os

def main():
    # 1. 读取 tool use event
    event = json.loads(sys.stdin.read())

    # 2. 【关键】检查是否在目标仓库/项目（有作用域限定时必须）
    cwd = os.getcwd()
    is_target_context = os.path.basename(cwd) == "<target-repo>" or \
                        "<target-repo>" in cwd
    
    if not is_target_context:
        # 不在目标上下文，放行所有操作
        return

    # 3. 工具类型过滤
    if event.get("tool_name") != "<expected-tool>":
        return  # 不拦截，静默返回

    # 4. 提取关键参数（嵌在 tool_input 下）
    param = event.get("tool_input", {}).get("<param-name>", "")

    # 5. 危险模式检测
    dangerous_patterns = [
        # 正则模式列表
    ]

    for pattern in dangerous_patterns:
        if re.search(pattern, param, re.IGNORECASE):
            # 6. 拦截：原因写 stderr，exit 2 阻断工具调用
            print(
                f"<拦截原因>\n<详细说明>\n<如何继续>",
                file=sys.stderr,
            )
            sys.exit(2)

    # 7. 通过检查，静默返回（不输出任何内容）

if __name__ == "__main__":
    main()
```

#### 5.2 全局规则的脚本模板

对于全局禁止的操作（如 DROP TABLE）：

```python
#!/usr/bin/env python3
import sys
import json
import re
import os

def main():
    # 1. 读取 tool use event
    event = json.loads(sys.stdin.read())

    # 2. 工具类型过滤（全局规则不需要检查 cwd）
    if event.get("tool_name") != "<expected-tool>":
        return  # 不拦截，静默返回

    # 3. 提取关键参数（嵌在 tool_input 下）
    param = event.get("tool_input", {}).get("<param-name>", "")

    # 4. 危险模式检测
    dangerous_patterns = [
        # 正则模式列表
    ]

    for pattern in dangerous_patterns:
        if re.search(pattern, param, re.IGNORECASE):
            # 5. 拦截：原因写 stderr，exit 2 阻断工具调用
            print(
                f"<拦截原因>\n<详细说明>\n<如何继续>",
                file=sys.stderr,
            )
            sys.exit(2)

    # 6. 通过检查，静默返回（不输出任何内容）

if __name__ == "__main__":
    main()
```

**脚本要求**：
- ✅ 必须可独立运行（`python3 script.py`）
- ✅ 必须处理 stdin 输入（tool use event JSON）
- ✅ 必须用 **exit code 2** 阻断危险操作，原因写入 **stderr**（Claude Code 的 PreToolUse 阻断机制）
- ✅ 通过时必须静默（不输出任何内容，exit 0）
- ✅ 必须加执行权限（`chmod +x`）

### 6. 生成测试用例

为每个 Hook 生成对应的测试脚本：

**测试模板** (`.claude/hooks/test-<guard-name>.sh`)：
```bash
#!/bin/bash
# 契约：拦截 = exit 2，放行 = exit 0
GUARD_SCRIPT=".claude/hooks/<guard-name>.py"
fail=0

echo "Testing <guard-name>..."

# Test 1: 应该拦截的情况
echo '{"tool_name": "Bash", "tool_input": {"command": "DROP TABLE users;"}}' | python3 "$GUARD_SCRIPT"
if [ $? -ne 2 ]; then echo "❌ Test 1 failed: should DENY dangerous command"; fail=1; else echo "✅ Test 1 passed"; fi

# Test 2: 应该通过的情况
echo '{"tool_name": "Bash", "tool_input": {"command": "SELECT * FROM users;"}}' | python3 "$GUARD_SCRIPT"
if [ $? -ne 0 ]; then echo "❌ Test 2 failed: should ALLOW safe command"; fail=1; else echo "✅ Test 2 passed"; fi

[ $fail -eq 0 ] && echo "All tests passed!" || { echo "Some tests FAILED"; exit 1; }
```

### 7. 更新配置文件

**更新策略**：
```python
def merge_hook_config(existing: dict, new_hooks: list) -> dict:
    """合并新 Hook 到现有配置"""
    if "hooks" not in existing:
        existing["hooks"] = {}
    
    if "PreToolUse" not in existing["hooks"]:
        existing["hooks"]["PreToolUse"] = []
    
    existing_commands = {h.get("command") for h in existing["hooks"]["PreToolUse"]}
    
    for hook in new_hooks:
        if hook["command"] not in existing_commands:
            existing["hooks"]["PreToolUse"].append(hook)
    
    return existing
```

**写入位置判断**：
- 如果是通用危险操作（sql-guard、delete-guard）且用户未明确指定项目 → 全局 `~/.claude/settings.json`
- 如果是项目特定约束（config-guard、archive-guard）→ 项目 `.claude/settings.json`
- 冲突时询问用户

### 8. 验证配置

**验证清单**：
1. ✅ settings.json 可以被 JSON 解析
2. ✅ Hook 脚本文件存在且可执行
3. ✅ Hook 脚本通过测试用例
4. ✅ 所有 Dangerous Operations 都有对应的 Hook 覆盖

**验证命令**：
```bash
# 1. JSON 语法检查
python3 -m json.tool .claude/settings.json > /dev/null

# 2. 脚本存在性检查
for script in .claude/hooks/*.py; do
  [ -f "$script" ] && [ -x "$script" ] || echo "Missing or not executable: $script"
done

# 3. 运行测试用例
for test in .claude/hooks/test-*.sh; do
  bash "$test" || echo "Test failed: $test"
done
```

## Conflict Arbitration

**Conflict Priority** (highest to lowest):

1. 用户明确指定的 Hook 配置（当前对话）
2. 现有 settings.json 中已配置的 Hook（不要删除用户手动添加的 Hook）
3. CLAUDE.md Dangerous Operations 标注的高风险操作
4. openspec/config.yaml 标注需要硬门禁的规则

**STOP if**:
- settings.json 存在但无法解析（语法错误）
- Hook 脚本路径与现有配置冲突
- 用户要求删除 Hook 但该 Hook 保护关键操作
- 多项目仓库且目标不明确

## Common Mistakes

| Mistake | Fix |
|---------|-----|
| **【最常见】规则有作用域限定但脚本不检查 cwd** | 读到"不要在本仓库..."时，Hook 脚本必须先检查 `os.getcwd()` 是否在目标仓库 |
| **正则过度匹配导致误拦截** | `r"(sqlite3?\|\.db)"` 会拦截文档中提到 "sqlite3" 的内容 → 改用 `r"sqlite3?\s+[^\s]*\.db"` |
| **例外路径列表不完整** | 只写了 `fixtures/` 例外，缺少 `tests/`, `examples/`, `docs/examples/` → 补全常见测试/示例路径 |
| 把所有规则都配置 Hook | 只有 Dangerous Operations 和明确需要硬门禁的规则才配置 Hook |
| Hook 拦截所有工具（matcher: "*"） | 按需选择 Bash / Write / Edit，避免性能影响 |
| Hook 脚本没有执行权限 | 生成后必须 `chmod +x` |
| Hook 脚本不处理 stdin | 必须从 stdin 读取 tool use event JSON |
| 使用错误的字段名 | 工具名是 `tool_name`，参数嵌在 `tool_input` 下（如 `tool_input.command`） |
| 使用错误的 DENY 格式 | 用 `sys.exit(2)` 并把原因写 stderr，不要用 `{"action": "DENY"}` |
| 通过时也返回 JSON | 通过时必须静默（不输出任何内容，exit 0） |
| settings.json 结构缺一层嵌套 | 必须是三层：matcher → hooks 数组 → {type: "command", command: "..."} |
| 覆盖现有 Hook 配置 | 追加而非覆盖，保留用户手动配置的 Hook |
| 不生成测试用例 | 每个 Hook 必须有对应的测试脚本 |
| 拦截原因不清晰 | reason 必须说明：为什么拦截、如何继续 |
| 把软约束也配置为 Hook | 只有不可逆/高风险操作才需要 Hook |
| 不验证配置就写入 | 写入前必须验证 JSON 语法、脚本存在性 |

## Red Flags - 停止并重新评估

出现以下情况时，停止当前操作，回到 Critical Guidelines 重新评估：

- **准备生成 Hook 脚本但没有先确认作用域边界**（规则有"本仓库"/"仅在"等限定词时必须检查 cwd）
- **准备用 Pattern Library 模板但未修改工作目录检查逻辑**（模板可能过时，必须根据规则作用域调整）
- **正则表达式过于宽泛**（如 `r"(sqlite3?|\.db)"` 会误拦截文档）
- 准备把代码风格、命名规范等软约束配置为 Hook
- 准备生成 Hook 脚本但跳过测试用例
- 准备直接覆盖 settings.json 而没有先读取
- 准备让脚本在通过时输出内容
- 准备用 `matcher: "*"` 拦截所有工具
- settings.json 无法解析但仍准备写入

完整的合理化规避表和基线测试场景见 `testing/test-history.md`。

处理已有 Hook 配置的项目：

1. **读取现有配置**
   - 检查全局和项目级 settings.json
   - 识别已配置的 Hook 脚本
   - 保留不是本技能生成的 Hook

2. **分类现有 Hook**
   - 本技能生成的（路径在 `.claude/hooks/`，符合命名规范）
   - 用户手动配置的（其他路径或自定义命名）

3. **更新策略**
   - 本技能生成的 Hook：可以更新或删除
   - 用户手动配置的 Hook：保留，询问是否需要调整
   - 冲突时：询问用户选择保留哪个

## Quality Checklist

完成前检查：

- [ ] 已读取 CLAUDE.md、config.yaml、现有 settings.json
- [ ] 已确认目标配置位置（全局 vs 项目级）
- [ ] **已确认每个规则的作用域边界**（是特定仓库限定还是全局禁止）
- [ ] **有作用域限定的 Hook 脚本包含 cwd 检查**（检查 `os.getcwd()` 是否在目标仓库）
- [ ] **正则表达式足够精确**（不会误拦截文档、注释、路径中的关键词）
- [ ] **例外路径列表完整**（包含 fixtures/, tests/, examples/, docs/examples/ 等）
- [ ] 所有 Dangerous Operations 都有对应的 Hook 覆盖
- [ ] 生成的 Hook 脚本存在于 `.claude/hooks/` 目录
- [ ] Hook 脚本有执行权限（`chmod +x`）
- [ ] Hook 脚本通过测试用例
- [ ] settings.json 可以被 JSON 解析
- [ ] 未覆盖用户手动配置的 Hook
- [ ] 拦截 reason 清晰说明原因和继续方法
- [ ] 只配置了高风险/不可逆操作的 Hook，未把软约束也配置为 Hook

## Output Requirements

向用户汇报：

1. **配置位置**：全局还是项目级 settings.json
2. **新增 Hook**：列出新增的 Hook 及其保护的操作
3. **生成的脚本**：列出生成的 Hook 脚本文件路径
4. **测试结果**：Hook 脚本测试是否通过
5. **覆盖情况**：哪些 Dangerous Operations 已有 Hook 覆盖，哪些未覆盖
6. **保留的 Hook**：现有用户配置的 Hook 是否保留
7. **验证命令**：如何手动测试 Hook（提供测试命令）
8. **后续建议**：如何调整 Hook、如何添加自定义模式

## Integration with Other Skills

**From `harness-claude-setup`**:
- 当生成或更新 CLAUDE.md 的 Dangerous Operations 时，提示用户："这些操作建议配置 Hook 硬门禁，使用 /harness-hook-setup 配置"

**From `harness-openspec-setup`**:
- 当配置 archive 规则时，提示用户："archive 门禁建议配置 Hook 硬拦截，使用 /harness-hook-setup 配置"

**To user**:
- 目标配置位置不明确
- settings.json 语法错误
- Hook 脚本测试失败
- 与现有 Hook 冲突需要决策

## Example Workflow

用户请求："配置 Hook 拦截危险的 SQL 和配置文件修改"

1. **读取来源**
   - 取 `CLAUDE.md` → 发现 "涉及 DROP TABLE → STOP" 和 "涉及修改 src/config/ → STOP"
   - 读取 `.claude/settings.json` → 不存在或为空

2. **生成 Hook 配置**
   ```json
   {
     "hooks": {
       "PreToolUse": [
         {
           "matcher": "Bash",
           "hooks": [
             { "type": "command", "command": "python3 .claude/hooks/sql-guard.py" }
           ]
         },
         {
           "matcher": "Write|Edit",
           "hooks": [
             { "type": "command", "command": "python3 .claude/hooks/config-guard.py" }
           ]
         }
       ]
     }
   }
   ```

3. **生成拦截脚本**
   - `.claude/hooks/sql-guard.py`（使用 Hook Pattern Library 模板）
   - `.claude/hooks/config-guard.py`（使用 Hook Pattern Library 模板）
   - `chmod +x .claude/hooks/*.py`

4. **生成测试用例**
   - `.claude/hooks/test-sql-guard.sh`
   - `.claude/hooks/test-config-guard.sh`

5. **运行测试**
   ```bash
   bash .claude/hooks/test-sql-guard.sh
   bash .claude/hooks/test-config-guard.sh
   ```

6. **写入配置**
   - 写入 `.claude/settings.json`

7. **验证**
   ```bash
   python3 -m json.tool .claude/settings.json
   ```

8. **汇报**
   ```
   ✅ Hook 配置完成
   
   配置位置：.claude/settings.json
   
   新增 Hook：
   - sql-guard: 拦截 DROP TABLE、DELETE 无 WHERE、TRUNCATE
   - config-guard: 保护 src/config/ 目录
   
   生成的脚本：
   - .claude/hooks/sql-guard.py ✅
   - .claude/hooks/config-guard.py ✅
   - .claude/hooks/test-sql-guard.sh ✅
   - .claude/hooks/test-config-guard.sh ✅
   
   测试结果：全部通过 ✅
   
   验证命令：
   bash .claude/hooks/test-sql-guard.sh
   bash .claude/hooks/test-config-guard.sh
   ```

## Resources

- `references/hook-patterns.md` - 完整的 Hook 拦截脚本库（SQL/Config/Delete/Archive/Deployment Guard），含脚本模板和测试模板
- `testing/test-history.md` - TDD 验证记录、基线测试场景、合理化规避表
