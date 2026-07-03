# Hook Pattern Library

本文档提供完整的 Hook 拦截脚本示例，按危险操作类型分类。

所有脚本遵循统一模板：
1. 从 stdin 读取 tool use event JSON
2. **先缩小作用域**：如果规则有作用域限定（"不要在本仓库..."、"仅在 X 项目"），先检查 `os.getcwd()` 是否在目标上下文；如果存在例外目录，先放行例外目录
3. 检查工具类型（字段名 `tool_name`）
4. 提取关键参数（嵌在 `tool_input` 下，如 `tool_input.command`、`tool_input.file_path`）
5. 匹配危险模式
6. 拦截时把原因写入 **stderr**，并 `sys.exit(2)`（exit code 2 是 PreToolUse 阻断工具调用的机制，stderr 作为原因回传模型）
7. 通过时静默返回，`exit 0`（不输出任何内容）

**推荐判定顺序**：`作用域` → `工具类型` → `例外路径/安全场景` → `危险模式`。顺序不要反过来，否则很容易出现过度拦截。

> **关于作用域边界**：
> - **全局规则**（如 `DROP TABLE`）：在任何目录都禁止 → 不需要检查 cwd
> - **特定仓库规则**（如"不要在 novel-generator 仓库保存 .novel/"）：只在目标仓库禁止 → 必须先检查 cwd
> - **常见错误**：读到"不要在本仓库..."就直接检测操作，导致在其他合法目录也被误拦截

> **关于 DENY 机制**：Claude Code 的 PreToolUse Hook 用 **exit code** 决定放行/拦截——`exit 2` 阻断并把 stderr 回传给模型；`exit 0` 放行。本库统一采用 exit 2 + stderr。
> 另一种等价方式是 `exit 0` 且向 stdout 输出 `{"hookSpecificOutput": {"hookEventName": "PreToolUse", "permissionDecision": "deny", "permissionDecisionReason": "..."}}`。
> **注意**：不存在 `{"action": "DENY"}` 这种格式，且「输出 JSON 后 exit 0」若字段写错会被当作放行（fail-open），因此本库优先用 exit 2，语义最不易出错。

## 完整 Hook 配置速查

在 `settings.json` 中，PreToolUse Hook 的最小完整写法如下：

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

字段契约：
- `hooks.PreToolUse`：数组，不是单个对象
- `matcher`：只匹配必要工具，默认不要用 `*`
- `hooks`：数组，支持同一 matcher 下挂多个 command hook
- `type`：固定写 `"command"`
- `command`：调用可独立测试的脚本

脚本契约：
- 输入：从 stdin 读取 event JSON
- 事件字段：`tool_name`、`tool_input.command`、`tool_input.file_path`
- 放行：exit 0，且不输出任何内容
- 拦截：stderr 输出原因与继续方式，并 `sys.exit(2)`

## 1. SQL Guard - 危险 SQL 操作拦截

**保护场景**：
- DROP TABLE
- DELETE 无 WHERE 条件
- TRUNCATE TABLE

**作用域**：全局（在任何目录都禁止）→ **不需要检查 cwd**

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

**拦截脚本** (`.claude/hooks/sql-guard.py`)：
```python
#!/usr/bin/env python3
import sys
import json
import re

def main():
    event = json.loads(sys.stdin.read())

    if event.get("tool_name") != "Bash":
        return

    command = event.get("tool_input", {}).get("command", "").strip()

    # 检测危险 SQL 模式
    dangerous_patterns = [
        r'\bDROP\s+TABLE\b',
        r'\bDELETE\s+FROM\s+\w+\s*;',  # DELETE 无 WHERE
        r'\bTRUNCATE\s+TABLE\b',
    ]

    for pattern in dangerous_patterns:
        if re.search(pattern, command, re.IGNORECASE):
            print(
                f"危险 SQL 操作被拦截：{pattern}\n命令：{command}\n\n"
                "如需执行，请明确添加 WHERE 条件或手动确认。",
                file=sys.stderr,
            )
            sys.exit(2)

if __name__ == "__main__":
    main()
```

**测试脚本** (`.claude/hooks/test-sql-guard.sh`)：
```bash
#!/bin/bash
# 契约：拦截 = exit 2，放行 = exit 0
GUARD_SCRIPT=".claude/hooks/sql-guard.py"
fail=0

echo "Testing sql-guard..."

# Test 1: 应该拦截 DROP TABLE
echo '{"tool_name": "Bash", "tool_input": {"command": "DROP TABLE users;"}}' | python3 "$GUARD_SCRIPT"
if [ $? -ne 2 ]; then echo "❌ Test 1 failed: should DENY DROP TABLE"; fail=1; else echo "✅ Test 1 passed"; fi

# Test 2: 应该拦截 DELETE 无 WHERE
echo '{"tool_name": "Bash", "tool_input": {"command": "DELETE FROM users;"}}' | python3 "$GUARD_SCRIPT"
if [ $? -ne 2 ]; then echo "❌ Test 2 failed: should DENY DELETE without WHERE"; fail=1; else echo "✅ Test 2 passed"; fi

# Test 3: 应该通过 SELECT
echo '{"tool_name": "Bash", "tool_input": {"command": "SELECT * FROM users;"}}' | python3 "$GUARD_SCRIPT"
if [ $? -ne 0 ]; then echo "❌ Test 3 failed: should ALLOW SELECT"; fail=1; else echo "✅ Test 3 passed"; fi

# Test 4: 应该通过 DELETE 有 WHERE
echo '{"tool_name": "Bash", "tool_input": {"command": "DELETE FROM users WHERE id = 1;"}}' | python3 "$GUARD_SCRIPT"
if [ $? -ne 0 ]; then echo "❌ Test 4 failed: should ALLOW DELETE with WHERE"; fail=1; else echo "✅ Test 4 passed"; fi

[ $fail -eq 0 ] && echo "All tests passed!" || { echo "Some tests FAILED"; exit 1; }
```

## 2. Config Guard - 配置文件保护

**保护场景**：
- src/config/ 目录
- config/production/ 目录
- .env.production 文件

**作用域**：特定项目（假设只在 backend 项目保护）→ **需要检查 cwd**

**Hook 配置**：
```json
{
  "hooks": {
    "PreToolUse": [
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

**拦截脚本** (`.claude/hooks/config-guard.py`)：
```python
#!/usr/bin/env python3
import sys
import json
import os

def main():
    event = json.loads(sys.stdin.read())

    # ✅ 第一步：检查是否在目标项目（假设是 backend 项目）
    # 如果规则是全局的，删除这个检查
    cwd = os.getcwd()
    is_target_project = os.path.basename(cwd) == "backend" or "backend" in cwd
    
    if not is_target_project:
        # 不在目标项目，放行
        return

    # 第二步：工具类型过滤
    if event.get("tool_name") not in ("Write", "Edit"):
        return

    # 第三步：提取文件路径
    file_path = event.get("tool_input", {}).get("file_path", "")

    # 检测受保护的配置目录
    protected_paths = [
        "src/config/",
        "config/production/",
        ".env.production",
    ]

    for protected in protected_paths:
        if protected in file_path:
            print(
                f"配置文件修改被拦截：{file_path}\n\n受保护路径：{protected}\n"
                "请确认该修改已经过 review，或在确认安全后手动执行。",
                file=sys.stderr,
            )
            sys.exit(2)

if __name__ == "__main__":
    main()
```

**测试脚本** (`.claude/hooks/test-config-guard.sh`)：
```bash
#!/bin/bash
# 契约：拦截 = exit 2，放行 = exit 0
GUARD_SCRIPT=".claude/hooks/config-guard.py"
fail=0

echo "Testing config-guard..."

# Test 1: 应该拦截 src/config/ 修改
echo '{"tool_name": "Write", "tool_input": {"file_path": "src/config/database.js"}}' | python3 "$GUARD_SCRIPT"
if [ $? -ne 2 ]; then echo "❌ Test 1 failed: should DENY config file write"; fail=1; else echo "✅ Test 1 passed"; fi

# Test 2: 应该拦截 .env.production 修改
echo '{"tool_name": "Edit", "tool_input": {"file_path": ".env.production"}}' | python3 "$GUARD_SCRIPT"
if [ $? -ne 2 ]; then echo "❌ Test 2 failed: should DENY .env.production edit"; fail=1; else echo "✅ Test 2 passed"; fi

# Test 3: 应该通过普通文件修改
echo '{"tool_name": "Write", "tool_input": {"file_path": "src/components/Button.tsx"}}' | python3 "$GUARD_SCRIPT"
if [ $? -ne 0 ]; then echo "❌ Test 3 failed: should ALLOW normal file write"; fail=1; else echo "✅ Test 3 passed"; fi

[ $fail -eq 0 ] && echo "All tests passed!" || { echo "Some tests FAILED"; exit 1; }
```

## 3. Delete Guard - 危险删除操作拦截

**保护场景**：
- rm -rf / 开头的路径
- rm -rf *
- rm -rf . 或 ..
- find with -delete

**Hook 配置**：
```json
{
  "hooks": {
    "PreToolUse": [
      {
        "matcher": "Bash",
        "hooks": [
          { "type": "command", "command": "python3 .claude/hooks/delete-guard.py" }
        ]
      }
    ]
  }
}
```

**拦截脚本** (`.claude/hooks/delete-guard.py`)：
```python
#!/usr/bin/env python3
import sys
import json
import re

def main():
    event = json.loads(sys.stdin.read())

    if event.get("tool_name") != "Bash":
        return

    command = event.get("tool_input", {}).get("command", "").strip()

    # 检测危险删除模式
    dangerous_patterns = [
        r'\brm\s+-rf?\s+/',           # rm -rf / 开头的路径
        r'\brm\s+-rf?\s+\*',          # rm -rf *
        r'\brm\s+-rf?\s+\.',          # rm -rf . 或 ..
        r'\bfind\s+.*-delete',        # find with -delete
    ]

    for pattern in dangerous_patterns:
        if re.search(pattern, command):
            print(
                f"危险删除操作被拦截：{pattern}\n命令：{command}\n\n"
                "如需执行，请明确指定要删除的文件或目录路径。",
                file=sys.stderr,
            )
            sys.exit(2)

if __name__ == "__main__":
    main()
```

**测试脚本** (`.claude/hooks/test-delete-guard.sh`)：
```bash
#!/bin/bash
# 契约：拦截 = exit 2，放行 = exit 0
GUARD_SCRIPT=".claude/hooks/delete-guard.py"
fail=0

echo "Testing delete-guard..."

# Test 1: 应该拦截 rm -rf <path>
echo '{"tool_name": "Bash", "tool_input": {"command": "rm -rf /tmp/dangerous"}}' | python3 "$GUARD_SCRIPT"
if [ $? -ne 2 ]; then echo "❌ Test 1 failed: should DENY rm -rf /"; fail=1; else echo "✅ Test 1 passed"; fi

# Test 2: 应该拦截 rm -rf *
echo '{"tool_name": "Bash", "tool_input": {"command": "rm -rf *"}}' | python3 "$GUARD_SCRIPT"
if [ $? -ne 2 ]; then echo "❌ Test 2 failed: should DENY rm -rf *"; fail=1; else echo "✅ Test 2 passed"; fi

# Test 3: 应该通过安全删除
echo '{"tool_name": "Bash", "tool_input": {"command": "rm temp.txt"}}' | python3 "$GUARD_SCRIPT"
if [ $? -ne 0 ]; then echo "❌ Test 3 failed: should ALLOW safe delete"; fail=1; else echo "✅ Test 3 passed"; fi

[ $fail -eq 0 ] && echo "All tests passed!" || { echo "Some tests FAILED"; exit 1; }
```

## 4. Archive Guard - OpenSpec 归档门禁

**保护场景**：
- tasks.md 存在未完成项
- verify.md 存在 CRITICAL 项或失败标记

**作用域**：特定仓库（假设只在使用 OpenSpec 的项目）→ **需要检查 cwd**

**Hook 配置**：
```json
{
  "hooks": {
    "PreToolUse": [
      {
        "matcher": "Bash",
        "hooks": [
          { "type": "command", "command": "python3 .claude/hooks/archive-guard.py" }
        ]
      }
    ]
  }
}
```

**拦截脚本** (`.claude/hooks/archive-guard.py`)：
```python
#!/usr/bin/env python3
import sys
import json
import re
import os

def main():
    event = json.loads(sys.stdin.read())

    # ✅ 第一步：检查是否在使用 OpenSpec 的项目
    # 简单判断：检查是否有 openspec/ 目录
    if not os.path.exists("openspec/config.yaml"):
        # 不是 OpenSpec 项目，放行
        return

    # 第二步：工具类型过滤
    if event.get("tool_name") != "Bash":
        return

    # 第三步：提取命令
    command = event.get("tool_input", {}).get("command", "").strip()

    # 检测 openspec archive 命令
    if not re.search(r'\bopenspec\s+archive\b', command):
        return

    # 提取 Change ID
    match = re.search(r'openspec\s+archive\s+(\S+)', command)
    if not match:
        return

    change_id = match.group(1)
    change_dir = f"openspec/changes/{change_id}"

    if not os.path.exists(change_dir):
        print(f"Change 目录不存在：{change_dir}", file=sys.stderr)
        sys.exit(2)

    # 检查 tasks.md 完成度
    tasks_file = f"{change_dir}/tasks.md"
    if os.path.exists(tasks_file):
        with open(tasks_file, 'r', encoding='utf-8') as f:
            content = f.read()
        unchecked = re.findall(r'- \[ \]', content)
        if unchecked:
            print(
                f"tasks.md 存在未完成项（{len(unchecked)} 个）\n\n"
                "归档前必须完成所有 checkbox 或标注 blocker。",
                file=sys.stderr,
            )
            sys.exit(2)

    # 检查 verify.md CRITICAL 项
    verify_file = f"{change_dir}/verify.md"
    if os.path.exists(verify_file):
        with open(verify_file, 'r', encoding='utf-8') as f:
            content = f.read()
        if '[CRITICAL]' in content or '❌' in content:
            print(
                "verify.md 存在 CRITICAL 项或失败标记\n\n"
                "归档前必须解决所有 CRITICAL 问题。",
                file=sys.stderr,
            )
            sys.exit(2)

if __name__ == "__main__":
    main()
```

**测试脚本** (`.claude/hooks/test-archive-guard.sh`)：
```bash
#!/bin/bash
# 契约：拦截 = exit 2，放行 = exit 0
GUARD_SCRIPT=".claude/hooks/archive-guard.py"
fail=0

echo "Testing archive-guard..."

mkdir -p openspec/changes/test-change

# Test 1: 应该拦截未完成的 tasks.md
echo "- [ ] Incomplete task" > openspec/changes/test-change/tasks.md
echo '{"tool_name": "Bash", "tool_input": {"command": "openspec archive test-change"}}' | python3 "$GUARD_SCRIPT"
if [ $? -ne 2 ]; then echo "❌ Test 1 failed: should DENY archive with incomplete tasks"; fail=1; else echo "✅ Test 1 passed"; fi

# Test 2: 应该通过完成的 tasks.md
echo "- [x] Complete task" > openspec/changes/test-change/tasks.md
echo '{"tool_name": "Bash", "tool_input": {"command": "openspec archive test-change"}}' | python3 "$GUARD_SCRIPT"
if [ $? -ne 0 ]; then echo "❌ Test 2 failed: should ALLOW archive with complete tasks"; fail=1; else echo "✅ Test 2 passed"; fi

# Test 3: verify.md 有 CRITICAL 应拦截
echo '[CRITICAL] something broke' > openspec/changes/test-change/verify.md
echo '{"tool_name": "Bash", "tool_input": {"command": "openspec archive test-change"}}' | python3 "$GUARD_SCRIPT"
if [ $? -ne 2 ]; then echo "❌ Test 3 failed: should DENY archive with CRITICAL"; fail=1; else echo "✅ Test 3 passed"; fi

rm -rf openspec/changes/test-change
[ $fail -eq 0 ] && echo "All tests passed!" || { echo "Some tests FAILED"; exit 1; }
```

## 5. Deployment Guard - 生产部署保护

**保护场景**：
- terraform apply
- kubectl delete
- deploy --prod
- 其他生产环境操作

**Hook 配置**：
```json
{
  "hooks": {
    "PreToolUse": [
      {
        "matcher": "Bash",
        "hooks": [
          { "type": "command", "command": "python3 .claude/hooks/deployment-guard.py" }
        ]
      }
    ]
  }
}
```

**拦截脚本** (`.claude/hooks/deployment-guard.py`)：
```python
#!/usr/bin/env python3
import sys
import json
import re

def main():
    event = json.loads(sys.stdin.read())

    if event.get("tool_name") != "Bash":
        return

    command = event.get("tool_input", {}).get("command", "").strip()

    # 检测生产部署模式
    dangerous_patterns = [
        (r'\bterraform\s+apply\b', "Terraform apply"),
        (r'\bkubectl\s+delete\b', "Kubernetes delete"),
        (r'\bdeploy\s+.*--prod\b', "Production deployment"),
        (r'\baws\s+.*--profile\s+prod', "AWS production operation"),
    ]

    for pattern, description in dangerous_patterns:
        if re.search(pattern, command, re.IGNORECASE):
            print(
                f"生产环境操作被拦截：{description}\n命令：{command}\n\n"
                "生产环境变更必须经过 review 和人工确认。",
                file=sys.stderr,
            )
            sys.exit(2)

if __name__ == "__main__":
    main()
```

**测试脚本** (`.claude/hooks/test-deployment-guard.sh`)：
```bash
#!/bin/bash
# 契约：拦截 = exit 2，放行 = exit 0
GUARD_SCRIPT=".claude/hooks/deployment-guard.py"
fail=0

echo "Testing deployment-guard..."

# Test 1: 应该拦截 terraform apply
echo '{"tool_name": "Bash", "tool_input": {"command": "terraform apply"}}' | python3 "$GUARD_SCRIPT"
if [ $? -ne 2 ]; then echo "❌ Test 1 failed: should DENY terraform apply"; fail=1; else echo "✅ Test 1 passed"; fi

# Test 2: 应该拦截 kubectl delete
echo '{"tool_name": "Bash", "tool_input": {"command": "kubectl delete pod foo"}}' | python3 "$GUARD_SCRIPT"
if [ $? -ne 2 ]; then echo "❌ Test 2 failed: should DENY kubectl delete"; fail=1; else echo "✅ Test 2 passed"; fi

# Test 3: 应该通过 terraform plan
echo '{"tool_name": "Bash", "tool_input": {"command": "terraform plan"}}' | python3 "$GUARD_SCRIPT"
if [ $? -ne 0 ]; then echo "❌ Test 3 failed: should ALLOW terraform plan"; fail=1; else echo "✅ Test 3 passed"; fi

[ $fail -eq 0 ] && echo "All tests passed!" || { echo "Some tests FAILED"; exit 1; }
```

## 脚本模板

所有拦截脚本遵循统一模板。根据规则作用域选择对应模板：

### 模板 A：有作用域限定的规则

适用于："不要在本仓库..."、"仅在 X 项目"、"不要在 backend 保存..." 等有上下文限定的规则。

```python
#!/usr/bin/env python3
import sys
import json
import re
import os

def main():
    # 1. 读取 tool use event
    event = json.loads(sys.stdin.read())

    # 2. 【关键】检查是否在目标仓库/项目
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

    # 7. 通过检查，静默返回（exit 0，不输出任何内容）

if __name__ == "__main__":
    main()
```

### 模板 B：全局规则

适用于：在任何目录都禁止的操作（如 `DROP TABLE`、`rm -rf /`）。

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

    # 6. 通过检查，静默返回（exit 0，不输出任何内容）

if __name__ == "__main__":
    main()
```

## 测试脚本模板

所有测试脚本遵循统一模板：

```bash
#!/bin/bash
# 契约：拦截 = exit 2，放行 = exit 0
GUARD_SCRIPT=".claude/hooks/<guard-name>.py"
fail=0

echo "Testing <guard-name>..."

# Test 1: 应该拦截的情况
echo '{"tool_name": "<tool>", "tool_input": {"<param>": "<dangerous-value>"}}' | python3 "$GUARD_SCRIPT"
if [ $? -ne 2 ]; then echo "❌ Test 1 failed: should DENY dangerous operation"; fail=1; else echo "✅ Test 1 passed"; fi

# Test 2: 应该通过的情况
echo '{"tool_name": "<tool>", "tool_input": {"<param>": "<safe-value>"}}' | python3 "$GUARD_SCRIPT"
if [ $? -ne 0 ]; then echo "❌ Test 2 failed: should ALLOW safe operation"; fail=1; else echo "✅ Test 2 passed"; fi

[ $fail -eq 0 ] && echo "All tests passed!" || { echo "Some tests FAILED"; exit 1; }
```

## 使用指南

### 1. 选择合适的模板

根据规则表述判断作用域：

| 规则表述 | 使用模板 | 示例 |
|---------|---------|------|
| "不要在本仓库..." | 模板 A（有作用域） | "不要在 novel-generator 仓库保存 .novel/" |
| "不要在 X 项目中..." | 模板 A（有作用域） | "不要在 backend 项目修改 config/" |
| "[仅在 X] 不要..." | 模板 A（有作用域） | "[仅在主仓库] 不要直接访问数据库" |
| "不要 DROP TABLE..." | 模板 B（全局） | 数据库删除操作全局禁止 |
| "不要 rm -rf /..." | 模板 B（全局） | 危险删除操作全局禁止 |

**判断原则**：
- 如果规则在**某些目录允许，某些目录禁止** → 模板 A
- 如果规则在**任何目录都禁止** → 模板 B

### 2. 自定义保护路径或模式

修改脚本中的 `protected_paths` 或 `dangerous_patterns` 列表：

```python
# 示例：扩展受保护的配置路径
protected_paths = [
    "src/config/",
    "config/production/",
    ".env.production",
    "infrastructure/",      # 新增
    "deploy/secrets.yaml",  # 新增
]
```

### 3. 处理例外路径

对于有作用域限定的 Hook，可能需要仓库内的例外：

```python
cwd = os.getcwd()
if "target-repo" not in cwd:
    return

# 仓库内的例外目录
ALLOWED_PATHS = ["fixtures/", "examples/", "tests/", "docs/examples/"]
if any(prefix in file_path for prefix in ALLOWED_PATHS):
    return  # 例外目录放行

# 其他路径检测
if dangerous_pattern_match:
    sys.exit(2)
```

### 4. 正则表达式最佳实践

❌ **过度匹配的正则**：
```python
# 错误：会拦截文档中提到 "sqlite3" 的内容
if re.search(r"(sqlite3?|\.db)", command):
```

✅ **精确匹配的正则**：
```python
# 正确：只检测真正的数据库文件操作
if re.search(r"sqlite3?\s+[^\s]*\.db", command) or \
   re.search(r"(touch|echo|>|>>)\s+[^\s]*\.db(\s|$)", command):
```

### 5. 测试验证

生成后立即运行测试脚本确保正确性：

```bash
# 运行单个测试
bash .claude/hooks/test-<guard-name>.sh

# 运行所有测试
for test in .claude/hooks/test-*.sh; do
  bash "$test" || echo "Test failed: $test"
done
```

### 6. 添加执行权限

生成脚本后必须添加执行权限：

```bash
chmod +x .claude/hooks/*.py .claude/hooks/test-*.sh
```

### 7. 合并配置

多个 Hook 可以合并到同一个脚本中，减少 matcher 数量：

```python
# 合并示例：sql-guard + delete-guard
def main():
    event = json.loads(sys.stdin.read())
    
    if event.get("tool_name") != "Bash":
        return
    
    command = event.get("tool_input", {}).get("command", "")
    
    # SQL 危险模式
    sql_patterns = [r'\bDROP\s+TABLE\b', r'\bDELETE\s+FROM\s+\w+\s*;']
    for pattern in sql_patterns:
        if re.search(pattern, command, re.IGNORECASE):
            print("危险 SQL 操作被拦截", file=sys.stderr)
            sys.exit(2)
    
    # 删除危险模式
    delete_patterns = [r'\brm\s+-rf?\s+/', r'\brm\s+-rf?\s+\*']
    for pattern in delete_patterns:
        if re.search(pattern, command):
            print("危险删除操作被拦截", file=sys.stderr)
            sys.exit(2)
```
