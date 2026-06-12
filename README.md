# x-ui-deploy · AI 一键搭建 VPN 翻墙节点

> **让 Claude 替你搭梯子**——在 VPS 上自动化部署 VLESS + XHTTP + TLS + Cloudflare CDN 架构的自建 VPN，从 0 到能用 10 分钟搞定。

[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](https://opensource.org/licenses/MIT)
[![Skill Version](https://img.shields.io/badge/skill-v4.3-blue)](https://github.com/henrywen98/claude-vpn-skill/releases)
[![Claude Code](https://img.shields.io/badge/Claude%20Code-Skill-8B5CF6)](https://claude.com/claude-code)
[![Stars](https://img.shields.io/github/stars/henrywen98/claude-vpn-skill?style=social)](https://github.com/henrywen98/claude-vpn-skill/stargazers)

**关键词**：自建 VPN · 科学上网 · 3X-UI 一键部署 · VLESS 搭建 · Xray 配置 · Cloudflare CDN 隐藏 IP · Claude Code Skill · VPS 翻墙教程 · 梯子搭建 · AI 自动化运维

[English README →](./README.en.md)

---

## 这是什么

一个 **Claude Code Skill**（Claude Code 的扩展能力包）。安装之后，你只要对 Claude 说一句"帮我部署一个 VPN"，它会：

1. 逐步问你 9 个必要参数（服务器 IP / 域名 / Cloudflare 凭据等）
2. SSH 登录你的 VPS，按 15 步完整部署 Nginx + Xray + 3X-UI 面板 + SSL 证书
3. 输出一条可以直接导入 Shadowrocket / v2rayN / Clash 的 VLESS 链接

**目标**：把梯子搭起来能用。不教架构哲学，不搞花哨配置，直接交付一个能连的节点。

## 适合什么人用

- ✅ **想要自己控制的梯子**，不想再续费机场、不想被卷跑路
- ✅ **手上有 VPS 但不想啃 15 篇博客**，希望 AI 一步步带着搭
- ✅ **跑过一键脚本但被 GFW 封过 IP**，想要 Cloudflare CDN 隐藏真实 IP 的方案
- ✅ **嫌 WireGuard / OpenVPN / Shadowsocks / Trojan 太老**，想用更抗检测的 VLESS + XHTTP
- ✅ **已经用 Claude Code / Codex CLI / OpenCode 干活**，想顺手把运维也交给 AI

不适合：完全没有 VPS、没有域名、不想接触命令行的人 — 建议直接买商业机场。

## 为什么选这个，而不是其他自建教程

| | 手抄 blog 教程 | 一键脚本 | **本 skill** |
|---|---|---|---|
| 能配合你实际情况调整 | ❌ 脚本固定 | ❌ 脚本固定 | ✅ AI 会问你实际参数 |
| 出错能诊断 | ❌ 自己摸索 | ⚠️ 报错就卡住 | ✅ AI 会排查并修 |
| 安全加固 | ⚠️ 看博主良心 | ⚠️ 通常很差 | ✅ UFW + Fail2Ban + 面板 localhost + 伪装站 |
| 加用户 / 续证书 / 改配置 | 📖 翻博客 | 🤷 重新跑 | ✅ AI 直接帮你改 |
| 伪装站指纹 | 🔴 模板化千篇一律 | 🔴 同上 | ✅ 部署时随机生成英文公司名 |

## 架构

```
客户端 → Cloudflare CDN (443) → Nginx (TLS 反代) → Xray (127.0.0.1:10000) → Internet
         ↑ 隐藏真实 IP         ↑ 伪装普通网站     ↑ 实际代理核心
```

| 组件 | 技术 | 作用 |
|------|------|------|
| VPN 协议 | **VLESS** | 轻量、无额外加密，性能顶 |
| 传输 | **XHTTP over TLS** | 替代老式 WebSocket，抗 GFW 检测更强 |
| 面板 | **3X-UI** (MHSanaei) | Web 管理界面，加用户/看流量 |
| 反代 | **Nginx** | TLS 终止 + 伪装站点 |
| 证书 | **acme.sh** + Cloudflare DNS | 自动申请通配符证书 + 每 60 天续期 |
| CDN | **Cloudflare** | 隐藏 VPS 真实 IP，防被 GFW 封 |

## 30 秒快速开始

### 1️⃣ 准备

- VPS：Debian 11/12 或 Ubuntu 20.04+（搬瓦工、Vultr、DigitalOcean、AWS Lightsail 都行）
- 域名：已接入 Cloudflare（域名本身放哪家注册商都行）
- Cloudflare API：API Token（推荐）或 Global API Key（用来自动签证书 + 自动配置 CF 控制台）
- [Claude Code](https://claude.com/claude-code)（免费 CLI 工具，使用时需 Anthropic 账号授权：Claude.ai Pro 订阅 **或** API Key 按量付费，二选一）

### 2️⃣ 安装 skill

#### 🚀 方式 A：一句话让 AI 自己装（最简单）

把下面这段话直接发给 Claude Code / Codex CLI / OpenCode 的任一个：

```
安装这个 skill：https://github.com/henrywen98/claude-vpn-skill

做法：git clone 后，把 skills/x-ui-deploy 这个目录复制到你当前 CLI 对应的 skills
目录（Claude Code 是 ~/.claude/skills/，Codex CLI 是 ~/.codex/skills/，OpenCode
是 ~/.config/opencode/skills/ 或 ~/.claude/skills/）。装完验证一下能被识别，然后
删掉 clone 出来的临时目录。
```

AI 会自动识别自己是哪个 CLI、git clone、复制、清理、确认。整个过程一条指令搞定。

> Gemini CLI 因为走 Extensions 体系不是 1:1 映射，需要手动处理——见方式 B 的备注。

#### 📦 方式 B：手动复制（或者想精细控制位置）

先把仓库拉下来：

```bash
git clone https://github.com/henrywen98/claude-vpn-skill.git
```

然后根据你用的 AI CLI 把 skill 放到对应目录：

| CLI | 安装命令 | 备注 |
|---|---|---|
| **[Claude Code](https://claude.com/claude-code)** | `cp -r claude-vpn-skill/skills/x-ui-deploy ~/.claude/skills/` | 原生 skill 支持 |
| **[OpenAI Codex CLI](https://developers.openai.com/codex/cli)** | `cp -r claude-vpn-skill/skills/x-ui-deploy ~/.codex/skills/` | 识别同样的 `SKILL.md` 格式 |
| **[OpenCode](https://opencode.ai)** | `cp -r claude-vpn-skill/skills/x-ui-deploy ~/.claude/skills/` | 原生读取 `~/.claude/skills/`，与 Claude 共用目录 |
| **[Gemini CLI](https://github.com/google-gemini/gemini-cli)** | `cat claude-vpn-skill/skills/x-ui-deploy/SKILL.md > ./GEMINI.md` | Gemini 用 Extensions 系统，简化做法是把 SKILL.md 作为项目级 context 注入 |

> 💡 **项目级安装**：把路径里的 `~/.claude/skills/`（或 `~/.codex/skills/`）换成仓库内的 `.claude/skills/`（或 `.codex/skills/`），skill 只对该项目生效。

### 3️⃣ 对你的 AI 说这句话

```
帮我部署一个 VPN
```

然后回答它问的问题，剩下的交给它。

## 使用场景

除了新部署，skill 也能处理运维：

| 你说什么 | Claude 会做什么 |
|---|---|
| "帮我部署一个 VPN" | 新部署全流程 |
| "VPN 连不上了" | 按 `troubleshooting.md` 诊断 + 修复 |
| "帮我加一个 VPN 用户" | 按 `maintenance.md` 添加客户端 |
| "证书要续期吗" | 检查证书状态，必要时强制续期 |
| "想加一个直连节点" | 读 `cf-dns-strategy.md` 给出改造方案 |

## 安全设计

默认开启，无需你配置：

- 🛡️ **UFW 防火墙**：仅开 SSH / 80 / 443
- 🔒 **X-UI 面板仅 localhost**：通过 SSH 隧道访问，不暴露公网
- 🚫 **Fail2Ban**：SSH 暴力破解 3 次 → 封 2 小时
- 📦 **3X-UI 订阅端口已禁用**（该端口默认公网暴露，是已知漏洞面）
- 🎭 **伪装站点随机化**：每台 VPS 生成一个随机英文公司名（如 `Atlas Ventures`），防 GFW 批量指纹
- 🔐 **TLSv1.2/1.3 only**，HSTS + 安全头齐全
- ☁️ **Cloudflare 隐藏真实 IP**：就算伪装被识破，CF 前置也让攻击者打不到源站

## 支持的客户端

| 平台 | 客户端 |
|------|--------|
| iOS | [Shadowrocket](https://apps.apple.com/app/shadowrocket/id932747118), V2Box |
| Android | [v2rayNG](https://github.com/2dust/v2rayNG) |
| Windows | [v2rayN](https://github.com/2dust/v2rayN), [Clash Verge](https://github.com/MetaCubeX/Clash.Meta) |
| macOS | [V2RayXS](https://github.com/tzmax/V2RayXS), Clash Verge |

## 文件结构

```
skills/x-ui-deploy/
├── SKILL.md                      # 主工作流（Claude 的行为指令）
└── references/                   # 按需加载的参考文档
    ├── manual-deploy.md           # 15 步完整部署蓝图
    ├── troubleshooting.md         # 故障排查 + 踩坑清单
    ├── maintenance.md             # 日常运维 + 安全加固
    └── cf-dns-strategy.md         # 进阶：直连 + CF 兜底双线路
```

## 常见问题

<details>
<summary><b>会暴露我的真实 IP 吗？</b></summary>

默认架构下 **不会**。所有流量走 Cloudflare CDN，GFW 和第三方只看到 Cloudflare 的 IP。你的 VPS 真实 IP 只有 Cloudflare 和你自己知道。
</details>

<details>
<summary><b>走 CF 会不会很慢？</b></summary>

CF 免费版长连接确实有软限速。如果日常使用觉得卡，可以按 `references/cf-dns-strategy.md` 再加一条直连节点作为主力，CF 节点留着当 IP 被封时的兜底。双线路并存。
</details>

<details>
<summary><b>IP 被 GFW 封了怎么办？</b></summary>

客户端切到 CF 节点立刻恢复（CF→VPS 那段不过 GFW）。治本需要在 VPS 控制台换 IP（搬瓦工面板有"Change IP"功能）或迁机房。
</details>

<details>
<summary><b>证书会过期吗？</b></summary>

acme.sh 自动续期，每 60 天一次，续期后自动 reload Nginx。正常不用手动管。
</details>

<details>
<summary><b>加用户要重新部署吗？</b></summary>

不用。通过 SSH 隧道进 3X-UI 面板，在现有入站里"添加客户端"，秒级生效。skill 会帮你走这个流程。
</details>

<details>
<summary><b>这个和 x-ui 官方脚本有什么区别？</b></summary>

官方脚本只装 x-ui 本身。这个 skill 在它之上加了：Nginx 反代 + 通配符 SSL + Cloudflare CDN + 伪装站 + UFW/Fail2Ban 硬化 + 自动生成客户端链接。而且 AI 帮你处理所有交互，包括出错时的排障。
</details>

<details>
<summary><b>不用 Claude Code 能用吗？</b></summary>

可以。`references/manual-deploy.md` 是完整的 15 步 SSH 命令清单，复制粘贴到你的终端也能部署。skill 只是让 AI 帮你跑这个流程 + 处理异常。
</details>

## 反馈 · 贡献

遇到问题或想法，欢迎：

- 提 [Issue](https://github.com/henrywen98/claude-vpn-skill/issues)
- 发 [Pull Request](https://github.com/henrywen98/claude-vpn-skill/pulls)
- 给这个仓库一个 ⭐ 如果它帮到你（也让更多人能搜到）

## 许可证

[MIT](./LICENSE.txt) — 随便用、随便改、随便分发。

---

**免责声明**：本 skill 仅用于个人学习、网络研究、合法访问国际互联网资源。使用者自行承担相关法律责任，作者不对任何滥用行为负责。
