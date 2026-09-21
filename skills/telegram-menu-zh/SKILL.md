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
#    ⚠️ 不要只用 "$HOME/..." —— Hermes 会把 HOME 改写成 profile 目录
#    （如 /root/.hermes/profiles/<name>/home），此时 $HOME/.hermes 并不存在。
#    必须补一个不依赖 HOME 的绝对路径回退。
[ -x "$PY" ] || PY="$HOME/.hermes/hermes-agent/venv/bin/python"
[ -x "$PY" ] || PY="$HOME/.hermes/hermes-agent/.venv/bin/python"
[ -x "$PY" ] || PY="/root/.hermes/hermes-agent/venv/bin/python"
[ -x "$PY" ] || PY="/root/.hermes/hermes-agent/.venv/bin/python"
echo "PY=$PY"
"$PY" -V
```

同时定位 Hermes 仓库根目录（用于 `sys.path`）：

```bash
HERMES_SRC="$(dirname "$(dirname "$(readlink -f "$(command -v hermes)")")")"
[ -d "$HERMES_SRC/hermes_cli" ] || HERMES_SRC="$HOME/.hermes/hermes-agent"
[ -d "$HERMES_SRC/hermes_cli" ] || HERMES_SRC="/root/.hermes/hermes-agent"
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
from hermes_constants import get_hermes_home

menu, hidden = telegram_menu_commands(max_commands=100)
# ⚠️ 必须用 get_hermes_home()，不要用 os.path.expanduser("~/.hermes/...")
#    网关读的就是 get_hermes_home()/"telegram_menu_zh.json"（见 telegram.py
#    _zh_translations_path）。而 Hermes 会把 HOME 改写成 profile 的 home 目录，
#    expanduser("~/.hermes/...") 在 profile 下会解析到错误路径 → 翻译静默失效。
zh_path = get_hermes_home() / "telegram_menu_zh.json"
try:
    with open(zh_path) as f:
        zh = json.load(f)
except (FileNotFoundError, json.JSONDecodeError):
    zh = {}

names = {n for n, _ in menu}
translated   = {n for n, _ in menu if n in zh}
untranslated = [(n, d) for n, d in menu if n not in zh]
# ⚠️ telegram_menu_commands() 已排除 hub skill 命令（源码按 .hub 路径过滤），
#    但翻译文件里为它们保留了条目——这些是「未进菜单故未用到」的预备翻译，
#    不是僵尸。只有既不在菜单、又对应源技能已卸载的条目才算真僵尸。
zombie       = [k for k in zh if k not in names]

print(f"总命令: {len(menu)} | 已翻译: {len(translated)} | 未翻译: {len(untranslated)}")
print(f"未进菜单但已备翻译(正常): {len(zombie)} | 隐藏(超出100上限): {hidden}\n")
if untranslated:
    print("=== 未翻译（需补全） ===")
    for n, d in untranslated:
        print(f"  /{n}  →  {d}")

if zombie:
    print("=== 未进菜单的翻译条目（多为 hub skill 的预备翻译，勿删） ===")
    for k in zombie:
        print(f"  {k}  →  {zh[k]}")
```

> **`zombie` 多半不是真僵尸。** `telegram_menu_commands()` 只返回核心 + 插件 + 内置 skill 命令，
> 用户自行安装（hub）的 skill 命令已被源码排除，但它们的中文翻译仍保留在 JSON 中备用。
> 判断真僵尸：该命令名既不在菜单、其对应 skill 目录也已从本机卸载 —— 只有这种情况才考虑清理，
> 且**必须先询问用户**。

> 若 `命令含 ≥100 条`，`hidden` 会 > 0——这些命令不展示，但**翻译仍应保留**在 JSON 中。

### Step 2: 补全未翻译命令

**翻译所有未翻译命令，无论是否超过 100 条。**

Agent 自行生成中文翻译，遵守[翻译规范](#翻译规范)，然后：

```python
import json
from hermes_constants import get_hermes_home
zh_path = get_hermes_home() / "telegram_menu_zh.json"
try:                                    # 文件可能尚不存在（新 profile 首次运行）
    with open(zh_path, encoding="utf-8") as f:
        zh = json.load(f)
except (FileNotFoundError, json.JSONDecodeError):
    zh = {}

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
import json
from hermes_constants import get_hermes_home
lang_path = get_hermes_home() / "menu_lang.json"
lang_path.parent.mkdir(parents=True, exist_ok=True)
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
- 翻译文件的**权威路径**是 `get_hermes_home() / "telegram_menu_zh.json"`（源码 `telegram.py:_zh_translations_path`）；
  `menu_lang.json` 同目录。**不要用 `~/.hermes/...`**——profile 下 `HOME` 被改写会解析错
- `_localize_telegram_menu()` 自动对中文描述做 UTF-8 截断（45 字节软限制）
- Telegram 菜单上限 **100 条**，超出从 skill 命令末端裁剪
- 改 JSON 后需**重启 gateway** 触发 `set_my_commands` 重新注册

## Common Pitfalls

1. **别用系统 python3** — 会 `TypeError: unsupported operand type(s) for |`；必须用 venv Python
2. **别硬编码 `/root/...`** — 优先用 `command -v hermes` 推导路径，兼容其他机器
2b. **别只用 `$HOME/.hermes/hermes-agent` 做回退** — Hermes 会把 `HOME` 改写成 profile 目录（如 `/root/.hermes/profiles/<name>/home`），该路径不存在；必须补不依赖 `HOME` 的绝对路径回退（见上）
2c. **别用 `os.path.expanduser("~/.hermes/...")` 定位翻译文件** — 网关读的是 `get_hermes_home()/"telegram_menu_zh.json"`（源码 `telegram.py:_zh_translations_path`）。`HOME` 被改写后 `expanduser` 会解析到 `<profile>/home/.hermes/...` 这个**不存在的路径**，导致翻译静默失效；必须 `from hermes_constants import get_hermes_home` 再拼路径
3. **别把「未进菜单的翻译条目」当真僵尸** — hub skill 命令被 `telegram_menu_commands()` 排除，其翻译属正常预备保留；确认真僵尸（命令和 skill 都不存在）后**先报告并询问用户**，不要静默删
4. **别只翻译前 100 条** — 全部翻译，超限部分保留备用
5. **别忘重启 gateway** — 否则菜单不更新

## Verification Checklist

- [ ] 用的是 venv Python（`$PY -V` 为 3.10+）
- [ ] 诊断输出中未翻译数 = 0（或已全部补全）
- [ ] `telegram_menu_zh.json` 是合法 JSON 且含新增条目
- [ ] `menu_lang.json` 为 `{"lang": "zh"}`
- [ ] 已提示用户重启 gateway
