## Overview

### Purpose

This document provides comprehensive instructions for implementing a restricted sudo access model on Ubuntu systems with integrated Wazuh monitoring. The solution enables administrative privileges for designated users while preventing direct root shell access and maintaining full audit trails for compliance and security monitoring.

### Objectives

- Create five user accounts with restricted sudo privileges
- Prevent direct root shell access (blocking `su`, `sudo su`, `sudo -i`, etc.)
- Enable comprehensive logging and session recording
- Integrate with Wazuh SIEM for centralized monitoring and alerting
- Maintain individual user accountability and traceability

### Scope

This implementation applies to Ubuntu 20.04 LTS and later versions with Wazuh agent installed.

## Prerequisites

### System Requirements

- Ubuntu 20.04 LTS or later
- Root or sudo access for initial configuration
- Minimum 2GB free disk space for logs
- Wazuh agent installed and configured

### Required Packages

```bash
sudo apt update
sudo apt install -y auditd audispd-plugins
```

### Network Requirements

- Connectivity to Wazuh manager (typically port 1514)
- NTP configured for accurate timestamps

## Architecture
### Security Model

```text
┌─────────────────────────────────────────────┐
│          Restricted Sudo Users              │
│  (user1, user2, user3, user4, user5)        │
└──────────────────┬──────────────────────────┘
                   │
                   ▼
        ┌──────────────────────┐
        │   Sudo with Logging  │
        │  - Command logging   │
        │  - I/O recording     │
        │  - Restrictions      │
        └──────────┬───────────┘
                   │
        ┌──────────┴───────────┐
        │                      │
        ▼                      ▼
   ┌─────────┐          ┌──────────┐
   │ Auditd  │          │ Syslog   │
   └────┬────┘          └─────┬────┘
        │                     │
        └──────────┬──────────┘
                   ▼
            ┌─────────────┐
            │ Wazuh Agent │
            └──────┬──────┘
                   │
                   ▼
            ┌─────────────┐
            │Wazuh Manager│
            └─────────────┘

```
              

### Log Flow

1. User executes sudo command
2. Sudo logs to `/var/log/sudo.log` and `/var/log/sudo-io/`
3. System logs authentication events to `/var/log/auth.log`
4. Auditd captures system calls to `/var/log/audit/audit.log`
5. Wazuh agent monitors all log sources
6. Events forwarded to Wazuh manager for analysis

## Implementation Steps

**Step 1: User Account Creation**

Create five user accounts with strong passwords:

```bash
# Create users
sudo adduser user1
sudo adduser user2
sudo adduser user3
sudo adduser user4
sudo adduser user5
```

**Best Practice:** Enforce password complexity requirements:

```bash
sudo apt install libpam-pwquality
```

Edit `/etc/security/pwquality.conf`:

```
minlen=12
dcredit=-1
ucredit=-1
ocredit=-1
lcredit=-1
```

**Step 2: Configure Sudo Logging**

Edit the main sudoers file to enable comprehensive logging:

```bash
sudo visudo
```

Add the following directives after the existing `Defaults` lines:

```
# Enhanced logging and session recording
Defaults        logfile="/var/log/sudo.log"
Defaults        log_input, log_output
Defaults        iolog_dir=/var/log/sudo-io
Defaults        iolog_file=%{seq}
Defaults        iolog_flush
Defaults        timestamp_timeout=15
Defaults        passwd_timeout=5
Defaults        passwd_tries=3

# Exclude certain commands from I/O logging to reduce noise
Defaults!/usr/bin/sudoreplay !log_output
Defaults!/sbin/reboot !log_output
Defaults!/sbin/shutdown !log_output
```

**Configuration Explanation:**

- `logfile`: Central log file for all sudo commands
- `log_input, log_output`: Record all terminal input/output
- `iolog_dir`: Directory for session recordings
- `iolog_flush`: Write logs immediately (prevents data loss)
- `timestamp_timeout`: Sudo password cache duration (15 minutes)

**Step 3: Create Log Directories**

```bash
# Create sudo I/O log directory
sudo mkdir -p /var/log/sudo-io

# Set appropriate permissions
sudo chmod 750 /var/log/sudo-io
sudo chown root:adm /var/log/sudo-io
```

**Step 4: Configure Restricted Sudo Access**

Create a dedicated sudoers configuration file for restricted users:

```bash
sudo visudo -f /etc/sudoers.d/restricted_users
```

Add the following configuration:

```
# Restricted Sudo Configuration for Monitored Users
# Created: [DATE]
# Purpose: Allow sudo privileges while blocking direct root access

# User: user1
user1 ALL=(ALL:ALL) ALL, \
    !/bin/su, !/usr/bin/su, \
    !/bin/bash, !/usr/bin/bash -i, \
    !/bin/sh, !/bin/dash, \
    !/usr/bin/sudo -i, !/usr/bin/sudo -s, \
    !/usr/bin/sudo su, !/usr/bin/sudo /bin/su, \
    !/usr/bin/passwd root, \
    !/usr/sbin/visudo

# User: user2
user2 ALL=(ALL:ALL) ALL, \
    !/bin/su, !/usr/bin/su, \
    !/bin/bash, !/usr/bin/bash -i, \
    !/bin/sh, !/bin/dash, \
    !/usr/bin/sudo -i, !/usr/bin/sudo -s, \
    !/usr/bin/sudo su, !/usr/bin/sudo /bin/su, \
    !/usr/bin/passwd root, \
    !/usr/sbin/visudo

# User: user3
user3 ALL=(ALL:ALL) ALL, \
    !/bin/su, !/usr/bin/su, \
    !/bin/bash, !/usr/bin/bash -i, \
    !/bin/sh, !/bin/dash, \
    !/usr/bin/sudo -i, !/usr/bin/sudo -s, \
    !/usr/bin/sudo su, !/usr/bin/sudo /bin/su, \
    !/usr/bin/passwd root, \
    !/usr/sbin/visudo

# User: user4
user4 ALL=(ALL:ALL) ALL, \
    !/bin/su, !/usr/bin/su, \
    !/bin/bash, !/usr/bin/bash -i, \
    !/bin/sh, !/bin/dash, \
    !/usr/bin/sudo -i, !/usr/bin/sudo -s, \
    !/usr/bin/sudo su, !/usr/bin/sudo /bin/su, \
    !/usr/bin/passwd root, \
    !/usr/sbin/visudo

# User: user5
user5 ALL=(ALL:ALL) ALL, \
    !/bin/su, !/usr/bin/su, \
    !/bin/bash, !/usr/bin/bash -i, \
    !/bin/sh, !/bin/dash, \
    !/usr/bin/sudo -i, !/usr/bin/sudo -s, \
    !/usr/bin/sudo su, !/usr/bin/sudo /bin/su, \
    !/usr/bin/passwd root, \
    !/usr/sbin/visudo
```

Set correct permissions:

```bash
sudo chmod 440 /etc/sudoers.d/restricted_users
```

**Restriction Explanation:**

- `!/bin/su, !/usr/bin/su`: Blocks `su` and `sudo su` commands
- `!/usr/bin/sudo -i, !/usr/bin/sudo -s`: Prevents interactive root shells
- `!/usr/bin/passwd root`: Prevents changing root password
- `!/usr/sbin/visudo`: Prevents sudoers file modification

**Step 5: Configure Auditd**

Install and configure auditd for enhanced system call monitoring:

```bash
# Install auditd
sudo apt install -y auditd audispd-plugins

# Enable and start service
sudo systemctl enable auditd
sudo systemctl start auditd
```

Create audit rules for sudo-related activities:

```bash
sudo nano /etc/audit/rules.d/sudo-monitoring.rules
```

Add the following rules:

```
# Audit sudo-related commands
-w /usr/bin/sudo -p x -k sudo_execution
-w /bin/su -p x -k su_execution
-w /usr/bin/su -p x -k su_execution

# Monitor sudoers file changes
-w /etc/sudoers -p wa -k sudoers_changes
-w /etc/sudoers.d/ -p wa -k sudoers_changes

# Monitor user authentication
-w /var/log/auth.log -p wa -k auth_log_changes
-w /var/log/sudo.log -p wa -k sudo_log_changes

# Monitor privileged command execution
-a always,exit -F arch=b64 -S execve -F euid=0 -k root_commands
-a always,exit -F arch=b32 -S execve -F euid=0 -k root_commands

# Monitor password changes
-w /usr/bin/passwd -p x -k passwd_changes
-w /etc/shadow -p wa -k shadow_changes

# Monitor session changes
-w /var/run/utmp -p wa -k session_changes
-w /var/log/wtmp -p wa -k session_changes
-w /var/log/btmp -p wa -k session_changes
```

Load the audit rules:

```bash
sudo augenrules --load
sudo systemctl restart auditd
```

Verify rules are loaded:

```bash
sudo auditctl -l
```

**Step 6: Configure Wazuh Agent**

Edit the Wazuh agent configuration:

```bash
sudo nano /var/ossec/etc/ossec.conf
```

Add or verify the following log monitoring sections:

```xml
<ossec_config>
  <!-- Sudo command logging -->
  <localfile>
    <log_format>syslog</log_format>
    <location>/var/log/sudo.log</location>
  </localfile>

  <!-- Authentication logs -->
  <localfile>
    <log_format>syslog</log_format>
    <location>/var/log/auth.log</location>
  </localfile>

  <!-- Audit logs -->
  <localfile>
    <log_format>audit</log_format>
    <location>/var/log/audit/audit.log</location>
  </localfile>

  <!-- System logs -->
  <localfile>
    <log_format>syslog</log_format>
    <location>/var/log/syslog</location>
  </localfile>

  <!-- Monitor sudoers file changes -->
  <syscheck>
    <directories check_all="yes" realtime="yes">/etc/sudoers</directories>
    <directories check_all="yes" realtime="yes">/etc/sudoers.d</directories>
  </syscheck>

  <!-- Command monitoring -->
  <wodle name="command">
    <disabled>no</disabled>
    <tag>last_logins</tag>
    <command>last -n 20</command>
    <interval>5m</interval>
    <ignore_output>no</ignore_output>
    <run_on_start>yes</run_on_start>
    <timeout>10</timeout>
  </wodle>
</ossec_config>
```

Restart Wazuh agent:

```bash
sudo systemctl restart wazuh-agent
sudo systemctl status wazuh-agent
```

**Step 7: Configure Log Rotation**

Create log rotation configuration to manage disk space:

```bash
sudo nano /etc/logrotate.d/sudo-custom
```

Add the following:

```
/var/log/sudo.log {
    weekly
    rotate 12
    compress
    delaycompress
    missingok
    notifempty
    create 0640 root adm
    sharedscripts
    postrotate
        systemctl reload rsyslog > /dev/null 2>&1 || true
    endscript
}

/var/log/sudo-io/*/* {
    monthly
    rotate 6
    compress
    delaycompress
    missingok
    notifempty
    create 0640 root adm
}
```

Test log rotation:

```bash
sudo logrotate -d /etc/logrotate.d/sudo-custom
```

## Verification & Testing

**Test 1: Verify Sudo Access**

Login as one of the restricted users and test basic sudo functionality:

```bash
# Switch to user1
su - user1

# Test allowed command
sudo apt update

# Expected: Command executes successfully
```

**Test 2: Verify Root Shell Restrictions**

Attempt to access root shell using various methods:

```bash
# These should all be DENIED
sudo su
sudo su -
sudo -i
sudo -s
sudo bash
sudo /bin/bash
su
su -
sudo passwd root
```

**Expected Output:**

`Sorry, user user1 is not allowed to execute '/bin/su' as root on [hostname].`

**Test 3: Verify Logging**

Check that commands are being logged:

```bash
# Execute a sudo command as user1
sudo ls /root

# Check sudo log
sudo tail -f /var/log/sudo.log

# Check auth log
sudo tail -f /var/log/auth.log

# Check audit log
sudo ausearch -k sudo_execution -i
```

**Test 4: Verify Session Recording**

```bash
# Execute a sudo command
sudo apt update

# List recorded sessions
sudo ls -la /var/log/sudo-io/

# Replay a session (replace 00/00/01 with actual path)
sudo sudoreplay /var/log/sudo-io/00/00/01
```

**Test 5: Verify Wazuh Integration**

Check Wazuh agent logs:

```bash
sudo tail -f /var/ossec/logs/ossec.log
```

On Wazuh manager, verify events are being received:

&limit=10\"">

```bash
# Search for events from the agent
curl -u <user>:<password> -k -X GET "https://localhost:55000/events?pretty=true&agent_id=<agent_id>&limit=10"
```

## Monitoring & Alerts

### Key Wazuh Rules for Monitoring

The following Wazuh rule IDs are relevant for monitoring restricted users:

| Rule ID | Description | Severity |
| --- | --- | --- |
| 5402 | Successful sudo execution | Low |
| 5403 | Failed sudo execution | Medium |
| 5501 | Login session opened | Low |
| 5551 | Multiple authentication failures | High |
| 80792 | Sudoers file modified | Critical |
| 80793 | Unauthorized sudo attempt | High |

**Custom Wazuh Rules**

Add custom rules to `/var/ossec/etc/rules/local_rules.xml` on the Wazuh manager:

```xml
<group name="sudo,">
  <!-- Alert on blocked sudo su attempts -->
  <rule id="100001" level="10">
    <if_sid>5403</if_sid>
    <match>not allowed to execute '/bin/su'|not allowed to execute '/usr/bin/sudo -i'</match>
    <description>Attempt to bypass sudo restrictions detected</description>
    <group>policy_violation,pci_dss_10.2.5,gdpr_IV_35.7.d,</group>
  </rule>

  <!-- Alert on multiple failed sudo attempts -->
  <rule id="100002" level="12" frequency="3" timeframe="300">
    <if_matched_sid>5403</if_matched_sid>
    <same_source_user />
    <description>Multiple failed sudo attempts from same user</description>
    <group>authentication_failures,pci_dss_10.2.4,pci_dss_10.2.5,</group>
  </rule>

  <!-- Alert on sudoers file changes -->
  <rule id="100003" level="15">
    <if_sid>550</if_sid>
    <match>/etc/sudoers</match>
    <description>Critical: Sudoers configuration file modified</description>
    <group>policy_violation,pci_dss_10.6.1,gdpr_IV_35.7.d,</group>
  </rule>

  <!-- Alert on passwd root attempts -->
  <rule id="100004" level="12">
    <if_sid>5403</if_sid>
    <match>not allowed to execute '/usr/bin/passwd root'</match>
    <description>Attempt to change root password detected</description>
    <group>policy_violation,pci_dss_10.2.5,</group>
  </rule>
</group>
```

Restart Wazuh manager:

```bash
sudo systemctl restart wazuh-manager
```

### Recommended Alerts

Configure email or SIEM integration for the following events:

**Critical Alerts (Immediate notification)**

- Sudoers file modifications
- Multiple failed sudo attempts (3+ in 5 minutes)
- Attempts to bypass restrictions

**Warning Alerts (Daily digest)**

- Failed sudo executions
- Unusual command patterns
- Off-hours administrative activity

**Informational Alerts (Weekly reports)**

- Sudo usage statistics per user
- Most frequently executed commands
- Login patterns

## Troubleshooting

**Issue 1: User Cannot Execute Any Sudo Commands**

**Symptoms:**

`user1 is not in the sudoers file. This incident will be reported.`

**Solution:**

```bash
# Verify sudoers.d file exists and has correct permissions
sudo ls -la /etc/sudoers.d/restricted_users
sudo chmod 440 /etc/sudoers.d/restricted_users

# Verify syntax
sudo visudo -c -f /etc/sudoers.d/restricted_users
```

**Issue 2: Logging Not Working**

**Symptoms:** No entries in `/var/log/sudo.log`

**Solution:**

```bash
# Verify log file exists and has correct permissions
sudo touch /var/log/sudo.log
sudo chmod 640 /var/log/sudo.log
sudo chown root:adm /var/log/sudo.log

# Verify sudo configuration
sudo grep logfile /etc/sudoers

# Test with a sudo command and check
sudo -u user1 sudo ls
sudo tail /var/log/sudo.log
```

**Issue 3: Wazuh Not Receiving Logs**

**Symptoms:** Events not appearing in Wazuh dashboard

**Solution:**

```bash
# Check Wazuh agent status
sudo systemctl status wazuh-agent

# Verify log file permissions
sudo ls -la /var/log/sudo.log /var/log/auth.log

# Check Wazuh configuration
sudo /var/ossec/bin/wazuh-control info

# Restart agent
sudo systemctl restart wazuh-agent

# Check connectivity to manager
sudo grep "Connected to the server" /var/ossec/logs/ossec.log
```

**Issue 4: Audit Rules Not Loading**

**Symptoms:**

`sudo auditctl -l
# Returns: No rules`

**Solution:**

```bash
# Check audit service status
sudo systemctl status auditd

# Manually load rules
sudo augenrules --load

# Verify rules file syntax
sudo auditctl -R /etc/audit/rules.d/sudo-monitoring.rules

# If issues persist, restart auditd
sudo systemctl restart auditd
```

**Issue 5: Session Recording Filling Disk**

**Symptoms:** `/var/log/sudo-io/` consuming excessive disk space

**Solution:**

```bash
# Check disk usage
sudo du -sh /var/log/sudo-io/

# Implement more aggressive log rotation
sudo nano /etc/logrotate.d/sudo-custom
# Change: monthly -> weekly
# Change: rotate 6 -> rotate 4

# Manually clean old sessions (older than 30 days)
sudo find /var/log/sudo-io/ -type f -mtime +30 -delete

# Consider excluding high-volume low-value commands
sudo visudo
# Add: Defaults!/usr/bin/apt !log_output
```

## Security Considerations

### Defense in Depth

This implementation provides multiple security layers:

1. **Access Control:** Sudo restrictions prevent direct root access
2. **Logging:** Comprehensive command and session logging
3. **Auditing:** System-level call monitoring via auditd
4. **Monitoring:** Real-time SIEM analysis via Wazuh
5. **Alerting:** Immediate notification of policy violations

**Known Limitations**

**Bypass Possibilities:**

While this configuration significantly restricts access, determined users with sudo privileges could potentially:

1. **Edit files directly:** Users can still edit system files with `sudo vim`, `sudo nano`, etc.
    
    **Mitigation:** Monitor file integrity with Wazuh syscheck
    
2. **Execute scripted shells:** Users might use `sudo python`, `sudo perl`, etc. to spawn shells
    
    **Mitigation:** Add specific restrictions if needed:
    
    `!/usr/bin/python* -c*,
    !/usr/bin/perl* -e*,
    !/usr/bin/ruby* -e*`
    
3. **Copy and execute binaries:** Users could copy `/bin/bash` and execute it
    
    **Mitigation:** Monitor `/tmp` and user home directories with auditd
    

**Hardening Recommendations**

1. **Implement Multi-Factor Authentication (MFA):**
    
    ```bash
    sudo apt install libpam-google-authenticator
    ```
    
2. **Enforce SSH Key-Based Authentication:**
    
    ```bash
    # Edit /etc/ssh/sshd_config
    PasswordAuthentication no
    PubkeyAuthentication yes
    ```
    
3. **Implement Account Lockout Policies:**
    
    ```bash
    # Edit /etc/pam.d/common-auth
    auth required pam_faillock.so preauth audit deny=3 unlock_time=900
    ```
    
4. **Regular Security Audits:**
    - Weekly review of sudo logs
    - Monthly access rights review
    - Quarterly penetration testing
5. **Principle of Least Privilege:**
    
    Consider creating role-based sudo configurations instead of ALL=(ALL:ALL) ALL
    

## Appendix

**Complete Sudoers Configuration Example**

Full example of `/etc/sudoers` with all recommended settings:

```
#
# This file MUST be edited with the 'visudo' command as root.
#
Defaults        env_reset
Defaults        mail_badpass
Defaults        secure_path="/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin:/snap/bin"
Defaults        use_pty
Defaults        logfile="/var/log/sudo.log"
Defaults        log_input, log_output
Defaults        iolog_dir=/var/log/sudo-io
Defaults        iolog_file=%{seq}
Defaults        iolog_flush
Defaults        timestamp_timeout=15
Defaults        passwd_timeout=5
Defaults        passwd_tries=3
Defaults        badpass_message="Invalid password attempt. This incident will be reported."
Defaults        lecture=always
Defaults        lecture_file=/etc/sudo_lecture.txt
Defaults!/usr/bin/sudoreplay !log_output
Defaults!/sbin/reboot !log_output
Defaults!/sbin/shutdown !log_output

# Host alias specification
# User alias specification
# Cmnd alias specification

# User privilege specification
root    ALL=(ALL:ALL) ALL

# Members of the admin group may gain root privileges
%admin ALL=(ALL) ALL

# Allow members of group sudo to execute any command
%sudo   ALL=(ALL:ALL) ALL

# Include sudoers.d files
#includedir /etc/sudoers.d
```

**Audit Rules Reference**

Complete audit rules for comprehensive monitoring:

```
## Audit Rules for Sudo Monitoring
## /etc/audit/rules.d/sudo-monitoring.rules

# Remove any existing rules
-D

# Set buffer size
-b 8192

# Set failure mode (0=silent, 1=printk, 2=panic)
-f 1

# Audit sudo execution
-w /usr/bin/sudo -p x -k sudo_execution
-w /bin/su -p x -k su_execution
-w /usr/bin/su -p x -k su_execution

# Monitor configuration changes
-w /etc/sudoers -p wa -k sudoers_changes
-w /etc/sudoers.d/ -p wa -k sudoers_changes
-w /etc/pam.d/ -p wa -k pam_changes
-w /etc/security/ -p wa -k security_changes

# Monitor logs
-w /var/log/sudo.log -p wa -k sudo_log_changes
-w /var/log/auth.log -p wa -k auth_log_changes
-w /var/log/secure -p wa -k secure_log_changes

# Monitor privileged commands
-a always,exit -F arch=b64 -S execve -F euid=0 -k root_commands
-a always,exit -F arch=b32 -S execve -F euid=0 -k root_commands

# Monitor authentication
-w /usr/bin/passwd -p x -k passwd_changes
-w /etc/shadow -p wa -k shadow_changes
-w /etc/gshadow -p wa -k gshadow_changes
-w /etc/group -p wa -k group_changes

# Monitor session activity
-w /var/run/utmp -p wa -k session_changes
-w /var/log/wtmp -p wa -k session_changes
-w /var/log/btmp -p wa -k session_changes

# Monitor network configuration
-w /etc/hosts -p wa -k network_changes
-w /etc/network/ -p wa -k network_changes

# Make configuration immutable (requires reboot to change)
-e 2
```

**Useful Commands Reference**

| Task | Command |
| --- | --- |
| View sudo log | `sudo tail -f /var/log/sudo.log` |
| View auth log | `sudo tail -f /var/log/auth.log` |
| View audit log | `sudo tail -f /var/log/audit/audit.log` |
| Search audit events | `sudo ausearch -k sudo_execution -i` |
| List sudo sessions | `sudo ls -la /var/log/sudo-io/` |
| Replay sudo session | `sudo sudoreplay /var/log/sudo-io/00/00/01` |
| Check sudo config | `sudo visudo -c` |
| Test audit rules | `sudo auditctl -l` |
| View Wazuh logs | `sudo tail -f /var/ossec/logs/ossec.log` |
| Check Wazuh status | `sudo /var/ossec/bin/wazuh-control status` |
| View user sudo history | `sudo grep user1 /var/log/sudo.log` |
| List active sessions | `who` or `w` |
| View last logins | `last -n 20` |
| View failed logins | `sudo lastb` |

**Log Sample Formats**

**Sudo Log Entry:**

```
Jan 26 10:15:23 hostname sudo: user1 : TTY=pts/0 ; PWD=/home/user1 ; USER=root ; COMMAND=/usr/bin/apt update
```

**Auth Log Entry:**

```
Jan 26 10:15:23 hostname sudo: pam_unix(sudo:session): session opened for user root by user1(uid=0)
```

**Audit Log Entry:**

```
type=EXECVE msg=audit(1706265323.123:456): argc=3 a0="sudo" a1="apt" a2="update"
```

**Compliance Mapping**

| Requirement | Implementation |
| --- | --- |
| PCI DSS 10.2.2 | Sudo command logging |
| PCI DSS 10.2.5 | Privilege escalation monitoring |
| PCI DSS 10.3 | Audit trails with user identification |
| GDPR Article 32 | Access control and logging |
| HIPAA 164.308(a)(4) | Activity tracking |
| SOC 2 CC6.1 | Logical access controls |
| ISO 27001 A.9.2.3 | Privileged access management |
| NIST 800-53 AU-2 | Audit events |
| NIST 800-53 AC-6 | Least privilege |

**Support Contacts**

| Role | Responsibility | Contact |
| --- | --- | --- |
| System Administrator | Day-to-day operations | [Contact Info] |
| Security Team | Security incidents | [Contact Info] |
| Wazuh Administrator | SIEM management | [Contact Info] |
| IT Manager | Escalations | [Contact Info] |

Document Version: 1.0 | Last Updated: January 26, 2026 | Author: Rabiul Hasan
Review Cycle: Quarterly | Next Review Date: April 26, 2026
