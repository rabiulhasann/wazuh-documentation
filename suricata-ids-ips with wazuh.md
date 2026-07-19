# 🛡️ Suricata IDS & IPS Setup Guide

Complete Production-Ready Deployment Documentation

## 📑 Table of Contents

### 📑 Table of Contents

1. Overview & Architecture
2. Prerequisites & Requirements
3. System Preparation
4. Suricata IDS Installation (AF-PACKET Mode)
5. Suricata IPS Installation (NFQUEUE Mode)
6. Custom Rule Configuration
7. Wazuh SIEM Integration
8. Testing & Verification
9. Grafana Dashboard Setup
10. Maintenance & Updates
11. Troubleshooting Guide

## 🎯 Overview & Architecture

This guide covers the complete deployment of **Suricata** in both **IDS** (Intrusion Detection System) and **IPS** (Intrusion Prevention System) modes, integrated with **Wazuh SIEM** for centralized monitoring and **Grafana** for visualization.

### 🔍 IDS Mode

**Purpose:** Passive monitoring and detection

**Method:** AF-PACKET capture

**Log:** /var/log/suricata/ids-eve.json

### 🛡️ IPS Mode

**Purpose:** Active blocking and prevention

**Method:** NFQUEUE inline

**Log:** /var/log/suricata/ips-eve.json

### 📊 Wazuh SIEM

**Purpose:** Centralized log management

**Backend:** OpenSearch

**Features:** Alerts, correlation, compliance

### 📈 Grafana

**Purpose:** Real-time visualization

**Source:** OpenSearch datasource

**Dashboards:** Custom security metrics

### System Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                      NETWORK TRAFFIC                         │
│                    (All Inbound/Outbound)                    │
└───────────────┬────────────────────┬────────────────────────┘
                │                    │
                │                    │
        ┌───────▼────────┐   ┌───────▼────────┐
        │  SURICATA IDS  │   │  SURICATA IPS  │
        │  (AF-PACKET)   │   │   (NFQUEUE)    │
        │                │   │                │
        │  Port: Mirror  │   │  Port: Inline  │
        │  Mode: Passive │   │  Mode: Active  │
        └───────┬────────┘   └───────┬────────┘
                │                    │
                │  Alerts & Flows    │  Alerts & Drops
                │                    │
                ▼                    ▼
        ┌─────────────────────────────────────┐
        │     /var/log/suricata/              │
        │  ├─ ids-eve.json (IDS logs)         │
        │  └─ ips-eve.json (IPS logs)         │
        └──────────────┬──────────────────────┘
                       │
                       │  JSON logs
                       │
                ┌──────▼──────┐
                │ Wazuh Agent │
                │ (Forwarder) │
                └──────┬──────┘
                       │
                       │  Encrypted (1514/TCP)
                       │
                ┌──────▼────────────────────┐
                │   Wazuh Manager           │
                │   + OpenSearch Backend    │
                │                           │
                │  • Indexing (wazuh-alerts)│
                │  • Correlation            │
                │  • Enrichment             │
                └──────┬────────────────────┘
                       │
              ┌────────┴────────┐
              │                 │
       ┌──────▼──────┐   ┌──────▼──────┐
       │   Wazuh     │   │   Grafana   │
       │  Dashboard  │   │  Dashboard  │
       │             │   │             │
       │ https://    │   │ http://     │
       │ :443        │   │ :3000       │
       └─────────────┘   └─────────────┘
```

### Key Features

- ✅ **Dual Mode Operation:** IDS and IPS run simultaneously without conflict
- ✅ **Separate Logging:** Independent log files for clear separation
- ✅ **45+ Custom Rules:** Web attack detection (SQLi, XSS, RCE, etc.)
- ✅ **Real-time Alerts:** Integrated with Wazuh for immediate notification
- ✅ **Visual Monitoring:** Grafana dashboards with 11+ panels
- ✅ **Production Ready:** Optimized for performance and reliability

## 📋 Prerequisites & Requirements

### Hardware Requirements

| Component | Minimum | Recommended | Notes |
| --- | --- | --- | --- |
| **CPU** | 4 cores | 8+ cores | More cores = better packet processing |
| **RAM** | 8 GB | 16+ GB | Rule compilation and packet buffers |
| **Disk** | 50 GB | 100+ GB SSD | For logs and packet capture |
| **Network** | 1 Gbps | 10 Gbps | Depends on traffic volume |

### Software Requirements

### 🖥️ Operating System

- **Ubuntu 22.04 LTS** (Recommended)
- Ubuntu 20.04 LTS (Supported)
- Debian 11/12 (Compatible)
- Kernel: 5.x or higher

### 📦 Required Packages

- build-essential
- libpcap-dev
- libnet1-dev
- libyaml-dev
- python3-yaml
- jq (for JSON processing)

### Network Configuration

| Service | Port | Protocol | Purpose |
| --- | --- | --- | --- |
| Wazuh Agent → Manager | 1514 | TCP | Log forwarding |
| Wazuh Agent Registration | 1515 | TCP | Agent enrollment |
| Wazuh API | 55000 | TCP | Management API |
| Wazuh Dashboard | 443 | TCP | Web UI |
| OpenSearch | 9200 | TCP | Grafana datasource |
| Grafana | 3000 | TCP | Dashboard UI |

### ⚠️ Important Notes

- Ensure **root/sudo access** is available
- Wazuh Manager must be **installed and running** before proceeding
- Know your **network interface name** (e.g., eth0, ens33)
- Backup existing configurations before starting
- Plan for **log rotation** to prevent disk space issues

### Pre-Installation Checklist

### ✅ Before You Begin

1. ☐ Ubuntu 22.04 LTS installed and updated
2. ☐ Wazuh Manager running and accessible
3. ☐ Network interface identified
4. ☐ Internet connectivity available
5. ☐ Firewall rules reviewed
6. ☐ Sufficient disk space available (100GB+)
7. ☐ Root/sudo access confirmed

## 🚀 System Preparation

Before installing Suricata, we need to prepare the system with proper configuration and dependencies.

### Update System Packages

Update the package list and upgrade existing packages to the latest versions.

```bash
sudo apt-get update
sudo apt-get upgrade -y
sudo apt-get dist-upgrade -y
```

**What this does:** Ensures all system packages are up-to-date, reducing security vulnerabilities and compatibility issues.

### Install Essential Packages

Install required dependencies for Suricata compilation and operation.

```bash
sudo apt-get install -y \\
    build-essential \\
    software-properties-common \\
    apt-transport-https \\
    ca-certificates \\
    curl \\
    wget \\
    git \\
    vim \\
    net-tools \\
    jq \\
    python3-yaml \\
    libpcap-dev \\
    libnet1-dev \\
    libyaml-dev
```

### Configure System Limits

Increase file descriptor limits for Suricata to handle high traffic volumes.

### Edit /etc/security/limits.conf

```bash
sudo nano /etc/security/limits.conf
```

Add these lines at the end of the file:

```
# Suricata limits
*               soft    nofile          65535
*               hard    nofile          65535
suricata        soft    nofile          131072
suricata        hard    nofile          131072
```

Save and exit: `Ctrl+O`, `Enter`, `Ctrl+X`

### Configure Network Tuning

Optimize kernel network parameters for packet processing.

### Edit /etc/sysctl.conf

```bash
sudo nano /etc/sysctl.conf
```

Add these lines at the end:

```
# Suricata network tuning
net.core.rmem_default = 33554432
net.core.rmem_max = 33554432
net.core.wmem_default = 33554432
net.core.wmem_max = 33554432
net.ipv4.tcp_rmem = 4096 87380 33554432
net.ipv4.tcp_wmem = 4096 65536 33554432
net.core.netdev_max_backlog = 50000
net.ipv4.tcp_max_syn_backlog = 30000
```

Apply the settings:

```bash
sudo sysctl -p
```

**✅ Expected output:** You should see the parameters being applied without errors.

### Create Suricata System User

Create a dedicated user for running Suricata services (security best practice).

```bash
sudo groupadd -r suricata
sudo useradd -r -g suricata -s /bin/false -c "Suricata IDS/IPS" suricata
```

**Explanation:**

- `r`: Create system account
- `g suricata`: Add to suricata group
- `s /bin/false`: No shell access (security)

### Identify Network Interface

Find the network interface name that Suricata will monitor.

```bash
ip addr show
```

**📝 Note:** Look for your primary interface (e.g., `eth0`, `ens33`, `enp0s3`).You'll need this name in the next sections.

Example output:

```
2: ens33: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc fq_codel state UP
    link/ether 00:0c:29:xx:xx:xx brd ff:ff:ff:ff:ff:ff
    inet 192.168.18.3/24 brd 192.168.18.255 scope global ens33
```

In this example, the interface is: ens33

### ✅ System Preparation Complete

Your system is now ready for Suricata installation. Proceed to the next section.

## 🔍 Suricata IDS Installation AF-PACKET Mode

Suricata IDS operates in **passive mode** using AF-PACKET for high-performance packet capture without blocking traffic.

### Add Suricata PPA Repository

Add the official Suricata stable repository to get the latest version.

```bash
sudo add-apt-repository ppa:oisf/suricata-stable -y
sudo apt-get update
```

### Install Suricata Package

Install Suricata from the repository.

```bash
sudo apt-get install -y suricata
```

Verify installation:

```bash
suricata --version
```

**✅ Expected output:** Something like `This is Suricata version 8.0.2 RELEASE`

### Create Log Directory

Set up the directory structure for Suricata logs.

```bash
sudo mkdir -p /var/log/suricata
sudo chown -R suricata:suricata /var/log/suricata
sudo chmod 755 /var/log/suricata
```

### Backup Original Configuration

Always keep a backup of the default configuration.

```bash
sudo cp /etc/suricata/suricata.yaml /etc/suricata/suricata.yaml.original
```

### Configure IDS Mode

Edit the main Suricata configuration file.

```bash
sudo nano /etc/suricata/suricata.yaml
```

### Key Configuration Changes:

**a) Set Network Interface** (around line 76-80)

```yaml
af-packet:
  - interface: ens33    # ⬅️ CHANGE THIS to your interface
    cluster-id: 99
    cluster-type: cluster_flow
    defrag: yes
```

**b) Configure Eve Log Output** (around line 500-550)

```yaml
outputs:
  - eve-log:
      enabled: yes
      filetype: regular
      filename: ids-eve.json    # ⬅️ IDS log filename
      types:
        - alert:
            payload: yes
            payload-printable: yes
            packet: yes
            metadata: yes
        - http:
            extended: yes
        - dns:
            query: yes
            answer: yes
        - tls:
            extended: yes
        - files:
            force-magic: no
        - ssh
        - flow
        - stats:
            totals: yes
```

**c) Enable HTTP App-Layer** (around line 800-850)

```yaml
app-layer:
  protocols:
    http:
      enabled: yes          # ⬅️ Change from 'no' to 'yes'
      memcap: 64mb
      request-body-limit: 100kb
      response-body-limit: 100kb
```

### ⚠️ Important YAML Syntax Rules

- Use **spaces, not tabs** for indentation
- Maintain **consistent indentation** (2 spaces per level)
- Ensure colons are followed by a space: `key: value`
- File must start with: `%YAML 1.1` and `--`

Save and exit: `Ctrl+O`, `Enter`, `Ctrl+X`

### Test Configuration

Validate the configuration file for syntax errors.

```bash
sudo suricata -c /etc/suricata/suricata.yaml -T
```

**✅ Success output:**

```
i: suricata: This is Suricata version 8.0.2 RELEASE running in SYSTEM mode
i: suricata: Configuration provided was successfully loaded.Exiting.
```

### ❌ If you see errors:

- Check for **typos** in interface name
- Verify **indentation** (use spaces, not tabs)
- Ensure **colons** have spaces after them
- Look at the **line number** in the error message

### Update Suricata Rules

Download the latest threat intelligence rules.

```bash
sudo suricata-update
```

**What this does:** Downloads Emerging Threats Open ruleset (free) and other configured sources.

List enabled rule sources:

```bash
sudo suricata-update list-enabled-sources
```

### Create Systemd Service

Set up Suricata IDS as a system service for automatic startup.

```bash
sudo nano /etc/systemd/system/suricata.service
```

Add this content (replace `ens33` with your interface):

```
[Unit]
Description=Suricata IDS (AF-PACKET Mode)
Documentation=https://suricata.io/
After=network.target network-online.target
Wants=network-online.target

[Service]
Type=simple
ExecStartPre=/usr/bin/suricata -c /etc/suricata/suricata.yaml -T
ExecStart=/usr/bin/suricata -c /etc/suricata/suricata.yaml -i ens33 --user suricata --group suricata
ExecReload=/bin/kill -USR2 $MAINPID
Restart=on-failure
RestartSec=5s
User=suricata
Group=suricata
LimitNOFILE=65535

[Install]
WantedBy=multi-user.target
```

Save and exit: `Ctrl+O`, `Enter`, `Ctrl+X`

### Enable and Start IDS Service

Start Suricata IDS and enable it to run on boot.

```bash
sudo systemctl daemon-reload
sudo systemctl enable suricata.service
sudo systemctl start suricata.service
```

Check service status:

```bash
sudo systemctl status suricata.service
```

**✅ Expected status:**

```
● suricata.service - Suricata IDS (AF-PACKET Mode)
   Loaded: loaded (/etc/systemd/system/suricata.service; enabled)
   Active: active (running) since ...
```

### Verify IDS Operation

Check that Suricata IDS is capturing traffic and generating logs.

**Check log file:**

```bash
ls -lh /var/log/suricata/ids-eve.json
```

**View recent events:**

```bash
sudo tail -f /var/log/suricata/ids-eve.json | jq '.'
```

**Count events by type:**

```bash
sudo jq -r '.event_type' /var/log/suricata/ids-eve.json | sort | uniq -c
```

**✅ You should see events like:**

- alert (security events)
- flow (connection metadata)
- http (web traffic)
- dns (DNS queries)
- stats (performance metrics)

### 🎉 Suricata IDS Installation Complete!

Your IDS is now running and monitoring network traffic.Proceed to IPS setup for active blocking.

## 🛡️ Suricata IPS Installation NFQUEUE Mode

Suricata IPS operates in **inline mode** using NFQUEUE to actively block malicious traffic in real-time.

### ⚠️ Important: IPS vs IDS

- **IDS (Previous Section):** Monitors and alerts (passive)
- **IPS (This Section):** Monitors, alerts, AND blocks (active)
- Both can run **simultaneously** on the same system
- IPS requires **iptables NFQUEUE** configuration

### Create IPS Configuration

Copy the IDS configuration as a base for IPS.

```bash
sudo cp /etc/suricata/suricata.yaml /etc/suricata/suricata-ips.yaml
```

### Configure IPS Mode

Edit the IPS configuration file to enable NFQUEUE mode.

```bash
sudo nano /etc/suricata/suricata-ips.yaml
```

### Key Configuration Changes:

**a) Change Eve Log Filename** (around line 500-550)

```yaml
outputs:
  - eve-log:
      enabled: yes
      filetype: regular
      filename: ips-eve.json    # ⬅️ Changed from ids-eve.json
      types:
        - alert:
            payload: yes
        - drop                   # ⬅️ ADD THIS for IPS drop events
        - flow
        - http:
            extended: yes
```

**b) Add NFQUEUE Configuration** (add at the end of file, before closing)

```yaml
# NFQUEUE configuration for IPS mode
nfqueue:
  - mode: accept
    fail-open: yes
    queue-num: 1

# Action order for IPS
action-order:
  - pass
  - drop
  - reject
  - alert
```

**Configuration Explained:**

- `mode: accept` - Default action if no rule matches
- `fail-open: yes` - Allow traffic if Suricata crashes (safety)
- `queue-num: 1` - NFQUEUE number (must match iptables)
- `action-order` - Priority: pass > drop > reject > alert

Save and exit: `Ctrl+O`, `Enter`, `Ctrl+X`

### Test IPS Configuration

Validate the IPS configuration file.

```bash
sudo suricata -c /etc/suricata/suricata-ips.yaml -T
```

**✅ Expected output:** "Configuration provided was successfully loaded. Exiting."

### Create IPS Systemd Service

Set up Suricata IPS as a separate system service.

```bash
sudo nano /etc/systemd/system/suricata-ips.service
```

Add this content:

```
[Unit]
Description=Suricata IPS (NFQUEUE Mode)
Documentation=https://suricata.io/
After=network.target network-online.target
Wants=network-online.target

[Service]
Type=simple

# Clean up any existing rules and processes FIRST
ExecStartPre=+/bin/sh -c 'iptables -D INPUT -j NFQUEUE --queue-num 1 2>/dev/null || true'
ExecStartPre=+/bin/sh -c 'iptables -D OUTPUT -j NFQUEUE --queue-num 1 2>/dev/null || true'
ExecStartPre=+/bin/sh -c 'pkill -9 suricata 2>/dev/null || true'
ExecStartPre=/bin/sleep 2

# Add iptables rules
ExecStartPre=+/bin/sh -c 'iptables -I INPUT -j NFQUEUE --queue-num 1'
ExecStartPre=+/bin/sh -c 'iptables -I OUTPUT -j NFQUEUE --queue-num 1'

ExecStart=/usr/bin/suricata -c /etc/suricata/suricata-ips.yaml -q 1 --user suricata --group suricata

# Give it time to stop gracefully
TimeoutStopSec=10
KillMode=mixed
KillSignal=SIGTERM

# Clean up after stop
ExecStopPost=+/bin/sh -c 'pkill -9 suricata 2>/dev/null || true'
ExecStopPost=+/bin/sh -c 'iptables -D INPUT -j NFQUEUE --queue-num 1 2>/dev/null || true'
ExecStopPost=+/bin/sh -c 'iptables -D OUTPUT -j NFQUEUE --queue-num 1 2>/dev/null || true'

Restart=on-failure
RestartSec=5s
TimeoutStartSec=30
LimitNOFILE=65535

[Install]
WantedBy=multi-user.target
```

**What this does:**

- `ExecStartPre`: Adds iptables rules before starting
- `q 1`: Listen on NFQUEUE queue number 1
- `ExecStopPost`: Removes iptables rules after stopping

Save and exit: `Ctrl+O`, `Enter`, `Ctrl+X`

### Enable and Start IPS Service

Start Suricata IPS and enable it to run on boot.

```bash
sudo systemctl daemon-reload
sudo systemctl enable suricata-ips.service
sudo systemctl start suricata-ips.service
```

Check service status:

```bash
sudo systemctl status suricata-ips.service
```

**✅ Expected status:** "active (running)"

### Verify IPS Operation

Confirm that IPS is running and iptables rules are in place.

**Check NFQUEUE iptables rules:**

```bash
sudo iptables -L INPUT -n -v | grep NFQUEUE
sudo iptables -L OUTPUT -n -v | grep NFQUEUE
```

**✅ Expected output:** You should see NFQUEUE rules for queue 1

**Check NFQUEUE status:**

```bash
cat /proc/net/netfilter/nfnetlink_queue
```

**Check IPS log file:**

```bash
ls -lh /var/log/suricata/ips-eve.json
sudo tail -f /var/log/suricata/ips-eve.json | jq '.'
```

### 🎉 Suricata IPS Installation Complete!

Your IPS is now running and actively protecting your network. Both IDS and IPS are operational!

## 📜 Custom Rule Configuration

Add custom detection rules for web attacks, malware, and other threats.

### Create Custom Rules File

Create a dedicated file for your custom detection rules.

```bash
sudo nano /var/lib/suricata/rules/attack-detection.rules
```

Add the following custom rules (comprehensive web attack detection):

```
# Custom Attack Detection Rules
# Author: rabiulhasanjisan
# Date: 2025-12-08

# SQL Injection Detection
alert http any any -> any any (msg:"SQL Injection - Single Quote"; flow:established,to_server; http.uri; content:"|27|"; sid:100010; rev:1; classtype:web-application-attack;)
alert http any any -> any any (msg:"SQL Injection - OR Pattern"; flow:established,to_server; http.uri; content:"OR"; nocase; content:"="; distance:0; within:10; sid:100011; rev:1; classtype:web-application-attack;)
alert http any any -> any any (msg:"SQL Injection - UNION SELECT"; flow:established,to_server; http.uri; content:"UNION"; nocase; content:"SELECT"; nocase; distance:0; within:20; sid:100012; rev:1; classtype:web-application-attack;)

# XSS Detection
alert http any any -> any any (msg:"XSS Attack - Script Tag"; flow:established,to_server; http.uri; content:"|3c|script"; nocase; sid:100020; rev:1; classtype:web-application-attack;)
alert http any any -> any any (msg:"XSS Attack - JavaScript Protocol"; flow:established,to_server; http.uri; content:"javascript:"; nocase; sid:100021; rev:1; classtype:web-application-attack;)

# Directory Traversal
alert http any any -> any any (msg:"Directory Traversal - Dot Dot Slash"; flow:established,to_server; http.uri; content:"../"; sid:100030; rev:1; classtype:web-application-attack;)
alert http any any -> any any (msg:"Directory Traversal - etc passwd"; flow:established,to_server; http.uri; content:"etc/passwd"; nocase; sid:100032; rev:1; classtype:web-application-attack;)

# Command Injection
alert http any any -> any any (msg:"Command Injection - Semicolon"; flow:established,to_server; http.uri; content:"|3b|"; content:"cat"; nocase; distance:0; within:10; sid:100040; rev:1; classtype:web-application-attack;)
alert http any any -> any any (msg:"Command Injection - Pipe Character"; flow:established,to_server; http.uri; content:"|7c|"; sid:100041; rev:1; classtype:web-application-attack;)

# Suspicious User-Agents
alert http any any -> any any (msg:"Suspicious User-Agent - SQLMap"; flow:established,to_server; http.user_agent; content:"sqlmap"; nocase; sid:100080; rev:1; classtype:web-application-attack;)
alert http any any -> any any (msg:"Suspicious User-Agent - Nikto"; flow:established,to_server; http.user_agent; content:"Nikto"; nocase; sid:100081; rev:1; classtype:attempted-recon;)

# IPS Blocking Visibility
alert tcp any any -> any 23 (msg:"IPS BLOCKING - TELNET Connection"; flow:to_server; threshold: type limit, track by_src, count 1, seconds 60; sid:100200; rev:1; classtype:policy-violation;)
alert tcp any any -> any 21 (msg:"IPS BLOCKING - FTP Connection"; flow:to_server; threshold: type limit, track by_src, count 1, seconds 60; sid:100201; rev:1; classtype:policy-violation;)
```

Save and exit: `Ctrl+O`, `Enter`, `Ctrl+X`

### Set Proper Permissions

Ensure the rules file has correct ownership and permissions.

```bash
sudo chown suricata:suricata /var/lib/suricata/rules/attack-detection.rules
sudo chmod 644 /var/lib/suricata/rules/attack-detection.rules
```

### Add Rules to IDS Configuration

Include the custom rules in the IDS configuration.

```bash
sudo nano /etc/suricata/suricata.yaml
```

Find the `rule-files:` section (around line 1900) and add:

```yaml
rule-files:
  - suricata.rules
  - attack-detection.rules    # ⬅️ ADD THIS LINE
```

Save and exit.

### Add Rules to IPS Configuration

Include the custom rules in the IPS configuration as well.

```bash
sudo nano /etc/suricata/suricata-ips.yaml
```

Find the `rule-files:` section and add the same line:

```yaml
rule-files:
  - suricata.rules
  - attack-detection.rules    # ⬅️ ADD THIS LINE
```

Save and exit.

### Test Configurations

Validate both configurations with the new rules.

**Test IDS config:**

```bash
sudo suricata -c /etc/suricata/suricata.yaml -T
```

**Test IPS config:**

```bash
sudo suricata -c /etc/suricata/suricata-ips.yaml -T
```

**✅ Both should show:** "Configuration provided was successfully loaded."

### Restart Services

Restart both IDS and IPS to load the new rules.

```bash
sudo systemctl restart suricata.service
sudo systemctl restart suricata-ips.service
```

Verify both are running:

```bash
sudo systemctl status suricata.service
sudo systemctl status suricata-ips.service
```

### ✅ Custom Rules Deployed!

Your custom rules are now active on both IDS and IPS systems.

## 📊 Wazuh SIEM Integration OpenSearch Backend

Integrate Suricata logs with Wazuh for centralized monitoring and alerting.

### ⚠️ Prerequisites

Before proceeding, ensure:

- Wazuh Manager is installed and running
- You have the Wazuh Manager IP address
- Port 1514/TCP is accessible from this host

### Install Wazuh Agent

Install the Wazuh agent on the Suricata server.

**Add Wazuh repository:**

```bash
curl -s <https://packages.wazuh.com/key/GPG-KEY-WAZUH> | gpg --no-default-keyring --keyring gnupg-ring:/usr/share/keyrings/wazuh.gpg --import && sudo chmod 644 /usr/share/keyrings/wazuh.gpg

echo "deb [signed-by=/usr/share/keyrings/wazuh.gpg] <https://packages.wazuh.com/4.x/apt/> stable main" | sudo tee -a /etc/apt/sources.list.d/wazuh.list

sudo apt-get update
```

**Install agent (replace with your Wazuh Manager IP):**

```bash
WAZUH_MANAGER="192.168.18.215"  # ⬅️ CHANGE THIS
AGENT_NAME="Suricata-IDS-IPS"

sudo WAZUH_MANAGER="$WAZUH_MANAGER" WAZUH_AGENT_NAME="$AGENT_NAME" apt-get install -y wazuh-agent
```

### Configure Wazuh Agent for Suricata

Configure the agent to monitor Suricata log files.

```bash
sudo nano /var/ossec/etc/ossec.conf
```

Add this before the closing `</ossec_config>` tag:

```xml
  <!-- Suricata IDS Log -->
  <localfile>
    <log_format>json</log_format>
    <location>/var/log/suricata/ids-eve.json</location>
  </localfile>

  <!-- Suricata IPS Log -->
  <localfile>
    <log_format>json</log_format>
    <location>/var/log/suricata/ips-eve.json</location>
  </localfile>
```

Save and exit.

### Start Wazuh Agent

Enable and start the Wazuh agent service.

```bash
sudo systemctl daemon-reload
sudo systemctl enable wazuh-agent
sudo systemctl start wazuh-agent
```

Check agent status:

```bash
sudo systemctl status wazuh-agent
```

### Verify Agent Connection

Confirm the agent is connected to the Wazuh Manager.

```bash
sudo grep -i "connected" /var/ossec/logs/ossec.log | tail -5
```

**✅ Expected output:** You should see "Connected to the server" messages.

### View Logs in Wazuh Dashboard

Access the Wazuh web interface to see Suricata events.

### 📊 Accessing Wazuh Dashboard

1. Open browser: `https://<WAZUH_MANAGER_IP>`
2. Login with your credentials
3. Go to: **Security Events → Discover**
4. Add filter: `agent.name: "Suricata-IDS-IPS"`

**Useful Wazuh Filters:**

| Filter | Description |
| --- | --- |
| `location: "/var/log/suricata/ids-eve.json"` | IDS events only |
| `location: "/var/log/suricata/ips-eve.json"` | IPS events only |
| `data.event_type: "alert"` | Security alerts |
| `data.alert.severity: [1 TO 2]` | Critical alerts |
| `data.alert.signature: *SQL*` | SQL injection attempts |

### ✅ Wazuh Integration Complete!

Suricata logs are now being forwarded to Wazuh for centralized management.

## 🧪 Testing & Verification

Test your Suricata IDS/IPS deployment to ensure everything is working correctly.

### Test IDS Detection

Generate test traffic to trigger IDS alerts.

**SQL Injection Test:**

```bash
curl "<http://localhost/>? id=1' OR '1'='1"
curl "<http://localhost/?user=admin>' UNION SELECT null--"
```

**XSS Test:**

```bash
curl "<http://localhost/?q=><script>alert(1)</script>"
curl "<http://localhost/?search=javascript:alert('XSS')>"
```

**Directory Traversal Test:**

```bash
curl "<http://localhost/../../../../etc/passwd>"
```

**Suspicious User-Agent Test:**

```bash
curl -A "sqlmap/1.5.2" "<http://localhost/>"
```

### Test IPS Blocking

Test that IPS is actively blocking traffic.

**Try connecting to blocked port (TELNET):**

```bash
telnet localhost 23
```

**✅ Expected result:** Connection should timeout or be refused (IPS blocking)

### Check IDS Alerts

Verify that IDS captured the test attacks.

```bash
# Count alerts
sudo jq 'select(.event_type=="alert")' /var/log/suricata/ids-eve.json | wc -l

# Show recent alerts
sudo jq 'select(.event_type=="alert") | {timestamp, signature: .alert.signature, severity: .alert.severity}' /var/log/suricata/ids-eve.json | tail -10
```

### Check IPS Activity

Verify that IPS logged the blocking activity.

```bash
# Count all IPS events
sudo jq -r '.event_type' /var/log/suricata/ips-eve.json | sort | uniq -c

# Show recent IPS activity
sudo jq 'select(.dest_port==23)' /var/log/suricata/ips-eve.json | tail -5
```

### System Health Check

Verify both services are running properly.

```bash
# Check all services
echo "=== IDS Status ==="
systemctl is-active suricata.service

echo "=== IPS Status ==="
systemctl is-active suricata-ips.service

echo "=== Wazuh Agent Status ==="
systemctl is-active wazuh-agent

echo "=== Log Files ==="
ls -lh /var/log/suricata/*.json

echo "=== Rules Loaded ==="
sudo suricata -c /etc/suricata/suricata.yaml -T 2>&1 | grep "rule(s) loaded"
```

### ✅ Testing Complete!

If all tests passed, your Suricata IDS/IPS deployment is fully operational.

## 📈 Grafana Dashboard Setup

Create a real-time visualization dashboard for Suricata metrics using Grafana with OpenSearch datasource.

### 📋 Prerequisites

- Grafana installed and running
- OpenSearch datasource configured
- Access to Wazuh OpenSearch backend (port 9200)

### Configure OpenSearch Datasource

Add Wazuh OpenSearch as a datasource in Grafana.

**In Grafana UI:**

1. Go to: **Configuration → Data Sources**
2. Click: **Add data source**
3. Select: **OpenSearch**
4. Configure:
- **Name:** Wazuh-OpenSearch
- **URL:** https://192.168.18.215:9200
- **Index name:** wazuh-alerts-*
- **Time field:** timestamp
- **Version:** 2.x+
1. Under **Auth**:
- Enable: **Basic auth**
- **User:** admin
- **Password:** (your OpenSearch password)
1. Under **TLS/SSL Settings**:
- Enable: **Skip TLS Verify** (for self-signed certs)
1. Click: **Save & Test**

**✅ Expected result:** "Data source is working"

### Import Dashboard JSON

The Grafana dashboard JSON was created earlier.Import it into Grafana.

**In Grafana UI:**

1. Go to: **Dashboards → Import**
2. Click: **Upload JSON file**
3. Select: `suricata-security-dashboard.json`
4. Select datasource: **Wazuh-OpenSearch**
5. Click: **Import**

### 📊 Dashboard Features

- 11 visualization panels
- Real-time metrics (30s refresh)
- Agent filter dropdown
- Time range selector
- Auto-refresh options

### Dashboard Panels Overview

### 🚨 Total Alerts

Live count of all security alerts with threshold colors

### 🔴 Critical Alerts

High-severity events (severity 1-2)

### 📈 Alerts Timeline

Time series chart showing alert trends

### 🎯 Attack Categories

Donut chart breakdown of attack types

### 🔝 Top Signatures

Bar chart of most triggered rules

### ⚠️ Severity Distribution

Pie chart of alert severity levels

### 🌐 Top Source IPs

Bar chart of attacker origins

### 🎯 Top Destination IPs

Bar chart of targeted hosts

### 📡 Protocol Distribution

Time series of protocol usage

### 📊 Event Types

Donut chart of event categories

### 📋 Recent Alerts

Scrollable table of latest events

### ✅ Grafana Dashboard Ready!

You now have a comprehensive real-time visualization of your Suricata security events.

## 🔧 Maintenance & Updates

Regular maintenance tasks to keep your Suricata deployment healthy and up-to-date.

### Daily Tasks

### Check Service Status

```bash
sudo systemctl status suricata.service suricata-ips.service wazuh-agent
```

### Monitor Disk Space

```bash
df -h /var/log/suricata/
du -sh /var/log/suricata/*
```

### Weekly Tasks

### Update Suricata Rules

```bash
sudo suricata-update
sudo systemctl restart suricata.service suricata-ips.service
```

### Review Alert Summary

```bash
# Top 10 alerts this week
sudo jq 'select(.event_type=="alert") | .alert.signature' /var/log/suricata/ids-eve.json | sort | uniq -c | sort -rn | head -10
```

### Monthly Tasks

### Update System Packages

```bash
sudo apt-get update
sudo apt-get upgrade -y
```

### Rotate Log Files

Configure log rotation to prevent disk space issues.

```bash
sudo nano /etc/logrotate.d/suricata
```

Add this configuration:

```
/var/log/suricata/*.json {
    daily
    rotate 30
    compress
    delaycompress
    missingok
    notifempty
    create 0640 suricata suricata
    sharedscripts
    postrotate
        /bin/kill -HUP `cat /var/run/suricata.pid 2>/dev/null` 2>/dev/null || true
    endscript
}
```

### Backup Configurations

```bash
sudo tar -czf /backup/suricata-config-$(date +%Y%m%d).tar.gz \\
    /etc/suricata/ \\
    /var/lib/suricata/rules/attack-detection.rules \\
    /etc/systemd/system/suricata*.service
```

### Performance Monitoring

### Check Suricata Stats

```bash
# IDS stats
sudo jq 'select(.event_type=="stats") | .stats' /var/log/suricata/ids-eve.json | tail -1

# IPS stats
sudo jq 'select(.event_type=="stats") | .stats' /var/log/suricata/ips-eve.json | tail -1
```

## 🔍 Troubleshooting Guide

### Common Issues & Solutions

### ❌ Issue: Suricata service fails to start

**Symptoms:** Service shows "failed" status

**Solutions:**

1. Check configuration syntax:

```
sudo suricata -c /etc/suricata/suricata.yaml -T
```

1. Check permissions:

```
sudo chown -R suricata:suricata /var/log/suricata/
sudo chmod 755 /var/log/suricata/
```

1. Check logs:

```
sudo journalctl -u suricata.service -n 50
```

### ❌ Issue: No alerts being generated

**Symptoms:** Log file empty or no alert events

**Solutions:**

1. Verify rules are loaded:

```
sudo suricata -c /etc/suricata/suricata.yaml -T 2>&1 | grep "rule(s) loaded"
```

1. Check HTTP app-layer is enabled:

```
sudo grep -A 5 "app-layer:" /etc/suricata/suricata.yaml | grep -A 3 "http:"
```

1. Generate test traffic and check logs

### ❌ Issue: IPS not blocking traffic

**Symptoms:** Blocked ports are still accessible

**Solutions:**

1. Check NFQUEUE rules:

```
sudo iptables -L -n -v | grep NFQUEUE
```

1. Check NFQUEUE status:

```
cat /proc/net/netfilter/nfnetlink_queue
```

1. Verify IPS service is running:

```
sudo systemctl status suricata-ips.service
```

### ❌ Issue: Wazuh agent not connecting

**Symptoms:** No logs appearing in Wazuh dashboard

**Solutions:**

1. Check agent status:

```
sudo systemctl status wazuh-agent
```

1. Check connectivity:

```
telnet WAZUH_MANAGER_IP 1514
```

1. Check agent logs:

```
sudo tail -f /var/ossec/logs/ossec.log
```

### ❌ Issue: High CPU usage

**Symptoms:** System performance degradation

**Solutions:**

1. Check rule count (too many rules can impact performance)
2. Adjust worker threads in config
3. Consider hardware upgrade
4. Disable unnecessary protocols in app-layer

### ❌ Issue: Disk space filling up

**Symptoms:** /var/log/suricata/ consuming too much space

**Solutions:**

1. Implement log rotation (see Maintenance section)
2. Compress old logs:

```
sudo gzip /var/log/suricata/*.json.1
```

1. Delete very old logs if not needed

### Getting Help

### 📚 Resources

- **Suricata Documentation:** docs.suricata.io
- **Wazuh Documentation:** documentation.wazuh.com
- **Suricata Forum:** forum.suricata.io
- **GitHub Issues:** Report bugs on respective GitHub repositories

### 🛡️ Suricata IDS & IPS Setup Complete

Documentation by Rabiul Hasan | Version 1.0 | 2025-12-08

© 2025 All Rights Reserved
