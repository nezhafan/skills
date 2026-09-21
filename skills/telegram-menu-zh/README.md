---
name: telegram-menu-zh
type: skill-readme
skill_entrypoint: SKILL.md
platform: Hermes (Telegram gateway)
---

# telegram-menu-zh

把 Hermes 的 Telegram 命令菜单一键汉化：扫描未翻译命令 → 生成中文 → 写回 JSON → 设语言偏好。

## 这个 Skill 做什么

Hermes 在 Telegram 里注册了命令菜单（`/model`、`/new` 等）。本 skill 让这些命令的
**描述**显示成中文。

- **输入**：用户发送 `/telegram-menu-zh`
- **输出**：`<HERMES_HOME>/telegram_menu_zh.json` 补全 + `<HERMES_HOME>/menu_lang.json` 设为 `zh`
- **之后**：重启 gateway 使菜单更新

**纯汉化**，不做中英切换。

> 📁 **路径说明**：`<HERMES_HOME>` 由 `get_hermes_home()` 决定 —— 默认 profile 是 `~/.hermes`，
> profile 模式下是 `~/.hermes/profiles/<name>`。**不要写死 `~/.hermes/...`**，因为 Hermes 会把
> `HOME` 改写成 profile 的 home 目录，`expanduser` 会解析到错误位置。

## 快速开始

1. 安装本 skill 后，在 Telegram 发送 `/telegram-menu-zh`。
2. Agent 自动诊断 → 补全 → 提示重启 gateway。

## 关键依赖

| 项目 | 值 |
|------|-----|
| 运行环境 | Hermes（含 Telegram gateway） |
| Python | **必须用 Hermes venv 的 Python（3.10+）** |
| 翻译文件 | `<HERMES_HOME>/telegram_menu_zh.json`（经 `get_hermes_home()` 定位） |
| 语言偏好 | `<HERMES_HOME>/menu_lang.json` |

> ⚠️ 系统 `python3` 常为 3.9，导入 Hermes 模块会报
> `TypeError: unsupported operand type(s) for |`。必须用 venv 解释器。

## 生效方式

改完 JSON 后**需重启 gateway**：

```bash
sudo systemctl restart hermes-gateway   # 或发送 /restart
```

## 文件

- `SKILL.md` — Agent 执行的完整指令（诊断脚本、补全脚本、翻译规范、坑点）

## 目录结构

```
telegram-menu-zh/
└── SKILL.md
```
