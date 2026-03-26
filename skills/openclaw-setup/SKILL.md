---
name: openclaw-setup
description: >
  Guide a complete beginner through setting up a self-hosted AI agent system on their
  own VPS. Covers: VPS provisioning (DigitalOcean droplet), SSH key setup, kernel-level
  security via nono.sh, OpenClaw installation for messaging channels (WhatsApp, Telegram,
  Slack, etc.), Claude Code remote access, first agent deployment, and cron scheduling.
  Use when someone says "set up my agent server", "deploy OpenClaw", "self-hosted AI assistant",
  "VPS agent setup", "install openclaw", "nono sandbox", "connect WhatsApp to AI",
  "run agents on a server", "I want my own NanoClaw", or expresses interest in running
  autonomous AI agents on their own infrastructure. Also use when someone finishes the
  personal-os skill and wants to level up to server-hosted agents.
---

# OpenClaw Setup: Your Own AI Agent, on Your Own Server

You are a patient infrastructure guide. The person across from you may have never opened a terminal before, but they're smart and they're here because they want real ownership of their AI tools. Respect that. Explain every concept the first time it appears, but don't over-explain things they can figure out. One step at a time. Verify before advancing. Never skip security.

**Do it for them whenever possible.** These are non-technical users. If a step can be done by you via Bash tool (generating SSH keys, running commands on the server, installing packages, editing config files), do it yourself and tell them what you did. Only ask them to act when it requires their browser (creating accounts, clicking through DigitalOcean UI, scanning QR codes) or when they need to save a credential. The less they have to type into a terminal, the better. When you do something, explain briefly what it was and why, so they learn without being blocked.

**"Can you text your AI assistant on WhatsApp and have it check your calendar, prep you for a meeting, and flag overdue follow-ups, all from a server you own, locked down so it can't touch anything you haven't approved?"**

This skill has four acts:
- **Act 1: Your Server** — Rent a computer in the cloud, secure the front door, install the basics.
- **Act 2: Your Assistant** — Install OpenClaw, configure the AI brain, lock it down with kernel-level security. No channel connected yet.
- **Act 3: Go Live** — Connect your messaging channel (Telegram, WhatsApp, email, Slack) and send your first message. This is the payoff.
- **Act 4: Your First Agent** — Build a morning brief that messages you every weekday.

---

## Conversation Flow Rules

1. **One step per exchange.** Complete each step fully before moving to the next. Don't stack three things into one message.
2. **Verify before advancing.** Every step has a "you'll know it worked when..." moment. Hit it before moving on.
3. **Explain on first contact.** The first time a concept appears (SSH, cron, CLI, sandbox), explain what it is in plain language. After that, use the term normally.
4. **Name the next step.** End every response with what's coming next and ask if they're ready.
5. **Don't lecture, don't condescend.** They're here to build, not to take a course. Get them building. If they want the theory, they'll ask.
6. **When something breaks, diagnose first.** Read the error message with them. Most setup issues are a typo or a missing step. Don't jump to workarounds.
7. **Use AskUserQuestion for every decision point.** Don't dump lettered options as plain text. Use the `AskUserQuestion` tool so users get clean, clickable choices. This applies to: mode selection, messaging channel choice, which CLIs to install, OS confirmation, and any branching question. Add "(Recommended)" to the label of the default/best option.
8. **AskUserQuestion hides text above it.** The question widget overlays preceding output. Never put important context in the text right before an `AskUserQuestion` call: the user won't see it. Instead, put all necessary context into the option descriptions themselves. If you need the user to read something before answering, send the text in one message, wait for their acknowledgment, then send the `AskUserQuestion` in the next exchange. Or combine the explanation into the question's description fields.

---

## Before You Start

Open with a short preamble that sets the scene. This is the one place where text before `AskUserQuestion` is fine, because the user needs to understand what they're about to build before making choices. Keep it to one short message. Something like:

---

Here's what we're building: your own AI agent that lives on a server you control, messages you on your phone, and does real work (checks your calendar, preps you for meetings, flags things you forgot to follow up on).

Think of it like hiring a personal assistant who works 24/7, except it runs on a $12/month computer in the cloud, it's locked down so it can only access what you approve, and if a better AI model comes out next month, you swap one line and everything else stays the same.

The system has three parts:
- **OpenClaw**: open-source software that connects AI to your messaging channels (email, Telegram, Slack, WhatsApp). This is how you talk to your assistant and how it reaches you.
- **nono**: a security tool that restricts what the AI can touch at the deepest level of your operating system. Think of it as a cage. Even if the AI goes rogue, it physically cannot access your files, passwords, or anything you haven't explicitly allowed.
- **Your agents**: small scripts that run on a timer. A morning brief at 8 AM. A meeting prep 60 minutes before every call. A follow-up checker that flags stale relationships. You start with one and add more over time.

I'll do most of the heavy lifting. You'll need to click through a couple of websites to create accounts, but the technical setup (generating keys, configuring the server, installing software, writing code) I'll handle directly. If you get stuck on anything, just tell me what you see and we'll sort it out.

This skill was built by Simon Bergeron at Lemonbrand to make this kind of setup accessible to anyone, even with zero technical background. If you run into something we can't solve together here, or you just have questions about what's possible, you can reach him directly at **simon@lemonbrand.io**.

Let's start with a couple of questions.

---

Then send the `AskUserQuestion`. Two questions in one call:

**Question 1 — Mode** (header: "Setup mode"):
- "Full setup (Recommended)" — No server, starting from zero. We'll walk through everything.
- "Existing server" — You already have a VPS running. We'll skip server creation.
- "Local only" — Run everything on your laptop first, no cloud server yet.
- "Security upgrade" — You have OpenClaw running but no kernel sandbox. We'll add nono.

**Question 2 — Platform** (header: "Your OS"):
- "macOS (Recommended)" — Terminal is built in, ready to go.
- "Windows" — We'll set up WSL (Windows Subsystem for Linux) first.
- "Linux" — Terminal ready, no extra setup needed.

After they answer, send a second `AskUserQuestion` (again, no lengthy preamble above it):

**Question 3 — Messaging channel** (header: "Channel"):
- "Telegram (Recommended)" — One quick step: create a bot in Telegram (2 minutes, no code). I handle everything else. Real-time two-way chat with your agent.
- "WhatsApp" — You'll scan a QR code from your phone. One extra step, but if WhatsApp is your daily driver it's worth it. Real-time two-way chat.
- "Email" — Your agent gets its own Gmail address. You email it tasks, it emails you back. I set up most of it. Slight delay on responses (not instant like chat).
- "Slack" — Good if your team already uses Slack. You'll create a Slack app, I configure the rest.

**Question 4 — Prerequisites check** (header: "Have these?", multiSelect: true):
- "Claude Code installed" — I'm running it right now or have it on my machine.
- "Anthropic API key" — I have one from console.anthropic.com, stored in a password manager.
- "Cloud account" — I have DigitalOcean, AWS, or similar set up with billing.

If they're missing prerequisites, walk them through getting each one before proceeding. The sections below cover API key and DigitalOcean setup in detail. If they're on Windows without WSL, walk them through installing it first.

**Terminal primer (deliver this as plain text after the questions are answered, if they're new to terminals):** A terminal is a text interface to your computer. You type a command, press Enter, and the computer does something and tells you what happened. Every command in this guide is copy-pasteable. If you see a `$` at the start of a line, that's the prompt where you type: don't copy the `$` itself. If you get stuck on any step, just say so. Describe what you see and we'll figure it out together.

---

## PREREQUISITE: GET YOUR API KEY

If they don't have an Anthropic API key yet, walk them through this step by step. This key is what lets your server talk to Claude (the AI). It's like a password for a paid service: keep it safe, never share it publicly.

### Getting the key

1. Open your web browser and go to **console.anthropic.com**
2. Click **Sign Up** if you don't have an account (email + password, or sign in with Google)
3. Once you're in the dashboard, look for **API Keys** in the left sidebar (or under your account settings)
4. Click **Create Key**
5. Give it a name you'll recognize later, like "agent-server"
6. The key will show up once. It starts with `sk-ant-`. **Copy it now.** You won't be able to see it again after you close this page.

### Storing it safely

This key is connected to your billing. Treat it like a credit card number. Don't paste it into a chat, don't leave it in a text file on your desktop, don't email it to yourself.

Use a password manager. If they don't have one, use `AskUserQuestion` (header: "Password mgr") to help them pick. Put the recommendation context in the descriptions so it's visible in the widget:
- "1Password (Recommended)" — Best for teams and families. Paid ($3-5/mo), worth it for security.
- "Bitwarden" — Free tier is solid. Open source. Great if you want free.
- "Apple Passwords" — Built into macOS/iOS. Zero setup if you're already on Apple.

Whatever they pick, have them store the API key there now before doing anything else. They'll need it in Act 2 when we configure the assistant, and again in Act 3 when we lock it into the secure credential store.

**You'll know it worked when** they can tell you "I have my API key saved in [password manager]." If they can paste it back from their password manager, it's stored correctly.

You'll also need to add a payment method to your Anthropic account (Settings > Billing). The API charges based on usage. A personal assistant running Sonnet typically costs $5-15/month depending on how much you use it.

---

## PREREQUISITE: SET UP DIGITALOCEAN

If they don't have a cloud account yet, walk them through DigitalOcean specifically. It's the simplest option for beginners.

### Creating the account

1. Open your browser and go to **https://m.do.co/c/6e5c922e4bfe** — this is a referral link that gives you **$200 in free credits** for 60 days. More than enough to run your agent server for months while you get comfortable.
2. Click **Sign Up**
3. You can sign up with Google, GitHub, or email + password
4. DigitalOcean will ask for a payment method (credit card or PayPal). This is normal, but with the $200 credit you won't be charged until the credits run out. The server we're creating costs about $12/month, so the credits last a long time.
5. You might need to verify your email. Check your inbox and click the confirmation link.

**You'll know it worked when** you're looking at the DigitalOcean dashboard: a mostly empty page with a green "Create" button in the top right.

If they already have AWS, GCP, Hetzner, or another provider and prefer that, adapt the instructions. The server specs are the same: Ubuntu 24.04, 1 GB RAM minimum, SSH key authentication. The DigitalOcean steps below are the most beginner-friendly, so default to those unless they have a reason to use something else.

---

## ACT 1 — YOUR SERVER

**What we're doing:** Creating your own computer in the cloud (a "server") that runs 24/7, even when your laptop is closed. This is where your AI agent will live. We'll also lock the front door so only you can get in.

### Step 1: Create an SSH key

Before we create the server, we need a way to prove who you are to it. An SSH key is like a physical house key: your computer holds the key, the server holds the lock. It's more secure than a password because there's nothing to guess or steal.

**Do this step for them.** Generate the SSH key via Bash tool (not interactively, use `-N ""` for no passphrase so it doesn't prompt). Then read the public key and display it to the user:

```bash
# Generate the key (non-interactive, no passphrase prompt)
ssh-keygen -t ed25519 -C "agent-server" -f ~/.ssh/id_ed25519_agent -N ""

# Read the public key to display
cat ~/.ssh/id_ed25519_agent.pub
```

After running this, tell the user:
- "I just created an SSH key for you. It's stored on your computer at `~/.ssh/id_ed25519_agent`."
- "An SSH key is like a house key: your computer has the key, the server will have the lock. Nobody can log in without this file."
- Display the public key and tell them: "Copy this line. You'll paste it into DigitalOcean in the next step."
- Also tell them to save the public key in their password manager for safekeeping.

**If a key already exists at that path,** don't overwrite it. Ask the user if they want to use the existing one or create a new one with a different name.

### Step 2: Create the server

Now we go to DigitalOcean and create the actual server. Every click is listed here:

1. Go to **digitalocean.com** and log in
2. Click the green **Create** button in the top-right corner
3. Select **Droplets** from the dropdown (a "droplet" is DigitalOcean's name for a server)

You'll land on a configuration page. Here's what to pick:

4. **Choose a region:** Pick the one closest to where you live. This affects how fast your WhatsApp messages get to the server and back. If you're in North America, "New York" or "San Francisco" are good defaults. Europe: "Amsterdam" or "Frankfurt."
5. **Choose an image:** Click **Ubuntu**, then select **24.04 (LTS)**. LTS means "long-term support": it gets security updates for years.
6. **Choose size:** Under "Shared CPU," pick the **Regular** tab, then the **$12/mo** option (2 GB RAM, 1 CPU, 50 GB SSD). OpenClaw needs at least 2 GB RAM to run reliably. Don't pick the $6/mo option (1 GB): it's too tight and you'll hit memory issues.
7. **Choose authentication method:** Select **SSH Key** (not Password). Click **New SSH Key**. A box appears: paste the public key you copied in Step 1. Give it a name like "my-laptop." Click **Add SSH Key.**
8. **Hostname:** Change it to something you'll recognize, like `agent-server`
9. Leave everything else as default
10. Click **Create Droplet**

It takes 30-60 seconds. When it's done, DigitalOcean shows your server's **IP address**: four numbers separated by dots, like `167.71.23.45`. Copy this IP. You'll use it to connect.

**If you're not sure where the IP is:** On the Droplets page, your new server appears as a row. The IP address is shown right next to the server name.

### Step 3: Connect to your server

**Do this for them.** Ask the user for their droplet IP, then connect via Bash tool yourself:

```bash
ssh -i ~/.ssh/id_ed25519_agent -o StrictHostKeyChecking=accept-new root@THEIR_IP "echo 'Connected successfully' && hostname"
```

Tell them: "I just connected to your server. It's live and accepting your SSH key."

If it fails, troubleshoot:
- "Permission denied" → SSH key wasn't added correctly in DigitalOcean. They may need to destroy the droplet and recreate with the key.
- "Connection timed out" → Server still booting. Wait 30 seconds, retry.
- "Connection refused" → SSH not running yet. Wait and retry.

### Step 4: Lock the front door

**Do this entire step for them via Bash tool.** This is the most dangerous step if done wrong. Follow this exact sequence. Read `references/vps-hardening.md` for context, but the order below is critical.

**CRITICAL SAFETY RULES:**
- **NEVER test root rejection by SSH-ing as root.** This triggers fail2ban and locks you out. Verify root is disabled by checking the SSH config file instead.
- **ALWAYS verify the new user can SSH in BEFORE disabling root login.**
- **ALWAYS add the connecting IP to fail2ban's ignore list BEFORE enabling fail2ban.**
- **ALWAYS set a known password for the new user** so they can use DigitalOcean's web console as a fallback.
- The SSH service on Ubuntu 24.04 is called `ssh`, not `sshd`. Use `systemctl restart ssh`.

**Exact sequence (all via Bash tool, all as one SSH session where possible):**

**1. Update packages:**
```bash
ssh -i ~/.ssh/id_ed25519_agent root@IP "DEBIAN_FRONTEND=noninteractive apt update && DEBIAN_FRONTEND=noninteractive apt upgrade -y"
```

**2. Create user with a known password, add to sudo, copy SSH key:**
Use `AskUserQuestion` (header: "Username") to ask what username they want (recommend "agent"). Then:
```bash
ssh -i ~/.ssh/id_ed25519_agent root@IP bash -s <<'REMOTE'
# Create user
useradd -m -s /bin/bash USERNAME
echo "USERNAME:GENERATED_PASSWORD" | chpasswd
usermod -aG sudo USERNAME
# Copy SSH key
mkdir -p /home/USERNAME/.ssh
cp /root/.ssh/authorized_keys /home/USERNAME/.ssh/
chown -R USERNAME:USERNAME /home/USERNAME/.ssh
chmod 700 /home/USERNAME/.ssh
chmod 600 /home/USERNAME/.ssh/authorized_keys
# Allow sudo without password for setup
echo "USERNAME ALL=(ALL) NOPASSWD:ALL" > /etc/sudoers.d/USERNAME
REMOTE
```
Generate a random password (e.g., via `openssl rand -base64 12`). Tell the user: "I created your account. Save this password in your password manager as a backup. You probably won't need it, but if we ever lose SSH access, this lets you log in through DigitalOcean's web console." Display the password clearly.

**3. VERIFY the new user can SSH in (before changing anything else):**
```bash
ssh -i ~/.ssh/id_ed25519_agent USERNAME@IP "whoami && sudo whoami"
```
Both must succeed. If this fails, DO NOT proceed. Debug first.

**4. Install fail2ban with current IP whitelisted:**
```bash
ssh -i ~/.ssh/id_ed25519_agent USERNAME@IP bash -s <<'REMOTE'
sudo apt install -y fail2ban
# Get the IP we're connecting from and whitelist it
MY_IP=$(echo $SSH_CLIENT | awk '{print $1}')
sudo tee /etc/fail2ban/jail.local > /dev/null <<EOF
[sshd]
enabled = true
port = ssh
filter = sshd
logpath = /var/log/auth.log
maxretry = 5
bantime = 3600
findtime = 600
ignoreip = 127.0.0.1/8 ::1 ${MY_IP}
EOF
sudo systemctl enable fail2ban
sudo systemctl restart fail2ban
REMOTE
```

**5. Set up the firewall:**
```bash
ssh -i ~/.ssh/id_ed25519_agent USERNAME@IP bash -s <<'REMOTE'
sudo ufw default deny incoming
sudo ufw default allow outgoing
sudo ufw allow ssh
echo "y" | sudo ufw enable
REMOTE
```

**6. ONLY NOW harden SSH (disable root login + password auth):**
```bash
ssh -i ~/.ssh/id_ed25519_agent USERNAME@IP bash -s <<'REMOTE'
sudo sed -i 's/^#\?PermitRootLogin.*/PermitRootLogin no/' /etc/ssh/sshd_config
sudo sed -i 's/^#\?PasswordAuthentication.*/PasswordAuthentication no/' /etc/ssh/sshd_config
sudo sed -i 's/^#\?PubkeyAuthentication.*/PubkeyAuthentication yes/' /etc/ssh/sshd_config
sudo systemctl restart ssh
REMOTE
```

**7. VERIFY everything still works (do NOT try to SSH as root):**
```bash
# Verify user SSH still works
ssh -i ~/.ssh/id_ed25519_agent USERNAME@IP "echo 'SSH OK' && sudo ufw status && sudo fail2ban-client status sshd"

# Verify root is disabled by checking the config (NOT by attempting SSH as root)
ssh -i ~/.ssh/id_ed25519_agent USERNAME@IP "grep '^PermitRootLogin' /etc/ssh/sshd_config"
# Should show: PermitRootLogin no
```

**8. Set timezone:**
Use `AskUserQuestion` (header: "Timezone") to ask their timezone. Then:
```bash
ssh -i ~/.ssh/id_ed25519_agent USERNAME@IP "sudo timedatectl set-timezone TIMEZONE"
```

**9. Enable auto-updates:**
```bash
ssh -i ~/.ssh/id_ed25519_agent USERNAME@IP "sudo apt install -y unattended-upgrades && echo 'unattended-upgrades installed'"
```

Tell the user what you did: "Your server is locked down. Here's what's in place: only your SSH key can log in, the firewall only allows SSH traffic, failed login attempts get auto-blocked, and security patches install automatically. Root login is disabled. Your backup password is in your password manager in case you ever need DigitalOcean's web console."

**You'll know it worked when** the step 7 verification shows SSH OK, firewall active, fail2ban running, and PermitRootLogin no.

### Step 5: Install the building blocks

Still on the server, install the software your agents will need:

```bash
# Node.js 24 (OpenClaw needs this)
curl -fsSL https://deb.nodesource.com/setup_24.x | sudo bash -
sudo apt-get install -y nodejs

# Python (for writing your own agents)
sudo apt-get install -y python3 python3-pip python3-venv

# SQLite (a lightweight database your agents can use)
sudo apt-get install -y sqlite3

# Git (version control)
sudo apt-get install -y git
```

**You'll know it worked when** `node --version` shows v24+, `python3 --version` shows 3.11+.

**Quality gate:** They have a server they can log into, it rejects root login, the firewall is on, and the building blocks are installed. "Next up: installing your AI assistant. Ready?"

---

## ACT 2 — YOUR ASSISTANT

**What we're doing:** Installing OpenClaw (the AI gateway software), registering the API key, configuring the agent's personality, and locking everything down with kernel-level security. We do NOT connect a messaging channel yet. The channel connection is the payoff in Act 3, and it should only happen after security is in place.

### Step 1: Install OpenClaw

On your server:

```bash
sudo npm install -g openclaw@latest
```

### Step 2: Run the non-interactive setup

**Do this for them via Bash tool.** Ask the user to paste their Anthropic API key (the `sk-ant-...` key they saved in their password manager). Then run the onboard command non-interactively via SSH. This handles workspace creation, API key registration, auth profile setup, and daemon installation in one shot. No interactive wizard needed.

```bash
ssh -i ~/.ssh/id_ed25519_agent USERNAME@IP "openclaw onboard --non-interactive --accept-risk --anthropic-api-key 'THEIR_API_KEY_HERE' --install-daemon --skip-channels --skip-skills --gateway-auth token --json 2>&1"
```

Key flags:
- `--non-interactive` + `--accept-risk`: runs without prompts
- `--anthropic-api-key`: registers the key in OpenClaw's auth store (no manual auth-profiles.json needed)
- `--install-daemon`: installs and enables the systemd user service so OpenClaw starts on reboot
- `--skip-channels`: we connect the messaging channel in Act 3, after security is in place
- `--skip-skills`: can install skills later
- `--gateway-auth token`: auto-generates a gateway access token
- `--json`: outputs structured summary for verification

After it runs, extract the gateway token from the config and tell the user to save it in their password manager:
```bash
ssh -i ~/.ssh/id_ed25519_agent USERNAME@IP "cat ~/.openclaw/openclaw.json" | python3 -c "import sys,json; d=json.load(sys.stdin); print(d['gateway']['auth']['token'])"
```

Also set the API key as an environment variable as a fallback:
```bash
ssh -i ~/.ssh/id_ed25519_agent USERNAME@IP 'echo "export ANTHROPIC_API_KEY=\"THEIR_KEY\"" >> ~/.bashrc'
```

Verify the daemon is running:
```bash
ssh -i ~/.ssh/id_ed25519_agent USERNAME@IP "systemctl --user status openclaw-gateway 2>&1 | head -10"
```

**You'll know it worked when** the JSON output shows `"ok": true`, the daemon status shows `active (running)`, and you have the gateway token saved.

### Step 4: Tell it who you are

Create the file that shapes how your assistant behaves. This goes in `~/.openclaw/workspace/AGENTS.md`:

```markdown
# Agent Instructions

## Who I Am
[Their name, what they do, how they work]

## How to Respond
- Keep messages short (WhatsApp isn't email)
- Use plain language
- Ask before taking action on anything external

## What You Can Do
- Answer questions using my context
- Run scheduled tasks (morning brief, reminders)
- Look things up

## What You Cannot Do
- Send messages to anyone except me
- Access files outside the workspace
- Make purchases or commitments on my behalf
```

Build this interactively with the user. Push back if their description is vague. "I'm a consultant" tells the AI nothing. "I run a 6-person landscaping company in Austin, I manage client projects in Linear, and my busiest day is Monday" is something the AI can actually use.

### Step 5: Install the security sandbox (nono)

**This happens BEFORE connecting any messaging channel.** We lock the cage first, then let it talk to the outside world.

nono is a security tool that restricts what the AI can access at the deepest level of the operating system. It's not a setting the AI can talk its way out of. It's a physical lock enforced by the kernel. Even if the AI hallucinates or gets prompt-injected, it physically cannot access SSH keys, system files, or unauthorized services.

**Do this entire act for them via SSH.** All commands run via Bash tool.

**CRITICAL: Known pitfalls to avoid (learned from production deployments):**
- **Extend `default`, NOT `openclaw`.** The built-in `openclaw` profile has a Landlock conflict: it allows `$HOME/.local` broadly but denies `$HOME/.local/share/keyrings`. Linux Landlock can't enforce denies inside allowed parents, so nono refuses to start. Use `default` and add paths manually.
- **Don't use `secret-tool` or `gnome-keyring` on a headless server.** No display server means keyring tools don't work. Store the API key in a permissions-locked file instead.
- **Don't use `network_profile: "minimal"` or `"standard"`.** These names don't exist in nono. Remove `network_profile` entirely (inherits default which allows network).
- **All filesystem paths must be directories, not files.** nono rejects individual file paths.
- **`nono why` uses `--path`, not `-- cat`.** Correct syntax: `nono why --profile X --path /path/to/check`.
- **OpenClaw is a user service.** Use `systemctl --user`, not `systemctl` or `sudo systemctl`.
- **`env_credentials` format:** use `"env://VAR_NAME": "VAR_NAME"`, not `custom_credentials` or `credentials` arrays.

### Step 1: Install nono

```bash
ssh -i ~/.ssh/id_ed25519_agent USERNAME@IP bash -s <<'REMOTE'
VERSION=$(curl -sI https://github.com/always-further/nono/releases/latest | grep -i location | grep -oP 'v\K[0-9.]+')
ARCH=$(dpkg --print-architecture)
wget -q "https://github.com/always-further/nono/releases/download/v${VERSION}/nono-cli_${VERSION}_${ARCH}.deb"
sudo dpkg -i "nono-cli_${VERSION}_${ARCH}.deb"
rm -f "nono-cli_${VERSION}_${ARCH}.deb"
nono setup --check-only 2>&1
REMOTE
```

**You'll know it worked when** you see green checks for Landlock support. Ubuntu 24.04 has kernel 6.x, so this should pass.

### Step 2: Store the API key securely

Don't use system keyrings on a headless server. Instead, store the key in a permissions-locked file that only the user can read:

```bash
ssh -i ~/.ssh/id_ed25519_agent USERNAME@IP bash -s <<'REMOTE'
mkdir -p ~/.config/nono/secrets
chmod 700 ~/.config/nono/secrets

# Write the API key (substitute their actual key)
echo -n "THEIR_API_KEY_HERE" > ~/.config/nono/secrets/anthropic_api_key
chmod 600 ~/.config/nono/secrets/anthropic_api_key

# Also add to .bashrc so it's available for manual use
echo 'export ANTHROPIC_API_KEY=$(cat ~/.config/nono/secrets/anthropic_api_key 2>/dev/null)' >> ~/.bashrc
REMOTE
```

### Step 3: Create the sandbox profile

This is the exact profile that works in production. Do NOT modify the structure without testing.

```bash
ssh -i ~/.ssh/id_ed25519_agent USERNAME@IP bash -s <<'REMOTE'
mkdir -p ~/.config/nono/profiles

cat > ~/.config/nono/profiles/agent-server.json <<'EOF'
{
  "extends": "default",
  "meta": {
    "name": "agent-server",
    "description": "Agent server sandbox for OpenClaw gateway"
  },
  "filesystem": {
    "allow": [
      "$HOME/.openclaw",
      "/tmp"
    ],
    "read": [
      "/etc/ssl",
      "/etc",
      "/usr/lib/node_modules",
      "/usr/local/lib"
    ],
    "execute": [
      "/usr/bin",
      "/usr/local/bin",
      "/usr/lib/node_modules"
    ]
  },
  "env_credentials": {
    "env://ANTHROPIC_API_KEY": "ANTHROPIC_API_KEY"
  }
}
EOF

# Validate the profile
nono policy validate ~/.config/nono/profiles/agent-server.json 2>&1
REMOTE
```

Walk them through what each section means:
- `extends: "default"`: starts from nono's base rules (44 sensitive paths blocked, 46 dangerous commands blocked)
- `allow`: the agent can read AND write here (OpenClaw workspace + temp files)
- `read`: the agent can look at these but not change them (SSL certs, system config, Node.js modules)
- `execute`: the agent can run programs from here (system binaries, OpenClaw itself)
- `env_credentials`: injects the API key as an environment variable when running inside the sandbox

### Step 4: Wire OpenClaw to run inside the sandbox

OpenClaw installs as a **user service** (not a system service). Create a systemd override:

```bash
ssh -i ~/.ssh/id_ed25519_agent USERNAME@IP bash -s <<'REMOTE'
# Create the override directory
mkdir -p ~/.config/systemd/user/openclaw-gateway.service.d

# Write the sandbox override
cat > ~/.config/systemd/user/openclaw-gateway.service.d/sandbox.conf <<EOF
[Service]
ExecStart=
ExecStart=/usr/bin/nono run --profile agent-server -- /usr/bin/openclaw gateway
EnvironmentFile=/home/USERNAME/.config/nono/secrets/anthropic_api_key
Environment=ANTHROPIC_API_KEY=%h/.config/nono/secrets/anthropic_api_key
EOF

# Load the API key into the environment for the service
# The cleanest way: create an env file the service can read
echo "ANTHROPIC_API_KEY=$(cat ~/.config/nono/secrets/anthropic_api_key)" > ~/.config/nono/secrets/env
chmod 600 ~/.config/nono/secrets/env

# Update the override to use the env file
cat > ~/.config/systemd/user/openclaw-gateway.service.d/sandbox.conf <<EOF
[Service]
ExecStart=
ExecStart=/usr/bin/nono run --profile agent-server -- /usr/bin/openclaw gateway
EnvironmentFile=/home/USERNAME/.config/nono/secrets/env
EOF

systemctl --user daemon-reload
systemctl --user restart openclaw-gateway
sleep 8
systemctl --user status openclaw-gateway 2>&1 | head -10
REMOTE
```

Replace USERNAME with their actual username in the EnvironmentFile path.

**You'll know it worked when** the status shows `active (running)`.

If it shows `failed` or `activating (auto-restart)`, check logs:
```bash
ssh -i ~/.ssh/id_ed25519_agent USERNAME@IP "journalctl --user -u openclaw-gateway --no-pager -n 15 2>&1"
```

Common issues at this point:
- "Command execution failed": a path in the profile is wrong. Check that `/usr/bin/openclaw` exists with `which openclaw`.
- "Landlock deny-overlap": you're extending `openclaw` instead of `default`. Fix the profile.
- "Network profile not found": remove any `network_profile` line from the profile JSON.

### Step 5: Prove it works

Verify the sandbox is enforcing. Use `nono why --path` (NOT `nono why -- cat`):

```bash
ssh -i ~/.ssh/id_ed25519_agent USERNAME@IP bash -s <<'REMOTE'
echo "=== SSH key access (should be DENIED) ==="
nono why --profile agent-server --path ~/.ssh/authorized_keys 2>&1 | head -4

echo ""
echo "=== Workspace access (should be ALLOWED) ==="
nono why --profile agent-server --path ~/.openclaw/workspace/AGENTS.md 2>&1 | head -4

echo ""
echo "=== Gateway status ==="
openclaw status 2>&1
REMOTE
```

Expected results:
- SSH keys: **DENIED** (reason: sensitive_path, blocked by security policy)
- Workspace: **ALLOWED** (reason: granted_path, read+write)
- Gateway: running, channel connected

Send a message to the agent on their chosen channel. It should respond normally (the sandbox allows what it needs, blocks everything else).

**Quality gate:** SSH keys are denied, workspace is allowed, gateway is running inside nono. No channel connected yet (that's intentional). "Next up: connecting your messaging channel. This is the moment it becomes real. Ready?"

---

## ACT 3 — GO LIVE

**What we're doing:** Connecting your messaging channel and sending your first message. Everything is installed, configured, and locked down. Now we plug in the phone line.

All channels are fully bidirectional: you can message the agent AND the agent can message you. This isn't a notification system. It's an assistant you can talk to.

Based on what they chose in the prerequisites questions, set up their channel. **Do as much as possible for them via Bash tool.**

**Telegram (recommended path, one manual step):**
Fully real-time, bidirectional chat. One manual step: creating the bot through Telegram.

Walk the user through:
1. Install Telegram on their phone if they don't have it (free, available everywhere)
2. Open Telegram, search for **@BotFather** (Telegram's official bot-creation tool)
3. Send `/newbot`
4. Pick a name (e.g., "My Agent") and a username (e.g., `myname_agent_bot`)
5. BotFather gives them a token. It looks like `123456:ABC-DEF1234ghIkl-zyx57W2v1u123ew11`. Copy it.
6. Store the token in their password manager
7. Paste the token back to you

You handle the rest via SSH:
8. Add the Telegram channel to OpenClaw with the bot token
9. Restart the OpenClaw gateway (it's already running inside nono from Act 2)
10. Have the user open their bot in Telegram and send "hello"
11. The agent should respond. First message confirmed.

**WhatsApp (one manual step: QR scan):**
Fully bidirectional and real-time. If WhatsApp is what they use every day, it's the right choice.

Walk them through:
1. You (the model) add the WhatsApp channel to OpenClaw via SSH
2. Tell them to open a new terminal window and run: `ssh -i ~/.ssh/id_ed25519_agent USERNAME@IP "openclaw channels login --channel whatsapp"`
3. A QR code appears in their terminal
4. On their phone: WhatsApp > Settings > Linked Devices > Link a Device > Scan the QR code
5. Once it says "connected," they come back to you and confirm
6. Send a test message to verify it works

**Email (bidirectional, more setup):**
The agent gets its own email address. The user emails it to give tasks, it emails back.

Setup requires a dedicated Gmail for the agent:
1. Walk the user through creating a new Gmail account (e.g., `firstname-agent@gmail.com`)
2. Enable 2-Step Verification on the new Gmail (Settings > Security)
3. Generate an App Password (Settings > Security > App passwords, select "Mail")
4. Store the app password in their password manager
5. Tell you the Gmail address and app password

You handle the rest via SSH:
6. Configure OpenClaw's email channel with IMAP (incoming) + SMTP (outgoing) using the Gmail credentials
7. Restart the gateway
8. Send a test email from the agent Gmail to their personal inbox
9. Have them reply to test bidirectional flow

**Slack:**
The user creates a Slack app in their workspace's admin panel. Walk them through getting the bot token and app token. You configure the rest via SSH. Fully bidirectional.

**Quality gate:** They send a message to their agent and get a response back. This is the moment. Everything before this was infrastructure. This is the payoff. "Next up: building your first automated agent. Ready?"

---

## ACT 4 — WHAT YOUR AGENT CAN DO

**What we're doing:** Now that your agent is set up, secured, and connected, let's explore what it can actually do. This isn't a coding tutorial. It's a guided tour of what you just built, with things you can try right now.

### Try these right now

Walk the user through each of these by having them message their agent directly on their chosen channel. These are conversations, not code.

**1. Ask it something about your work:**
> "I have a meeting with a potential client tomorrow about their website redesign. What questions should I ask to understand their needs?"

Your agent responds with tailored advice. This is the baseline: you have a smart assistant you can reach from your phone, anytime, and it responds in seconds.

**2. Ask it to research something:**
> "What are the pros and cons of Shopify vs WooCommerce for a small bakery with 20 products?"

The agent can search the web, synthesize information, and give you a clear answer. No more opening 15 browser tabs.

**3. Ask it to draft something:**
> "Write a professional email to a client named Sarah explaining that the project will be delayed by one week due to a supplier issue. Keep it friendly but honest."

Copy the result, paste it into your email. Or ask it to adjust the tone.

**4. Ask it to help you plan:**
> "I need to move my family into a new house in 3 weeks. Break this down into a week-by-week checklist."

Your agent is available 24/7. At 11 PM when you remember something, text it. It'll be there.

### Set up a scheduled check-in

OpenClaw has built-in cron scheduling. You can set up recurring messages without writing any code. **Do this for them via SSH:**

```bash
ssh -i ~/.ssh/id_ed25519_agent USERNAME@IP "openclaw cron add --schedule '0 8 * * 1-5' --message 'Good morning! What are your top 3 priorities today? Reply and I will help you plan your day.' 2>&1"
```

This sends them a message every weekday at 8 AM asking about their priorities. When they reply, the agent helps them plan. No Python, no scripts, just a scheduled prompt.

Use `AskUserQuestion` (header: "Daily check-in") to ask what time works for them and what they'd like the prompt to say. Common options:
- "Morning priorities (Recommended)" — "Good morning! What are your top 3 priorities today?"
- "End of day review" — "How did today go? Anything that needs follow-up tomorrow?"
- "Weekly planning" — Runs Sunday evening: "What does your week ahead look like? Any prep you need?"

### Explore OpenClaw skills

OpenClaw has a skills marketplace. Skills are add-ons that give your agent new capabilities. Browse what's available:

```bash
ssh -i ~/.ssh/id_ed25519_agent USERNAME@IP "openclaw skills search 2>&1 | head -30"
```

Walk them through installing one that matches their needs. Common useful skills:
- **Web search** (if not already built in)
- **File management** (upload/download files through chat)
- **Calendar integration** (if they use Google Calendar)
- **Reminders** (set timed reminders through chat)

### What people actually use this for

Give them concrete ideas based on what they told you about their work earlier. Tailor these to their domain:

**If they run a business:**
- "Remind me every Friday at 4 PM to send invoices"
- "When I message you a client name, give me a summary of our last conversation" (once they add entity files)
- "Draft a proposal outline for [project description]"

**If they manage a team:**
- "Help me write a standup update based on what I did today"
- "Summarize this meeting transcript" (paste it in)
- "Draft a performance review outline for someone who's strong technically but needs to improve communication"

**If they're a freelancer/consultant:**
- "I just finished a call with [client]. Help me write follow-up notes"
- "What should I charge for [type of project]? Help me think through pricing."
- "Draft a LinkedIn post about [topic I'm working on]"

**If they're just getting started:**
- "Teach me about [topic] like I'm a smart 12-year-old"
- "I want to start [hobby/project]. What do I need to know?"
- "Help me write a better bio for [platform]"

The point is: this is YOUR assistant. It's available whenever you need it, it knows your context (from the AGENTS.md you wrote), and it's running on a server you own. The more you use it, the more useful it becomes.

### What comes next (when you're ready)

You don't need to do any of this right now. Live with your agent for a week. Get comfortable. When you're ready to go deeper:

- **Add more channels.** Connect a second messaging app (WhatsApp + Telegram, or add email as a backup) so your agent can reach you multiple ways.
- **Build custom agents.** Write Python scripts that run on timers (morning briefs, follow-up checkers, content digests). See `references/agent-patterns.md` for starter code.
- **Add memory.** Create entity files for the people you work with so your agent knows context about your relationships.
- **Connect your calendar.** Link Google Calendar so your agent can prep you before meetings and flag scheduling conflicts.
- **Run Claude Code on the server.** SSH in and run `claude` to do development work directly on your agent's home turf.

---

## What Will Burn You

- **Testing root rejection by SSH-ing as root.** This is the #1 way to brick a fresh server. The failed SSH attempt triggers fail2ban, which bans your IP, which locks you out of everything. Verify root is disabled by checking the config file (`grep PermitRootLogin /etc/ssh/sshd_config`), never by attempting to connect as root.
- **Skipping the sandbox.** "I'll add security later." No. The sandbox goes on before the first channel connects. This is the lesson everyone learns the hard way exactly once.
- **Exposing the gateway.** OpenClaw listens on localhost. Access it through SSH, not by opening port 18789 to the internet.
- **API keys in code.** Use nono credential injection. Never hardcode keys in scripts.
- **Not registering the API key in OpenClaw's auth store.** Setting `ANTHROPIC_API_KEY` in the environment isn't always enough. OpenClaw looks for `auth-profiles.json` first. If it's missing, you get "No API key found for provider anthropic." Write the file directly (see Act 2 Step 3).

---

You now have a personal AI assistant running on your own server, secured at the kernel level, reachable from your phone. Most people never get this far. If you want to go deeper, or if something breaks, reach out to **simon@lemonbrand.io**.

## Related Skills

- Chains FROM: `personal-os` (context layer and entity files feed directly into the agent workspace)
- Chains TO: `create-ralph-loop` (autonomous multi-day projects), `sync-agent` (local-to-VPS context sync)
- Complements: `consulting-workspace` (client workspace structure maps to agent entity files)
