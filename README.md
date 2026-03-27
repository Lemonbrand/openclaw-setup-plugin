# OpenClaw Setup Plugin

A Claude Code plugin that guides anyone (including complete beginners) through setting up a self-hosted AI agent on their own server.

![Openclaw-setup image](ralph-loop_diagram.html.png)

## What it builds

A personal AI assistant that:
- Lives on a $12/month cloud server you own
- Messages you on Telegram, WhatsApp, email, or Slack
- Is locked down with kernel-level security (nono) so it can only access what you approve
- Runs 24/7, even when your laptop is off
- Can send you scheduled check-ins, prep you for meetings, and flag overdue follow-ups

## What's covered

| Act | What happens |
|-----|-------------|
| **1. Your Server** | Rent a DigitalOcean droplet, generate SSH keys, harden security (firewall, fail2ban, auto-updates) |
| **2. Your Assistant** | Install OpenClaw, configure the AI brain (Anthropic API), lock it down with nono kernel sandbox |
| **3. Go Live** | Connect your messaging channel and send your first message |
| **4. Your First Agent** | Set up scheduled check-ins and explore what your agent can do |

## Install

```bash
/plugin install github.com/lemonbrand/openclaw-setup-plugin
```

Or in Claude Desktop: Plugins > Add from GitHub > `lemonbrand/openclaw-setup-plugin`

## Usage

After installing, run:

```
/openclaw-setup
```

The skill walks you through everything step by step. It does the technical work for you via terminal commands. You only need to click through a couple of websites (creating accounts) and save credentials.

## Prerequisites

- **Claude Code** installed and running
- A web browser (for creating DigitalOcean and Anthropic accounts)
- A password manager (1Password, Bitwarden, or Apple Passwords)

Everything else is set up during the walkthrough.

## What you'll need (provided during setup)

- **Anthropic API key** (~$5-15/month usage)
- **DigitalOcean account** ($12/month for the server, $200 free credit with referral link)
- **Telegram, WhatsApp, Gmail, or Slack** account for messaging

## Reference files

The plugin includes reference documentation for deeper customization:

- `references/vps-hardening.md` - Server security checklist
- `references/nono-profiles.md` - Kernel sandbox profile reference
- `references/agent-patterns.md` - Starter code for cron agents (morning brief, meeting prep, follow-up checker)

## Built by

Simon Bergeron at [Lemonbrand](https://lemonbrand.io). Questions or issues: simon@lemonbrand.io
