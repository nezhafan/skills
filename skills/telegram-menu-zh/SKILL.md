---
name: telegram-menu-zh
description: "Telegram 菜单汉化：一键检查未翻译命令 → 补全中文 → 设置语言偏好。纯汉化，无语言切换。Use when the user sends /telegram-menu-zh, asks to check/update Telegram menu translations, or wants the Telegram command menu shown in Chinese."
---

# Menu Zh — Telegram 命令菜单汉化

发送 `/telegram-menu-zh`，Agent 自动完成 Telegram 菜单汉化：
**扫描未翻译命令 → 生成中文翻译 → 写回 JSON → 设置语言偏好 → 提示重启 gateway**。

**本 skill 是纯汉化工具**，不提供英文回退或语言切换。

## When to Use

- 用户发送 `/telegram-menu-zh`（或 `/menu_zh`）
- 用户说「更新菜单翻译」「检查菜单翻译」「补全菜单中文」
- 用户安装新 skill 后，要求检查菜单翻译
- 菜单中某条命令显示英文

**Don't use for：** 修改命令本身的英文名/描述（改 `hermes_cli/commands.py`），或做中英语言切换。

---

## 关键前置：用对 Python

Hermes 代码使用 Python 3.10+ 语法（`int | None`），**系统 `python3`（常见 3.9）会直接报错**
`TypeError: unsupported operand type(s) for |`。

**必须使用 Hermes 自带的 venv Python。** 先定位它（不要硬编码路径）：

```bash
# 1) 优先用 hermes 可执行文件指向的解释器
PY="$(head -1 "$(command -v hermes)" 2>/dev/null | sed 's|^#!||')"
# 2) 回退到常见位置
[ -x "$PY" ] || PY="$HOME/.hermes/hermes-agent/venv/bin/python"
[ -x "$PY" ] || PY="$HOME/.hermes/hermes-agent/.venv/bin/python"
echo "PY=$PY"
"$PY" -V
```

同时定位 Hermes 仓库根目录（用于 `sys.path`）：

```bash
HERMES_SRC="$(dirname "$(dirname "$(readlink -f "$(command -v hermes)")")")"
[ -d "$HERMES_SRC/hermes_cli" ] || HERMES_SRC="$HOME/.hermes/hermes-agent"
echo "HERMES_SRC=$HERMES_SRC"
```

---

## 操作流程（按顺序执行）

### Step 1: 诊断当前状态

用上一步的 `$PY` 运行：

```python
import sys, json, os
sys.path.insert(0, os.environ["HERMES_SRC"])
from hermes_cli.commands import telegram_menu_commands

menu, hidden = telegram_menu_commands(max_commands=100)
zh_path = os.path.expanduser("~/.hermes/telegram_menu_zh.json")
try:
    with open(zh_path) as f:
        zh = json.load(f)
except (FileNotFoundError, json.JSONDecodeError):
    zh = {}

names = {n for n, _ in menu}
translated   = {n for n, _ in menu if n in zh}
untranslated = [(n, d) for n, d in menu if n not in zh]
zombie       = [k for k in zh if k not in names]

print(f"总命令: {len(menu)} | 已翻译: {len(translated)} | 未翻译: {len(untranslated)} | 僵尸条目: {len(zombie)}")
print(f"隐藏(超出上限): {hidden}\n")
if untranslated:
    print("=== 未翻译 ===")
    for n, d in untranslated:
        print(f"  /{n}  →  {d}")
if zombie:
    print("=== 僵尸条目（已不在菜单） ===")
    for k in zombie:
        print(f"  {k}  →  {zh[k]}")
```

> 若 `命令含 ≥100 条`，`hidden` 会 > 0——这些命令不展示，但**翻译仍应保留**在 JSON 中。

### Step 2: 补全未翻译命令

**翻译所有未翻译命令，无论是否超过 100 条。**

Agent 自行生成中文翻译，遵守[翻译规范](#翻译规范)，然后：

```python
import json, os
zh_path = os.path.expanduser("~/.hermes/telegram_menu_zh.json")
with open(zh_path) as f:
    zh = json.load(f)

new_translations = {
    # "command_name": "中文描述",
    # 例： "photo_frame": "生成照片画框",
}
zh.update(new_translations)
with open(zh_path, "w", encoding="utf-8") as f:
    json.dump(zh, f, ensure_ascii=False, indent=2)
print(f"已补全 {len(new_translations)} 条")
```

> 如需处理僵尸条目，**先询问用户**再删除，不要静默清理。

### Step 3: 设置中文语言偏好

```python
import json, os
lang_path = os.path.expanduser("~/.hermes/menu_lang.json")
with open(lang_path, "w", encoding="utf-8") as f:
    json.dump({"lang": "zh"}, f, ensure_ascii=False)
```

### Step 4: 告知用户

```
✅ 已补全 N 条翻译
📋 新增：
  /command1 → 中文描述
  /command2 → 中文描述

⚠️ 当前菜单共 X 条命令，超过 Telegram 100 条上限，
   末尾 Y 条不展示（翻译已保留，功能不受影响）。

🔄 重启 gateway 生效：sudo systemctl restart hermes-gateway（或发送 /restart）
```

---

## 翻译规范

| 规范 | 说明 |
|------|------|
| 长度 | ≤ 15 汉字（UTF-8 ≤ 45 字节，超出会被自动截断） |
| 风格 | 动词开头：切换、查看、管理、设置、浏览…… |
| 意译 | 不直译英文，用自然中文表达功能 |
| skill 命令 | 参考其 SKILL.md 的 `description`，但更精简 |

---

## 常见场景

- **首次汉化**：新装 Hermes → 发 `/telegram-menu-zh` → 补全所有翻译。
- **装了新 skill 后**：新命令进菜单但无翻译 → 发一次命令补全。
- **菜单显示英文**：某命令缺翻译 → 发命令自动定位补全。
- **清理僵尸条目**：翻译文件有已卸载 skill 的条目 → 诊断后**询问用户**是否清理。

---

## 技术背景

- `telegram_menu_commands(max_commands=100)` 从 `hermes_cli/commands.py` 获取全部注册命令，
  按「核心命令 → 插件命令 → skill 命令」优先级排序，仅 skill 层被上限裁剪。
- 翻译流水线（`gateway/platforms/telegram.py`）：
  `_build_telegram_menu()` → `_get_menu_lang()` → `_localize_telegram_menu()` → `set_my_commands()`
- `_ensure_zh_translations()` 在网关启动时合并内置翻译 + `telegram_menu_zh.json`（**用户文件优先**）
- `_localize_telegram_menu()` 自动对中文描述做 UTF-8 截断（45 字节软限制）
- Telegram 菜单上限 **100 条**，超出从 skill 命令末端裁剪
- 改 JSON 后需**重启 gateway** 触发 `set_my_commands` 重新注册

## Common Pitfalls

1. **别用系统 python3** — 会 `TypeError: unsupported operand type(s) for |`；必须用 venv Python
2. **别硬编码 `/root/...`** — 用 `command -v hermes` 推导路径，兼容其他机器
3. **别丢僵尸条目不管，也别静默删** — 先报告并询问
4. **别只翻译前 100 条** — 全部翻译，超限部分保留备用
5. **别忘重启 gateway** — 否则菜单不更新

## Verification Checklist

- [ ] 用的是 venv Python（`$PY -V` 为 3.10+）
- [ ] 诊断输出中未翻译数 = 0（或已全部补全）
- [ ] `telegram_menu_zh.json` 是合法 JSON 且含新增条目
- [ ] `menu_lang.json` 为 `{"lang": "zh"}`
- [ ] 已提示用户重启 gateway
