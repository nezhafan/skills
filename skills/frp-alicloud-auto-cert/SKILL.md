---
name: frp-alicloud-auto-cert
description: 家用设备（树莓派/NAS/旧笔记本/软路由等无公网IP主机）通过 frp 穿透暴露服务时，为托管在阿里云的域名定期自动签发与续期 Let's Encrypt 证书。方案A：本地 DNS-01 验证（经阿里云 DNS，不占 80/443、不依赖公网服务器、换服务器只改一行 frpc 配置）；方案B：SSH 密钥免密登录公网服务器远程续签并同步回本地。适用于 frp 内网穿透 + 阿里云域名 + 自建 HTTPS 的场景。
---

# 家用设备 + frp + 阿里云域名：本地自动签发证书

## 场景

你有一台**家用设备**（树莓派 / NAS / 旧笔记本 / 软路由 / 迷你主机……任何长期开机、能出网、但没有公网 IP 的机器），通过 **frp 内网穿透**把上面的服务暴露到公网服务器，域名解析托管在**阿里云**，希望对外提供 HTTPS。

典型链路：

```
家用设备（跑着你的服务）─── frpc ───► 公网服务器（frps）─── 用户浏览器
```

本 skill 解决的正是这条链路上最容易变旧的环节：**TLS 证书的签发与自动续期**。

## 为什么要在家用设备这侧做

传统做法是在**公网服务器**上装 certbot 签证书，但这样会带来几个问题：

- 证书签在公网机，与真正跑服务的家用设备分离，两边要手动同步
- HTTP-01 验证需要公网机的 80 端口可达，而 80 常常已被 frps 占用
- 一旦更换公网服务器，证书部署逻辑要跟着重搭一遍

改成**由家用设备主动签发**就都解决了：

- 证书签发、续期、部署 **全部在家用设备上完成**，与服务器解耦
- 用 **DNS-01** 验证 —— 完全不碰 80/443，不需要任何端口映射，不依赖公网机上的 web 服务
- 公网服务器只跑 frps 做纯转发，**零证书逻辑**
- 换公网服务器 = 改一行 `serverAddr`，证书侧完全不受影响

## 架构

```
家用设备（唯一控制端，无需公网 IP）
 ├─ certbot + certbot-dns-aliyun
 ├─ 凭证 ~/.secrets/alicloud.ini (chmod 600)
 ├─ systemd timer     定期自动检查并续期
 ├─ deploy-hook       续期成功后自动重启 frpc
 └─ frpc ────────────► 公网服务器 frps（纯转发，无任何证书配置）
```

## 适用前提

1. 域名解析托管在**阿里云**（其他 DNS 服务商见文末「换 DNS 服务商」）
2. 家用设备**能正常出网**，且有 root 或 sudo 权限
3. 家用设备上用 **frp**（或其他内网穿透方案）把服务暴露到公网
4. 公网侧用 frps 的 `https2http` 插件在穿透隧道内终结 TLS

> 系统要求：Linux（Debian/Ubuntu/Raspberry Pi OS 等）。非 Linux 设备可参考思路，改用对应的定时任务机制。

## 步骤

### 1. 在家用设备上安装 certbot + 阿里云 DNS 插件

系统自带的 certbot 版本通常太老（Debian 11 是 1.12，Ubuntu 22.04 是 1.21），与新版 DNS 插件兼容性差。**用 pip 装新版**：

```bash
pip3 install --upgrade pip setuptools wheel
pip3 install certbot certbot-dns-aliyun

# 验证插件已注册
certbot plugins | grep -A4 dns-aliyun
```

**坑**：pip 装的 `certbot` 在 `/usr/bin/certbot`（不是 `/usr/local/bin`），后面写 systemd unit 时注意路径。

### 2. 创建阿里云 AccessKey（子用户，最小权限）

> 绝对不要用主账号 AK。主账号 AK 泄露等于整个云账户沦陷。

1. 阿里云控制台 → RAM 访问控制 → 用户 → 创建用户（如 `certbot-dns`）
2. 勾选「OpenAPI 调用访问」，生成 AccessKey
3. 只授权一个策略：`AliyunDNSFullAccess`
4. 保存 AccessKey ID / Secret

### 3. 写凭证文件

```bash
mkdir -p ~/.secrets && chmod 700 ~/.secrets
```

文件 `~/.secrets/alicloud.ini`：

```ini
dns_aliyun_access_key = <你的AccessKeyId>
dns_aliyun_access_key_secret = <你的AccessKeySecret>
```

```bash
chmod 600 ~/.secrets/alicloud.ini
```

**重要坑（实测踩过）**：凭证 key 用**下划线**格式 `dns_aliyun_access_key`，**不要**加插件名前缀（如 `certbot_dns_aliyun:dns_aliyun_access_key`）。新版插件（2.0+）源码内读取的是 `access-key` / `access-key-secret`，certbot 的 INI 解析器会自动把下划线形式映射过去。写成带前缀的形式会报错：

```
Missing properties in credentials configuration file:
 * Property "dns_aliyun_access_key" not found
```

排查方法：`grep -n "_get_prop\|add('credentials'" <插件目录>/dns_aliyun.py`

**另一个坑**：命令行参数 `--dns-aliyun` 有歧义（会同时匹配 `--dns-aliyun-propagation-seconds` 和 `--dns-aliyun-credentials`），会报 `ambiguous option`。必须用完整形式 `--authenticator dns-aliyun`。

### 4. 先 dry-run 验证链路

```bash
certbot certonly \
  --authenticator dns-aliyun \
  --dns-aliyun-credentials ~/.secrets/alicloud.ini \
  -d example.com --dry-run --non-interactive --agree-tos \
  -m admin@example.com
```

看到 `The dry run was successful.` 说明 AK 有权限、能写 TXT 记录、LE 能验证成功。

### 5. 正式签发（多域名合并成一张）

```bash
certbot certonly \
  --authenticator dns-aliyun \
  --dns-aliyun-credentials ~/.secrets/alicloud.ini \
  --cert-name example.com \
  -d example.com -d blog.example.com -d sub.example.com \
  --non-interactive --agree-tos \
  -m admin@example.com
```

一张证书含所有 SAN，管理最简单。用 `--cert-name` 固定证书名，便于后续引用。

验证：

```bash
openssl x509 -in /etc/letsencrypt/live/example.com/fullchain.pem \
  -noout -subject -dates -ext subjectAltName
```

### 6. 接入 frpc（https2http 插件终结 TLS）

关键：**证书路径用软链指向 letsencrypt 目录**，这样续期后自动生效，无需改配置、无需重启以外的操作。

```bash
ln -sf /etc/letsencrypt/live/example.com/fullchain.pem /usr/local/frp/certs/merged.fullchain.pem
ln -sf /etc/letsencrypt/live/example.com/privkey.pem   /usr/local/frp/certs/merged.privkey.pem
```

`frpc.toml` 中每个 https 代理的写法：

```toml
[[proxies]]
name = "main-site"
type = "https"
customDomains = ["example.com"]

[proxies.plugin]
type = "https2http"
localAddr = "127.0.0.1:7007"
crtPath = "/usr/local/frp/certs/merged.fullchain.pem"
keyPath = "/usr/local/frp/certs/merged.privkey.pem"
```

重启生效：`systemctl restart frpc`

> 原理：frps 收到 443 域名请求后，把 TLS 握手透传给 frpc 的 `https2http` 插件，插件用这两个文件完成握手，再把明文请求转发给 `localAddr` 上的本地服务。**证书始终在设备本地，永不离开。**

### 7. 自动续期：systemd timer + deploy-hook

**deploy-hook** `/usr/local/frp/deploy-frpc-cert.sh`（记得 `chmod +x`）：

```bash
#!/bin/bash
# certbot deploy-hook: 证书续期成功后自动重启 frpc
LOG=/var/log/letsencrypt/deploy-frpc.log
{
  echo "===== $(date '+%F %T') ====="
  echo "RENEWED_LINEAGE=$RENEWED_LINEAGE"
  if [ "$RENEWED_LINEAGE" = "/etc/letsencrypt/live/example.com" ]; then
    systemctl restart frpc
    sleep 2
    systemctl is-active --quiet frpc && echo "frpc restarted OK" \
      || echo "ERROR: frpc failed to start!"
  else
    echo "skip: unrelated lineage"
  fi
} >> "$LOG" 2>&1
```

**注册 hook** —— 写入 renewal 配置的 `renew_hook`：

```bash
sed -i '/^\[renewalparams\]/a renew_hook = /usr/local/frp/deploy-frpc-cert.sh' \
  /etc/letsencrypt/renewal/example.com.conf
```

**systemd service** `/etc/systemd/system/certbot-renew.service`：

```ini
[Unit]
Description=Certbot renewal (DNS-01 via Alicloud)
After=network-online.target
Wants=network-online.target

[Service]
Type=oneshot
ExecStart=/usr/bin/certbot renew --quiet --no-random-sleep-on-renew
```

**systemd timer** `/etc/systemd/system/certbot-renew.timer`：

```ini
[Unit]
Description=Periodic certbot renewal check

[Timer]
OnCalendar=*-*-* 03,15:17:00
RandomizedDelaySec=3600
Persistent=true

[Install]
WantedBy=timers.target
```

启用：

```bash
systemctl daemon-reload
systemctl enable --now certbot-renew.timer
systemctl list-timers certbot-renew.timer
```

> `certbot renew` 会自动跳过有效期超过 30 天的证书，所以每天跑两次不会浪费 API 配额；`RandomizedDelaySec` 避免整点拥堵；`Persistent=true` 保证设备关机错过时间点后开机补跑。

### 8. 端到端演练

```bash
# 模拟续期（不触发 deploy-hook）
certbot renew --dry-run

# 手动触发 hook，验证「续期后自动重启 frpc」这条路径
RENEWED_LINEAGE=/etc/letsencrypt/live/example.com \
RENEWED_DOMAINS="example.com blog.example.com" \
  /usr/local/frp/deploy-frpc-cert.sh
cat /var/log/letsencrypt/deploy-frpc.log
```

## 更换公网服务器

只改家用设备上 `frpc.toml` 的 `serverAddr`，然后 `systemctl restart frpc`。

证书签发、续期、部署逻辑**完全不动** —— 这正是本方案的核心价值。公网服务器对方案来说是纯可替换的。

## 清理旧的「公网机 certbot」（如果之前那样部署过）

在公网服务器上执行：

```bash
# 1. 备份
tar czf /root/backup-letsencrypt-$(date +%Y%m%d).tar.gz -C /etc letsencrypt

# 2. 停用定时任务
systemctl disable --now certbot.timer certbot.service

# 3. 卸载
export DEBIAN_FRONTEND=noninteractive
apt-get remove --purge -y certbot python3-certbot python3-certbot-nginx
apt-get autoremove -y
rm -rf /etc/letsencrypt
rm -f /etc/cron.d/certbot
systemctl daemon-reload
```

**关键坑**：删掉 `/etc/letsencrypt` 后，如果公网机 nginx 里还残留 `ssl_certificate /etc/letsencrypt/live/...` 的引用，`nginx -t` 会直接失败，**下次 nginx 重启或 reload 会起不来**。

排查与处理：

```bash
nginx -t                                  # 明确报出哪个证书文件找不到
grep -rn 'ssl_certificate' /etc/nginx/    # 找出所有引用
ss -lntp | grep -E ':80|:443'             # 确认端口到底被谁占用
```

先把废弃的 vhost 配置备份后删除。**注意先确认该 vhost 是否真在监听** —— 常见情况是配了 `listen 443` 但 443 实际被 frps 占着（nginx 压根没跑），那些配置是纯死配置，直接删掉即可。

## 方案 B：SSH 远程续签（证书签在公网服务器上）

方案 A（上面 1–8 步）用 DNS-01 在**本地**签发，是首选方案。但如果你已经在**公网服务器（frps）上**用 certbot 签过证书、只是想把「定期检查 + 同步 + 重启」自动化，可以走这条更轻量的路径：**本地脚本通过 SSH 免密登录公网机，检查证书到期情况，到期就在远端续签，再 scp 回本地，最后重启 frpc。**

适用：域名已验证过、证书已存在于两侧，只想加一层自动巡检。不适用：从未签过证书的全新部署（那请用方案 A）。

### B1. 前置：配置 SSH 免密（必须先手动完成）

脚本依赖免密登录，**这一步必须人工做一次**，Agent 无法替用户输密码：

```bash
# 1) 生成密钥对（已存在则直接跳过，加 -N "" 表示不设密钥密码，便于脚本无人值守调用）
ssh-keygen -t rsa -f ~/.ssh/id_rsa -N ""

# 2) 把公钥推到公网服务器（会提示输入一次服务器密码）
ssh-copy-id root@<公网服务器IP>

# 3) 验证免密是否成功（不报错、直接回显 OK 即可）
ssh -o BatchMode=yes root@<公网服务器IP> "echo OK"
```

> **不要用 `sshpass` + 明文密码文件**。密码一旦在服务器侧被改，定时任务会静默失败；明文字符串落盘本身也是安全隐患。密钥认证是唯一推荐做法。
>
> `-N ""` 的意义：让脚本能无人值守调用。若你更看重安全、希望密钥本身也带口令，则需配合 `ssh-agent` 使用，否则 cron 里会因等待输入口令而挂起。

### B2. 续期脚本 `/root/www/scripts/cert-renew-check.sh`

要点：用**数组** `SSH_OPTS` 统一携带密钥与 `BatchMode`；开头做**前置自检**（密钥存在 + 免密可用），失败即退出，避免 cron 里静默挂起；每个域名独立判断成败，最后统一汇总。

```bash
#!/bin/bash
# 每月检查 HTTPS 证书是否本月到期，到期则重新签发并同步
set -uo pipefail

CERTS_DIR="/usr/local/frp/certs"
SSH_KEY="/root/.ssh/id_rsa"
FRPS_HOST="root@<公网服务器IP>"
# BatchMode=yes 让免密失败时立即报错，而不是挂起等密码输入
SSH_OPTS=(-i "$SSH_KEY" -o StrictHostKeyChecking=no -o BatchMode=yes -o ConnectTimeout=15)
FRPS_CERT_DIR="/etc/letsencrypt/live"
DOMAINS=("example.com" "blog.example.com")
THIS_MONTH=$(date +%Y-%m)
RENEWED=0
FAILED=0

# 前置自检
if [ ! -f "$SSH_KEY" ]; then
    echo "[ERR] SSH 密钥不存在: $SSH_KEY"
    echo "[ERR] 请先执行: ssh-keygen -t rsa -f $SSH_KEY -N \"\" && ssh-copy-id $FRPS_HOST"
    exit 1
fi
if ! ssh "${SSH_OPTS[@]}" "$FRPS_HOST" "true" 2>/dev/null; then
    echo "[ERR] 无法免密登录 $FRPS_HOST，请检查 ssh-copy-id 是否已完成"
    exit 1
fi

for domain in "${DOMAINS[@]}"; do
    cert="${CERTS_DIR}/${domain}.fullchain.pem"
    [ -f "$cert" ] || { echo "[SKIP] $domain: 证书文件不存在"; continue; }

    expiry=$(openssl x509 -enddate -noout -in "$cert" | cut -d= -f2)
    expiry_month=$(date -d "$expiry" +%Y-%m 2>/dev/null)

    if [ "$expiry_month" = "$THIS_MONTH" ]; then
        echo "[EXPIRE] $domain: $expiry → 重新签发..."
        # 注意：certbot 与 systemctl start 之间用 ; 而非 &&，
        # 否则续签失败时 frps 不会被重新拉起来
        if ! ssh "${SSH_OPTS[@]}" "$FRPS_HOST" \
            "systemctl stop frps && certbot certonly --standalone --force-renewal -d $domain --agree-tos --non-interactive --email admin@example.com; systemctl start frps"; then
            echo "[ERR] $domain: 远程签发失败"; FAILED=1; continue
        fi
        scp "${SSH_OPTS[@]}" "${FRPS_HOST}:${FRPS_CERT_DIR}/${domain}/fullchain.pem" \
            "${CERTS_DIR}/${domain}.fullchain.pem" || { echo "[ERR] fullchain 同步失败"; FAILED=1; continue; }
        scp "${SSH_OPTS[@]}" "${FRPS_HOST}:${FRPS_CERT_DIR}/${domain}/privkey.pem" \
            "${CERTS_DIR}/${domain}.privkey.pem" || { echo "[ERR] privkey 同步失败"; FAILED=1; continue; }
        echo "[OK] $domain 证书已更新"; RENEWED=1
    else
        echo "[OK] $domain: $expiry (本月不过期)"
    fi
done

[ "$RENEWED" -eq 1 ] && { systemctl restart frpc; echo "[DONE] frpc 已重启"; }
[ "$FAILED" -ne 0 ] && { echo "[DONE] 存在失败项，请手动检查！"; exit 1; }
```

挂进定时任务（每月 10 号凌晨 3 点）：

```bash
cronjob(action='create', name='HTTPS证书月度更新', schedule='0 3 10 * *',
        prompt='执行脚本 /root/www/scripts/cert-renew-check.sh，检查HTTPS域名证书是否本月到期，到期则通过SSH在frps服务器上续签并同步到本地，最后重启frpc。')
```

### B3. 方案 B 的坑

| 问题 | 原因 | 解决 |
|------|------|------|
| 定时任务静默失败，无任何输出 | 用密码文件（`sshpass -f`）认证，服务器改密码后失效 | 改用密钥认证；脚本开头加免密自检 |
| 脚本卡住不返回 | 免密失效后 ssh 转为交互式索要密码 | `SSH_OPTS` 里加 `-o BatchMode=yes` |
| frps 停掉后再没起来 | 远端命令链写成 `... certbot ... && systemctl start frps`，续签失败导致 `start` 不执行 | 用 `;` 分隔，保证 frps 一定被拉起 |
| 部分域名失败被忽略 | 循环内出错仍继续并最终报成功 | 用 `FAILED` 标志汇总，失败则 `exit 1` |

## Pitfalls 汇总

| 问题 | 原因 | 解决 |
|------|------|------|
| `ambiguous option: --dns-aliyun` | 参数前缀有歧义 | 用 `--authenticator dns-aliyun` |
| `Property "dns_aliyun_access_key" not found` | 凭证 key 名写法不对 | 去掉 `certbot_dns_aliyun:` 前缀，用 `dns_aliyun_access_key` |
| systemd unit 报 certbot 不存在 | pip 装在 `/usr/bin/` 而非 `/usr/local/bin/` | `which certbot` 确认真实路径后改 unit |
| 续期后线上证书没变 | deploy-hook 未注册，或未重启 frpc | 检查 renewal.conf 的 `renew_hook`；看 `/var/log/letsencrypt/deploy-frpc.log` |
| 删证书后 `nginx -t` 失败 | nginx 仍引用已删的证书路径 | 删除废弃 vhost；先用 `ss -lntp` 确认端口是否真被 nginx 占用 |
| 证书验证一直等待 | DNS TXT 传播需要时间，默认等 30s | 正常现象；可用 `--dns-aliyun-propagation-seconds` 调整 |
| Python 3.9 弃用警告 | certbot 4.x 即将弃用 3.9 | 仅警告可忽略；有条件升级 Python |
| 设备关机错过续期时间点 | 定时任务未执行 | timer 里保持 `Persistent=true`，开机后自动补跑 |

## 换 DNS 服务商

把 `certbot-dns-aliyun` 换成对应插件，凭证文件里的 key 名同理（安装方式一致）：

- **Cloudflare**：`certbot-dns-cloudflare`，key 为 `dns_cloudflare_api_token`
- **DNSPod / 腾讯云**：`certbot-dns-dnspod`，key 为 `dns_dnspod_api_id` / `dns_dnspod_api_key`
- **通用 RFC2136**：`certbot-dns-rfc2136`

## 验证清单

- [ ] `certbot plugins` 能看到 `dns-aliyun`
- [ ] `certbot certonly ... --dry-run` 成功
- [ ] `openssl x509` 输出包含全部 SAN 域名
- [ ] 公网 `openssl s_client -connect <域名>:443 -servername <域名>` 显示新证书
- [ ] 证书链验证 `Verify return code: 0 (ok)`
- [ ] `systemctl is-enabled certbot-renew.timer` 返回 enabled
- [ ] 手动触发 deploy-hook 后 frpc 正常重启
- [ ] `curl -sI https://<域名>` 返回预期状态码（非 SSL 错误）
