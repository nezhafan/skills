## Skills 列表

一组可复用的 Agent skills，覆盖环境桥接、系统集成、运维自动化等场景。

### 🔌 模型接入

| Skill | 说明 |
|-------|------|
| [codex-deepseek-bridge](skills/codex-deepseek-bridge/SKILL.md) | 将 DeepSeek 模型通过 Moon Bridge 转发层接入 OpenAI Codex，本地部署与配置 |

### 🖥️ 系统集成

| Skill | 说明 |
|-------|------|
| [mac-permission-popup](skills/mac-permission-popup/SKILL.md) | Claude 申请授权时联动 macOS 原生弹窗（注意会发起一次终端授权弹窗，需允许） |
| [telegram-menu-zh](skills/telegram-menu-zh/SKILL.md) | Telegram 命令菜单一键汉化 |

### 🛠️ 运维自动化

| Skill | 说明 |
|-------|------|
| [frp-alicloud-auto-cert](skills/frp-alicloud-auto-cert/SKILL.md) | 家用设备通过 frp 穿透 + 阿里云域名，定期自动签发/续期 Let's Encrypt 证书。DNS-01 验证不占端口，远程服务器只做转发，换服务器仅需改一行配置 |

### 📋 工具

| Skill | 说明 |
|-------|------|
| [skill-list](skills/skill-list/SKILL.md) | 列出已安装的 skill 并汉化总结其功能 |
