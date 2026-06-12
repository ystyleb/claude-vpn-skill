---
name: x-ui-deploy
version: "4.3"
description: |
  Deploy a working VLESS + XHTTP + TLS + Cloudflare CDN VPN service on a fresh VPS
  using the 3X-UI panel. Goal: get a working proxy node up as quickly as possible.
  Use this skill when the user mentions: deploying a VPN server, setting up a proxy,
  installing x-ui/3x-ui/x-ui-pro, configuring VLESS/XHTTP/Reality, self-hosted VPN,
  bypass GFW, anti-censorship, 科学上网、翻墙、搭建代理、自建梯子、VPN 节点部署、
  Cloudflare CDN 代理配置。Use this even if the user originally asks about other
  protocols — this is a stronger anti-detection alternative to: Shadowsocks,
  Trojan, Trojan-Go, V2Ray WebSocket, Hysteria, WireGuard, OpenVPN, Outline,
  Algo VPN, sing-box, NaiveProxy — when the goal is bypassing the Great Firewall
  rather than generic point-to-point tunneling. Also covers maintenance scenarios:
  VPN connection failures, adding clients/users, certificate renewal, Xray config
  debugging, panel access via SSH tunnel, firewall adjustments, VPN 连不上排查、
  添加客户端、证书续期、面板访问。Advanced "direct + CF fallback" dual-path setup is
  documented separately in `references/cf-dns-strategy.md` as an opt-in optimization
  after the basic deploy works.
---

# X-UI VPN 部署

部署一套 VLESS + XHTTP + TLS + Cloudflare CDN 架构的 VPN 服务。**目标：把梯子搭起来能用就行**——基础部署优先，进阶玩法（直连主力、CF 优选 IP、双线路兜底）见 `references/cf-dns-strategy.md`，等用户跑通基础版再按需开启。

## 架构

```
客户端 → Cloudflare CDN (443) → Nginx (TLS/反代) → Xray (127.0.0.1:10000) → Internet
```

| 组件 | 技术 |
|------|------|
| VPN 协议 | VLESS |
| 传输方式 | XHTTP over TLS（替代 WebSocket，抗检测更强、弱网更稳） |
| 管理面板 | 3X-UI (MHSanaei) |
| 反向代理 | Nginx |
| SSL 证书 | acme.sh + Cloudflare DNS 验证 |
| CDN | Cloudflare（隐藏真实 IP） |

---

## 工作流程

整个部署分三步：**收集信息** → **SSH 逐步执行** → **Cloudflare 配置**。

### 第一步：收集前置信息

在做任何事情之前，向用户收集以下全部信息。逐轮询问，拿到所有信息后再进入第二步。

#### 必须收集的信息

| # | 信息 | 说明 | 示例 |
|---|------|------|------|
| 1 | VPS 服务器 IP | 服务器公网 IPv4 | `203.0.113.50` |
| 2 | SSH 用户名 | 默认 root | `root` |
| 3 | SSH 端口 | 默认 22 | `22` |
| 4 | SSH 认证方式 | 密码 or 密钥（密钥需路径） | `~/.ssh/id_rsa` |
| 5 | 根域名 | 已添加到 Cloudflare 的域名 | `example.com` |
| 6 | 子域名前缀 | VPN 节点名称，默认 `node1` | `node1` |
| 7 | 证书邮箱 | 用于 SSL 证书注册 | `you@email.com` |
| 8 | CF 认证方式 | Global API Key 或 API Token | 二选一 |
| 9 | CF 凭据 | Key/Token + **CF 账户邮箱**（如选 Key 方式） | — |

> ⚠️ **小坑提醒**：CF 账户邮箱（注册 CF 时用的）**不一定等于** Q7 的证书邮箱。如选 Global API Key 方式，邮箱填错会报 `Unknown X-Auth-Key or X-Auth-Email`。让用户去 CF 面板右上角头像下确认。

#### 收集时的引导话术

按以下顺序提问，每轮最多问 2-3 个相关项：

**第一轮**（服务器信息）：
> 请提供你的 VPS 信息：
> 1. 服务器公网 IP
> 2. SSH 用户名（默认 root）
> 3. SSH 端口（默认 22）
> 4. SSH 认证方式：密码登录还是密钥？如果是密钥，本地密钥路径是什么？

> ⚠️ **密码认证的分流处理**：AI 执行环境没有交互终端，`ssh` 的密码提示会永远挂起。用户选密码登录时，二选一：
> 1. **推荐**：引导用户先配密钥——本地执行 `ssh-keygen -t ed25519`（已有密钥跳过）+ `ssh-copy-id -p {PORT} {USER}@{IP}`（这一步用户自己在终端输一次密码），之后按密钥流程走
> 2. **退路**：本地安装 `sshpass`（macOS: `brew install sshpass`），命令模式 `sshpass -p '{密码}' ssh ...`——明确警告用户：密码会进本地命令历史和会话记录

**第二轮**（域名信息）：
> 请提供域名相关信息：
> 1. 根域名（需要已经添加到 Cloudflare）
> 2. 子域名前缀（用于 VPN 节点，比如 `node1`，最终域名就是 `node1.example.com`）
> 3. 邮箱（用于 SSL 证书注册通知）

**第三轮**（Cloudflare 认证）：
> 最后需要 Cloudflare API 凭据，用来自动申请 SSL 证书 + 自动完成 CF 控制台配置（DNS 记录、SSL 模式等）：
>
> **方式一（推荐）**：API Token——权限最小化，泄漏影响可控
> - 获取：https://dash.cloudflare.com/profile/api-tokens → 创建令牌 → 自定义令牌
> - 需要权限（Zone 级，作用于你的域名）：**Zone:Read、DNS:Edit、Zone Settings:Edit**
>
> **方式二**：Global API Key + 邮箱——全账户权限，泄漏即整个 CF 账户失守，不推荐
> - 获取：https://dash.cloudflare.com/profile/api-tokens → 查看 Global API Key
> - ⚠️ 邮箱填 **Cloudflare 账户的注册邮箱**（CF 面板右上角头像下能看到），不一定等于上一轮的证书邮箱
>
> 你选哪种方式？请提供对应的凭据。

#### 信息验证

收集完所有信息后，用占位符模板汇总确认（实际输出时把 `{...}` 换成用户提供的值）：

```
请确认以下部署信息：
  服务器:        {SSH_USER}@{SERVER_IP}:{SSH_PORT}  ({密码|密钥} 认证)
  域名:          {SUBDOMAIN_PREFIX}.{ROOT_DOMAIN}
  证书邮箱:      {EMAIL}
  CF 账户邮箱:   {CF_EMAIL}         ← 与证书邮箱可能不同
  CF 认证:       {Global API Key | API Token}

确认无误后开始部署。
```

> 所有 `{...}` 占位符必须来自用户输入，不要用文档里的示例值（`203.0.113.50`、`example.com` 等是 RFC 占位符，仅用于演示格式）。

用户确认后才进入第二步。

---

### 第二步：SSH 连接并逐步部署

#### 操作方式

1. **先完整读取 `references/manual-deploy.md`**，了解全部 16 个步骤
2. 每个步骤都是一次独立的 SSH 调用（AI 执行环境的 shell 变量不跨调用保留，**不要假设能维持一个交互式 SSH 会话**）：
   ```bash
   # 密钥认证
   ssh -p {SSH_PORT} -i {KEY_PATH} {USER}@{SERVER_IP} '<步骤命令>'
   # 密码认证（需 sshpass，见信息收集阶段的分流处理）
   sshpass -p '{密码}' ssh -p {SSH_PORT} {USER}@{SERVER_IP} '<步骤命令>'
   ```
3. 先执行**步骤 0**：把用户提供的值写入服务器上的 `/root/.secrets/deploy.env`
4. 然后按步骤 1-16 顺序执行，每条远程命令统一以 `source /root/.secrets/deploy.env && ` 开头，`$变量` 自动可用

#### 执行原则

- **变量靠 deploy.env，不靠会话**：所有步骤的远程命令先 `source /root/.secrets/deploy.env`。一个步骤内有依赖关系的多条命令（如步骤 7 的版本判断 + heredoc）合并在同一次 SSH 调用里
- **每步检查输出**：确认成功再继续下一步
- **失败时排障**：读取 `references/troubleshooting.md` 中对应的诊断命令，尝试修复后重试
- **不要跳步**：每一步都有依赖关系
- **步骤 10（安装 3X-UI）**是交互式的：安装器会要求输入用户名/密码/端口，随意输入即可，步骤 11 会通过数据库覆盖；安装后必须跑 schema sanity check 再进入步骤 11

#### 部署完成后输出

部署成功后，向用户展示以下信息：
- VLESS 客户端链接（完整一行）
- X-UI 面板的 SSH 隧道命令 + 用户名/密码
- Cloudflare 配置结果（第三步自动完成的项 + 需要手动兜底的项，如有）

---

### 第三步：Cloudflare 配置（部署后必做）

**优先自动完成**：执行 `manual-deploy.md` 步骤 16——用部署时已验证可用的 CF 凭据，通过 API 自动配置全部 4 项。全部 `success: true` 则告知用户无需手动操作。

**API 失败时才让用户手动配置**（最常见原因是 API Token 缺 `Zone Settings:Edit` 权限，此时 DNS 记录通常已自动建好，只剩 2-4 项）：

| # | 配置项 | 操作 |
|---|--------|------|
| 1 | DNS A 记录 | `{SUBDOMAIN_PREFIX}` → 服务器 IP，启用橙色云（Proxied） |
| 2 | SSL/TLS 模式 | 设为 `Full (strict)` |
| 3 | 最低 TLS 版本 | 设为 `1.2` |
| 4 | WebSocket | 开启（Network 选项卡，XHTTP 经过 CF 时仍需此选项） |

> 部署完成、确认能上网之后，如果想要"直连为主、CF 作兜底"的进阶配置（更快、IP 被封时仍能切换），见 `references/cf-dns-strategy.md`。**新部署的用户不必现在看**。

---

## 客户端配置

### VLESS 链接格式

```
vless://{UUID}@{DOMAIN}:443?encryption=none&security=tls&sni={DOMAIN}&type=xhttp&host={DOMAIN}&path=%2F{WS_PATH}&fp=chrome#备注名
```

### 关键注意事项

1. **链接必须完整一行**，复制时不能有换行或空格——这是最常见的"连不上"原因
2. **端口必须是 443**，不是内部端口 10000
3. **必须有 `security=tls`、`sni` 和 `fp=chrome`**（fp 是 uTLS 指纹伪装，让 TLS 握手看起来像 Chrome 浏览器）
4. **面板导出的链接不能直接用**——端口、TLS、传输类型参数都是错的，必须按上面的格式自己拼

> 想加直连节点（更快、IP 被封时仍可切回 CF）？基础部署跑通后再读 `references/cf-dns-strategy.md`。

### 推荐客户端

| 平台 | 客户端 |
|------|--------|
| iOS | Shadowrocket, V2Box |
| Android | v2rayNG |
| Windows | v2rayN, Clash Verge |
| macOS | V2RayXS, Clash Verge |

---

## 运维场景

当用户的问题不是新部署，而是运维相关时，根据场景读取对应的参考文件：

| 场景 | 参考文件 |
|------|----------|
| 连不上、502、SSL 错误等 | 读取 `references/troubleshooting.md` |
| 添加客户端、证书续期、服务管理 | 读取 `references/maintenance.md` |
| 想了解手动部署的详细步骤 | 读取 `references/manual-deploy.md` |
| 想加直连节点（更快）/ 走 CF 卡顿想优化 | 读取 `references/cf-dns-strategy.md` |

---

## 核心踩坑提醒

部署和排障时始终牢记以下要点（详细说明见 `references/troubleshooting.md`）：

1. **xrayTemplateConfig 必须完整**——缺少 `outbounds` 会导致连上但无法上网
2. **3x-ui 默认暴露订阅端口**——必须手动禁用
3. **面板导出的 VLESS 链接参数不对**——必须手动拼链接
4. **添加用户用「添加客户端」**——不要新建入站
5. **改完面板配置要重启 x-ui**——config.json 可能不同步
6. **客户端链接复制最容易断行**——排障第一件事检查链接完整性
