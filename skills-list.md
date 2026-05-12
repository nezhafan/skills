---
name: skill-list
description: Use when the user asks "what skills are installed", "list skills", "show me all skills", "/skills", "/skill_list", or wants to see locally installed agent skills. Lists all installed Hermes skills grouped by category with Chinese descriptions.
version: 2.0.0
author: kikia
license: MIT
metadata:
  hermes:
    tags: [skills, list, catalog, local]
    related_skills: [hermes-agent-skill-authoring]
---

# Skill List — 本机已安装 Skill 列表

列出本机所有已安装的 Hermes Agent skills，按分类分组，展示中文名称和描述。

**自动注册 Telegram 命令：`/skill_list`**

## When to Use

- 用户问 "有哪些 skill" / "列出所有 skill" / "show skills"
- 用户发 `/skill_list` 或 `/skills`
- 用户想了解本机已安装了什么能力
- 用户想按分类浏览可用 skill

**Don't use for:**
- 搜索/发现网络上的新 skill → 那是 `hermes skills search` 的活
- 用户想看某个 skill 详情 → 用 `skill_view(name)`

## 操作步骤

### Step 1: 获取已安装 Skill 列表

```
skills_list()
```

### Step 2: 按分类整理 + 翻译描述

将结果按 `category` 分组，无分类的归入「未分类」。
将 description 翻译为简洁中文（≤15 字）。

### Step 3: 展示给用户

以分类分组的形式展示，每个 skill 显示名称（加粗）和中文描述。

**输出格式示例：**

```
📦 本机已安装 N 个 Skills

autonomous-ai-agents
• claude-code — Claude Code 开发代理
• codex — Codex 开发代理
  ...

未分类
• skill-list — 本机 Skill 列表
• yuanbao — 元宝群操作
```

### Step 4: 补充使用提示

在列表末尾告知用户：
- 查看详情：`skill_view(name)` 或直接说「看看 XXX skill」
- 日常使用：描述需求即可，agent 自动加载匹配的 skill

## Common Pitfalls

1. **不要用表格** — Telegram 不支持 markdown 表格，用分类+列表格式
2. **不要展开详情** — 这只是列表，按需查看
3. **不要搜索网络** — 这不是 skill 搜索，只列本地已安装
4. **不要跳过翻译** — description 要翻译成中文

## Verification Checklist

- [ ] `skills_list()` 成功返回
- [ ] 按 category 正确分组
- [ ] 每个 skill 展示名称 + 中文描述
- [ ] 末尾有使用提示
