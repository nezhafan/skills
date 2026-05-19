---
name: skill-list
description: Use when the user asks "what skills are installed", "list skills", "show me all skills", "/skills", "/skill_list", or wants to see locally installed agent skills. Lists all installed skills grouped by category with Chinese descriptions.
---

# Skill List — 本机已安装 Skill 列表

列出本机所有已安装的 Agent skills，展示中文描述。

## When to Use

- 用户问 "有哪些 skill" / "列出所有 skill" / "show skills"
- 用户发 `/skill_list`
- 用户想了解本机已安装了什么能力

**Don't use for:**
- 搜索/发现网络上的新 skill
- 用户想看某个 skill 详情 → 直接 `Read` 对应的 SKILL.md

## 操作步骤

### Step 1: 从上下文识别当前智能体

查看你自己的系统上下文（Environment 部分、system prompt 等），匹配以下关键字来识别当前运行在哪个智能体中：

| 上下文特征 | 智能体 | skills 路径 |
|-----------|--------|-------------|
| `Claude Code`, `claude-code`, `CLAUDECODE` | Claude Code | `~/.claude/skills` |
| `Codex`, `codex` | Codex | `~/.codex/skills` |
| `Hermes`, `hermes` | Hermes | `~/.hermes/skills` |
| `OpenClaw`, `openclaw` | OpenClaw | `~/.openclaw/skills` |

### Step 2: 提取所有 SKILL.md 的 description

确定 skills 路径后，用**一次** bash 调用提取：

```bash
find <SKILLS_DIR> -maxdepth 2 -name "SKILL.md" -exec grep -H "^\(name\|description\):" {} \;
```

如果路径不存在，向用户报告：「检测到当前智能体为 XXX，但 skills 目录 XXX 不存在，请确认。」

### Step 3: 整理并翻译

解析输出，提取每个 skill 的 `name` 和 `description`，将 description 翻译为简洁中文（≤15 字）。

### Step 3: 整理并翻译

解析 Step 2 的输出，提取每个 skill 的 `name` 和 `description`，将 description 翻译为简洁中文（≤15 字）。

### Step 4: 展示给用户

以列表形式展示，每个 skill 显示名称（加粗）和中文描述。末尾附加使用提示。

**输出格式示例：**

```
📦 本机已安装 N 个 Skills（检测到 XXX 智能体）

• brainstorming — 创意头脑风暴
• writing-plans — 编写实现计划
• writing-skills — 编写 Skill 文件

💡 查看详情：直接说「看看 XXX skill」即可
```

## Common Pitfalls

1. **不要用 `skills_list()` 等专有 API** — 从上下文识别智能体 + bash 提取，通用且只需一次权限
2. **不要遍历目录猜智能体** — 用户可能装了多个智能体，必须从上下文特征精确识别
3. **不要展开详情** — 这只是列表，按需查看
4. **不要搜索网络** — 只列本地已安装
5. **不要跳过翻译** — description 要翻译成中文

## Verification Checklist

- [ ] 从上下文正确识别了当前智能体
- [ ] `find` + `grep` 成功返回结果
- [ ] 每个 skill 展示名称 + 中文描述
- [ ] 末尾有使用提示
