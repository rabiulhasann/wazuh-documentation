# 🧭 Production Step-by-Step: Audit Commands Run by User (Wazuh)

## STEP 0 — Pre-flight Checklist (Do NOT Skip)

On each endpoint:

```bash
uname -m
id
```

You should confirm:

System is **64-bit**

You know which users are **human** (UID ≥ 1000)

## STEP 1 — Install & Enable Auditd

### Debian / Ubuntu

```bash
sudo apt update
sudo apt install -y auditd audispd-plugins
```

### RHEL / Rocky / Alma

```bash
sudo dnf install -y audit audit-libs
```

Enable:

```bash
sudo systemctl enable auditd
sudo systemctl start auditd
```

Verify:

```bash
auditctl -s
```

## STEP 2 — Configure Auditd for Production Safety

Edit:

```bash
sudo nano /etc/audit/auditd.conf
```

**Minimum safe production settings:**

```
max_log_file = 100
num_logs = 10
max_log_file_action = ROTATE
disk_full_action = SUSPEND
disk_error_action = SUSPEND
flush = INCREMENTAL_ASYNC
freq = 50
```

Restart:

```bash
sudo systemctl restart auditd
```

👉 Prevents disk-fill and CPU spikes.

## STEP 3 — Create Dedicated Audit Rule File (Best Practice)

Never edit `/etc/audit/audit.rules` directly.

Create:

```bash
sudo nano /etc/audit/rules.d/wazuh-user-commands.rules
```

## STEP 4 — Add Production Audit Rules

### Option A (Recommended): High-Risk Commands Only

```bash
-w /usr/bin/nc -p x -k wazuh-net
-w /usr/bin/ncat -p x -k wazuh-net
-w /usr/bin/tcpdump -p x -k wazuh-sniff
-w /usr/bin/nmap -p x -k wazuh-recon
-w /usr/bin/curl -p x -k wazuh-transfer
-w /usr/bin/wget -p x -k wazuh-transfer
```

### Option B (Advanced): Full Exec Tracking (Use Carefully)

```bash
-a always,exit -F arch=b64 -S execve \\
-F auid>=1000 -F auid!=4294967295 \\
-k wazuh-user-cmd
```

⚠️ Only use this on **bastion or critical servers**.

## STEP 5 — Load & Verify Audit Rules

```bash
sudo augenrules --load
sudo systemctl restart auditd
```

Verify:

```bash
auditctl -l
```

You **must** see `wazuh-*` rules listed.

## STEP 6 — Configure Wazuh Agent to Read Audit Logs

Edit:

```bash
sudo nano /var/ossec/etc/ossec.conf
```

Add:

```xml
<localfile>
<log_format>audit</log_format>
<location>/var/log/audit/audit.log</location>
</localfile>
```

Restart agent:

```bash
sudo systemctl restart wazuh-agent
```

## STEP 7 — Create Lookup Lists on Wazuh Manager

### 7.1 Audit Key Mapping

```bash
sudo nano /var/ossec/etc/lists/audit-keys
wazuh-net:network
wazuh-recon:reconnaissance
wazuh-sniff:sniffing
wazuh-transfer:data_transfer
wazuh-user-cmd:command_execution
```

### 7.2 Suspicious Programs List

```bash
sudo nano /var/ossec/etc/lists/suspicious-programs
nc:red
ncat:red
tcpdump:orange
nmap:orange
curl:yellow
wget:yellow
bash:yellow
python:yellow
```

## STEP 8 — Add Production Wazuh Detection Rules

Edit:

```bash
sudo nano /var/ossec/etc/rules/local_rules.xml
```

### 🔴 High Risk Commands

```xml
<group name="audit,command_execution">
<rule id="100210" level="12">
<if_sid>80792</if_sid>
<list field="audit.command" lookup="match_key_value" check_value="red">
etc/lists/suspicious-programs</list>
<description>HIGH RISK: Suspicious command executed by user $(audit.auid)</description>
<mitre>T1059</mitre>
</rule>
```

### 🟠 Medium Risk (Rate Limited)

```xml
<rule id="100211" level="7" frequency="5" timeframe="300">
<if_sid>80792</if_sid>
<list field="audit.command" lookup="match_key_value" check_value="orange">
etc/lists/suspicious-programs</list>
<description>Multiple risky commands executed by user $(audit.auid)</description>
</rule>
```

### 🟡 Low Risk (Context Only)

```xml
<rule id="100212" level="3">
<if_sid>80792</if_sid>
<list field="audit.command" lookup="match_key_value" check_value="yellow">
etc/lists/suspicious-programs</list>
<description>Suspicious shell or scripting execution</description>
</rule>
</group>
```

Restart manager:

```bash
sudo systemctl restart wazuh-manager
```

## STEP 9 — Add Noise Suppression (VERY IMPORTANT)

Exclude automation tools:

```xml
<rule id="100213" level="0">
<if_sid>80792</if_sid>
<field name="audit.exe">/usr/bin/ansible</field>
<description>Ignore Ansible automation</description>
</rule>
```

Repeat for:

- cron
- salt
- chef
- backup agents

## STEP 10 — Validate Detection

On an endpoint:

```bash
nc -v
```

On Wazuh dashboard:

```
rule.id:100210
```

You should see:

- Username
- Executed binary
- Full command
- TTY
- Parent process

## STEP 11 — Rollout Strategy (MANDATORY)

| Phase | Scope |
| --- | --- |
| Canary | 1–2 servers |
| Controlled | Bastions / jump hosts |
| Full | Production fleet |

```
/var/log/audit/audit.log
```

- growth
- Wazuh EPS
- CPU usage
