# 日常维护

## 服务管理

```bash
# 查看所有服务状态
systemctl is-active nginx x-ui fail2ban

# 重启服务
systemctl restart nginx
systemctl restart x-ui

# 查看日志
journalctl -u x-ui --since "1 hour ago"
tail -30 /var/log/nginx/error.log
```

---

## 证书管理

```bash
# 查看证书有效期
openssl x509 -in /root/cert/fullchain.cer -noout -enddate

# 查看 acme.sh 定时任务（自动续期）
crontab -l | grep acme

# 手动强制续期
~/.acme.sh/acme.sh --renew -d $(ls ~/.acme.sh/ | grep _ecc | head -1 | sed 's/_ecc//') --force --ecc
```

证书由 acme.sh 自动续期（每 60 天），续期后自动 reload Nginx。正常情况下不需要手动操作。

---

## 添加新客户端

**不要新建入站！** 在现有入站里「添加客户端」：

1. 通过 SSH 隧道进面板：
   ```bash
   ssh -L 54321:localhost:54321 user@SERVER_IP
   # 浏览器访问 http://localhost:54321
   ```
2. 入站列表 → 点 `+` 展开客户端列表
3. 点「添加客户端」→ 填备注 → UUID 自动生成 → 保存
4. 用以下模板拼链接发给用户：
   ```
   vless://<新UUID>@DOMAIN:443?encryption=none&security=tls&sni=DOMAIN&type=xhttp&host=DOMAIN&path=%2FWS_PATH&fp=chrome#用户备注
   ```

> 新建入站会导致：绕过 Nginx 伪装、端口暴露公网、无 TLS 保护。

---

## 面板修改后验证

面板修改入站后，Xray 运行配置可能不同步：

```bash
# 检查 config.json 是否正确
python3 -c "
import json
cfg = json.load(open('/usr/local/x-ui/bin/config.json'))
print('Outbounds:', cfg.get('outbounds'))
print('DNS:', cfg.get('dns'))
print('Clients:', len(cfg['inbounds'][0]['settings']['clients']))
"

# 不确定时直接重启
systemctl restart x-ui
```

---

## 安全加固清单

### 已由部署流程配置

- [x] UFW 防火墙仅开放 SSH/80/443
- [x] X-UI 面板仅 localhost 访问
- [x] 订阅端口已禁用
- [x] Fail2Ban 防暴力破解（SSH 3 次失败封 2 小时）
- [x] Nginx 隐藏版本号
- [x] TLSv1/TLSv1.1 已禁用
- [x] HSTS / X-Frame-Options / X-Content-Type-Options 安全头部
- [x] Cloudflare 隐藏真实 IP

### 建议额外配置

- [ ] SSH 密钥认证（禁用密码登录）
- [ ] 定期更换 XHTTP 路径
- [ ] 定期更换 Cloudflare API Token

### 敏感文件权限

```bash
chmod 700 /root/.secrets/
chmod 600 /root/.secrets/*.txt
chmod 600 /root/vpn-config.txt
```

---

## 配置文件位置速查

| 文件 | 路径 |
|------|------|
| 密钥目录 | `/root/.secrets/` |
| 部署变量文件 | `/root/.secrets/deploy.env`（所有远程命令 `source` 它获取变量；含 CF 凭据，chmod 600）|
| XHTTP 路径 | `/root/.secrets/ws_path.txt` |
| 客户端 UUID | `/root/.secrets/vless_uuid.txt` |
| X-UI 凭据 | `/root/.secrets/xui_username.txt` / `xui_password.txt` |
| 伪装站公司名 | `/root/.secrets/decoy_name.txt`（部署时随机生成的英文名，改了要同步改 `/var/www/$DOMAIN/index.html`）|
| Nginx 全局 | `/etc/nginx/nginx.conf` |
| Nginx 站点 | `/etc/nginx/sites-available/<子域名>` |
| Fail2Ban | `/etc/fail2ban/jail.local` |
| X-UI 数据库 | `/etc/x-ui/x-ui.db` |
| Xray 运行配置 | `/usr/local/x-ui/bin/config.json`（自动生成，勿手动改） |
| SSL 证书 | `/root/cert/` |
| acme.sh | `~/.acme.sh/<根域名>_ecc/` |
| 伪装网站 | `/var/www/<子域名>/index.html` |
| 配置汇总 | `/root/vpn-config.txt` |

---

## 定期维护计划

### 每月

- [ ] 检查 SSL 证书有效期
- [ ] 查看 Fail2Ban 拦截记录：`fail2ban-client status sshd`
- [ ] 更新系统软件包：`apt update && apt upgrade`
- [ ] 看一眼 VPS 控制台的当月流量用量（KiwiVM/Vultr/GCP 等首页通常都有），接近上限时准备应对

### 每季度

- [ ] 更换 XHTTP 路径（同步更新 Nginx + X-UI + 客户端）
- [ ] 备份配置文件
- [ ] 审查防火墙规则

### 每年

- [ ] 更换 Cloudflare API Token
- [ ] 审查 X-UI 用户列表
- [ ] 检查 Xray 版本更新和弃用通知
