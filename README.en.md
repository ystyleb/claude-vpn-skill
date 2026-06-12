# x-ui-deploy · AI-Powered VPN Server Setup

> **Let Claude set up your VPN.** Automatically deploy a self-hosted VPN (VLESS + XHTTP + TLS + Cloudflare CDN) on your VPS. Zero to working node in about 10 minutes.

[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](https://opensource.org/licenses/MIT)
[![Skill Version](https://img.shields.io/badge/skill-v4.3-blue)](https://github.com/henrywen98/claude-vpn-skill/releases)
[![Claude Code](https://img.shields.io/badge/Claude%20Code-Skill-8B5CF6)](https://claude.com/claude-code)
[![Stars](https://img.shields.io/github/stars/henrywen98/claude-vpn-skill?style=social)](https://github.com/henrywen98/claude-vpn-skill/stargazers)

**Keywords**: self-hosted VPN · bypass GFW · 3X-UI one-click · VLESS setup · Xray server · Cloudflare CDN · Claude Code Skill · AI DevOps · VPS proxy · censorship circumvention

[简体中文 README →](./README.md)

---

## What is this

A **Claude Code Skill** (an extension that plugs into [Claude Code](https://claude.com/claude-code)). Once installed, just tell Claude:

> "Help me deploy a VPN"

…and it will:

1. Ask you 9 necessary parameters (server IP, domain, Cloudflare credentials, etc.) across 3 rounds
2. SSH into your VPS and execute a 15-step deployment: Nginx + Xray + 3X-UI panel + SSL certificates
3. Output a VLESS link you can paste directly into Shadowrocket / v2rayN / Clash

**Goal**: Get a working VPN node up, fast. No architectural preaching, no feature bloat, just a node that works.

## When to use this skill

- ✅ **You want a VPN you fully control**, no more renewing commercial subscriptions or losing your provider overnight
- ✅ **You have a VPS but don't want to read 15 blog posts** — let an AI walk you through it step by step
- ✅ **You ran a one-click script and got your IP blocked by the GFW**, and want a Cloudflare-CDN-fronted setup that hides your origin IP
- ✅ **WireGuard / OpenVPN / Shadowsocks / Trojan feel dated** — you want VLESS + XHTTP for stronger anti-detection
- ✅ **You already use Claude Code / Codex CLI / OpenCode**, and want to hand off ops to your AI as well

Not for you if: you don't have a VPS, don't have a domain, or don't want to touch a terminal — buy a commercial proxy service instead.

## Why this over other self-hosting guides

| | Blog tutorials | One-click scripts | **This skill** |
|---|---|---|---|
| Adapts to your specifics | ❌ Fixed | ❌ Fixed | ✅ AI asks your actual params |
| Handles errors | ❌ Figure it out yourself | ⚠️ Crashes and stops | ✅ AI diagnoses and fixes |
| Security hardening | ⚠️ Depends on author | ⚠️ Usually weak | ✅ UFW + Fail2Ban + localhost panel + decoy site |
| Add users / renew certs / change config | 📖 Re-read blog | 🤷 Re-run | ✅ AI does it for you |
| Decoy-site fingerprinting | 🔴 Identical everywhere | 🔴 Same | ✅ Randomized English company name per deploy |

## Architecture

```
Client → Cloudflare CDN (443) → Nginx (TLS reverse proxy) → Xray (127.0.0.1:10000) → Internet
         ↑ hides real IP        ↑ disguised as normal site   ↑ actual proxy core
```

| Component | Tech | Role |
|-----------|------|------|
| Protocol | **VLESS** | Lightweight, no extra encryption, peak performance |
| Transport | **XHTTP over TLS** | Replaces old WebSocket, stronger anti-GFW detection |
| Panel | **3X-UI** (MHSanaei) | Web UI for adding users, monitoring traffic |
| Reverse Proxy | **Nginx** | TLS termination + decoy site |
| Certificates | **acme.sh** + Cloudflare DNS | Auto-issue wildcard cert, 60-day auto-renew |
| CDN | **Cloudflare** | Hides VPS real IP, protects against IP-based blocking |

## 30-second quickstart

### 1️⃣ Prepare

- A VPS running Debian 11/12 or Ubuntu 20.04+ (BandwagonHost, Vultr, DigitalOcean, AWS Lightsail, etc.)
- A domain attached to Cloudflare (registrar doesn't matter)
- Cloudflare API credentials (API Token recommended, or Global API Key) — used for SSL cert issuance and automatic CF dashboard configuration
- [Claude Code](https://claude.com/claude-code) (free CLI tool; requires an Anthropic account at runtime — either a Claude.ai Pro/Max subscription **or** an API key with pay-per-use billing)

### 2️⃣ Install the skill

#### 🚀 Option A: One-prompt install (easiest)

Paste this into Claude Code / Codex CLI / OpenCode:

```
Install this skill: https://github.com/henrywen98/claude-vpn-skill

How: git clone the repo, then copy the folder skills/x-ui-deploy into the skills
directory of the CLI you're currently running (Claude Code: ~/.claude/skills/,
Codex CLI: ~/.codex/skills/, OpenCode: ~/.config/opencode/skills/ or
~/.claude/skills/). Verify the skill is recognized, then remove the cloned
temp directory.
```

The AI will auto-detect which CLI it is, clone, copy, clean up, and verify — all in one shot.

> Gemini CLI uses a different Extensions system (not 1:1 with SKILL.md). Handle it manually via Option B.

#### 📦 Option B: Manual copy (or if you want to pick the location)

First clone the repo:

```bash
git clone https://github.com/henrywen98/claude-vpn-skill.git
```

Then drop the skill into your AI CLI's skills directory:

| CLI | Install command | Notes |
|---|---|---|
| **[Claude Code](https://claude.com/claude-code)** | `cp -r claude-vpn-skill/skills/x-ui-deploy ~/.claude/skills/` | Native skill support |
| **[OpenAI Codex CLI](https://developers.openai.com/codex/cli)** | `cp -r claude-vpn-skill/skills/x-ui-deploy ~/.codex/skills/` | Recognizes the same `SKILL.md` format |
| **[OpenCode](https://opencode.ai)** | `cp -r claude-vpn-skill/skills/x-ui-deploy ~/.claude/skills/` | Natively reads `~/.claude/skills/`, shares directory with Claude Code |
| **[Gemini CLI](https://github.com/google-gemini/gemini-cli)** | `cat claude-vpn-skill/skills/x-ui-deploy/SKILL.md > ./GEMINI.md` | Gemini uses an Extensions system. Simplest path: inject `SKILL.md` as project-level context |

> 💡 **Project-level install**: Replace `~/.claude/skills/` (or `~/.codex/skills/`) with in-repo `.claude/skills/` (or `.codex/skills/`) to scope the skill to one project.

### 3️⃣ Tell your AI

```
Help me deploy a VPN
```

Answer its questions, the rest is automatic.

## Use cases

Beyond first-time deploy, the skill also handles ops:

| Say this to Claude | Claude does |
|---|---|
| "Help me deploy a VPN" | Full fresh-deploy flow |
| "The VPN isn't connecting" | Diagnose per `troubleshooting.md` and fix |
| "Add a new VPN user" | Adds client per `maintenance.md` |
| "Do I need to renew the cert" | Checks cert, force-renews if needed |
| "I want to add a direct-connect node" | Reads `cf-dns-strategy.md` and proposes the conversion |

## Security

Enabled by default, no configuration needed:

- 🛡️ **UFW firewall**: only SSH / 80 / 443 open
- 🔒 **3X-UI panel bound to localhost**: access via SSH tunnel, never exposed to public internet
- 🚫 **Fail2Ban**: 3 failed SSH attempts → 2 hour ban
- 📦 **3X-UI subscription port disabled** (known exposed-by-default surface)
- 🎭 **Randomized decoy site**: each VPS gets a random English company name (e.g., `Atlas Ventures`) to defeat batch fingerprinting
- 🔐 **TLSv1.2/1.3 only**, HSTS + security headers in place
- ☁️ **Cloudflare hides real IP**: even if the decoy site is fingerprinted, attackers hit CF, not your origin

## Supported clients

| Platform | Clients |
|----------|---------|
| iOS | [Shadowrocket](https://apps.apple.com/app/shadowrocket/id932747118), V2Box |
| Android | [v2rayNG](https://github.com/2dust/v2rayNG) |
| Windows | [v2rayN](https://github.com/2dust/v2rayN), [Clash Verge](https://github.com/MetaCubeX/Clash.Meta) |
| macOS | [V2RayXS](https://github.com/tzmax/V2RayXS), Clash Verge |

## File layout

```
skills/x-ui-deploy/
├── SKILL.md                      # Main workflow (Claude's instructions)
└── references/                   # Loaded on demand
    ├── manual-deploy.md           # Full 15-step deployment script
    ├── troubleshooting.md         # Error diagnostics + gotchas
    ├── maintenance.md             # Day-to-day ops + security hardening
    └── cf-dns-strategy.md         # Advanced: direct + CF dual-path fallback
```

## FAQ

<details>
<summary><b>Does this expose my VPS real IP?</b></summary>

By default, **no**. All traffic goes through Cloudflare CDN. Observers (including the GFW) only see Cloudflare IPs. Your origin IP is known only to you and Cloudflare.
</details>

<details>
<summary><b>Is going through CF slow?</b></summary>

Cloudflare's free tier does soft-throttle long-lived connections. If daily use feels laggy, follow `references/cf-dns-strategy.md` to add a direct-connect node as primary, keeping CF as fallback for when the VPS IP gets blocked.
</details>

<details>
<summary><b>What if the GFW blocks my VPS IP?</b></summary>

Client switches to the CF node and immediately works again (the CF→VPS hop doesn't cross the GFW). For a permanent fix, change the VPS IP in your provider's control panel, or migrate datacenter.
</details>

<details>
<summary><b>Do certificates expire?</b></summary>

acme.sh handles auto-renewal every 60 days and reloads Nginx automatically. Normally no manual intervention needed.
</details>

<details>
<summary><b>Do I have to re-deploy to add users?</b></summary>

No. SSH-tunnel into the 3X-UI panel and "Add Client" in the existing inbound. Takes seconds. The skill can do this for you.
</details>

<details>
<summary><b>How is this different from the official 3X-UI install script?</b></summary>

The official script only installs 3X-UI itself. This skill adds: Nginx reverse proxy + wildcard SSL + Cloudflare CDN + decoy site + UFW/Fail2Ban hardening + auto-generated client links. Plus AI handles all interactive steps and troubleshooting.
</details>

<details>
<summary><b>Can I use this without Claude Code?</b></summary>

Yes. `references/manual-deploy.md` is a complete 15-step SSH command sequence you can copy-paste into any terminal. The skill just lets AI do it for you and handle the edge cases.
</details>

## Feedback · Contributing

Found an issue or have ideas? Open an [Issue](https://github.com/henrywen98/claude-vpn-skill/issues) or submit a [Pull Request](https://github.com/henrywen98/claude-vpn-skill/pulls).

If this helped you, a ⭐ would mean a lot — it also helps other people discover it.

## License

[MIT](./LICENSE.txt) — use, modify, and distribute freely.

---

**Disclaimer**: This skill is intended for personal education, network research, and lawful access to the international Internet. Users are responsible for compliance with their local laws. The author takes no responsibility for misuse.
