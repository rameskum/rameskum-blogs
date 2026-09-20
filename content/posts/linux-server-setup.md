---
title: 'New Linux Server? Do These 8 Things Before Anything Else'
date: 2026-09-20T15:05:00-04:00
draft: false
ShowToc: true
description: >-
  The minimum-viable hardening for a fresh Ubuntu or Debian server: system
  updates, a non-root user, SSH key login with password auth disabled, UFW
  with deny-by-default rules, SSH login logging, and the essential tools
  (git, fzf, curl, htop) — in the right order so you never lock yourself out.
tags:
  - linux
  - ubuntu
  - debian
  - ssh
  - security
categories: article
keywords:
  - ubuntu server setup
  - debian server hardening
  - ssh key authentication
  - ufw firewall
  - disable ssh password
---

Every fresh VPS (Hetzner, DigitalOcean, your homelab Proxmox box) boots up
with the same problem: a root login over password, no firewall, and nothing
installed. Before you deploy anything, spend 15 minutes on this checklist.

**Order matters.** Firewall rules and SSH hardening can lock you out
permanently if done in the wrong sequence. Do the steps in this order and you
won't have to rescue-boot your way back in.

## 1. Update everything first

Connect as root and patch the box before touching anything else.

```bash
apt update && apt upgrade -y
```

A fresh image is usually weeks behind. Running services against a known-CVE
kernel is the easiest way to get owned in the first hour.

## 2. Create a non-root user with sudo

Running everything as root is how one typo becomes a disaster. Create your
day-to-day user now:

```bash
adduser deploy
usermod -aG sudo deploy
```

`adduser` prompts for a password. Pick a strong one — this password doubles as
your sudo password until SSH keys are set up (next step).

Verify the sudo group took:

```bash
id deploy
# uid=1000(deploy) gid=1000(deploy) groups=1000(deploy),27(sudo)
```

## 3. Add your SSH key (from your laptop)

**Do this before touching any SSH setting.** Generate a key locally if you
don't have one:

```bash
# on your laptop, not the server
ssh-keygen -t ed25519 -C "laptop"
ssh-copy-id deploy@<server-ip>
```

`ssh-copy-id` appends your public key to `~/.ssh/authorized_keys` on the
server. Test it in a **new terminal** before going further:

```bash
ssh deploy@<server-ip>
# should log in without asking for a password
```

Keep this first SSH session open for the rest of the guide. If something
goes wrong, it stays connected.

## 4. Harden SSH: disable passwords, enable verbose logging

Now that key login works, kill password auth. Password logins are what bots
brute-force all day. Edit `/etc/ssh/sshd_config` (or drop a file in
`/etc/ssh/sshd_config.d/`):

```bash
sudo nano /etc/ssh/sshd_config
```

Set these lines (uncomment if needed):

```conf
PasswordAuthentication no
ChallengeResponseAuthentication no
PermitRootLogin prohibit-password
LogLevel VERBOSE
```

- `PasswordAuthentication no` — keys only from here on.
- `ChallengeResponseAuthentication no` — closes the keyboard-interactive
  backdoor, which would otherwise still prompt for passwords.
- `PermitRootLogin prohibit-password` — root can only log in with a key.
- `LogLevel VERBOSE` — logs the key fingerprint on every login attempt.
  This is your SSH audit trail.

Validate the config before restarting — a typo here breaks all SSH logins:

```bash
sudo sshd -t && sudo systemctl restart sshd
```

`sshd -t` exits silently on success and prints errors on failure. Only
restart when it passes.

**From a second terminal, test a fresh login before closing your first
session.** If the second session works, you're safe.

Check who logged in recently using the fingerprints verbose logging records:

```bash
journalctl -u ssh --since today | grep -E "Accepted|Failed"
```

`Accepted publickey for deploy` lines confirm key logins; `Failed password`
lines show brute-force attempts bouncing off your disabled password auth.

## 5. Firewall: deny everything, allow only SSH

UFW is Ubuntu/Debian's simple frontend to iptables. Default-deny all inbound
traffic, then punch a hole only for SSH:

```bash
sudo apt install -y ufw
sudo ufw default deny incoming
sudo ufw default allow outgoing
sudo ufw allow ssh
sudo ufw enable
```

Answer `y` when it warns you about disrupting SSH — `allow ssh` was added
*before* enabling, so your session survives. This is why step 5 comes after
SSH is working.

Verify the ruleset:

```bash
sudo ufw status verbose
```

Expected output:

```
Status: active
Logging: on (low)
Default: deny (incoming), allow (outgoing), deny (routed)
New profiles: skip

To                         Action      From
--                         ------      ----
22/tcp                     ALLOW IN    Anywhere
22/tcp (v6)                ALLOW IN    Anywhere (v6)
```

If you later run a service (say, Caddy on 443), open it the same way:
`sudo ufw allow 443/tcp`. When in doubt, `sudo ufw status numbered` and
`sudo ufw delete <number>` to remove a rule.

## 6. Install the essentials

Small, useful, nothing dev-stack-yet:

```bash
sudo apt install -y git fzf curl htop fail2ban
```

- **git** — you'll need it for cloning and config next.
- **fzf** — the fuzzy finder. `Ctrl+R` history search and `Ctrl+T` file
  completion become instant-fuzzy once it's installed. Load the keybindings
  with `source /usr/share/doc/fzf/examples/key-bindings.bash` or add it to
  `~/.bashrc`.
- **curl** — every install script and health check starts with curl.
- **htop** — `top` for humans; check load and memory at a glance.
- **fail2ban** — watches the SSH logs from step 4 and bans IPs after
  repeated failed attempts. Works out of the box on Debian/Ubuntu with the
  `sshd` jail enabled by default:

```bash
sudo systemctl enable --now fail2ban
sudo fail2ban-client status sshd
```

## 7. Configure git

Even on a server, git needs an identity before its first commit:

```bash
git config --global user.name "Ramesh Kumar"
git config --global user.email "ramesh@example.com"
git config --global init.defaultBranch main
git config --global core.editor nano
git config --global credential.helper "cache --timeout=3600"
```

Check it stuck:

```bash
git config --global --list
```

## 8. Turn on automatic security updates (optional but free)

Ubuntu and Debian can patch themselves for security CVEs without you:

```bash
sudo apt install -y unattended-upgrades
sudo dpkg-reconfigure -plow unattended-upgrades
```

Answer `Yes`. It installs security updates daily and emails root on
failures. Kernel updates still need a reboot — `cat /var/run/reboot-required`
tells you when.

## The final checklist

Run through this before you call the server done:

```bash
# 1. key login works, password login refused
ssh -o PreferredAuthentications=password deploy@<server-ip>
# -> Permission denied (publickey) ... good

# 2. firewall active with ssh open
sudo ufw status | grep -q "22/tcp.*ALLOW" && echo "ufw ok"

# 3. sshd config valid, password auth off
sudo sshd -T | grep -Ei "passwordauthentication|permitrootlogin"

# 4. fail2ban watching ssh
sudo fail2ban-client status sshd | grep -q "Status.*ok" && echo "fail2ban ok"
```

That's it. No password guessing, no open ports you don't know about, a login
audit trail, and the four tools every server needs. From here it's a clean
base — add your app's ports to UFW as you deploy, and you're production-ready.
