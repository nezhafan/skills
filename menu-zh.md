---
name: menu-zh
description: "Telegram 菜单汉化：/menu_zh 一键检查未翻译命令 → 补全中文 → 设置语言偏好。无语言切换，纯汉化。"
version: 5.0.0
author: kikia
license: MIT
metadata:
  hermes:
    tags: [telegram, menu, localization, i18n, zh]
    related_skills: [hermes-agent]
---

# Menu Zh — Telegram 命令菜单汉化

发送 `/menu_zh`，Agent 自动完成 Telegram 菜单汉化：扫描未翻译命令 → 生成中文翻译 → 写回 JSON → 设置语言偏好。

**本 skill 是纯汉化工具**，不提供英文回退或语言切换功能。

## 触发条件

- 用户发送 `/menu_zh`
- 用户说「更新菜单翻译」「检查菜单翻译」「补全菜单中文」
- 用户安装了新 skill 后，要求检查翻译

## 操作流程（Agent 按顺序执行）

### Step 1: 诊断当前状态

```python
import sys, json
sys.path.insert(0, '/root/.hermes/hermes-agent')
from hermes_cli.commands import telegram_menu_commands

menu, hidden = telegram_menu_commands(max_commands=100)

zh_path = '/root/.hermes/telegram_menu_zh.json'
try:
    with open(zh_path) as f:
        zh = json.load(f)
except (FileNotFoundError, json.JSONDecodeError):
    zh = {}

translated = {n for n, _ in menu if n in zh}
untranslated = [(n, d) for n, d in menu if n not in zh]
zombie = [k for k in zh if k not in {n for n, _ in menu}]

print(f"总命令数: {len(menu)} | 已翻译: {len(translated)} | 未翻译: {len(untranslated)} | 僵尸条目: {len(zombie)}")
print(f"隐藏命令: {hidden}")
print()
if untranslated:
    print("=== 未翻译命令 ===")
    for n, d in untranslated:
        print(f"  /{n}  →  {d}")
if zombie:
    print("=== 僵尸条目（已不在菜单） ===")
    for k in zombie:
        print(f"  {k}  →  {zh[k]}")
```

### Step 2: 补全未翻译命令

**翻译所有命令，不管是否超过 100 条。** 超出的命令虽不展示，翻译仍保留在 JSON 中。

Agent 自行生成中文翻译，要求：
- ≤ 15 个汉字（UTF-8 ≤ 45 字节）
- 动词开头、简洁自然（参考风格：「切换模型」「查看用量」）
- 不要逐字翻译英文描述，要意译

```python
import json

zh_path = '/root/.hermes/telegram_menu_zh.json'
with open(zh_path) as f:
    zh = json.load(f)

new_translations = {
    # Agent 在此填入生成的翻译
    # "command_name": "中文描述",
}
zh.update(new_translations)

with open(zh_path, 'w', encoding='utf-8') as f:
    json.dump(zh, f, ensure_ascii=False, indent=2)
```

### Step 3: 设置中文语言偏好

```python
import json
lang_path = '/root/.hermes/menu_lang.json'
with open(lang_path, 'w', encoding='utf-8') as f:
    json.dump({"lang": "zh"}, f, ensure_ascii=False)
```

### Step 4: 告知用户

```
✅ 已补全 N 条翻译
📋 新增翻译：
  /command1 → 中文描述
  /command2 → 中文描述
⚠️ 当前菜单共 X 条命令，超过 Telegram 100 条上限。
   末尾 Y 条命令不会展示（翻译已保留，功能不受影响）。
🔄 重启 gateway 生效：
  sudo systemctl restart hermes-gateway
  （或发送 /restart）
```

## 翻译规范

| 规范 | 说明 |
|------|------|
| 长度 | ≤ 15 汉字，UTF-8 ≤ 45 字节 |
| 风格 | 动词开头：切换、查看、管理、设置、浏览…… |
| 意译 | 不要直译英文，用自然中文表达功能 |
| skill 命令 | 参考 SKILL.md 的 `description`，但更精简 |

## 常见场景

### 首次汉化
用户新装 Hermes → 发 `/menu_zh` → Agent 补全所有翻译 → 菜单变为中文。

### 安装新 skill 后
用户装了新 skill → 其命令自动加入菜单但无翻译 → 发 `/menu_zh` 补全。

### 菜单出现英文
某条命令显示英文 → 发 `/menu_zh`，Agent 自动定位并补全。

### 清理僵尸条目
翻译文件中有已卸载 skill 的条目 → Agent 诊断后询问是否清理。

## 技术背景

- `telegram_menu_commands(max_commands=100)` 从 `hermes_cli/commands.py` 获取所有注册命令
- 翻译流水线（`telegram.py`）：`_build_telegram_menu()` → `_get_menu_lang()` → `_localize_telegram_menu()` → `set_my_commands()`
- `_ensure_zh_translations()` 在网关启动时合并内置翻译 + `telegram_menu_zh.json`（用户文件优先）
- `_localize_telegram_menu()` 自动对中文描述做 UTF-8 截断（45 字节软限制）
- Telegram 菜单上限 100 条，超出从 skill 命令末端裁剪
- 修改 JSON 后需重启 gateway 触发 `set_my_commands` 重新注册
