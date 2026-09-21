---
name: skill-list
description: Use when the user asks "what skills are installed", "list skills", "show me all skills", "/skills", "/skill_list", or wants to see locally installed agent skills grouped by category with Chinese descriptions. Lists locally installed agent skills.
---

# Skill List — 本机已安装 Skill 列表

列出本机所有已安装的 Agent skills，按分类分组，展示中文描述。

## When to Use

- 用户问「有哪些 skill」/「列出所有 skill」/「show skills」
- 用户发 `/skill_list`
- 用户想了解本机已安装了什么能力

**Don't use for:**
- 搜索/发现网络上的新 skill
- 查看某个 skill 详情 → 直接用文件读取工具打开对应 SKILL.md

## 操作步骤

### Step 1: 定位当前 Agent 的 skills 目录

从自身运行上下文（system prompt / environment 特征）判断当前 Agent，取其 skills 根目录：

- Hermes → `~/.hermes/skills`
- Claude Code → `~/.claude/skills`
- Codex → `~/.codex/skills`
- OpenClaw → `~/.openclaw/skills`

判断依据是**上下文里出现的标识字符串**（如 `Hermes`/`hermes`、`Claude Code`/`CLAUDECODE`、`Codex`/`codex`），不是目录是否存在——用户可能同时装了多个 Agent。

### Step 2: 一次性提取所有 SKILL.md 的 frontmatter

```bash
find <SKILLS_DIR> -maxdepth 3 -name "SKILL.md" -exec grep -H "^\(name\|description\):" {} \;
```

**关键：必须用 `-maxdepth 3`，不能用 `-maxdepth 2`。**
skills 常按分类分目录存放（如 `skills/creative/photo-frame/SKILL.md`，深度为 3），
用 `-maxdepth 2` 会漏掉绝大多数 skill。

若目录不存在或返回为空，向用户报告：「检测到当前 Agent 为 XXX，但未在 `XXX` 找到已安装的 skill，请确认路径。」

### Step 3: 解析 + 翻译

把每条结果解析为 `name` / `description`，并为 description 生成简洁中文（≤15 字），动词开头、意译而非直译。
按 SKILL.md 路径的首层目录名作为**分类**（顶层直接存放的 skill 归入「未分类」）。

### Step 4: 展示

按分类分组输出，skill 名加粗，末尾附使用提示：

```
📦 本机已安装 N 个 Skills

【分类 A】
• skill-a — 中文描述
• skill-b — 中文描述

【分类 B】
• skill-c — 中文描述

💡 查看详情：直接说「看看 XXX skill」即可
```

## Common Pitfalls

1. **`-maxdepth 2` 会漏 skill** — 真实安装目录多为 3 层，务必用 `-maxdepth 3`
2. **不要用专有 API（如 Hermes 的 `skills_list()`）** — 从上下文识别 + bash 提取，跨 Agent 通用
3. **不要靠遍历目录猜 Agent** — 必须从上下文字符串判断
4. **不要展开详情** — 这是列表，详情按需读取
5. **不要联网搜索** — 只列本地已安装
6. **不要跳过翻译** — description 一律转成中文

## Verification Checklist

- [ ] 用 `-maxdepth 3` 提取，数量与实际安装数一致（不是只有一两条）
- [ ] 每个 skill 展示「名称 + 中文描述」
- [ ] 结果按分类分组
- [ ] 末尾有使用提示
