---
name: codex-deepseek-bridge
type: skill-readme
skill_entrypoint: SKILL.md
platform: Codex (desktop / CLI)
---

# codex-deepseek-bridge

把 **DeepSeek 模型**接进 **OpenAI Codex**（桌面端 / CLI），通过本地 **Moon Bridge** 转发层完成协议转换。

## 这个 Skill 做什么

Codex 默认只认 OpenAI 协议的模型。本 skill 在本地跑一个 Moon Bridge 代理，
把 Codex 的请求转成 DeepSeek 能理解的协议，从而让 Codex 用上 DeepSeek。

```
Codex  →  Moon Bridge (Transform)  →  DeepSeek API
```

## 快速开始

1. 把本目录发给 Codex（或支持该 skill 的 Agent），说「安装」。
2. 说「用 codex-deepseek-bridge 部署」，按提示输入 DeepSeek API Key。
3. 重启 Codex 生效。

## 参数

| 参数 | 必填 | 默认 | 说明 |
|------|------|------|------|
| `install_dir` | 是 | — | Moon Bridge 安装目录；不存在则自动 `git clone` |
| `model_name` | 否 | `deepseek-v4-pro` | 也可选 `deepseek-v4-flash` |
| `codex_home` | 否 | `~/.codex` | Codex 配置目录 |

> **API Key 不通过参数传入**。写 `config.yml` 前直接让用户输入
> （取自 [platform.deepseek.com/api_keys](https://platform.deepseek.com/api_keys)），
> 每次部署都重新询问，不写入任何参数或预设环境变量。

## 适用症状

- Codex 一直显示 “Reconnecting”
- Codex 找不到 moonbridge 模型
- 需要本地部署 Moon Bridge 转发层

## 关键信息

| 项目 | 值 |
|------|-----|
| Moon Bridge 地址 | `http://127.0.0.1:38440`（默认） |
| 上游 API | DeepSeek API（Anthropic 协议） |
| 源码 | `https://github.com/ZhiYi-R/moon-bridge.git` |

## 文件

- `SKILL.md` — Agent 执行的完整部署指令

## 目录结构

```
codex-deepseek-bridge/
└── SKILL.md
```
