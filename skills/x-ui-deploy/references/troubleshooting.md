# 故障排查

## 核心原则：先查客户端，再查服务端

大多数"连不上"的问题出在客户端配置（链接断行、缺参数），不要急着改服务端。

## 整体故障定位

以伪装站点是否能访问作为分水岭：

| 症状 | 大概率原因 | 下一步 |
|---|---|---|
| 客户端连不上，但浏览器能打开 `https://{DOMAIN}` | VPN 链路问题（path / UUID / 客户端配置） | 走下方"客户端"和"服务端"两步排查 |
| 客户端连不上，伪装站也打不开 | VPS 故障 / IP 被封 / 防火墙改动 | 去 VPS 控制台看实例状态、流量上限、最近改动 |

> 如果你按 `cf-dns-strategy.md` 配了直连 + CF 兜底双节点，节点切换是 5 秒内生效的客户端单边操作，先切再排查。


---

## 第一步：检查客户端链接

按以下清单逐项排查：

- [ ] 链接是否完整一行？（复制时最容易在 path 参数处断行）
- [ ] 端口是否为 `443`？（不是内部端口 10000）
- [ ] `security=tls` 是否存在？
- [ ] `sni=子域名` 是否存在？
- [ ] `path` 中间有没有换行、空格、乱码？
- [ ] `host` 是否设置为子域名？

---

## 第二步：检查服务端

### 快速诊断表

| 问题 | 检查命令 | 解决方案 |
|------|----------|----------|
| Xray 没监听 | `ss -tlnp \| grep 10000` | `systemctl restart x-ui` |
| outbounds 为 null | 检查 config.json（见下方） | 重设 xrayTemplateConfig |
| 502 Bad Gateway | `systemctl status x-ui` | 重启 X-UI |
| SSL 错误 | `openssl x509 -in /root/cert/fullchain.cer -noout -enddate` | 续期证书 |
| XHTTP 不通 | curl 测试连通性（见下方） | 检查 Nginx location 配置 |
| Nginx 配置错误 | `nginx -t` | 根据错误信息修复 |
| 防火墙问题 | `ufw status` | 确认 443 端口开放 |
| 面板配置不同步 | 对比数据库和 config.json | `systemctl restart x-ui` |

### 详细诊断命令

```bash
# 1. 服务状态
systemctl is-active nginx x-ui fail2ban

# 2. 端口监听
ss -tlnp | grep -E ':80 |:443 |:10000 |:54321 '

# 3. Xray 配置完整性（outbounds 不能为 null）
python3 -c "
import json
cfg = json.load(open('/usr/local/x-ui/bin/config.json'))
print('Outbounds:', cfg.get('outbounds'))
print('DNS:', cfg.get('dns'))
"

# 4. XHTTP 连通性测试（返回非 502/503 = Xray 在响应）
WS_PATH=$(cat /root/.secrets/ws_path.txt)
curl -s -o /dev/null -w '%{http_code}' \
  -X POST http://127.0.0.1:10000/$WS_PATH

# 5. 服务器出网测试
curl -s -o /dev/null -w '%{http_code}' --max-time 5 https://www.google.com

# 6. 日志
tail -30 /var/log/nginx/error.log
journalctl -u x-ui --since "30 min ago" --no-pager
```

---

## 常见踩坑经验

### 1. xrayTemplateConfig 不完整 → outbounds=null

**症状**：客户端显示已连接，但打不开任何网页。

**原因**：3x-ui 用数据库中 `xrayTemplateConfig` 生成 Xray 运行配置，不完整的模板导致缺失字段被设为 null。

**修复**：重新设置完整的模板（参考 `manual-deploy.md` 第 12 步），模板必须至少包含：`dns`、`routing`、`outbounds`（direct + blocked）、`policy`、`api`、`stats`。

### 2. 3x-ui 默认开启订阅端口

**症状**：`ss -tlnp` 发现公网上多了一个意外端口（如 2096）。

**修复**：
```bash
sqlite3 /etc/x-ui/x-ui.db "INSERT OR REPLACE INTO settings (key, value) VALUES ('subEnable', 'false');"
sqlite3 /etc/x-ui/x-ui.db "INSERT OR REPLACE INTO settings (key, value) VALUES ('subListen', '127.0.0.1');"
systemctl restart x-ui
```

### 3. 面板导出的 VLESS 链接不能直接用

面板显示的是 Xray 内部配置（端口 10000、TLS 关闭），不适用于 Cloudflare CDN 架构。必须手动拼接链接：端口改 443、加 `security=tls`、加 `sni`。

### 4. 客户端链接复制断行

VLESS 链接很长，复制时经常在 path 参数处断行，导致路径中间出现 `\n` 或空格。这是最常见的"连不上"原因。

### 5. Xray 运行配置与面板不同步

在面板里删除/修改入站后，`/usr/local/x-ui/bin/config.json` 可能没有及时更新。不确定时执行 `systemctl restart x-ui` 强制重新生成。

### 6. CF API 自动配置（步骤 16）部分失败

**症状**：DNS 记录创建成功，但 SSL 模式 / 最低 TLS / WebSocket 三项返回 `success: false`（403）。

**原因**：API Token 只有 `Zone:Read + DNS:Edit`，缺 `Zone Settings:Edit` 权限——acme.sh 签证书和建 DNS 记录都不需要这个权限，所以前面步骤全部正常，到设置项才暴露。

**修复**：二选一：(a) 让用户去 Token 编辑页补 `Zone Settings:Edit` 权限后重跑步骤 16 对应命令；(b) 按 SKILL.md 第三步表格手动配置剩余项（只剩 2-3 项，1 分钟搞定）。

### 7. 传输协议演进

本 skill 已使用 XHTTP 替代 WebSocket（Xray 官方推荐的迁移方向）。XHTTP 分片成多个短 HTTP 请求，流量特征更像正常网页浏览，抗检测更强，弱网环境下也更稳定。
