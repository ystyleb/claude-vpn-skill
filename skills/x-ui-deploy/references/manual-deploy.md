# 部署执行蓝图

Claude 按此文件的步骤逐条在服务器上执行命令。

> **执行模式**：AI agent 的每条远程命令都是一次独立的 SSH 调用，shell 变量不会跨调用保留。因此步骤 0 把所有变量持久化到服务器上的 env 文件，后续每个步骤统一用以下模式执行（一个步骤的多条命令可以合并在同一次 SSH 调用里）：
>
> ```bash
> ssh -p {SSH_PORT} [-i {KEY_PATH}] {USER}@{SERVER_IP} 'source /root/.secrets/deploy.env && <步骤命令>'
> ```

## 步骤 0：写入部署变量文件

第一条 SSH 命令：把用户提供的值写入 `/root/.secrets/deploy.env`（把示例值替换为用户实际值后执行）：

```bash
mkdir -p /root/.secrets && chmod 700 /root/.secrets
cat > /root/.secrets/deploy.env << "ENVEOF"
# === 用户提供的值 ===
ROOT_DOMAIN="example.com"
SUBDOMAIN_PREFIX="node1"
EMAIL="you@email.com"
SSH_PORT="22"

# Cloudflare 认证（二选一，删掉不用的那组）
# 用 export：acme.sh 的 dns_cf 钩子和步骤 16 的 curl 都是子进程，必须从环境继承凭据
# 方式一：API Token（推荐，最小权限）
export CF_Token="your_api_token"
# 方式二：Global API Key
# export CF_Key="your_global_api_key"
# export CF_Email="your_cf_email"

# === 自动拼接（source 时求值，无需手动改） ===
DOMAIN="${SUBDOMAIN_PREFIX}.${ROOT_DOMAIN}"
XUI_PORT="54321"
ENVEOF
chmod 600 /root/.secrets/deploy.env
```

> heredoc 分隔符带引号（`"ENVEOF"`），`DOMAIN="${SUBDOMAIN_PREFIX}..."` 以字面量写入文件、每次 `source` 时才求值——派生变量永远跟随基础变量。
>
> 后续所有步骤先 `source /root/.secrets/deploy.env` 再执行命令，变量自动可用。

---

## 步骤 1：检查系统环境

```bash
whoami  # 必须是 root
cat /etc/os-release | head -3  # 必须是 Debian/Ubuntu
ss -tlnp | grep -E ':80 |:443 '  # 检查端口占用
```

如果 80/443 已被占用，提醒用户现有服务可能被覆盖，确认后再继续。

## 步骤 2：安装依赖

```bash
apt update && apt install -y nginx ufw fail2ban curl wget openssl sqlite3 socat cron jq python3-bcrypt
```

> `python3-bcrypt` 用于后续生成面板密码哈希。如果包不存在，后续会 fallback 到 `pip3 install bcrypt`。`jq` 用于步骤 16 解析 Cloudflare API 响应。

## 步骤 3：生成安全参数

```bash
mkdir -p /root/.secrets && chmod 700 /root/.secrets

# XHTTP 路径（16 字符 hex，短路径容易被扫描发现）
openssl rand -hex 8 > /root/.secrets/ws_path.txt

# 客户端 UUID
cat /proc/sys/kernel/random/uuid > /root/.secrets/vless_uuid.txt

# X-UI 面板凭据
echo "admin_$(openssl rand -hex 4)" > /root/.secrets/xui_username.txt
openssl rand -base64 18 > /root/.secrets/xui_password.txt

# 伪装站点公司名（随机英文，避免所有部署都叫同一个名字被 GFW 指纹识别）
DECOY_ADJS=(Alpine Summit Ridge Vertex Meridian Cascade Atlas Zenith Stratum Lumina Crestwood Northwind Silverline Boreal Solstice)
DECOY_NOUNS=(Systems Solutions Labs Partners Group Dynamics Ventures Networks Analytics Consulting)
echo "${DECOY_ADJS[RANDOM % ${#DECOY_ADJS[@]}]} ${DECOY_NOUNS[RANDOM % ${#DECOY_NOUNS[@]}]}" > /root/.secrets/decoy_name.txt

chmod 600 /root/.secrets/*.txt
```

把生成的值注册进 deploy.env（以 `$(cat ...)` 字面量追加，每次 source 时从 `.txt` 文件动态读取——以后轮换 WS_PATH/UUID 等只需替换对应 `.txt` 文件，deploy.env 不用动；但 CF 凭据轮换仍需编辑 deploy.env 本身）：

```bash
cat >> /root/.secrets/deploy.env << "ENVEOF"

# === 步骤 3 生成的安全参数（source 时动态读取） ===
WS_PATH="$(cat /root/.secrets/ws_path.txt)"
UUID="$(cat /root/.secrets/vless_uuid.txt)"
XUI_USER="$(cat /root/.secrets/xui_username.txt)"
XUI_PASS="$(cat /root/.secrets/xui_password.txt)"
DECOY_NAME="$(cat /root/.secrets/decoy_name.txt)"
ENVEOF

source /root/.secrets/deploy.env
echo "WS_PATH: $WS_PATH"
echo "UUID: $UUID"
echo "XUI_USER: $XUI_USER"
echo "XUI_PASS: $XUI_PASS"
echo "DECOY_NAME: $DECOY_NAME"
```

## 步骤 4：安装 acme.sh 并申请 SSL 证书

```bash
curl https://get.acme.sh | sh -s email="$EMAIL"
```

CF 凭据已在步骤 0 的 deploy.env 里用 `export` 定义，`source` 后即在环境中，acme.sh 的 `dns_cf` 子进程能直接继承——无需在此重复 export。

申请并安装证书（显式指定 Let's Encrypt——acme.sh 默认 CA 是 ZeroSSL，偶发签发失败率更高）：

```bash
~/.acme.sh/acme.sh --issue -d "$ROOT_DOMAIN" -d "*.$ROOT_DOMAIN" --dns dns_cf --server letsencrypt --keylength ec-256

mkdir -p /root/cert
~/.acme.sh/acme.sh --install-cert -d "$ROOT_DOMAIN" \
  --cert-file "/root/cert/$ROOT_DOMAIN.cer" \
  --key-file "/root/cert/$ROOT_DOMAIN.key" \
  --fullchain-file /root/cert/fullchain.cer \
  --ca-file /root/cert/ca.cer \
  --reloadcmd "systemctl reload nginx"

chmod 600 /root/cert/*.key
```

> 如果证书申请失败，通常是 Cloudflare API 凭据不对或权限不足。让用户重新检查。

## 步骤 5：创建伪装站点

**防指纹**：每次部署的伪装站"公司名"都随机生成（见步骤 3 的 `DECOY_NAME`），避免多台 VPS 共享同一特征被批量识别。模板是英文的，避免中文命名在 `.com`/`.uk` 这类域名上看起来突兀。

```bash
mkdir -p "/var/www/$DOMAIN"
cat > "/var/www/$DOMAIN/index.html" << SITEEOF
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>${DECOY_NAME}</title>
    <meta name="description" content="Professional digital solutions and consulting services.">
    <style>
        * { margin: 0; padding: 0; box-sizing: border-box; }
        body { font-family: -apple-system, BlinkMacSystemFont, "Segoe UI", Roboto, "Helvetica Neue", Arial, sans-serif; color: #1a1a2e; line-height: 1.6; }
        .nav { background: #fff; padding: 1rem 2rem; border-bottom: 1px solid #e8e8e8; display: flex; justify-content: space-between; align-items: center; }
        .nav-brand { font-size: 1.3rem; font-weight: 600; color: #2563eb; text-decoration: none; }
        .nav-links { display: flex; gap: 2rem; list-style: none; }
        .nav-links a { text-decoration: none; color: #555; font-size: 0.95rem; }
        .hero { background: linear-gradient(135deg, #667eea 0%, #764ba2 100%); color: white; padding: 5rem 2rem; text-align: center; }
        .hero h1 { font-size: 2.5rem; font-weight: 700; margin-bottom: 1rem; }
        .hero p { font-size: 1.2rem; opacity: 0.9; max-width: 600px; margin: 0 auto 2rem; }
        .btn { display: inline-block; background: #fff; color: #764ba2; padding: 0.8rem 2rem; border-radius: 6px; text-decoration: none; font-weight: 600; }
        .features { display: grid; grid-template-columns: repeat(auto-fit, minmax(280px, 1fr)); gap: 2rem; padding: 4rem 2rem; max-width: 1100px; margin: 0 auto; }
        .feature { text-align: center; padding: 2rem; }
        .feature h3 { margin: 1rem 0 0.5rem; font-size: 1.2rem; }
        .feature p { color: #666; font-size: 0.95rem; }
        .icon { font-size: 2.5rem; }
        .footer { background: #f8f9fa; padding: 2rem; text-align: center; color: #888; font-size: 0.85rem; border-top: 1px solid #e8e8e8; }
    </style>
</head>
<body>
    <nav class="nav">
        <a href="#" class="nav-brand">${DECOY_NAME}</a>
        <ul class="nav-links">
            <li><a href="#">Services</a></li>
            <li><a href="#">About</a></li>
            <li><a href="#">Contact</a></li>
        </ul>
    </nav>
    <section class="hero">
        <h1>Digital Solutions for Modern Business</h1>
        <p>We help companies transform their operations with cutting-edge technology and data-driven strategies.</p>
        <a href="#" class="btn">Learn More</a>
    </section>
    <section class="features">
        <div class="feature"><div class="icon">&#9729;</div><h3>Cloud Infrastructure</h3><p>Scalable and secure cloud solutions tailored to your business needs.</p></div>
        <div class="feature"><div class="icon">&#9881;</div><h3>Process Automation</h3><p>Streamline workflows and reduce operational costs with smart automation.</p></div>
        <div class="feature"><div class="icon">&#128200;</div><h3>Data Analytics</h3><p>Turn your data into actionable insights with advanced analytics platforms.</p></div>
    </section>
    <footer class="footer">&copy; $(date +%Y) ${DECOY_NAME}. All rights reserved.</footer>
</body>
</html>
SITEEOF
```

## 步骤 6：加固 Nginx 全局配置

```bash
sed -i 's/ssl_protocols TLSv1 TLSv1.1 TLSv1.2 TLSv1.3;/ssl_protocols TLSv1.2 TLSv1.3;/' /etc/nginx/nginx.conf
sed -i 's/# server_tokens off;/server_tokens off;/' /etc/nginx/nginx.conf
```

## 步骤 7：配置 Nginx 站点

nginx 1.25.1 起 `listen ... http2` 写法已 deprecated（改用独立的 `http2 on;` 指令），先按版本选择语法：

```bash
NGINX_VER=$(nginx -v 2>&1 | grep -oE '[0-9]+\.[0-9]+\.[0-9]+')
if dpkg --compare-versions "$NGINX_VER" ge 1.25.1; then
  LISTEN_443='listen 443 ssl;
    listen [::]:443 ssl;
    http2 on;'
else
  LISTEN_443='listen 443 ssl http2;
    listen [::]:443 ssl http2;'
fi
```

```bash
cat > "/etc/nginx/sites-available/$DOMAIN" << NGINXEOF
server {
    listen 80;
    listen [::]:80;
    server_name $DOMAIN;
    return 301 https://\$server_name\$request_uri;
}

server {
    $LISTEN_443
    server_name $DOMAIN;

    ssl_certificate /root/cert/fullchain.cer;
    ssl_certificate_key /root/cert/$ROOT_DOMAIN.key;
    ssl_protocols TLSv1.2 TLSv1.3;
    ssl_ciphers 'ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA256:ECDHE-ECDSA-AES256-GCM-SHA384:ECDHE-RSA-AES256-GCM-SHA384:ECDHE-ECDSA-CHACHA20-POLY1305:ECDHE-RSA-CHACHA20-POLY1305';
    ssl_prefer_server_ciphers on;
    ssl_session_cache shared:SSL:10m;
    ssl_session_timeout 10m;

    add_header Strict-Transport-Security "max-age=31536000; includeSubDomains; preload" always;
    add_header X-Frame-Options DENY always;
    add_header X-Content-Type-Options nosniff always;

    server_tokens off;
    root /var/www/$DOMAIN;
    index index.html;

    # XHTTP 代理 → Xray
    location /$WS_PATH {
        proxy_pass http://127.0.0.1:10000;
        proxy_http_version 1.1;
        proxy_set_header Host \$host;
        proxy_set_header X-Real-IP \$remote_addr;
        proxy_set_header X-Forwarded-For \$proxy_add_x_forwarded_for;
        proxy_read_timeout 86400s;
        proxy_send_timeout 86400s;

        # XHTTP mode=auto 时也需要支持 WebSocket upgrade
        proxy_set_header Upgrade \$http_upgrade;
        proxy_set_header Connection "upgrade";

        # 禁用缓冲，确保流式传输
        proxy_buffering off;
        proxy_cache off;
    }

    location / { try_files \$uri \$uri/ =404; }
    location ~ /\. { deny all; }
}
NGINXEOF
```

> heredoc 不带引号，`$DOMAIN`、`$ROOT_DOMAIN`、`$WS_PATH`、`$LISTEN_443` 会被 bash 展开为实际值。Nginx 变量（`$server_name`、`$host` 等）用 `\$` 转义以保留。
>
> ⚠️ 上面的版本判断和 heredoc 必须放在**同一次 SSH 调用**里执行（`$LISTEN_443` 是普通 shell 变量，不在 deploy.env 中）。

启用配置：

```bash
ln -sf "/etc/nginx/sites-available/$DOMAIN" /etc/nginx/sites-enabled/
rm -f /etc/nginx/sites-enabled/default
nginx -t && systemctl reload nginx
```

## 步骤 8：配置防火墙

```bash
ufw default deny incoming
ufw default allow outgoing
ufw allow "$SSH_PORT/tcp"
ufw allow 80/tcp
ufw allow 443/tcp
ufw deny "$XUI_PORT/tcp"
echo "y" | ufw enable
```

## 步骤 9：配置 Fail2Ban

```bash
cat > /etc/fail2ban/jail.local << JAILEOF
[DEFAULT]
bantime = 3600
findtime = 600
maxretry = 5
banaction = ufw

[sshd]
enabled = true
port = $SSH_PORT
logpath = /var/log/auth.log
maxretry = 3
bantime = 7200
JAILEOF

systemctl enable fail2ban && systemctl restart fail2ban
```

## 步骤 10：安装 3X-UI

Pin 到指定版本安装（master 分支的安装脚本和数据库 schema 随时可能变化，导致后续步骤的交互处理和 sqlite 写入失效）：

```bash
XUI_VERSION="v3.3.0"   # 本 skill 验证过的版本；升级前先确认步骤 11-13 的表结构仍兼容
bash <(curl -Ls "https://raw.githubusercontent.com/MHSanaei/3x-ui/$XUI_VERSION/install.sh") "$XUI_VERSION"
```

> **交互处理**：安装器会依次要求输入：
> 1. 是否继续安装 → 输入 `y`
> 2. 用户名 → 随意输入（如 `temp`），后续会通过数据库覆盖
> 3. 密码 → 随意输入（如 `temp`），后续会通过数据库覆盖
> 4. 端口 → 输入 `54321`
>
> 使用 **MHSanaei/3x-ui**，不要用已停维的 vaxilu/x-ui。

等待安装完成，确认数据库文件存在、且表结构符合步骤 11-13 的预期（防上游 schema 变更导致静默写错）：

```bash
ls -la /etc/x-ui/x-ui.db
sqlite3 /etc/x-ui/x-ui.db ".schema settings" | grep -q '"key"\|key ' && echo "settings 表 OK"
sqlite3 /etc/x-ui/x-ui.db ".schema users" | grep -q 'username' && echo "users 表 OK"
sqlite3 /etc/x-ui/x-ui.db ".schema inbounds" | grep -q 'stream_settings' && echo "inbounds 表 OK"
```

任一项不输出 OK → 上游 schema 已变，停下来人工核对表结构，不要盲跑步骤 11-13。

## 步骤 11：配置 3X-UI 面板

> ⚠️ 本步骤整个代码块必须在**同一次 SSH 调用**里执行（`HASHED`、`NEW_SECRET` 是步骤内临时变量，拆开执行会丢失，导致 sqlite 写入空值）。

```bash
systemctl stop x-ui

# 面板仅 localhost 访问
sqlite3 /etc/x-ui/x-ui.db "INSERT OR REPLACE INTO settings (key, value) VALUES ('webListen', '127.0.0.1');"
sqlite3 /etc/x-ui/x-ui.db "INSERT OR REPLACE INTO settings (key, value) VALUES ('webPort', '$XUI_PORT');"

# 更新面板凭据（bcrypt 密码哈希）
# 确保 bcrypt 模块可用（Debian 12+/Ubuntu 24.04+ 的 pip 受 PEP 668 管控，需要 --break-system-packages）
python3 -c "import bcrypt" 2>/dev/null || pip3 install bcrypt -q 2>/dev/null || pip3 install bcrypt -q --break-system-packages
# 注意：openssl rand -base64 生成的密码仅含 A-Za-z0-9+/= 字符，不含单引号，此处安全
HASHED=$(python3 -c "
import bcrypt
password = '$XUI_PASS'
print(bcrypt.hashpw(password.encode(), bcrypt.gensalt(10)).decode())
")
sqlite3 /etc/x-ui/x-ui.db "UPDATE users SET username='$XUI_USER', password='$HASHED' WHERE id=1;"

# 生成新 secret
NEW_SECRET=$(openssl rand -hex 16)
sqlite3 /etc/x-ui/x-ui.db "UPDATE settings SET value='$NEW_SECRET' WHERE key='secret';"

# 禁用订阅端口（默认会在公网暴露！）
sqlite3 /etc/x-ui/x-ui.db "INSERT OR REPLACE INTO settings (key, value) VALUES ('subEnable', 'false');"
sqlite3 /etc/x-ui/x-ui.db "INSERT OR REPLACE INTO settings (key, value) VALUES ('subListen', '127.0.0.1');"
```

## 步骤 12：设置 Xray 模板

模板必须完整，否则 outbounds 为 null 导致流量无法出站：

```bash
sqlite3 /etc/x-ui/x-ui.db "INSERT OR REPLACE INTO settings (key, value) VALUES ('xrayTemplateConfig', '{
  \"log\": {\"access\": \"none\", \"dnsLog\": false, \"loglevel\": \"warning\"},
  \"dns\": {\"servers\": [\"8.8.8.8\", \"1.1.1.1\"]},
  \"routing\": {
    \"domainStrategy\": \"IPIfNonMatch\",
    \"rules\": [
      {\"type\": \"field\", \"inboundTag\": [\"api\"], \"outboundTag\": \"api\"},
      {\"type\": \"field\", \"outboundTag\": \"blocked\", \"protocol\": [\"bittorrent\"]}
    ]
  },
  \"outbounds\": [
    {\"tag\": \"direct\", \"protocol\": \"freedom\", \"settings\": {}},
    {\"tag\": \"blocked\", \"protocol\": \"blackhole\", \"settings\": {}}
  ],
  \"policy\": {\"levels\": {\"0\": {\"statsUserDownlink\": true, \"statsUserUplink\": true}}, \"system\": {\"statsInboundDownlink\": true, \"statsInboundUplink\": true}},
  \"api\": {\"tag\": \"api\", \"services\": [\"HandlerService\", \"LoggerService\", \"StatsService\"]},
  \"stats\": {}
}');"
```

## 步骤 13：创建 VLESS 入站

> ⚠️ 本步骤整个代码块必须在**同一次 SSH 调用**里执行（`SETTINGS`/`STREAM`/`SNIFFING` 是步骤内临时变量）。

```bash
SETTINGS="{\"clients\":[{\"id\":\"$UUID\",\"flow\":\"\",\"email\":\"default-user\",\"limitIp\":0,\"totalGB\":0,\"expiryTime\":0,\"enable\":true,\"tgId\":\"\",\"subId\":\"\",\"reset\":0}],\"decryption\":\"none\",\"fallbacks\":[]}"
STREAM="{\"network\":\"xhttp\",\"security\":\"none\",\"xhttpSettings\":{\"path\":\"/$WS_PATH\",\"host\":\"$DOMAIN\",\"mode\":\"auto\"}}"
SNIFFING="{\"enabled\":true,\"destOverride\":[\"http\",\"tls\",\"quic\"],\"metadataOnly\":false,\"routeOnly\":true}"

sqlite3 /etc/x-ui/x-ui.db "INSERT INTO inbounds (user_id, up, down, total, all_time, remark, enable, expiry_time, listen, port, protocol, settings, stream_settings, tag, sniffing) VALUES (1, 0, 0, 0, 0, 'VLESS-XHTTP-TLS-CF', 1, 0, '127.0.0.1', 10000, 'vless', '$SETTINGS', '$STREAM', 'inbound-10000', '$SNIFFING');"

systemctl start x-ui
sleep 3
```

## 步骤 14：健康检查

逐项检查，全部通过才算部署成功：

```bash
# 服务状态
systemctl is-active nginx x-ui fail2ban

# 端口监听
ss -tlnp | grep -E ':80 |:443 |:10000 |:54321 '

# Xray 监听在 localhost
ss -tlnp | grep '127.0.0.1:10000'

# X-UI 面板仅 localhost
ss -tlnp | grep "127.0.0.1:$XUI_PORT"

# Xray outbounds 不为 null
python3 -c "
import json
cfg = json.load(open('/usr/local/x-ui/bin/config.json'))
print('Outbounds:', cfg.get('outbounds'))
print('DNS:', cfg.get('dns'))
"

# 防火墙状态
ufw status

# SSL 证书有效期
openssl x509 -in /root/cert/fullchain.cer -noout -enddate
```

如果任一项不通过，参考 `troubleshooting.md` 对应的诊断和修复方法。

## 步骤 15：保存配置并输出结果

```bash
SERVER_IP=$(curl -4 -s --max-time 10 ifconfig.me)  # 强制 IPv4：A 记录和客户端直连都用 IPv4，双栈机 ifconfig.me 可能返回 IPv6

VLESS_LINK="vless://${UUID}@${DOMAIN}:443?encryption=none&security=tls&sni=${DOMAIN}&type=xhttp&host=${DOMAIN}&path=%2F${WS_PATH}&fp=chrome#VLESS-XHTTP-TLS-CF"

cat > /root/vpn-config.txt << CFGEOF
========================================
  VPN Configuration - $DOMAIN
  Generated: $(date -u '+%Y-%m-%d %H:%M:%S UTC')
========================================

## Architecture
Client → Cloudflare CDN (443) → Nginx (TLS) → Xray (127.0.0.1:10000) → Internet

## Domain
  Domain:     $DOMAIN
  Server IP:  $SERVER_IP (hidden behind Cloudflare)

## VLESS Client Link (copy as ONE LINE, no line breaks!)
$VLESS_LINK

## Manual Client Config
  Protocol:   VLESS
  Address:    $DOMAIN
  Port:       443 (NOT 10000)
  UUID:       $UUID
  Encryption: none
  Transport:  XHTTP
  Path:       /$WS_PATH
  Host:       $DOMAIN
  TLS:        Enabled (MUST be on)
  SNI:        $DOMAIN (MUST be set)
  Fingerprint: chrome

## X-UI Panel Access (SSH tunnel ONLY)
  SSH Tunnel: ssh -L $XUI_PORT:localhost:$XUI_PORT root@$SERVER_IP
  URL:        http://localhost:$XUI_PORT
  Username:   $XUI_USER
  Password:   $XUI_PASS

## Adding New Clients
  Do NOT create new inbounds! Add clients to existing inbound:
  Panel → Inbound list → VLESS-XHTTP-TLS-CF → "+" → Add Client
  Then generate link using this template:
  vless://<UUID>@$DOMAIN:443?encryption=none&security=tls&sni=$DOMAIN&type=xhttp&host=$DOMAIN&path=%2F$WS_PATH&fp=chrome#<NAME>
  WARNING: Panel-exported links have WRONG port, TLS, and transport settings!

## Security
  - UFW: only $SSH_PORT/80/443 open
  - X-UI: localhost only (127.0.0.1:$XUI_PORT)
  - Xray: localhost only (127.0.0.1:10000)
  - Fail2Ban: SSH $SSH_PORT, 3 attempts → 2hr ban
  - Subscription port: disabled

## Key Files
  - Secrets:     /root/.secrets/
  - Nginx Site:  /etc/nginx/sites-available/$DOMAIN
  - Fail2Ban:    /etc/fail2ban/jail.local
  - X-UI DB:     /etc/x-ui/x-ui.db
  - Xray Config: /usr/local/x-ui/bin/config.json (auto-generated, don't edit)
  - Certs:       /root/cert/

## Optional: 直连主力 + CF 兜底（进阶，跑通基础后再看）
  references/cf-dns-strategy.md
CFGEOF

chmod 600 /root/vpn-config.txt
cat /root/vpn-config.txt
```

将输出中的 VLESS 链接、面板凭据、SSH 隧道命令整理后展示给用户。

## 步骤 16：Cloudflare API 自动配置

部署用的 CF 凭据（步骤 4 签证书时已验证可用）同样可以完成 SKILL.md 第三步的全部配置——DNS A 记录、SSL 模式、最低 TLS 版本、WebSocket。在服务器上执行（curl + jq 已在步骤 2 安装）。

> ⚠️ 本步骤整个代码块在**同一次 SSH 调用**里执行（`AUTH`/`ZONE_ID`/`REC_ID` 是步骤内临时变量）。`exit 1` 只退出远程 shell——执行后检查输出有没有 `FAIL:` 前缀，有则转手动配置，不要继续当成功处理。

```bash
API="https://api.cloudflare.com/client/v4"
if [ -n "$CF_Token" ]; then
  AUTH=(-H "Authorization: Bearer $CF_Token")
else
  AUTH=(-H "X-Auth-Email: $CF_Email" -H "X-Auth-Key: $CF_Key")
fi

# 1. 获取 Zone ID
ZONE_ID=$(curl -s "${AUTH[@]}" "$API/zones?name=$ROOT_DOMAIN" | jq -r '.result[0].id')
if [ -z "$ZONE_ID" ] || [ "$ZONE_ID" = "null" ]; then
  echo "FAIL: 找不到 zone（凭据权限不足或域名不在此账户），转手动配置"; exit 1
fi

SERVER_IP=$(curl -4 -s --max-time 10 ifconfig.me)  # 强制 IPv4：A 记录和客户端直连都用 IPv4，双栈机 ifconfig.me 可能返回 IPv6

# 2. A 记录（已存在则更新，否则创建），开橙色云
REC_ID=$(curl -s "${AUTH[@]}" "$API/zones/$ZONE_ID/dns_records?type=A&name=$DOMAIN" | jq -r '.result[0].id')
BODY="{\"type\":\"A\",\"name\":\"$SUBDOMAIN_PREFIX\",\"content\":\"$SERVER_IP\",\"ttl\":1,\"proxied\":true}"
if [ -z "$REC_ID" ] || [ "$REC_ID" = "null" ]; then
  curl -s -X POST "${AUTH[@]}" -H "Content-Type: application/json" \
    "$API/zones/$ZONE_ID/dns_records" -d "$BODY" | jq '{step:"dns_record", success, errors}'
else
  curl -s -X PUT "${AUTH[@]}" -H "Content-Type: application/json" \
    "$API/zones/$ZONE_ID/dns_records/$REC_ID" -d "$BODY" | jq '{step:"dns_record", success, errors}'
fi

# 3. SSL 模式 → Full (strict)
curl -s -X PATCH "${AUTH[@]}" -H "Content-Type: application/json" \
  "$API/zones/$ZONE_ID/settings/ssl" -d '{"value":"strict"}' | jq '{step:"ssl_mode", success, errors}'

# 4. 最低 TLS 版本 → 1.2
curl -s -X PATCH "${AUTH[@]}" -H "Content-Type: application/json" \
  "$API/zones/$ZONE_ID/settings/min_tls_version" -d '{"value":"1.2"}' | jq '{step:"min_tls", success, errors}'

# 5. WebSocket → 开启（XHTTP mode=auto 在 CF 链路上可能升级到 WS 通道，关闭会导致部分客户端不通）
curl -s -X PATCH "${AUTH[@]}" -H "Content-Type: application/json" \
  "$API/zones/$ZONE_ID/settings/websockets" -d '{"value":"on"}' | jq '{step:"websockets", success, errors}'
```

逐项检查输出的 `success` 字段：

- **全部 `true`** → CF 配置完成，告知用户无需再去控制台手动操作
- **任一项 `false`** → 把对应项的 `errors` 展示给用户，并给出 SKILL.md 第三步的手动配置表作为 fallback。最常见原因：API Token 缺 **Zone Settings:Edit** 权限（settings 三项会 403，但 DNS 记录能成功——此时只需手动改 SSL/TLS 设置）

> 等待 DNS 生效后（通常 1-2 分钟），可验证：`curl -s -o /dev/null -w '%{http_code}' https://$DOMAIN` 返回 200 即整条链路打通。
