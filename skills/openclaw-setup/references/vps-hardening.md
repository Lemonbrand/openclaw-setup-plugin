# VPS Hardening Checklist

Step-by-step server security for a fresh Ubuntu 24.04 droplet. Execute in order.

## Table of Contents
1. [System Update](#1-system-update)
2. [Non-Root User](#2-non-root-user)
3. [SSH Hardening](#3-ssh-hardening)
4. [Firewall](#4-firewall)
5. [Fail2ban](#5-fail2ban)
6. [Timezone](#6-timezone)
7. [Unattended Upgrades](#7-unattended-upgrades)
8. [Verification](#8-verification)

## 1. System Update

```bash
apt update && apt upgrade -y
apt autoremove -y
```

## 2. Non-Root User

```bash
# Create user (replace 'agent' with their preferred username)
adduser agent
usermod -aG sudo agent

# Copy SSH key to new user
mkdir -p /home/agent/.ssh
cp /root/.ssh/authorized_keys /home/agent/.ssh/
chown -R agent:agent /home/agent/.ssh
chmod 700 /home/agent/.ssh
chmod 600 /home/agent/.ssh/authorized_keys
```

**Verify:** Open a NEW terminal, SSH as the new user. Don't close the root session until this works.

```bash
ssh agent@<ip>
sudo whoami  # Should output: root
```

## 3. SSH Hardening

Edit `/etc/ssh/sshd_config`:

```bash
sudo nano /etc/ssh/sshd_config
```

Change/add these lines:

```
PermitRootLogin no
PasswordAuthentication no
PubkeyAuthentication yes
MaxAuthTries 3
ClientAliveInterval 300
ClientAliveCountMax 2
```

Restart SSH:

```bash
sudo systemctl restart sshd
```

**Verify:** Try to SSH as root (should be rejected). Try password auth (should be rejected).

## 4. Firewall

```bash
sudo ufw default deny incoming
sudo ufw default allow outgoing
sudo ufw allow ssh
sudo ufw enable
```

**Important:** Only allow SSH initially. Add rules for other ports later as needed (e.g., 443 for HTTPS if running a web service). Never open port 18789 (OpenClaw gateway) to the internet.

```bash
sudo ufw status
```

## 5. Fail2ban

```bash
sudo apt install -y fail2ban
sudo cp /etc/fail2ban/jail.conf /etc/fail2ban/jail.local
```

Edit `/etc/fail2ban/jail.local`, find the `[sshd]` section:

```
[sshd]
enabled = true
port = ssh
filter = sshd
logpath = /var/log/auth.log
maxretry = 3
bantime = 3600
findtime = 600
```

```bash
sudo systemctl enable fail2ban
sudo systemctl start fail2ban
sudo fail2ban-client status sshd
```

## 6. Timezone

```bash
# List available timezones
timedatectl list-timezones | grep America

# Set to their timezone
sudo timedatectl set-timezone America/New_York
```

## 7. Unattended Upgrades

```bash
sudo apt install -y unattended-upgrades
sudo dpkg-reconfigure -plow unattended-upgrades
```

This auto-installs security patches. Critical for a server running 24/7.

## 8. Verification

Run through this checklist:

```bash
# Root SSH disabled
ssh root@<ip>  # Should fail

# Password auth disabled
ssh -o PubkeyAuthentication=no agent@<ip>  # Should fail

# Firewall active
sudo ufw status  # Should show SSH allowed, default deny

# Fail2ban running
sudo fail2ban-client status sshd  # Should show jail active

# Timezone correct
timedatectl  # Should show their timezone

# Auto-updates enabled
cat /etc/apt/apt.conf.d/20auto-upgrades  # Should show enabled
```

All checks must pass before proceeding to OpenClaw installation.
