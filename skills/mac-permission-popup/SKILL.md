---
name: mac-permission-popup
description: 在 macOS 上部署 Claude Code 权限拦截弹窗。当 Claude 请求授权（如 rm、sudo 等危险操作）且用户不在终端前台时，弹出 macOS 原生对话框显示请求详情，用户点选 拒绝/允许一次/会话允许 后决定自动联动回 Claude Code。支持 setup / status / uninstall。
---

# macOS 权限弹窗 — Claude Code PermissionRequest Hook

**作用**：拦截 Claude Code 的 `PermissionRequest` 事件。当用户不在终端前台时，用
`osascript` 弹出 macOS 原生对话框展示权限请求；用户点选后，决策通过 hook 输出
JSON 自动联动回 Claude Code 的权限系统。

**平台**：仅 macOS（依赖 `osascript`）。
**输入参数**：无。部署/卸载/检查分别对应下面三种操作模式。

| 项目 | 值 |
|------|-----|
| 脚本路径 | `~/.claude/hooks/permission-alert.sh` |
| Hook 事件 | `PermissionRequest` |
| 配置文件 | `~/.claude/settings.json` |
| 弹窗超时 | 60 秒（超时=不输出 JSON，回落内置提示流程） |

---

## 工作原理

```
Claude 请求权限（如 Bash rm）
        │
        ▼
PermissionRequest hook 触发 → permission-alert.sh
        │
        ├─ 前台是终端 App → 直接 exit 0，走 Claude Code 内置提示
        │
        └─ 前台非终端    → osascript 弹原生对话框
                              ├─ 拒绝     → deny（拒绝并回传原因）
                              ├─ 允许一次  → allow
                              └─ 会话允许  → alwaysAllow
```

---

## 操作模式

### mode = setup（默认）— 部署

#### Step 1: 前置检查

```bash
[ "$(uname)" = "Darwin" ] || { echo "仅支持 macOS"; exit 1; }
osascript -e 'display dialog "测试" buttons {"OK"} default button "OK" giving up after 2' >/dev/null 2>&1 \
  && echo "osascript OK" || echo "osascript 不可用（检查终端是否被授予自动化权限）"
```

#### Step 2: 写入脚本

将下方[脚本内容](#脚本内容)写入 `~/.claude/hooks/permission-alert.sh`，然后 `chmod +x`：

```bash
mkdir -p ~/.claude/hooks
# （用文件写入工具把脚本内容写到 ~/.claude/hooks/permission-alert.sh）
chmod +x ~/.claude/hooks/permission-alert.sh
```

> 若脚本已存在：默认**不覆盖**；如需覆盖，先备份为 `.bak`，避免丢失用户自定义。

#### Step 3: 合并 hook 配置

读取 `~/.claude/settings.json`，在 `hooks.PermissionRequest` 下**追加**（不覆盖已有条目）：

```json
{
  "hooks": {
    "PermissionRequest": [
      {
        "matcher": "*",
        "hooks": [
          { "type": "command", "command": "bash ~/.claude/hooks/permission-alert.sh" }
        ]
      }
    ]
  }
}
```

**必须用 Python 读写 JSON，不要手写文本**（settings.json 可能已有内容，且注释/转义易错）：

```python
import json, os
p = os.path.expanduser("~/.claude/settings.json")
try:
    with open(p) as f: s = json.load(f)
except (FileNotFoundError, json.JSONDecodeError):
    s = {}
hooks = s.setdefault("hooks", {})
entries = hooks.setdefault("PermissionRequest", [])
cmd = "bash ~/.claude/hooks/permission-alert.sh"
if not any(h.get("command") == cmd
           for e in entries for h in e.get("hooks", [])):
    entries.append({"matcher": "*",
                    "hooks": [{"type": "command", "command": cmd}]})
    os.makedirs(os.path.dirname(p), exist_ok=True)
    with open(p, "w") as f: json.dump(s, f, indent=2, ensure_ascii=False)
    print("已写入 hook 配置")
else:
    print("hook 已存在，跳过")
```

> 首次注册 hook 时，Claude Code 可能弹出**终端授权提示**，需手动允许一次。

#### Step 4: 验证

```bash
ls -l ~/.claude/hooks/permission-alert.sh
python3 - <<'EOF'
import json, os
s = json.load(open(os.path.expanduser("~/.claude/settings.json")))
pr = s.get("hooks", {}).get("PermissionRequest", [])
print("PermissionRequest hook:", "已配置" if pr else "未配置")
for e in pr:
    for h in e.get("hooks", []):
        print("  command:", h.get("command", "?"))
EOF
```

---

### mode = status — 检查

```bash
ls -l ~/.claude/hooks/permission-alert.sh 2>/dev/null || echo "脚本未部署"
python3 -c "import json,os; s=json.load(open(os.path.expanduser('~/.claude/settings.json'))); print('hook 已配置' if s.get('hooks',{}).get('PermissionRequest') else 'hook 未配置')"
```

---

### mode = uninstall — 卸载

1. 用 Python 从 `settings.json` 中移除 `hooks.PermissionRequest` 里 command 匹配 `permission-alert.sh` 的条目（若数组清空则删除该键）。
2. 询问用户是否删除脚本文件 `~/.claude/hooks/permission-alert.sh`。

---

## 脚本内容

完整写入 `~/.claude/hooks/permission-alert.sh`：

```bash
#!/bin/bash
# PermissionRequest hook — 用户不在终端前台时，弹出交互式 macOS 授权对话框
# 决策直接联动回 Claude Code 权限系统
set -uo pipefail

request=$(cat)

# 解析字段：写入临时 JSON 再逐个取值，避免命令含空格被 read 拆错
tmp=$(mktemp)
printf '%s' "$request" | python3 -c '
import sys, json
try:
    d = json.load(sys.stdin)
except Exception:
    d = {}
tn = (d.get("tool_name") or "未知工具").replace("\n", " ")
ti = d.get("tool_input") or {}
if isinstance(ti, dict):
    v = ti.get("command") or ti.get("file_path") or ti.get("description") or json.dumps(ti, ensure_ascii=False)
else:
    v = str(ti)
msg = (d.get("reason") or d.get("message") or "").replace("\n", " ")
json.dump({"tool_name": tn, "tool_input": str(v), "message": msg},
          open(sys.argv[1], "w"), ensure_ascii=False)
' "$tmp" 2>/dev/null

tool_name=$(python3 -c 'import json,sys;print(json.load(open(sys.argv[1]))["tool_name"])' "$tmp" 2>/dev/null || echo "未知工具")
tool_input=$(python3 -c 'import json,sys;print(json.load(open(sys.argv[1]))["tool_input"])' "$tmp" 2>/dev/null || echo "")
message=$(python3 -c 'import json,sys;print(json.load(open(sys.argv[1]))["message"])' "$tmp" 2>/dev/null || echo "")
rm -f "$tmp"

# 检测前台 App（osascript 优先，回退 AppKit — 无需辅助功能权限）
front_app=$(osascript -e 'tell application "System Events" to get name of first application process whose frontmost is true' 2>/dev/null || true)
if [[ -z "$front_app" ]]; then
  front_app=$(python3 -c 'from AppKit import NSWorkspace; print(NSWorkspace.sharedWorkspace().frontmostApplication().localizedName())' 2>/dev/null || true)
fi

# 前台是终端应用时不弹窗，走内置权限流程
terminal_apps="Terminal iTerm2 Warp kitty Alacritty WezTerm Ghostty Hyper Tabby"
for app in $terminal_apps; do
  [[ "$front_app" == "$app" ]] && exit 0
done

# 组装对话框文案
if [[ -n "$message" ]]; then
  dialog_text="$message"
elif [[ -n "$tool_input" ]]; then
  dialog_text="Claude Code 请求执行 [$tool_name]: $tool_input"
else
  dialog_text="Claude Code 请求使用工具: $tool_name"
fi

# 转义双引号和反斜杠，防止 AppleScript 注入/语法错误
esc() { printf '%s' "$1" | sed 's/\\/\\\\/g; s/"/\\"/g'; }
safe_text=$(esc "$dialog_text")
safe_title=$(esc "Claude Code 授权请求")

result=$(osascript \
  -e "display dialog \"$safe_text\" with title \"$safe_title\" buttons {\"拒绝\", \"允许一次\", \"会话允许\"} default button \"允许一次\" cancel button \"拒绝\" with icon caution giving up after 60" 2>&1)

# 超时检测：AppleScript 超时会返回 "... gave up:true"，此时不应视为用户选择
if [[ "$result" == *"gave up:true"* ]]; then
  exit 0   # 超时 → 走正常权限提示流程
fi

# 顺序重要：「会话允许」含「允许」子串，须先判断更具体的
if [[ "$result" == *"会话允许"* ]]; then
  behavior="alwaysAllow"
elif [[ "$result" == *"允许一次"* ]]; then
  behavior="allow"
elif [[ "$result" == *"拒绝"* ]]; then
  behavior="deny"
else
  exit 0   # 超时/关闭 → 不输出 JSON，走正常流程
fi

python3 -c '
import json, sys
b = sys.argv[1]
out = {"hookSpecificOutput": {"hookEventName": "PermissionRequest",
       "decision": {"behavior": b}}}
if b == "deny":
    out["hookSpecificOutput"]["decision"]["message"] = "用户通过对话框拒绝了此操作"
print(json.dumps(out, ensure_ascii=False))
' "$behavior"
```

---

## 自定义

- **增加终端 App**：修改脚本中 `terminal_apps` 变量。
- **修改超时**：改 `giving up after 60` 的秒数。
- **修改按钮**：改 `buttons {...}`，**同时**更新下方对应的 `if/elif` 判断分支。

---

## 故障排查

| 现象 | 原因 | 解决 |
|------|------|------|
| 弹窗不出现 | 用户正在终端前台 | 切到浏览器/编辑器等非终端 App 再触发 |
| 弹窗中文乱码 | dialog 文本未转义 | 脚本已内置 `esc` 转义；检查是否有其他未转义字符 |
| 点击无效 | 终端缺自动化权限 | 系统设置 → 隐私与安全性 → 自动化/辅助功能，勾选终端 |
| hook 不生效 | settings.json JSON 语法错误 | `python3 -c "import json;json.load(open('$HOME/.claude/settings.json'))"` |
| 脚本不执行 | 无执行权限 | `chmod +x ~/.claude/hooks/permission-alert.sh` |
| hook 事件名报错 | 使用了旧事件名 | 确认事件为 `PermissionRequest`（非 `PreToolUse`） |

## Verification Checklist

- [ ] `uname` 为 `Darwin`（仅 macOS）
- [ ] 脚本已写入且可执行
- [ ] `settings.json` 中 `hooks.PermissionRequest` 含 `permission-alert.sh` 命令
- [ ] `settings.json` 是合法 JSON
- [ ] 非终端前台触发时能看到中文对话框
