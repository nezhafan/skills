---
name: menu-lang
description: "Use when setting up Chinese/English language switching for the Telegram bot command menu. Adds a /menu_lang command trigger that shows 「🌐 中文菜单」and「🌐 English Menu」inline buttons."
version: 2.1.0
author: kikia
license: MIT
metadata:
  hermes:
    tags: [telegram, menu, localization, i18n, zh, setup]
    related_skills: [hermes-agent]
---

# Menu Lang — Telegram 命令菜单语言切换（按钮式）

## 概述

为 Hermes Agent Telegram bot 的 `/menu_lang` 命令添加**内联按钮**支持：用户发送 `/menu_lang` 后 bot 弹出两个按钮供选择语言。

## 安装后如何生效

安装此 skill 后，Telegram 菜单中会出现 `/menu_lang` 命令（显示为 `/menu_lang`）。但此时发送它只会进入普通对话——因为 gateway 代码中还缺少命令触发入口。

**下一步**：发 `/menu_lang` 或说「授权修改」，agent 会提示：

> 需要修改的文件：`telegram.py`（1 处，添加 `/menu_lang` 命令触发 handler）
> 
> 已存在的代码无需改动：按钮发送、回调处理、菜单构建、中文翻译均已就绪。

用户回复「授权」后 agent 自动完成修改，重启 gateway 即可使用按钮切换语言。

## When to Use

- 用户说"帮我设置菜单语言切换"
- 用户说"让 /menu_lang 弹按钮"
- 用户发 `/menu_lang` 或说「授权修改」
- 用户问"为什么 /menu_lang 没有按钮"

## 代码现状

以下组件已在 `telegram.py` 中实现，**无需修改**：

| 组件 | 行号 | 说明 |
|------|------|------|
| `send_menu_lang_picker()` | 1908 | 发送 🌐中文/English 按钮 |
| `_handle_menu_lang_callback()` | 2187 | 处理按钮点击→切换语言 |
| `_handle_callback_query()` | 2259 | `ml:` 前缀转发到菜单处理 |
| `_build_telegram_menu()` | 3528 | 构建菜单 + 本地化 |
| `_localize_telegram_menu()` | 3597 | 中文翻译应用 |
| `_ensure_zh_translations()` | 3642 | 补全缺失翻译（28 条内置） |
| `_get_menu_lang()` / `_set_menu_lang()` | 3564/3580 | 语言偏好读写 |
| `menu_lang` 翻译条目 | 3671 | `"menu_lang": "切换菜单语言"` |

**仅需添加 1 处**：在 handler 注册区为 `/menu_lang` 添加特定触发器，调用已有 `send_menu_lang_picker()`。

## 操作流程

Agent 收到授权后：

1. **在 handler 注册区添加 menu_lang 拦截器**
   - 位置：`_handle_command` 的 `filters.COMMAND` handler 之前（约 1148 行）
   - 添加一个 `TelegramMessageHandler`，匹配 `/menu_lang` 或 `/menu-lang`
   - 回调方法调用 `send_menu_lang_picker()`

2. **添加 handler 方法**：
   ```python
   async def _handle_menu_lang_command(self, update, context):
       chat_id = str(update.message.chat_id)
       current_lang = self._get_menu_lang()
       await self.send_menu_lang_picker(chat_id, current_lang)
   ```

3. **告知用户重启 gateway**

4. **验证**：在 Telegram 中发送 `/menu_lang`，确认弹出两个按钮

## 禁止事项

- **禁止擅自修改 telegram.py 或重启 gateway 而不先让用户授权**
- **禁止 kill 进程后在未授权的情况下用其他方式（pkill 等）重试**

## 验证清单

- [ ] `/menu_lang` 在菜单中可见
- [ ] 发送 `/menu_lang` 后弹出两个按钮
- [ ] 点「🌐 中文菜单」后菜单切换为中文
- [ ] 点「🌐 English Menu」后菜单恢复英文
- [ ] 重启 gateway 后语言偏好保持

## 常见陷阱

1. **只安装 skill 不改代码** → `/menu_lang` 只是普通对话
2. **忘了重启 gateway** → 改了 telegram.py 但 handler 不生效
3. **Telegram 客户端缓存** → 退出重进才看到菜单变化
