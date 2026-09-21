---
name: pi-driven-certbot-dns01
description: 从内网机器（树莓派等无公网IP主机）主动签发/续签 Let's Encrypt 证书，用 DNS-01 验证 + 阿里云 DNS 插件。证书与自动化 100% 在本地闭环，公网服务器只做端口转发。适用于 frp 内网穿透、自建反代等场景。换公网服务器时零成本迁移。
---

# Pi 端驱动的 Let's Encrypt 证书签发（DNS-01 / 阿里云）

## 这个 skill 解决什么问题

传统做法是"在公网服务器上跑 certbot"：证书签在公网机、需要 80/443 做 HTTP-01 验证、换服务器证书逻辑要重搭。

本方案反过来：**完全由内网机器（Pi）主动推送**。

- 证书签发、续期、部署 **全部在内网机器上完成**
- 用 **DNS-01** 验证 —— 不碰 80/443，不需要端口映射，不依赖公网机上的任何 web 服务
- 公网服务器只跑 frps 做纯转发，**零证书逻辑**
- 换公网机 = 改一行 `serverAddr`，证书侧完全不受影响

## 适用前提

1. 域名 DNS 托管在**阿里云**（其他 DNS 商换对应 certbot 插件即可，见文末）
2. 有一台**能出网的内网机器**（Pi / NAS / VPS 都行），可以是 root
3. 内网机器上用 frp（或其他内网穿透）把服务暴露到公网
4. 公网侧用 frps 的 `https2http` 插件终结 TLS

## 架构

```
内网机器（唯一控制端）
 ├─ certbot + certbot-dns-aliyun
 ├─ 凭证 ~/.secrets/<dns>.ini (chmod 600)
 ├─ systemd timer   每周两检，自动续期
 ├─ deploy-hook     续期成功后自动重启 frpc
 └─ frpc ──────► 公网 frps（纯转发，无任何证书配置）
```

## 步骤

### 1. 安装 certbot + 阿里云 DNS 插件

系统自带的 certbot 版本通常太老（Debian 11 是 1.12，Ubuntu 22.04 是 1.21），与新版 DNS 插件兼容性差。**用 pip 装新版**：

```bash
pip3 install --upgrade pip setuptools wheel
pip3 install certbot certbot-dns-aliyun

# 验证插件已注册
certbot plugins | grep -A4 dns-aliyun
```

**坑**：`certbot` 会被装到 `/usr/bin/certbot`（不是 `/usr/local/bin`），写 systemd unit 时注意路径。

### 2. 创建阿里云 AccessKey（子用户，最小权限）

> 绝对不要用主账号 AK。主账号 AK 泄露等于整个账户沦陷。

1. 阿里云控制台 → RAM 访问控制 → 用户 → 创建用户（如 `certbot-dns`）
2. 勾选「OpenAPI 调用访问」，生成 AccessKey
3. 只授权一个策略：`AliyunDNSFullAccess`
4. 保存 AccessKey ID / Secret

### 3. 写凭证文件

```bash
mkdir -p ~/.secrets && chmod 700 ~/.secrets
```

文件 `~/.secrets/aliyun.ini`：

```ini
dns_aliyun_access_key = <你的AccessKeyId>
dns_aliyun_access_key_secret = <你的AccessKeySecret>
```

```bash
chmod 600 ~/.secrets/aliyun.ini
```

**重要坑（踩过）**：凭证文件的 key 名用**下划线**格式 `dns_aliyun_access_key`，**不要**带插件名前缀（如 `certbot_dns_aliyun:dns_aliyun_access_key`）。新版插件（2.0+）源码里读的是 `access-key` / `access-key-secret`，certbot 的 INI 解析器会自动把 `dns_aliyun_access_key` 映射过去。写成带前缀的形式会报：

```
Missing properties in credentials configuration file:
 * Property "dns_aliyun_access_key" not found
```

验证方法：`grep -n "_get_prop\|add('credentials'" <插件目录>/dns_aliyun.py`

**另一个坑**：命令行参数 `--dns-aliyun` 有歧义（会匹配到 `--dns-aliyun-propagation-seconds` 和 `--dns-aliyun-credentials`），会报 `ambiguous option`。必须用完整形式 `--authenticator dns-aliyun`。

### 4. 先 dry-run 验证链路

```bash
certbot certonly \
  --authenticator dns-aliyun \
  --dns-aliyun-credentials ~/.secrets/aliyun.ini \
  -d example.com --dry-run --non-interactive --agree-tos \
  -m admin@example.com
```

看到 `The dry run was successful.` 说明 AK 有权限、能写 TXT 记录、LE 能验证。

### 5. 正式签发（多域名合并成一张）

```bash
certbot certonly \
  --authenticator dns-aliyun \
  --dns-aliyun-credentials ~/.secrets/aliyun.ini \
  --cert-name example.com \
  -d example.com -d blog.example.com -d sub.example.com \
  --non-interactive --agree-tos \
  -m admin@example.com
```

一张证书含所有 SAN，管理最简单。用 `--cert-name` 固定证书名，方便后续引用。

验证：

```bash
openssl x509 -in /etc/letsencrypt/live/example.com/fullchain.pem \
  -noout -subject -dates -ext subjectAltName
```

### 6. 接入 frpc（https2http 插件）

关键：**证书路径用软链指向 letsencrypt 目录**，这样续期后自动生效，不用改配置。

```bash
ln -sf /etc/letsencrypt/live/example.com/fullchain.pem /usr/local/frp/certs/merged.fullchain.pem
ln -sf /etc/letsencrypt/live/example.com/privkey.pem   /usr/local/frp/certs/merged.privkey.pem
```

`frpc.toml` 中每个 https 代理：

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

重启：`systemctl restart frpc`

### 7. 自动续期：systemd timer + deploy-hook

**deploy-hook** `/usr/local/frp/deploy-frpc-cert.sh`（`chmod +x`）：

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

**注册 hook** —— 写进 renewal 配置（`renew_hook`）：

```bash
sed -i '/^\[renewalparams\]/a renew_hook = /usr/local/frp/deploy-frpc-cert.sh' \
  /etc/letsencrypt/renewal/example.com.conf
```

**systemd timer** `/etc/systemd/system/certbot-renew.service`：

```ini
[Unit]
Description=Certbot renewal (DNS-01)
After=network-online.target
Wants=network-online.target

[Service]
Type=oneshot
ExecStart=/usr/bin/certbot renew --quiet --no-random-sleep-on-renew
```

`/etc/systemd/system/certbot-renew.timer`：

```ini
[Unit]
Description=Twice-daily certbot renewal check

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

### 8. 端到端演练

```bash
# 模拟续期（不触发 deploy-hook）
certbot renew --dry-run

# 手动触发 hook 验证重启逻辑
RENEWED_LINEAGE=/etc/letsencrypt/live/example.com \
RENEWED_DOMAINS="example.com blog.example.com" \
  /usr/local/frp/deploy-frpc-cert.sh
cat /var/log/letsencrypt/deploy-frpc.log
```

## 迁移到新公网服务器

只改内网机器上 `frpc.toml` 的 `serverAddr`，然后 `systemctl restart frpc`。证书签发逻辑完全不动 —— 这就是本方案的核心价值。

## 清理旧的"公网机 certbot"（如果之前是那样部署的）

在公网机上：

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

**关键坑**：删完 `/etc/letsencrypt` 后，如果公网机 nginx 里还有 `ssl_certificate /etc/letsencrypt/live/...` 的引用，`nginx -t` 会直接失败，**下次 nginx 重启/reload 起不来**。

排查：

```bash
nginx -t                                  # 会明确报哪个证书文件找不到
grep -rn 'ssl_certificate' /etc/nginx/    # 找出所有引用
```

处理：把已废弃的 vhost 配置删掉或备份移走。**注意先确认该 vhost 是否真在监听** —— 常见情况是 `listen 443` 但 443 实际被 frps 占着，nginx 根本没跑，那些配置是纯死配置。用 `ss -lntp | grep 443` 确认端口归属。

## Pitfalls 汇总

| 问题 | 原因 | 解决 |
|------|------|------|
| `ambiguous option: --dns-aliyun` | 参数前缀有歧义 | 用 `--authenticator dns-aliyun` |
| `Property "dns_aliyun_access_key" not found` | 凭证 key 名写法不对 | 去掉 `certbot_dns_aliyun:` 前缀，用 `dns_aliyun_access_key` |
| systemd unit 报 certbot 不存在 | pip 装在 `/usr/bin/` 而非 `/usr/local/bin/` | `which certbot` 确认真实路径 |
| 续期后线上证书没变 | deploy-hook 没注册或没重启 frpc | 检查 renewal.conf 的 `renew_hook`，看 `/var/log/letsencrypt/deploy-frpc.log` |
| 删证书后 `nginx -t` 失败 | nginx 仍引用已删的证书路径 | 删掉废弃 vhost 配置；先用 `ss -lntp` 确认端口是否真被 nginx 占用 |
| 证书验证一直等 | DNS TXT 传播需要时间，默认 30s | 正常；可用 `--dns-aliyun-propagation-seconds` 调整 |
| Python 3.9 警告 | certbot 4.x 即将弃用 3.9 | 仅警告，可忽略；有条件升 Python |

## 换其他 DNS 服务商

把 `certbot-dns-aliyun` 换成对应插件，凭证文件里的 key 名和安装方式同理：

- Cloudflare: `certbot-dns-cloudflare`，`dns_cloudflare_api_token`
- DNSPod: `certbot-dns-dnspod`，`dns_dnspod_api_id` / `dns_dnspod_api_key`
- 通用 RFC2136: `certbot-dns-rfc2136`

## 验证清单

- [ ] `certbot plugins` 能看到 dns-aliyun
- [ ] `certbot certonly ... --dry-run` 成功
- [ ] `openssl x509` 看到所有 SAN 域名
- [ ] 公网 `openssl s_client -connect <域名>:443 -servername <域名>` 显示新证书
- [ ] `Verify return code: 0 (ok)`
- [ ] `systemctl is-enabled certbot-renew.timer` 为 enabled
- [ ] 手动触发 deploy-hook 后 frpc 正常重启
- [ ] `curl -sI https://<域名>` 返回 200/301/403（非 SSL 错误）
