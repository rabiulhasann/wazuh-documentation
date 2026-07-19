**Open Source Security**

# CrowdSec

Implementation Guide

// Firewall bouncer · kernel-level blocking · DDoS mitigation · real-time crowd-sourced threat intel

**Metadata:**

- **TIME:** 30–45 min
- **DIFFICULTY:** Intermediate
- **UPDATED:** April 2026

## 00 — Architecture

## How CrowdSec Works

CrowdSec separates threat *detection* from threat *enforcement*. The engine reads logs and creates ban decisions. The firewall bouncer enforces them at the kernel. The Central API connects your instance to a global crowd-sourced blocklist.

```
📄 Your Logs (nginx / sshd) → 🔍 Parser (extract IP + fields) → ⚡ Scenario (leaky bucket logic) → ⚖️ Decision (ban IP N hours) → 🔥 nftables (kernel firewall) → 🚫 BLOCKED (traffic dropped)
```

| Component | Role | Location |
| --- | --- | --- |
| Security Engine | Reads logs, detects attacks, creates decisions | Your server |
| LAPI (Local API) | Stores and serves decisions to bouncers | Your server |
| Firewall Bouncer | Writes nftables rules to enforce bans | Your server |
| Collections | Bundles of parsers + scenarios per service | CrowdSec Hub |
| CAPI (Central API) | Community blocklist sync | api.crowdsec.net |

## 01 — Before You Begin

## Prerequisites

| Requirement | Details |
| --- | --- |
| OS | Ubuntu 20.04 / 22.04 / 24.04 · Debian 11/12 · RHEL / Rocky / AlmaLinux 8/9 |
| Access | Root or sudo privileges |
| Init | systemd |
| Firewall | nftables (Ubuntu 22.04+) or iptables |
| Service | At least one running service with logs (Nginx, Apache, SSH) |

### Check firewall backend

```bash
sudo nft list ruleset &>/dev/null && echo "nftables available"
sudo iptables -L -n &>/dev/null && echo "iptables available"
```

> ⚠️
**Production safety:** Take a snapshot or backup before proceeding. If anything goes wrong you will have a restore point.
> 

## 02 — Installation

## Install CrowdSec

### Add the official repository

```bash
curl -s <https://install.crowdsec.net> | sudo bash
```

### Install the security engine

```bash
sudo apt install crowdsec -y
```

```bash
sudo dnf install crowdsec -y
```

### Enable and start

```bash
sudo systemctl enable --now crowdsec
sudo systemctl status crowdsec
```

> ✅
You should see `active (running)`. CrowdSec auto-detects services like Nginx and SSH and pre-installs relevant collections.
> 

## 03 — Enforcement

## Install a Bouncer

The security engine detects attacks but cannot block by itself. A bouncer enforces decisions at the firewall level.

> ℹ️
On **Ubuntu 24.04 with the official Nginx repo (v1.28+)**, `crowdsec-nginx-bouncer` fails with a Lua ABI mismatch. We use `crowdsec-firewall-bouncer-nftables` instead — blocks at the kernel, equally effective. See Step 06 for full details.
> 

### Ubuntu 24.04 — nftables (recommended)

```bash
sudo apt install crowdsec-firewall-bouncer-nftables -y
```

### Older Ubuntu / Debian — iptables

```bash
sudo apt install crowdsec-firewall-bouncer-iptables -y
```

### RHEL / Rocky / AlmaLinux

```bash
sudo dnf install crowdsec-firewall-bouncer-iptables -y
```

### Enable and verify

```bash
sudo systemctl enable --now crowdsec-firewall-bouncer
sudo cscli bouncers list
```

```
 Name                         IP Address  Valid  Last pull       Type
────────────────────────────────────────────────────────────────────────
 FirewallBouncer-xxxxxxxxxxxx  127.0.0.1   true   2 seconds ago   crowdsec-firewall-bouncer
```

## 04 — Detection Rules

## Install Collections

Collections are bundles of parsers and detection scenarios for a specific service. Install one and CrowdSec knows how to parse and protect that service automatically.

```bash
# Core Linux (auth logs, syslog) — usually pre-installed
sudo cscli collections install crowdsecurity/linux

# SSH brute-force detection
sudo cscli collections install crowdsecurity/sshd

# Nginx web server attacks
sudo cscli collections install crowdsecurity/nginx

# Apache (if using Apache instead of Nginx)
sudo cscli collections install crowdsecurity/apache2

# WordPress (if running WP)
sudo cscli collections install crowdsecurity/wordpress

# HTTP CVEs, path traversal, scanners
sudo cscli collections install crowdsecurity/http-cve

# Reload after installing
sudo systemctl reload crowdsec
```

### Check what is installed

```bash
sudo cscli hub list
sudo cscli collections list
```

## 05 — Log Sources

## Configure Log Acquisition

CrowdSec needs to know which log files to watch. This is configured in `/etc/crowdsec/acquis.yaml`. Auto-detection often fills this in — verify your web server logs are present.

### Inspect current config

```bash
cat /etc/crowdsec/acquis.yaml
```

### Edit acquis.yaml

```bash
sudo nano /etc/crowdsec/acquis.yaml
```

### For Nginx

```yaml
filenames:
  - /var/log/nginx/access.log
  - /var/log/nginx/error.log
labels:
  type: nginx
---
```

### For Apache

```yaml
filenames:
  - /var/log/apache2/access.log
  - /var/log/apache2/error.log
labels:
  type: apache2
---
```

### For SSH — file-based

```yaml
filenames:
  - /var/log/auth.log       # Debian / Ubuntu
  - /var/log/secure         # RHEL / Rocky
labels:
  type: syslog
---
```

### For SSH — journald (no file logs)

```yaml
source: journalctl
journalctl_filter:
  - "_SYSTEMD_UNIT=sshd.service"
labels:
  type: syslog
---
```

### Apply and verify

```bash
sudo systemctl restart crowdsec
# Look for non-zero reads under "Acquisition Metrics"
sudo cscli metrics
```

> ⚠️
If a log file shows **0 reads**, the path is wrong. Find the correct path with: `sudo nginx -T | grep access_log`
> 

## 06 — Bouncer Setup · Option A

## Firewall Bouncer (Recommended)

> ⚠️
**Why not the Nginx bouncer?** On Ubuntu 24.04 with Nginx from the official repo (v1.28+), `crowdsec-nginx-bouncer` fails — `libnginx-mod-http-lua` requires `nginx-abi-1.24.0-1` but your Nginx ABI is newer. The firewall bouncer has no such dependency and blocks at the kernel level before traffic even reaches Nginx.
> 

| Bouncer | Blocks at | CAPTCHA | Ubuntu 24.04 + Nginx repo |
| --- | --- | --- | --- |
| crowdsec-firewall-bouncer-nftables | Kernel / nftables | — | ✅ Works |
| crowdsec-nginx-bouncer | Application (HTTP) | ✅ | ❌ ABI mismatch |
| crowdsec-openresty-bouncer | Application (HTTP) | ✅ | ⚠️ Replace Nginx first |

### Install nftables bouncer

```bash
sudo apt install crowdsec-firewall-bouncer-nftables -y
```

### Enable and start

```bash
sudo systemctl enable --now crowdsec-firewall-bouncer
sudo systemctl status crowdsec-firewall-bouncer
```

### Verify bouncer registration

```bash
sudo cscli bouncers list
```

### Verify nftables rules are applied

```bash
# nftables (Ubuntu 24.04 default)
sudo nft list ruleset | grep -A5 crowdsec
# iptables (older systems)
sudo iptables -L -n | grep crowdsec
```

> ✅
**How it works:** The bouncer polls the LAPI every 10 seconds and writes ban decisions directly into nftables. Banned IPs are dropped at the kernel — before any traffic reaches Nginx. This is actually faster than application-level blocking.
> 

### Want CAPTCHA support later?

Migrate to **OpenResty** — a drop-in Nginx replacement with Lua built in. Install from `openresty.org/package/ubuntu`, then install `crowdsec-openresty-bouncer`. Your existing Nginx config is fully compatible.

## 07 — Web Dashboard

## CrowdSec Console (Web UI)

The CrowdSec Console at **app.crowdsec.net** is the free web dashboard — real-time attack maps, metrics, decision management, and community blocklist access.

### Create a free account

Visit **https://app.crowdsec.net** and register. No credit card required.

### Get the enrollment key

In the Console navigate to **Security Engines → Add Security Engine** and copy the enrollment key shown.

### Enroll your server

```bash
sudo cscli console enroll YOUR_ENROLLMENT_KEY_HERE
```

Then go back to the Console and **accept the enrollment request**.

### Restart and verify

```bash
sudo systemctl restart crowdsec
# Should say: "You can successfully interact with Central API (CAPI)"
sudo cscli capi status
```

## 08 — Critical Safety Step

## Whitelist Your IPs

> 🔴
**Do this before testing.** Without a whitelist, CrowdSec can ban your own IP during load tests or if your monitoring tools trigger detection rules.
> 

### Create the whitelist file

```bash
sudo nano /etc/crowdsec/parsers/s02-enrich/mywhitelists.yaml
```

```yaml
name: myorg/whitelists
description: "Whitelist for our trusted IPs"
whitelist:
  reason: "Our own infrastructure"
ip:
    - "YOUR.OFFICE.IP.HERE"
    - "YOUR.VPN.EXIT.IP"
    - "YOUR.MONITORING.SERVER.IP"
cidr:
    - "10.0.0.0/8"
    - "192.168.0.0/16"
```

### Apply the whitelist

```bash
sudo systemctl reload crowdsec
# Your whitelisted IPs should never appear here
sudo cscli decisions list
```

## 09 — Testing

## Verify & Test

### Full health check

```bash
sudo systemctl status crowdsec
sudo systemctl status crowdsec-firewall-bouncer
sudo cscli hub list
sudo cscli metrics
sudo cscli decisions list
sudo cscli alerts list
```

### Simulate an SSH brute-force (safe test)

```bash
# Run from the server itself — NOT from a whitelisted IP
for i in {1..6}; do ssh baduser@127.0.0.1; done

# Check for the ban decision
sudo cscli decisions list

# Remove the test ban when done
sudo cscli decisions delete --ip 127.0.0.1
```

### Confirm nftables rules are active

```bash
sudo nft list ruleset | grep crowdsec
sudo iptables -L -n | grep crowdsec
```

## 10 — Reference

## Commands Cheatsheet

### Decision management

```bash
# View all active bans
sudo cscli decisions list

# Manually ban an IP for 24h
sudo cscli decisions add --ip 1.2.3.4 --duration 24h --reason "manual ban"

# Unban an IP
sudo cscli decisions delete --ip 1.2.3.4

# Ban an entire subnet
sudo cscli decisions add --range 1.2.3.0/24

# Clear ALL decisions (use with caution)
sudo cscli decisions delete --all
```

### Monitoring and alerts

```bash
sudo cscli alerts list
sudo cscli alerts list --since 1h
sudo journalctl -u crowdsec -f
sudo cscli metrics
```

### Collections and hub

```bash
sudo cscli hub search nginx
sudo cscli hub update && sudo cscli hub upgrade
sudo cscli scenarios list
```

## 11 — Diagnostics

## Troubleshooting

### Nginx bouncer ABI mismatch (your current error)

```bash
# Error: libnginx-mod-http-lua: Depends: nginx-abi-1.24.0-1
# Fix: use the firewall bouncer — no Lua dependency
sudo apt install crowdsec-firewall-bouncer-nftables -y
sudo systemctl enable --now crowdsec-firewall-bouncer
sudo cscli bouncers list
```

### CrowdSec not detecting attacks

```bash
# Check if logs are being read
sudo cscli metrics | grep -A5 "Acquisition"

# Enable debug mode temporarily
sudo sed -i 's/level: info/level: debug/' /etc/crowdsec/config.yaml
sudo systemctl restart crowdsec
sudo journalctl -u crowdsec -f

# Set back to info after debugging
sudo sed -i 's/level: debug/level: info/' /etc/crowdsec/config.yaml
sudo systemctl restart crowdsec
```

### Bouncer not blocking traffic

```bash
sudo cscli bouncers list
sudo nft list ruleset | grep crowdsec
sudo journalctl -u crowdsec-firewall-bouncer -f
```

### False positive — legitimate user banned

```bash
sudo cscli decisions list
sudo cscli decisions delete --ip AFFECTED_IP
sudo nano /etc/crowdsec/parsers/s02-enrich/mywhitelists.yaml
```

### Wrong log file paths

```bash
sudo nginx -T 2>/dev/null | grep access_log
sudo apache2ctl -S 2>/dev/null | grep access_log
```

## 12 — Before Going Live

## Production Checklist

Click each item to mark it as done.

- ✅ CrowdSec engine running and enabled on boot
- ✅ Firewall bouncer (nftables) installed — kernel-level blocking active
- ✅ Bouncer registered — cscli bouncers list shows Valid: true
- ✅ Nginx / Apache collection installed and logs wired in acquis.yaml
- ✅ SSH collection installed and auth.log path configured
- ✅ Own IPs whitelisted in s02-enrich/mywhitelists.yaml
- ✅ Console enrollment completed — Web UI at app.crowdsec.net accessible
- ✅ Community blocklist active — cscli capi status shows OK
- ✅ Hub content updated — cscli hub update and hub upgrade run
- ✅ SSH brute-force simulation tested and ban decision confirmed
- ✅ Monitoring set up — metrics and alerts reviewed
- ✅ Server snapshot or backup taken before deployment

## 13 — Reference

## Important File Paths

- `/etc/crowdsec/config.yaml` — Main engine configuration
- `/etc/crowdsec/acquis.yaml` — Log sources to watch
- `/etc/crowdsec/profiles.yaml` — Decision profiles
- `/etc/crowdsec/parsers/s02-enrich/` — Whitelist YAML files go here
- `/var/lib/crowdsec/data/crowdsec.db` — SQLite decisions database
- `/etc/crowdsec/bouncers/` — Bouncer configuration files
- `/var/log/crowdsec/` — CrowdSec engine log files
- `/etc/crowdsec/local_api_credentials.yaml` — LAPI authentication credentials

> 📖
Docs: **docs.crowdsec.net**  ·  Console: **app.crowdsec.net**  ·  Hub: **hub.crowdsec.net**
>
