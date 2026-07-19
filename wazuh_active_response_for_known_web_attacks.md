# 🛡️ Wazuh Active Response Documentation

Automated Web Attack Blocking & Security Response System

- **Overview**
- **Alert Analysis**
- **Configuration**
- **Custom Rules Testing**
- **Troubleshooting**
- **Quick Reference**

## 📋 Overview

### 🎯 Purpose

This documentation provides a complete guide to implementing automated blocking of web attacks using Wazuh Active Response. The system detects various attack patterns including SQL injection, XSS, path traversal, command injection, and web shells, then automatically blocks attacking IPs at the firewall level across all monitored agents.

### 🔑 Key Features

- **Real-time Detection:** Monitors web access logs for attack patterns
- **Automated Response:** Automatically blocks malicious IPs using iptables/firewalld
- **Multi-Agent Support:** Applies blocking across all Wazuh agents
- **Tiered Blocking:** Different timeout periods based on attack severity
- **MITRE ATT&CK Mapping:** Aligns with cybersecurity frameworks
- **Customizable Rules:** Easily add new attack patterns

### 📊 Attack Detection Coverage

| Attack Type | Rule ID | Severity Level | Block Duration |
| --- | --- | --- | --- |
| SQL Injection | 100100, 100101 | Level 8-10 | 1-24 hours |
| Time-based SQL Injection | 100102 | Level 10 | 24 hours |
| Path Traversal | 100103, 100104 | Level 8-10 | 1-24 hours |
| Command Injection | 100105, 100106 | Level 8-10 | 1-24 hours |
| Web Shell | 100107 | Level 12 | Permanent |
| Known Exploits (Log4j, etc.) | 100112 | Level 12 | Permanent |

### 🏗️ Architecture

### System Flow

1. **Log Collection:** Wazuh agent monitors nginx/apache access logs
2. **Pattern Detection:** Custom rules analyze logs for attack patterns
3. **Alert Generation:** Matching patterns trigger alerts with severity levels
4. **Active Response:** Manager sends firewall-drop command to agents
5. **IP Blocking:** Agent adds iptables rule to block attacker IP
6. **Auto Timeout:** Block automatically expires after configured duration

## 🔍 Alert Analysis

### Original Attack Alert

### ⚠️ SQL Injection Attack Detected

**Source IP:** 46.151.182.62 (Ukraine)

**Target:** /eonapi/getApiKey

**Attack Pattern:** `union select sleep(6)`

**HTTP Response:** 200 (Success)

**MITRE ATT&CK:** T1190 - Exploit Public-Facing Application

```bash
{
  "rule": {
    "id": "31106",
    "level": 6,
    "description": "A web attack returned code 200 (success).",
    "groups": ["web", "accesslog", "attack"],
    "mitre": {
      "technique": ["Exploit Public-Facing Application"],
      "id": ["T1190"],
      "tactic": ["Initial Access"]
    }
  },
  "data": {
    "protocol": "GET",
    "srcip": "46.151.182.62",
    "id": "200",
    "url": "/eonapi/getApiKey?username=%27%20union%20select%20sleep(6)..."
  },
  "GeoLocation": {
    "country_name": "Ukraine",
    "location": {"lon": 30.5233, "lat": 50.45}
  }
}
```

### Attack Breakdown

SQL Injection Components

### Attack Vector Analysis

- **union:** SQL UNION operator to combine query results
- **select:** SQL SELECT statement to extract data
- **sleep(6):** Time-based blind SQL injection technique
- **HTTP 200:** Attack was successful (vulnerable endpoint)

### Indicators of Compromise

- URL-encoded single quotes (%27)
- SQL keywords in GET parameters
- Time-based injection function
- Successful response code

**Remediation Steps**

### Immediate Actions

1. **Block IP:** Use Wazuh Active Response (automated)
2. **Review Logs:** Check for successful data exfiltration
3. **Patch Application:** Fix SQL injection vulnerability
4. **Audit Database:** Check for unauthorized access

### Long-term Solutions

- Implement parameterized queries
- Add input validation and sanitization
- Deploy Web Application Firewall (WAF)
- Apply principle of least privilege to database users
- Regular security testing and code reviews

## ⚙️ Configuration

### Step 1: Active Response Configuration

Configure active response in the Wazuh manager's `ossec.conf` file to automatically block attacking IPs across all agents.

 - /var/ossec/etc/ossec.conf

```html
<!-- Command Definition -->
<ossec_config>
  <command>
    <name>firewall-drop</name>
    <executable>firewall-drop</executable>
    <timeout_allowed>yes</timeout_allowed>
  </command>
</ossec_config>

<!-- Active Response - Block Level 6+ for 1 Hour -->
<ossec_config>
  <active-response>
    <command>firewall-drop</command>
    <location>all</location>
    <rules_group>web,attack</rules_group>
    <level>6</level>
    <timeout>3600</timeout>
  </active-response>
</ossec_config>

<!-- Block Successful Attacks (Level 10+) for 24 Hours -->
<ossec_config>
  <active-response>
    <command>firewall-drop</command>
    <location>all</location>
    <rules_group>attack_success</rules_group>
    <level>10</level>
    <timeout>86400</timeout>
  </active-response>
</ossec_config>

<!-- Permanent Block for Web Shells and Exploits -->
<ossec_config>
  <active-response>
    <command>firewall-drop</command>
    <location>all</location>
    <rules_group>webshell,exploit</rules_group>
    <level>12</level>
    <timeout>0</timeout>
  </active-response>
</ossec_config>
```

### 📝 Configuration Parameters

- **location:** `all` = applies to all agents, `local` = only on affected agent
- **rules_group:** Trigger on specific rule groups
- **level:** Minimum severity level to trigger (1-15)
- **timeout:** Block duration in seconds (0 = permanent)

### Step 2: Verify Agent Configuration

Ensure active response is enabled on all agents:

 Agent /var/ossec/etc/ossec.conf

```html
<ossec_config>
  <active-response>
    <disabled>no</disabled>
  </active-response>
</ossec_config>
```

### Step 3: Apply Configuration

```bash
# On Wazuh Manager
sudo systemctl restart wazuh-manager
sudo systemctl status wazuh-manager

# Restart all agents
sudo /var/ossec/bin/agent_control -R -a

# Verify agents are connected
sudo /var/ossec/bin/agent_control -lc
```

### ⚠️ Important Notes

- Always backup `ossec.conf` before making changes
- Test in a non-production environment first
- Ensure you have alternative access to servers in case of accidental lockout
- Monitor active-responses.log for the first 24 hours after deployment

## 📜 Custom Detection Rules

### Complete Rule Set

Add these custom rules to `/var/ossec/etc/rules/local_rules.xml`:

local_rules.xml

```bash
<!-- Custom Web Attack Rules -->
<group name="web,attack,accesslog">

  <!-- SQL Injection Detection -->
  <rule id="100100" level="8">
    <if_sid>31100</if_sid>
    <url>union|select|insert|update|delete|drop|create|alter|exec|script|javascript|onerror</url>
    <description>SQL Injection or XSS attempt detected</description>
    <mitre>
      <id>T1190</id>
    </mitre>
  </rule>

  <!-- SQL Injection with Success Response -->
  <rule id="100101" level="10">
    <if_sid>100100</if_sid>
    <id>200|301|302</id>
    <description>SQL Injection or XSS attack succeeded</description>
    <group>attack_success</group>
    <mitre>
      <id>T1190</id>
    </mitre>
  </rule>

  <!-- Time-based SQL Injection -->
  <rule id="100102" level="10">
    <if_sid>31100</if_sid>
    <url>sleep\(|waitfor|benchmark\(|pg_sleep</url>
    <description>Time-based SQL injection attack detected</description>
    <group>attack_success</group>
    <mitre>
      <id>T1190</id>
    </mitre>
  </rule>

  <!-- Path Traversal Attempts -->
  <rule id="100103" level="8">
    <if_sid>31100</if_sid>
    <url>\.\./|\.\.\|etc/passwd|boot\.ini|win\.ini</url>
    <description>Path traversal attack attempt</description>
    <mitre>
      <id>T1190</id>
    </mitre>
  </rule>

  <!-- Web Shell Detection -->
  <rule id="100107" level="12">
    <if_sid>31100</if_sid>
    <url>c99\.php|r57\.php|shell\.php|cmd\.php|phpshell|webshell</url>
    <description>Web shell access attempt detected</description>
    <group>attack_success,webshell</group>
    <mitre>
      <id>T1505.003</id>
    </mitre>
  </rule>

  <!-- Multiple Attack Attempts -->
  <rule id="100110" level="10" frequency="3" timeframe="60">
    <if_matched_sid>31100</if_matched_sid>
    <same_source_ip />
    <description>Multiple web attack attempts from same source IP</description>
    <group>attack_success,multiple_attacks</group>
    <mitre>
      <id>T1190</id>
    </mitre>
  </rule>

  <!-- Known Exploit Patterns -->
  <rule id="100112" level="12">
    <if_sid>31100</if_sid>
    <url>log4j|jndi|struts|spring4shell|cve-</url>
    <description>Known exploit pattern detected (Log4j, Struts, etc.)</description>
    <group>attack_success,exploit</group>
    <mitre>
      <id>T1190</id>
    </mitre>
  </rule>

</group>
```

### Rule Severity Levels

| Level | Severity | Description | Response |
| --- | --- | --- | --- |
| 6 | Medium | Attack attempt detected | Block for 1 hour |
| 8 | High | Confirmed attack pattern | Block for 1 hour |
| 10 | Very High | Successful attack | Block for 24 hours |
| 12 | Critical | Critical threat (webshell, exploit) | Permanent block |

### Validate and Apply Rules

```bash
# Validate XML syntax
xmllint --noout /var/ossec/etc/rules/local_rules.xml

# Test rules with wazuh-logtest
sudo /var/ossec/bin/wazuh-logtest -t

# Restart manager to apply
sudo systemctl restart wazuh-manager

# Verify rules loaded
sudo grep "rule id=\"1001" /var/ossec/etc/rules/local_rules.xml | wc -l
```

## 🧪 Testing & Validation

### Method 1: Manual Active Response Test

```bash
# List available agents
sudo /var/ossec/bin/agent_control -l

# List available active responses
sudo /var/ossec/bin/agent_control -L

# Block test IP on specific agent (replace 013 with your agent ID)
sudo /var/ossec/bin/agent_control -b 1.2.3.4 -f firewall-drop0 -u 013

# Check active response log
sudo tail -f /var/ossec/logs/active-responses.log
```

### Method 2: Test with Log Injection

```bash
# On the agent, inject a test SQL injection log
echo '88.77.66.55 - - [16/Feb/2026:12:00:00 +0600] "GET /test?id=1%20union%20select%201 HTTP/1.1" 200 100 "-" "TestBot"' | sudo tee -a /var/log/nginx/access.log

# Monitor on manager
sudo tail -f /var/ossec/logs/active-responses.log
```

### Method 3: Use wazuh-logtest

```bash
# Start interactive log test
sudo /var/ossec/bin/wazuh-logtest

# Paste test log entry:
46.151.182.62 - - [16/Feb/2026:11:50:00 +0600] "GET /api?q=union+select+sleep(5) HTTP/1.1" 200 100 "-" "Mozilla"

# Expected output will show:
# - Rule ID: 100102
# - Level: 10
# - Active Response: firewall-drop
```

### Verify IP Block on Agent

```bash
# SSH to agent
ssh MGC-WEB-FRONTEND

# Check iptables for blocked IP
sudo iptables -L INPUT -v -n | grep "1.2.3.4"

# Expected output:
# 0  0 DROP  all  --  *  *  1.2.3.4  0.0.0.0/0

# List all Wazuh blocks
sudo iptables -L INPUT -v -n | grep DROP

# For firewalld systems
sudo firewall-cmd --list-rich-rules
```

### Monitoring Commands

### ✅ Real-time Monitoring

Open multiple terminals to monitor different logs simultaneously:

Bash - Terminal 1: Watch Alerts

```bash
sudo tail -f /var/ossec/logs/alerts/alerts.json | jq 'select(.rule.id | startswith("1001"))'
```

Bash - Terminal 2: Watch Active Response

```bash
sudo tail -f /var/ossec/logs/active-responses.log
```

Bash - Terminal 3: Watch Firewall

```bash
watch -n 2 'sudo iptables -L INPUT -v -n | grep DROP | wc -l'
```

### Clean Up Test Blocks

```bash
# Remove specific IP block
sudo iptables -D INPUT -s 1.2.3.4 -j DROP

# List current rules with line numbers
sudo iptables -L INPUT --line-numbers

# Delete by line number
sudo iptables -D INPUT 1
```

### ⚠️ Testing Precautions

- Always test in a non-production environment first
- Use test IPs that won't affect real traffic
- Keep alternative access to servers (console, KVM, etc.)
- Document all test IPs used for cleanup
- Monitor for false positives during the first 48 hours

## 🔧 Troubleshooting

❌ Active Response Not Working

### Symptoms

- Command sent but no entry in active-responses.log
- IP not blocked on agent
- No firewall rules added

### Diagnostic Steps

```bash
# Check agent status
sudo /var/ossec/bin/agent_control -i 013

# Check if execd is running on manager
sudo ps aux | grep wazuh-execd

# Check manager logs for errors
sudo tail -50 /var/ossec/logs/ossec.log | grep -i error

# Verify active response configuration
sudo grep -A10 "active-response" /var/ossec/etc/ossec.conf

# Check if active response is disabled
sudo grep "disabled" /var/ossec/etc/ossec.conf | grep active
```

### Common Fixes

- Ensure `<disabled>no</disabled>` in active-response section
- Restart wazuh-manager: `sudo systemctl restart wazuh-manager`
- Restart agent: `sudo /var/ossec/bin/agent_control -R -u 013`
- Check firewall script exists: `ls -la /var/ossec/active-response/bin/firewall-drop`

❌ Rules Not Triggering

### Symptoms

- Attack logs present but no alerts generated
- Custom rules not matching
- Wrong rule ID triggering

### Diagnostic Steps

```bash
# Validate XML syntax
xmllint --noout /var/ossec/etc/rules/local_rules.xml

# Check if rules loaded
sudo grep "rule id=\"1001" /var/ossec/etc/rules/local_rules.xml

# Test with wazuh-logtest
sudo /var/ossec/bin/wazuh-logtest

# Check manager startup logs
sudo journalctl -u wazuh-manager -n 50
```

### Common Issues

- **XML syntax error:** Check for unclosed tags, special characters
- **Wrong if_sid:** Ensure parent rule exists (31100 for web logs)
- **Regex issues:** Test patterns separately, escape special chars
- **Rule not loaded:** Restart manager after adding rules

❌ Agent Disconnected

### Check Agent Status

```bash
# List disconnected agents
sudo /var/ossec/bin/agent_control -ln

# Get detailed agent info
sudo /var/ossec/bin/agent_control -i 013

# Check agent service on remote host
ssh MGC-WEB-FRONTEND "sudo systemctl status wazuh-agent"

# Check agent logs
ssh MGC-WEB-FRONTEND "sudo tail -50 /var/ossec/logs/ossec.log"
```

### Common Fixes

- Restart agent service
- Check network connectivity between agent and manager
- Verify firewall rules allow port 1514 (UDP) and 1515 (TCP)
- Check manager IP in agent configuration

❌ False Positives

### Identifying False Positives

- Legitimate traffic being blocked
- Internal tools triggering alerts
- Search engine crawlers marked as attacks

### Solutions

XML - Add to local_rules.xml

```bash
<!-- Whitelist specific IP -->
<rule id="100200" level="0">
  <if_sid>100100,100102,100103</if_sid>
  <srcip>10.0.0.0/8</srcip>
  <description>Ignore internal network</description>
</rule>

<!-- Whitelist specific user agent -->
<rule id="100201" level="0">
  <if_sid>100100</if_sid>
  <match>Googlebot|bingbot</match>
  <description>Ignore search engine bots</description>
</rule>
```

### Best Practices

- Monitor alerts for first 48 hours
- Create whitelist rules for known IPs/networks
- Adjust rule severity levels if needed
- Use longer timeouts initially, then adjust

### Diagnostic Commands Summary

```bash
# Full system check
sudo systemctl status wazuh-manager
sudo /var/ossec/bin/agent_control -lc
sudo /var/ossec/bin/agent_control -L
sudo tail -50 /var/ossec/logs/ossec.log
sudo tail -20 /var/ossec/logs/active-responses.log
sudo ps aux | grep wazuh
```

## 📚 Quick Reference

### Common Commands

| Action | Command |
| --- | --- |
| List agents | `sudo /var/ossec/bin/agent_control -l` |
| List active agents | `sudo /var/ossec/bin/agent_control -lc` |
| Agent details | `sudo /var/ossec/bin/agent_control -i <id>` |
| List active responses | `sudo /var/ossec/bin/agent_control -L` |
| Block IP manually | `sudo /var/ossec/bin/agent_control -b <ip> -f firewall-drop0 -u <id>` |
| Restart manager | `sudo systemctl restart wazuh-manager` |
| Restart agent | `sudo /var/ossec/bin/agent_control -R -u <id>` |
| Test rules | `sudo /var/ossec/bin/wazuh-logtest` |
| View alerts | `sudo tail -f /var/ossec/logs/alerts/alerts.json` |
| View active responses | `sudo tail -f /var/ossec/logs/active-responses.log` |

### File Locations

| File/Directory | Path | Purpose |
| --- | --- | --- |
| Manager Config | `/var/ossec/etc/ossec.conf` | Main configuration file |
| Custom Rules | `/var/ossec/etc/rules/local_rules.xml` | User-defined rules |
| Manager Logs | `/var/ossec/logs/ossec.log` | Main log file |
| Alerts | `/var/ossec/logs/alerts/alerts.json` | Alert output |
| Active Response Log | `/var/ossec/logs/active-responses.log` | AR execution log |
| AR Scripts | `/var/ossec/active-response/bin/` | Response scripts |
| Agent Config | `/var/ossec/etc/ossec.conf` (on agent) | Agent configuration |

### Severity Levels

| Level | Category | Description |
| --- | --- | --- |
| 0 | Ignored | No action required |
| 1-3 | Low | Informational events |
| 4-5 | Low | System notification |
| 6-7 | Medium | Suspicious activity |
| 8-10 | High | Attack detected |
| 11-15 | Critical | Severe threat |

### MITRE ATT&CK Techniques

| Technique ID | Name | Tactic |
| --- | --- | --- |
| T1190 | Exploit Public-Facing Application | Initial Access |
| T1505.003 | Web Shell | Persistence |
| T1595 | Active Scanning | Reconnaissance |

### Useful Resources

- **Wazuh Documentation:** documentation.wazuh.com
- **MITRE ATT&CK:** attack.mitre.org
- **OWASP Top 10:** owasp.org/top-ten
- **Wazuh GitHub:** github.com/wazuh/wazuh

### 💡 Pro Tips

- Always backup configurations before making changes
- Test rules in a staging environment first
- Monitor false positives for the first week
- Document custom rules and their purposes
- Regularly review blocked IPs and adjust rules
- Keep Wazuh and agents updated to latest version
- Integrate with SIEM for centralized monitoring

↑

© 2026 Wazuh Active Response Documentation
