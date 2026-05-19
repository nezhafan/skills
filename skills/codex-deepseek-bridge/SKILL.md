---
name: codex-deepseek-bridge
description: Use when the user wants to connect Codex (desktop or CLI) to DeepSeek models through Moon Bridge, or when Codex shows "Reconnecting", can't find the moonbridge model, or needs local deployment of the Moon Bridge proxy layer.
---

# Skill: codex-deepseek-bridge
**Description:** 将 DeepSeek 模型通过 Moon Bridge 转发层接入 OpenAI Codex，实现本地部署和配置。核心链路：Codex → Moon Bridge (Transform) → DeepSeek API (Anthropic 协议)

**Input Parameters:**
- `install_dir` (string, required): Moon Bridge 安装目录的绝对路径。若目录不存在则自动执行 `git clone https://github.com/ZhiYi-R/moon-bridge.git`，可选值：`~/.moonbridge` 目录下。
- `model_name` (string, optional, default: `"deepseek-v4-pro"`): 模型选择。可选值：`deepseek-v4-pro`（旗舰）或 `deepseek-v4-flash`（快速）
- `codex_home` (string, optional): Codex 配置目录。默认为 `~/.codex`

> **API Key 不通过参数传入**。执行 Step 2 写入 config.yml 前，直接让用户输入 DeepSeek API Key（从 [platform.deepseek.com/api_keys](https://platform.deepseek.com/api_keys) 获取），不提供选项。不预设到环境变量，不写入任何参数。每次部署都重新询问。

**Outputs:**
- `deploy_status` (object): 部署结果，包含以下字段：
  - `success` (boolean): 部署是否成功
  - `moonbridge_url` (string): Moon Bridge 监听地址（默认 `http://127.0.0.1:38440`）
  - `config_toml` (string): 生成的 Codex 配置文件路径
  - `models_catalog` (string): 生成的模型目录文件路径
- `skill_markdown` (string): 用户后续操作的指引文本（重启 Codex、下次启动命令等）

**Example Usage (Python):**
```python
# api_key 不通过参数传入，执行时直接让用户输入，不提供选项
input_data = {
    "install_dir": "/home/user/moon-bridge",
    "model_name": "deepseek-v4-pro",
    "codex_home": "/home/user/.codex"
}

output = moonbridge_codex_setup(inputs=input_data)

print("Deploy Status:", output["deploy_status"])
print("Guide:", output["skill_markdown"])
```

---

## 平台说明

本 skill 仅支持 Linux/macOS。执行前设置变量：

| 变量 | 值 |
|------|-----|
| `CODEX_HOME` | `${CODEX_HOME:-$HOME/.codex}` |
| `TEMP_DIR` | `/tmp` |
| 后台启动 | `nohup ... &` |
| 查看日志 | `tail -f /tmp/moonbridge.log` |
| 退出桌面版 | `Cmd+Q` (macOS) |
| Shell 配置 | `~/.zshrc` / `~/.bashrc` |

---

## 部署步骤

### Step 1: 检查前置条件

```bash
go version 2>&1           # 需要 1.25+
```

确认 Codex 配置目录存在（通常在 `~/.codex`，桌面版用户默认已有）：
```bash
ls -d "${CODEX_HOME:-$HOME/.codex}" 2>/dev/null || echo "Codex 目录不存在，将在 Step 5 自动创建"
```

### Step 2: 获取 API Key + 创建 data 目录 + 写入 config.yml

**首先**，直接让用户输入 DeepSeek API Key，不提供选项：

> 请输入你的 DeepSeek API Key（从 platform.deepseek.com/api_keys 获取）

用户输入后得到 `api_key` 值，继续后续步骤。

```bash
mkdir -p $MB_DIR/data
```

然后写入 `$MB_DIR/config.yml`，**将模板中的 `$API_KEY`、`$MODEL_NAME` 占位符替换为用户实际值**（`$API_KEY` 替换为上一步交互获取的值）。关键结构：`providers`、`models`、`routes`、`defaults` 全部是顶层键，不能包在 `provider:` 下。

```yaml
mode: "Transform"
server:
  addr: "127.0.0.1:38440"

persistence:
  active_provider: db_sqlite

extensions:
  deepseek_v4:
    config:
      reinforce_instructions: true
  db_sqlite:
    enabled: true
    config:
      path: ./data/moonbridge.db
      wal: true
      busy_timeout_ms: 5000
      max_open_conns: 1

cache:
  mode: "explicit"
  ttl: "5m"
  prompt_caching: true

defaults:
  model: "moonbridge"
  max_tokens: 65536

models:
  $MODEL_NAME:
    context_window: 1000000
    max_output_tokens: 384000
    default_reasoning_level: "high"
    supported_reasoning_levels:
      - effort: "high"
        description: "High reasoning effort"
      - effort: "xhigh"
        description: "Extra high reasoning effort"
    supports_reasoning_summaries: true
    default_reasoning_summary: "auto"
    extensions:
      deepseek_v4:
        enabled: true

providers:
  deepseek:
    base_url: "https://api.deepseek.com/anthropic"
    api_key: "$API_KEY"
    version: "2023-06-01"
    offers:
      - model: $MODEL_NAME
        pricing:
          input_price: 2
          output_price: 8
          cache_write_price: 1
          cache_read_price: 0.2

routes:
  moonbridge:
    model: $MODEL_NAME
    provider: deepseek
```

### Step 3: 验证编译

```bash
cd $MB_DIR
go build ./cmd/moonbridge 2>&1
```

编译通过后，**不要在这里启动服务**。告诉用户用以下命令自行在终端启动：

```bash
cd $MB_DIR && go run ./cmd/moonbridge --config config.yml
```

后台运行：

```bash
# Linux/macOS
cd $MB_DIR && nohup go run ./cmd/moonbridge --config config.yml > /tmp/moonbridge.log 2>&1 &
```

成功标志：`Moon Bridge 监听于 127.0.0.1:38440`

### Step 4: 验证连通性（用户启动 Moon Bridge 后执行）

```bash
curl -s http://127.0.0.1:38440/v1/models
curl -s http://127.0.0.1:38440/v1/responses \
  -H "Content-Type: application/json" \
  -d '{"model":"moonbridge","input":"用一句话打招呼","max_output_tokens":50}'
```

应返回 DeepSeek 回复。`401` = key 有问题，`402` = 余额不足。

### Step 5: 生成 Codex 配置

```bash
cd $MB_DIR
CODEX_HOME_DIR="${CODEX_HOME:-$HOME/.codex}"
mkdir -p "$CODEX_HOME_DIR"

# 备份现有配置
cp "$CODEX_HOME_DIR/config.toml" "$CODEX_HOME_DIR/config.toml.bak" 2>/dev/null || true
cp "$CODEX_HOME_DIR/models_catalog.json" "$CODEX_HOME_DIR/models_catalog.json.bak" 2>/dev/null || true

MODEL="$(go run ./cmd/moonbridge --config config.yml --print-codex-model)"
go run ./cmd/moonbridge \
  --config config.yml \
  --print-codex-config "$MODEL" \
  --codex-base-url "http://127.0.0.1:38440/v1" \
  --codex-home "$CODEX_HOME_DIR" \
  > /tmp/codex_config_new.toml
```

生成后检查新文件（`/tmp/codex_config_new.toml`）第一行必须是 `model = "moonbridge"`（不能用 `2>&1` 重定向，否则日志行混入导致 TOML 解析失败）。如果 `$CODEX_HOME_DIR/config.toml` 已有 `[marketplaces...]`、`[plugins...]`、`[projects...]` 等段，将它们追加到新文件末尾再覆盖回去。

生成的两个文件：
- `~/.codex/config.toml` — provider 配置，`wire_api = "responses"`（旧文件备份为 `config.toml.bak`）
- `~/.codex/models_catalog.json` — 模型能力元数据（旧文件备份为 `models_catalog.json.bak`）

### Step 6: 后续操作

**立即生效：**
- Codex 桌面版：完全退出后重新打开（macOS: Cmd+Q）
- Codex CLI：`CODEX_HOME="$HOME/.codex" codex --cd /path/to/project`

**下次启动 Moon Bridge：**
```bash
cd $MB_DIR && go run ./cmd/moonbridge --config config.yml
```

可建议加到 shell 配置文件做 alias。查日志：
```bash
tail -f /tmp/moonbridge.log
```

---

## 常见问题

| 现象 | 原因 | 解决 |
|------|------|------|
| `field provider not found` | config.yml 用了 `provider:` 嵌套旧格式 | 用本 skill 模板重写 |
| Codex 一直 Reconnecting | Moon Bridge 没启动 | `curl http://127.0.0.1:38440/v1/models` |
| `unable to open database file (14)` | 没有 `data/` 目录 | `mkdir -p $MB_DIR/data` |
| `config.toml` 第一行是日志 | 生成用 `2>&1` 混入 stderr | 重新生成，只用 `>` |
| `401` | API Key 无效 | 检查 config.yml 中 `api_key` |
| `402` | DeepSeek 欠费 | platform.deepseek.com 充值 |
| Codex 配错了想还原 | config.toml 被覆盖 | `cp ~/.codex/config.toml.bak ~/.codex/config.toml` |
| Codex 看不到 Moonbridge | `models_catalog.json` 未生成 | 重新执行 Step 5 |
