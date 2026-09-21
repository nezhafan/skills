---
name: mac-permission-popup
type: skill-readme
skill_entrypoint: SKILL.md
platform: macOS + Claude Code
---

# mac-permission-popup

在 macOS 上给 Claude Code 装一个**原生授权弹窗**：当 Claude 请求危险权限（如 `rm`、`sudo`）
且你不在终端前台时，弹出系统对话框，点选 拒绝 / 允许一次 / 会话允许 即联动回 Claude。

## 这个 Skill 做什么

Claude Code 每次要执行敏感操作都会请求授权。如果你此时在浏览器/编辑器里，
终端里的提示看不到。本 skill 用一个 `PermissionRequest` hook 拦截请求，
通过 macOS `osascript` 弹原生对话框，把你的点击结果回传给 Claude 权限系统。

```
Claude 请求权限 → PermissionRequest hook → permission-alert.sh
    ├─ 前台是终端     → exit 0，走内置提示
    └─ 前台非终端     → 弹原生对话框 → deny / allow / alwaysAllow
```

## 快速开始

1. 把本目录发给 Claude Code，说「安装这个 skill」。
2. 说「用 mac-permission-popup 部署」。首次注册 hook 时终端会弹一次授权，**允许**即可。
3. 切到浏览器后让 Claude 执行 `rm` 之类操作，即可看到弹窗。

## 关键信息

| 项目 | 值 |
|------|-----|
| 平台 | **仅 macOS** |
| 脚本路径 | `~/.claude/hooks/permission-alert.sh` |
| Hook 事件 | `PermissionRequest` |
| 配置文件 | `~/.claude/settings.json` |
| 弹窗超时 | 60 秒（超时回落内置提示） |

## 操作模式

- `setup`（默认）— 部署脚本 + 合并 hook 配置 + 验证
- `status` — 检查部署状态
- `uninstall` — 移除 hook 配置（脚本是否删除由你决定）

## 依赖

- macOS（`osascript`）
- Python 3（脚本内解析 hook JSON）
- 终端需要有「自动化」权限（首次弹窗时授权）

## 文件

- `SKILL.md` — Agent 执行的完整指令，含可直接部署的 `permission-alert.sh` 全文

## 目录结构

```
mac-permission-popup/
└── SKILL.md
```
