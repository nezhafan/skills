---
name: mac-permission-popup
description: 设置 macOS 原生弹窗拦截 Claude Code 权限请求。当 Claude 需要授权时（如 rm、sudo 等危险操作），弹出 macOS 对话框显示请求详情和选项，用户点击后决定自动联动到 Claude Code 继续执行。
---

# Skill: mac-permission-popup
**Description:** 部署 macOS 原生授权弹窗系统。拦截 Claude Code 的 PermissionRequest 事件，通过 osascript 弹出交互对话框（拒绝/允许一次/会话允许），用户点击后 decision 自动联动到 Claude Code 权限系统。

**Input Parameters:**
- `action` (string, optional, default: `"setup"`): 操作模式。`setup` = 完整部署, `status` = 检查当前状态, `uninstall` = 移除弹窗系统
- `terminal_apps` (string, optional): 额外需要识别为终端应用的 app 名称列表（逗号分隔），在这些 app 前台时不弹窗，走内置权限流程

**Outputs:**
- `status` (object): 部署状态
  - `script_deployed` (boolean): 脚本是否已部署
  - `hook_configured` (boolean): hook 是否已配置
  - `script_path` (string): 脚本路径
- `guide` (string): 后续操作指引

---

## 平台说明

本 skill 仅支持 macOS。利用 `osascript display dialog` 实现原生弹窗。

| 项目 | 值 |
|------|-----|
| 脚本路径 | `~/.claude/hooks/permission-alert.sh` |
| Hook 事件 | `PermissionRequest` |
| 配置文件 | `~/.claude/settings.json` |
| 弹窗超时 | 60 秒（超时后走内置权限提示流程） |

---

## 工作原理

```
Claude 需要权限 (如 Bash rm)
       │
       ▼
PermissionRequest hook 触发
       │
       ▼
permission-alert.sh 执行
       │
       ├─ 用户在终端前台 → 退出码 0，走 Claude Code 内置权限提示
       │
       └─ 用户不在终端   → osascript 弹出原生对话框
                              │
                              ├─ 拒绝     → deny
                              ├─ 允许一次  → allow
                              └─ 会话允许  → alwaysAllow
```

---

## 部署步骤

### Step 1: 检查前置条件

确认 macOS 支持 osascript：

```bash
osascript -e 'display dialog "测试" buttons {"OK"} default button "OK" giving up after 3' 2>/dev/null && echo "osascript OK" || echo "osascript 不可用"
```

### Step 2: 部署脚本

创建 `~/.claude/hooks/permission-alert.sh`（若已存在则跳过，可用 `action=force` 覆盖）。

```bash
mkdir -p ~/.claude/hooks
chmod +x ~/.claude/hooks/permission-alert.sh
```

### Step 3: 配置 Hook

在 `~/.claude/settings.json` 中添加 `hooks.PermissionRequest` 配置。

读取现有 settings.json，在 `"hooks"` 键下添加/合并：

```json
"hooks": {
  "PermissionRequest": [
    {
      "matcher": "",
      "hooks": [
        {
          "type": "command",
          "command": "bash ~/.claude/hooks/permission-alert.sh"
        }
      ]
    }
  ]
}
```

若 settings.json 中已有 `hooks.PermissionRequest` 且包含该命令，则跳过。

### Step 4: 验证部署

```bash
# 检查脚本存在且可执行
ls -la ~/.claude/hooks/permission-alert.sh

# 检查 hook 配置
python3 -c "
import json
with open('$HOME/.claude/settings.json') as f:
    s = json.load(f)
hooks = s.get('hooks', {})
pr = hooks.get('PermissionRequest', [])
print('PermissionRequest hook:', '已配置' if pr else '未配置')
for entry in pr:
    for h in entry.get('hooks', []):
        print(f'  command: {h.get(\"command\", \"?\")}')
"
```

---

## 移除

设置 `action=uninstall` 时：
1. 从 settings.json 中移除 `hooks.PermissionRequest` 配置
2. 可选择保留或删除 `~/.claude/hooks/permission-alert.sh`

---

## 脚本内容

以下是 `permission-alert.sh` 完整内容（部署时写入 `~/.claude/hooks/permission-alert.sh`）：

```bash
#!/bin/bash
# PermissionRequest hook - 当用户不在终端时，弹出可交互的 macOS 授权对话框
# decision 直接联动到 Claude Code 的权限系统
set -euo pipefail

request=$(cat)

# 提取关键信息
tool_name=$(echo "$request" | python3 -c "
import sys, json
d = json.load(sys.stdin)
print(d.get('tool_name', '未知工具'))
" 2>/dev/null || echo "未知工具")

message=$(echo "$request" | python3 -c "
import sys, json
d = json.load(sys.stdin)
print(d.get('message', ''))
" 2>/dev/null || echo "")

tool_input=$(echo "$request" | python3 -c "
import sys, json
d = json.load(sys.stdin)
ti = d.get('tool_input', {})
if 'command' in ti:
    print(ti['command'])
elif 'file_path' in ti:
    print(ti['file_path'])
elif 'description' in ti:
    print(ti['description'])
else:
    print(json.dumps(ti, ensure_ascii=False))
" 2>/dev/null || echo "")

# 检测当前前台 app（优先用 osascript，失败则回退到 AppKit——无需 Accessibility 权限）
front_app=$(osascript -e 'tell application "System Events" to get name of first application process whose frontmost is true' 2>/dev/null || true)
if [[ -z "$front_app" ]]; then
  front_app=$(python3 -c "from AppKit import NSWorkspace; print(NSWorkspace.sharedWorkspace().frontmostApplication().localizedName())" 2>/dev/null || true)
fi

# 在这些终端应用中不弹窗，走内置权限流程
terminal_apps="Terminal iTerm2 Warp kitty Alacritty WezTerm Ghostty Hyper Tabby"

is_terminal=false
for app in $terminal_apps; do
  if [[ "$front_app" == "$app" ]]; then
    is_terminal=true
    break
  fi
done

if [[ "$is_terminal" == "true" ]]; then
  exit 0
fi

# 构建对话框内容
dialog_title="Claude Code 授权请求"

if [[ -n "$message" ]]; then
  dialog_text="$message"
elif [[ -n "$tool_input" ]]; then
  dialog_text="Claude Code 请求执行 [$tool_name]: $tool_input"
else
  dialog_text="Claude Code 请求使用工具: $tool_name"
fi

# 弹出交互对话框
result=$(osascript \
  -e "set theDialog to \"$dialog_text\"" \
  -e "display dialog theDialog with title \"$dialog_title\" buttons {\"拒绝\", \"允许一次\", \"会话允许\"} default button \"允许一次\" cancel button \"拒绝\" with icon caution giving up after 60" 2>&1)

# 注意：检测顺序很重要 — "会话允许" 包含 "允许" 子串，先检测更具体的
if [[ "$result" == *"会话允许"* ]]; then
  cat <<'DECISION'
{
  "hookSpecificOutput": {
    "hookEventName": "PermissionRequest",
    "decision": {
      "behavior": "alwaysAllow"
    }
  }
}
DECISION
elif [[ "$result" == *"允许一次"* ]]; then
  cat <<'DECISION'
{
  "hookSpecificOutput": {
    "hookEventName": "PermissionRequest",
    "decision": {
      "behavior": "allow"
    }
  }
}
DECISION
elif [[ "$result" == *"拒绝"* ]]; then
  cat <<'DECISION'
{
  "hookSpecificOutput": {
    "hookEventName": "PermissionRequest",
    "decision": {
      "behavior": "deny",
      "message": "用户通过对话框拒绝了此操作"
    }
  }
}
DECISION
else
  # 超时或关闭对话框 → 不输出 JSON，走正常权限提示流程
  exit 0
fi
```

---

## 自定义

### 添加更多终端应用

修改脚本中的 `terminal_apps` 变量，或通过 skill 参数 `terminal_apps` 传入额外的 app 名称。

### 修改弹窗超时

修改脚本中 `giving up after 60` 的数字（秒）。

### 修改按钮文本

修改 `osascript` 的 `buttons` 参数，同时更新对应的检测逻辑。

---

## 故障排查

| 现象 | 原因 | 解决 |
|------|------|------|
| 弹窗不出现 | 用户在终端前台 | 切换到非终端 app（浏览器、编辑器等）再触发 |
| 弹窗出现但点击无效 | osascript 权限不足 | 系统设置 → 隐私与安全性 → 辅助功能，确保终端有权限 |
| hook 配置不生效 | JSON 格式错误 | `python3 -c "import json; json.load(open('$HOME/.claude/settings.json'))"` 检查 |
| 脚本权限错误 | 不可执行 | `chmod +x ~/.claude/hooks/permission-alert.sh` |
| 对话框乱码 | 特殊字符未转义 | 检查 `dialog_text` 中是否有未转义的双引号 |
