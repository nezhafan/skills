---
name: skill-list
type: skill-readme
skill_entrypoint: SKILL.md
platform: any
---

# skill-list

列出本机已安装的 Agent skills，按分类分组并翻译成中文描述。

## 这个 Skill 做什么

用户想知道「这台机器上装了哪些 skill」时，扫描当前 Agent 的 skills 目录，
提取每个 `SKILL.md` 的 `name` / `description`，按分类分组，用中文展示。

- **输入**：用户的一句话（或 `/skill_list`）
- **输出**：按分类分组的 skill 列表 + 中文描述
- **不涉及**：联网搜索、安装/卸载、skill 详情展开

## 支持的 Agent

| Agent | skills 目录 |
|-------|-------------|
| Hermes | `~/.hermes/skills` |
| Claude Code | `~/.claude/skills` |
| Codex | `~/.codex/skills` |
| OpenClaw | `~/.openclaw/skills` |

Agent 由**运行上下文中的标识字符串**判断（如 `Hermes`、`CLAUDECODE`），
不靠目录是否存在猜测。

## 快速开始

1. 把本目录发给你的 Agent，说「帮我安装这个 skill」。
2. 直接问 Agent「有哪些 skill」即可触发。

## 核心逻辑

```bash
# 关键：-maxdepth 3（skill 常按分类分目录，深度为 3）
find <SKILLS_DIR> -maxdepth 3 -name "SKILL.md" -exec grep -H "^\(name\|description\):" {} \;
```

> ⚠️ 用 `-maxdepth 2` 会漏掉绝大多数 skill（只找到顶层那一两个）。

## 文件

- `SKILL.md` — Agent 执行的完整指令（含步骤、坑点、校验清单）

## 目录结构

```
skill-list/
└── SKILL.md
```
