# nono.sh Profile Reference

Custom security profiles for agent server setups. Profiles define the kernel-level sandbox rules that restrict what an agent process can access.

## Table of Contents
1. [Profile Location](#profile-location)
2. [Built-in Profiles](#built-in-profiles)
3. [Profile Structure](#profile-structure)
4. [Filesystem Rules](#filesystem-rules)
5. [Network Rules](#network-rules)
6. [Credential Injection](#credential-injection)
7. [Common Profiles](#common-profiles)
8. [Debugging](#debugging)

## Profile Location

User profiles: `~/.config/nono/profiles/<name>.json`
Built-in profiles: installed with nono (read-only)

## Built-in Profiles

| Profile | Use Case |
|---------|----------|
| `default` | Minimal sandbox, CWD only |
| `claude-code` | Claude Code sessions (workspace + claude config) |
| `openclaw` | OpenClaw gateway (workspace + gateway config) |
| `python-dev` | Python development (pip, venv, site-packages) |
| `node-dev` | Node.js development (npm, node_modules) |

Always extend a built-in profile rather than building from scratch:

```json
{
  "extends": "openclaw"
}
```

## Profile Structure

```json
{
  "extends": "openclaw",
  "description": "Human-readable description",
  "filesystem": {
    "allow": ["paths with read+write"],
    "read": ["paths with read-only"],
    "write": ["paths with write-only"]
  },
  "network": {
    "profile": "minimal|developer|enterprise",
    "allow_domains": ["extra.domain.com"],
    "block": false
  },
  "credentials": [...],
  "rollback": {
    "enabled": true,
    "max_sessions": 10,
    "max_storage_gb": 2
  }
}
```

## Filesystem Rules

**Path variables:**
- `$HOME` — user home directory
- `$WORKDIR` — current working directory
- `$TMPDIR` — system temp directory
- `$XDG_CONFIG_HOME` — `~/.config` by default
- `$UID` — numeric user ID

**Always blocked (cannot override):**
- `~/.ssh/` (SSH keys)
- `~/.aws/`, `~/.azure/`, `~/.gcloud/` (cloud credentials)
- `~/.gnupg/` (GPG keys)
- `~/.docker/config.json` (container registry auth)
- `~/.npmrc`, `~/.pypirc` (package manager tokens)
- Browser profile directories
- Shell history files

**Rules:**

```json
{
  "filesystem": {
    "allow": [
      "$HOME/.openclaw/workspace",
      "$HOME/agent-workspace"
    ],
    "read": [
      "$HOME/.openclaw/openclaw.json",
      "/etc/ssl/certs",
      "/usr/local/bin/gws",
      "/usr/bin/gh"
    ]
  }
}
```

- `allow` = read + write (recursive for directories)
- `read` = read-only (recursive for directories)
- `write` = write-only (rare, for log files)
- Single files use exact path, directories are recursive

## Network Rules

**Profile-based (recommended):**

```json
{
  "network": {
    "profile": "minimal"
  }
}
```

| Profile | What's Allowed |
|---------|---------------|
| `minimal` | LLM APIs only (api.anthropic.com, api.openai.com, etc.) |
| `developer` | LLMs + package registries + GitHub + docs sites |
| `enterprise` | Developer + cloud providers (AWS, GCP, Azure) |

**Custom domain additions:**

```json
{
  "network": {
    "profile": "minimal",
    "allow_domains": [
      "oauth2.googleapis.com",
      "www.googleapis.com",
      "api.github.com"
    ]
  }
}
```

**Always blocked (cannot override):**
- `169.254.169.254` (cloud metadata)
- `metadata.google.internal`
- IPv4/IPv6 link-local ranges

## Credential Injection

Two methods. Proxy injection is preferred (agent never sees the raw key).

### Proxy Injection (Recommended)

The nono proxy intercepts HTTP requests and injects credentials before forwarding. The sandboxed process only sees a placeholder.

```json
{
  "credentials": [
    {
      "name": "anthropic_api_key",
      "source": "keyring",
      "inject": "proxy",
      "header": "x-api-key",
      "domains": ["api.anthropic.com"]
    }
  ]
}
```

**Storing credentials:**

```bash
# Linux (Secret Service / gnome-keyring)
echo -n "sk-ant-..." | secret-tool store --label="nono: anthropic_api_key" \
    service nono username anthropic_api_key target default

# macOS (Keychain)
security add-generic-password -s "nono" -a "anthropic_api_key" -w "sk-ant-..."
```

**Supported sources:**
- `keyring` — system keyring (Secret Service on Linux, Keychain on macOS)
- `1password` — 1Password (`op://vault/item/field`)
- `env` — parent process environment (least secure, use only for dev)

### Environment Injection

Secrets loaded before sandbox applies, then keyring access is revoked:

```json
{
  "credentials": [
    {
      "name": "GITHUB_TOKEN",
      "source": "keyring",
      "inject": "env"
    }
  ]
}
```

## Common Profiles

### Agent Server (Basic)

For a server running OpenClaw + cron agents with Google Workspace and GitHub access:

```json
{
  "extends": "openclaw",
  "description": "Agent server with Google Workspace + GitHub CLI access",
  "filesystem": {
    "allow": [
      "$HOME/.openclaw/workspace",
      "$HOME/agent-workspace",
      "$HOME/logs"
    ],
    "read": [
      "$HOME/.openclaw/openclaw.json",
      "/etc/ssl/certs",
      "/usr/local/bin/gws",
      "/usr/bin/gh",
      "$HOME/.config/gws"
    ]
  },
  "network": {
    "profile": "minimal",
    "allow_domains": [
      "oauth2.googleapis.com",
      "www.googleapis.com",
      "gmail.googleapis.com",
      "api.github.com"
    ]
  },
  "credentials": [
    {
      "name": "anthropic_api_key",
      "source": "keyring",
      "inject": "proxy",
      "header": "x-api-key",
      "domains": ["api.anthropic.com"]
    }
  ],
  "rollback": {
    "enabled": true,
    "max_sessions": 10,
    "max_storage_gb": 2
  }
}
```

### Claude Code Session (VPS)

For running Claude Code on the VPS (interactive development):

```json
{
  "extends": "claude-code",
  "description": "Claude Code on VPS with agent workspace access",
  "filesystem": {
    "allow": [
      "$HOME/agent-workspace"
    ],
    "read": [
      "/usr/local/bin/gws",
      "/usr/bin/gh"
    ]
  }
}
```

## Debugging

**Check if a path is allowed:**

```bash
nono why --profile agent-server -- cat /path/to/file
# Outputs: ALLOWED (reason) or DENIED (reason)
```

**Trace what a command needs:**

```bash
nono learn -- python3 ~/agent-workspace/agents/morning_brief.py
# Outputs: list of all files/network the command accessed
# Use this output to build your profile
```

**Compare profiles:**

```bash
nono policy compare agent-server openclaw
# Shows differences between two profiles
```

**Validate profile:**

```bash
nono policy validate ~/.config/nono/profiles/agent-server.json
# Checks for syntax errors, conflicting rules
```
