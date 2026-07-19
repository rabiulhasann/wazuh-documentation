# 🛡️ OWASP WAF Control & Juice Shop

## Complete Setup Guide for Beginners

> **Purpose:** This guide demonstrates how to set up a secure web application testing environment using **OWASP WAF Control** as a Web Application Firewall and **OWASP Juice Shop** as a vulnerable web application for testing.
> 

---

## 📑 Table of Contents

- 📖 Overview
- 🏗️ Architecture Overview
- 🛡️ VM1: WAF Control Setup
- 🧃 VM2: Juice Shop Setup
- 🧪 Testing WAF Protection
- 🔧 Troubleshooting Guide
- ⭐ Best Practices & Security Tips
- 🎓 Summary & Key Takeaways
- 📚 References & Additional Resources

---

## 📖 Overview

This guide demonstrates how to set up a secure web application environment using **OWASP WAF Control** as a Web Application Firewall and **OWASP Juice Shop** as a vulnerable web application for testing.

### 🎯 What You'll Learn

- Installing and configuring OWASP WAF Control with ModSecurity
- Setting up NGINX as a reverse proxy
- Deploying OWASP Juice Shop in Docker
- Configuring firewall rules to restrict direct access
- Testing WAF protection against SQL injection attacks

### 🖥️ System Requirements

| Component | Requirement |
| --- | --- |
| Virtual Machines | 2 VMs, Ubuntu 20.04+ recommended |
| RAM | Minimum 2 GB per VM |
| Disk Space | 20 GB per VM |
| Network | VMs must be able to communicate |
| Software | Docker, NGINX, Git, UFW |

---

## 🏗️ Architecture Overview

### Traffic Flow

```
👤 Client/User Browser
        |
        v
🛡️ WAF VM: 192.168.18.3:81
        |
        v
🔀 NGINX Reverse Proxy + ModSecurity + OWASP CRS
        |
        v
🧃 Juice Shop VM: 192.168.18.120:3000
```

### 📋 VM Configuration

### 🖥️ VM1 - WAF Control Server

- **IP Address:** `192.168.18.3`
- **Services:** NGINX + ModSecurity + OWASP CRS
- **Port:** `81` Reverse Proxy

### 🖥️ VM2 - Juice Shop Application Server

- **IP Address:** `192.168.18.120`
- **Services:** Docker + OWASP Juice Shop
- **Port:** `3000` Restricted Access

> ⚠️ **Security Note:**
Juice Shop on VM2 is configured to accept connections only from the WAF VM `192.168.18.3`. Direct access from other IP addresses is blocked using UFW and iptables.
> 

---

## 🛡️ VM1: WAF Control Setup

**Target VM:** `192.168.18.3`

### 1. Download Installation Script

Download the official OWASP WAF Control installation script from `wafcontrol.org`.

```bash
curl -fsSL https://wafcontrol.org/download/install.sh -o install.sh
chmod +x install.sh
```

### 2. Install Dependencies

Install required libraries for ModSecurity compilation, including PCRE2 for regular expression support.

```bash
# Update package lists
sudo apt-get update

# Install PCRE2 and dependencies
sudo apt-get install -y libpcre2-dev libpcre2-8-0 libpcre3 libpcre3-dev
```

### 💡 What is PCRE2?

PCRE2, or Perl Compatible Regular Expressions 2, is a library that provides regex pattern-matching capabilities required by ModSecurity for analyzing HTTP requests.

### 3. Run Installation Script

Execute the installation script. This installs NGINX with the ModSecurity module and OWASP Core Rule Set.

```bash
sudo ./install.sh
```

> ⏳ **Note:** Installation may take 5-10 minutes depending on your system. The script compiles NGINX with ModSecurity from source.
> 

### 4. Configure Core Rule Set CRS

The default CRS version 4.x may have compatibility issues in some environments. This guide uses stable version `v3.3.5`.

```bash
# Navigate to ModSecurity directory
cd /etc/nginx/modsec

# Remove CRS v4 if installed
sudo rm -rf coreruleset-4.*

# Clone OWASP Core Rule Set repository
sudo git clone https://github.com/coreruleset/coreruleset.git

# Switch to stable v3.3.5
cd coreruleset
sudo git checkout v3.3.5
```

### 5. Configure CRS Setup File

Copy the example configuration file to create the active CRS setup file.

```bash
sudo cp /etc/nginx/modsec/coreruleset/crs-setup.conf.example \\
  /etc/nginx/modsec/coreruleset/crs-setup.conf
```

### 6. Update ModSecurity Main Configuration

Edit the main ModSecurity configuration file.

```bash
sudo nano /etc/nginx/modsec/main.conf
```

Add these lines at the end of the file:

```
Include /etc/nginx/modsec/coreruleset/crs-setup.conf
Include /etc/nginx/modsec/coreruleset/rules/*.conf
```

### 7. Configure NGINX Reverse Proxy

Create a reverse proxy configuration for Juice Shop.

```bash
sudo nano /etc/nginx/conf.d/juiceshop.conf
```

Add the following configuration:

```
server {
    listen 81;
    server_name _;

    # Enable ModSecurity WAF
    modsecurity on;
    modsecurity_rules_file /etc/nginx/modsec/main.conf;

    # Logs
    access_log /var/log/nginx/juiceshop_access.log;
    error_log /var/log/nginx/juiceshop_error.log;

    location / {
        # Proxy to Juice Shop on VM2
        proxy_pass http://192.168.18.120:3000;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
```

### 📝 Configuration Breakdown

- `listen 81` — WAF listens on port 81.
- `modsecurity on` — Enables WAF protection.
- `proxy_pass` — Forwards traffic to the Juice Shop VM.
- `proxy_set_header` — Preserves client request information.

### 8. Fix REQUEST-922 Rule If Needed

Some installations may show syntax errors in the multipart attack rule.

```bash
sudo nano /etc/nginx/modsec/coreruleset/rules/REQUEST-922-MULTIPART-ATTACK.conf
```

> ⚠️ Only edit this file if `nginx -t` shows errors related to this rule. Check for syntax issues around the reported line number, such as line 139, and ensure proper `SecRule` formatting.
> 

### 9. Test and Restart NGINX

Validate the configuration and restart NGINX.

```bash
# Test configuration
sudo nginx -t

# If test passes, restart NGINX
sudo systemctl restart nginx

# Check status
sudo systemctl status nginx
```

### ✅ Success Indicators

- `nginx -t` shows `syntax is ok`.
- `nginx -t` shows `test is successful`.
- `systemctl status nginx` shows `active (running)`.

### 10. Verify ModSecurity Installation

Confirm that ModSecurity is properly compiled or loaded with NGINX.

```bash
nginx -V 2>&1 | grep -i modsecurity
```

Expected output may include something similar to:

```
--add-dynamic-module=/path/to/ModSecurity-nginx
```

### 🔍 VM1 Complete Command History

```bash
curl -fsSL https://wafcontrol.org/download/install.sh -o install.sh
chmod +x install.sh
sudo ./install.sh
nano install.sh -l
sudo apt-get update
sudo apt-get install -y libpcre2-dev
sudo apt-get install -y libpcre2-8-0 libpcre2-dev libpcre3 libpcre3-dev
./install.sh
nginx -t
cd /etc/nginx/modsec
rm -rf coreruleset-4.*
git clone https://github.com/coreruleset/coreruleset.git
cd coreruleset
git checkout v3.3.5
nginx -t
systemctl reload nginx
nano /etc/nginx/modsec/main.conf
cp /etc/nginx/modsec/coreruleset/crs-setup.conf.example /etc/nginx/modsec/coreruleset/crs-setup.conf
nginx -t
sudo nano /etc/nginx/conf.d/juiceshop.conf
sudo nginx -t
sudo systemctl restart nginx
```

---

## 🧃 VM2: Juice Shop Setup

**Target VM:** `192.168.18.120`

### 1. Deploy Juice Shop with Docker

Run OWASP Juice Shop as a Docker container accessible on port `3000`.

```bash
# Pull and run Juice Shop container
docker run -d -p 0.0.0.0:3000:3000 bkimminich/juice-shop

# Verify container is running
docker ps
```

### 🐳 Docker Command Breakdown

- `d` — Run in detached mode.
- `p 0.0.0.0:3000:3000` — Expose container port 3000 on host port 3000.
- `bkimminich/juice-shop` — Official Juice Shop image.

### 2. Configure UFW Firewall

Set up basic firewall rules to allow SSH and prepare access restrictions.

```bash
# Reset UFW to start fresh
sudo ufw reset

# Allow SSH first
sudo ufw allow 22/tcp
sudo ufw allow 22/udp

# Enable firewall
sudo ufw enable

# Check status
sudo ufw status verbose
```

> 🚨 **Critical Warning:** Always allow SSH port `22` before enabling UFW, or you may lock yourself out of the VM.
> 

### 3. Allow WAF VM Access

Configure UFW to allow connections from the WAF VM only.

```bash
# Allow WAF VM to access port 3000
sudo ufw allow from 192.168.18.3 to any port 3000 proto tcp
sudo ufw allow from 192.168.18.3 to any port 3000 proto udp

# Allow management IP if needed
sudo ufw allow from 192.168.69.8 to any port 3000 proto tcp
sudo ufw allow from 192.168.69.8 to any port 3000 proto udp

# Deny all other access to port 3000
sudo ufw deny 3000/tcp

# Reload firewall
sudo ufw reload
sudo ufw status verbose
```

### 4. Configure Docker iptables Rules

Docker can bypass UFW, so add iptables rules directly to the `DOCKER-USER` chain.

```bash
# Allow WAF VM through Docker networking
sudo iptables -I DOCKER-USER -s 192.168.18.3 -p tcp --dport 3000 -j ACCEPT
sudo iptables -I DOCKER-USER -s 192.168.18.3 -p udp --dport 3000 -j ACCEPT

# Allow management IP if needed
sudo iptables -I DOCKER-USER -s 192.168.69.8 -p tcp --dport 3000 -j ACCEPT
sudo iptables -I DOCKER-USER -s 192.168.69.8 -p udp --dport 3000 -j ACCEPT

# Drop all other traffic to port 3000
sudo iptables -I DOCKER-USER -p tcp --dport 3000 -j DROP

# Verify rules
sudo iptables -S DOCKER-USER
```

### 🔍 Understanding the DOCKER-USER Chain

Docker creates its own iptables rules that can bypass UFW. The `DOCKER-USER` chain allows you to add custom rules that Docker will process. Rules are evaluated in order, so `ACCEPT` rules must appear before `DROP` rules.

> ⚠️ If rules are inserted with `-I`, they are inserted at the top. Verify rule order with `sudo iptables -L DOCKER-USER -n --line-numbers`.
> 

### 5. Make iptables Rules Persistent

Save iptables rules so they persist after reboot.

```bash
# Install iptables-persistent
sudo apt-get install -y iptables-persistent

# Save current rules
sudo netfilter-persistent save
```

Alternatively, add rules to UFW after-rules:

```bash
sudo nano /etc/ufw/after.rules
```

Add these lines before the final `COMMIT` if appropriate for your environment:

```
# Docker bypass rules for Juice Shop
*filter
:DOCKER-USER - [0:0]
-A DOCKER-USER -s 192.168.18.3 -p tcp --dport 3000 -j ACCEPT
-A DOCKER-USER -s 192.168.18.3 -p udp --dport 3000 -j ACCEPT
-A DOCKER-USER -s 192.168.69.8 -p tcp --dport 3000 -j ACCEPT
-A DOCKER-USER -s 192.168.69.8 -p udp --dport 3000 -j ACCEPT
-A DOCKER-USER -p tcp --dport 3000 -j DROP
COMMIT
```

### 6. Verify Configuration

Test that the firewall is working correctly.

### ✅ Test from WAF VM

```bash
curl http://192.168.18.120:3000
```

Expected result: Juice Shop HTML content should be returned.

### ❌ Test from another machine

```bash
curl http://192.168.18.120:3000
```

Expected result: Connection timeout or refused.

### 🔍 VM2 Complete Command History

```bash
docker run -d -p 0.0.0.0:3000:3000 bkimminich/juice-shop
docker ps
sudo ufw reset
sudo ufw status verbose
sudo ufw allow 22/tcp
sudo ufw enable
sudo ufw status verbose
sudo ufw allow 22/udp
sudo ufw status verbose
ip a
sudo ufw allow from 192.168.18.3 to any port 3000 proto tcp
sudo ufw allow from 192.168.18.3 to any port 3000 proto udp
sudo ufw allow from 192.168.69.8 to any port 3000 proto tcp
sudo ufw allow from 192.168.69.8 to any port 3000 proto udp
sudo ufw deny 3000/tcp
sudo ufw reload
sudo ufw status verbose
sudo iptables -I DOCKER-USER -s 192.168.18.3 -p tcp --dport 3000 -j ACCEPT
sudo iptables -I DOCKER-USER -s 192.168.69.8 -p tcp --dport 3000 -j ACCEPT
sudo iptables -I DOCKER-USER -s 192.168.69.8 -p udp --dport 3000 -j ACCEPT
sudo iptables -I DOCKER-USER -s 192.168.18.3 -p udp --dport 3000 -j ACCEPT
sudo iptables -I DOCKER-USER -p tcp --dport 3000 -j DROP
sudo iptables -S DOCKER-USER
nano /etc/ufw/after.rules
sudo ufw reload
sudo ufw status verbose
sudo iptables -S DOCKER-USER
```

---

## 🧪 Testing WAF Protection

### 🎯 Testing Objective

Verify that the WAF successfully blocks SQL injection attacks while allowing legitimate traffic.

### ✅ Test 1: Normal Request

From any client machine:

```bash
curl http://192.168.18.3:81/
```

Expected result:

- HTTP `200 OK`
- Juice Shop HTML content returned
- No WAF blocking

### 🚫 Test 2: SQL Injection Attack

```bash
curl "http://192.168.18.3:81/rest/products/search?q=' OR 1=1--"
```

Expected result:

- HTTP `403 Forbidden`
- ModSecurity blocks the request
- Attack is logged in the audit log

### 📊 Test 3: Monitor Audit Logs

On the WAF VM:

```bash
sudo tail -f /var/log/modsec_audit.log
```

Look for entries similar to:

```
Message: SQL Injection Attack Detected
Rule ID: 942100
Action: Denied
```

### 🌐 Test 4: Browser Access

1. Open a browser and navigate to:

```
http://192.168.18.3:81
```

1. Browse Juice Shop normally.
2. Try a search payload such as:

```
' OR 1=1--
```

1. Confirm that a `403 Forbidden` response is shown.

### 🔒 Test 5: Direct Access Block

```bash
curl http://192.168.18.120:3000
```

Expected result:

- Connection timeout or refused
- No response from Juice Shop
- Confirms firewall rules are working

### 📝 Additional Test Scenarios

### Cross-Site Scripting XSS Test

```bash
curl "http://192.168.18.3:81/? q=<script>alert('XSS')</script>"
```

### Path Traversal Test

```bash
curl "http://192.168.18.3:81/../../../../etc/passwd"
```

### Command Injection Test

```bash
curl "http://192.168.18.3:81/?cmd=cat%20/etc/passwd"
```

Expected result: These malicious requests should be blocked by the WAF with HTTP `403`.

---

## 🔧 Troubleshooting Guide

### ❌ Problem: NGINX Fails to Start After Installation

Possible causes:

- Configuration syntax errors
- Port conflicts on port `80` or `81`
- ModSecurity rule syntax errors

Solutions:

```bash
# Check configuration
sudo nginx -t

# Check for port conflicts
sudo lsof -i :81
sudo lsof -i :80

# View NGINX error logs
sudo tail -f /var/log/nginx/error.log

# Check NGINX service logs
sudo journalctl -u nginx -n 50
```

### ❌ Problem: 502 Bad Gateway When Accessing WAF

Possible causes:

- Juice Shop container is not running
- Firewall is blocking WAF-to-Juice-Shop communication
- Incorrect IP address in `proxy_pass`

Solutions:

```bash
# On Juice Shop VM, check container
docker ps
docker logs container_id

# Test connectivity from WAF VM
curl http://192.168.18.120:3000

# Check firewall rules on Juice Shop VM
sudo iptables -S DOCKER-USER
sudo ufw status verbose

# Verify NGINX proxy configuration
sudo cat /etc/nginx/conf.d/juiceshop.conf | grep proxy_pass
```

### ❌ Problem: WAF Not Blocking Malicious Requests

Possible causes:

- ModSecurity is not enabled
- CRS rules are not loaded
- Rules are in `DetectionOnly` mode

Solutions:

```bash
# Check ModSecurity status in config
sudo cat /etc/nginx/conf.d/juiceshop.conf | grep modsecurity

# Verify rules are loaded
sudo ls /etc/nginx/modsec/coreruleset/rules/

# Check SecRuleEngine mode
sudo cat /etc/nginx/modsec/main.conf | grep SecRuleEngine
```

Expected setting:

```
SecRuleEngine On
```

Not this:

```
SecRuleEngine DetectionOnly
```

Edit if needed:

```bash
sudo nano /etc/nginx/modsec/main.conf
sudo systemctl restart nginx
```

### ❌ Problem: Can Still Access Juice Shop Directly

Possible causes:

- UFW is not enabled
- iptables rules are not applied correctly
- You are testing from an allowed IP address

Solutions:

```bash
# Check UFW status
sudo ufw status verbose

# Verify iptables rule order
sudo iptables -L DOCKER-USER -n --line-numbers

# Check source IP from remote client if internet-based
curl ifconfig.me
```

Rules should be ordered with `ACCEPT` rules first and `DROP` rules last.

### ❌ Problem: Lost SSH Access After Enabling UFW

Prevention:

```bash
sudo ufw allow 22/tcp
sudo ufw allow 22/udp
sudo ufw enable
```

If locked out:

1. Access the VM through the hypervisor console.
2. Disable UFW:

```bash
sudo ufw disable
```

1. Allow SSH:

```bash
sudo ufw allow 22
```

1. Re-enable UFW:

```bash
sudo ufw enable
```

### ❌ Problem: ModSecurity Rules Causing False Positives

Find the rule ID from the audit log:

```bash
sudo tail /var/log/modsec_audit.log | grep 'id "'
```

Disable a specific rule, for example rule `920100`, by adding this before the CRS include statements in `main.conf`:

```
SecRuleRemoveById 920100
```

Or increase the inbound anomaly score threshold in `crs-setup.conf`:

```
SecAction "id:900110,phase:1,setvar:tx.inbound_anomaly_score_threshold=10"
```

---

## ⭐ Best Practices & Security Tips

### 🔐 Regular Updates

Keep OWASP CRS, ModSecurity, NGINX, Docker, and the operating system updated.

### 📊 Log Monitoring

Regularly review WAF logs to identify attack patterns and tune your rules.

### 🎯 Rule Tuning

Start with `DetectionOnly` mode, analyze false positives, tune rules, and then switch to blocking mode.

### 🔒 SSL/TLS

Use HTTPS in production with valid certificates from Let's Encrypt or another trusted Certificate Authority.

### 🚨 Alerting

Set up alerts for critical attacks using fail2ban, SIEM tools, or custom scripts.

### 💾 Backup Configuration

Regularly back up WAF configurations, CRS rules, NGINX configurations, and custom modifications.

### 📋 Production Deployment Checklist

- [ ]  WAF is enabled and in blocking mode: `SecRuleEngine On`
- [ ]  OWASP CRS is properly configured and loaded
- [ ]  SSL/TLS certificates are installed and valid
- [ ]  Firewall rules restrict direct application access
- [ ]  SSH access is secured using key-based authentication
- [ ]  Log rotation is configured for WAF audit logs
- [ ]  Monitoring and alerting are in place
- [ ]  Regular backups are scheduled
- [ ]  Update procedures are documented
- [ ]  Incident response plan is prepared

---

## ⚙️ Performance Optimization

### 🚀 Performance Tips

- **Rule Optimization:** Disable unused CRS rules to reduce processing overhead.
- **Request Body Limit:** Set an appropriate `SecRequestBodyLimit`.
- **Audit Logging:** Use `RelevantOnly` mode to log only relevant security events.
- **NGINX Workers:** Configure `worker_processes` based on CPU cores.
- **Connection Limits:** Set appropriate `client_max_body_size` and rate limits.

---

## 🎓 Advanced Configuration

### 🔧 Custom Rule Examples

### Block Specific User Agents

Add to `main.conf`:

```
SecRule REQUEST_HEADERS:User-Agent "@contains badbot" \\
    "id:1001,phase:1,deny,status:403,msg:'Bad bot blocked'"
```

### Rate Limiting by IP

Add to `main.conf`:

```
SecAction "id:1002,phase:1,initcol:ip=%{REMOTE_ADDR}"
SecRule IP:requests "@gt 100" \\
    "id:1003,phase:1,deny,status:429,msg:'Rate limit exceeded'"
SecAction "id:1004,phase:5,setvar:ip.requests=+1,expirevar:ip.requests=60"
```

### Whitelist Specific IPs

Add before CRS rules in `main.conf`:

```
SecRule REMOTE_ADDR "@ipMatch 192.168.1.0/24,10.0.0.0/8" \\
    "id:1005,phase:1,pass,ctl:ruleEngine=Off"
```

---

## 📈 Monitoring & Analytics

### Parse Audit Logs with CLI Tools

Count attacks by rule ID:

```bash
grep -o 'id "[0-9]*"' /var/log/modsec_audit.log | sort | uniq -c | sort -rn
```

Extract blocked IPs:

```bash
grep "403" /var/log/nginx/juiceshop_access.log | awk '{print $1}' | sort | uniq -c | sort -rn
```

Monitor real-time attacks:

```bash
tail -f /var/log/modsec_audit.log | grep -i "msg:"
```

### Install Log Analysis Tools

Install GoAccess:

```bash
sudo apt-get install -y goaccess
```

Analyze NGINX access logs:

```bash
goaccess /var/log/nginx/juiceshop_access.log --log-format=COMBINED
```

---

## 🔄 Maintenance Procedures

### Update OWASP CRS

```bash
cd /etc/nginx/modsec/coreruleset
sudo git fetch --tags
sudo git checkout latest-version
sudo nginx -t
sudo systemctl reload nginx
```

### Rotate Audit Logs

Create a logrotate configuration file:

```bash
sudo nano /etc/logrotate.d/modsec
```

Add configuration:

```
/var/log/modsec_audit.log {
    daily
    rotate 14
    compress
    delaycompress
    missingok
    notifempty
    create 0640 www-data www-data
    postrotate
        systemctl reload nginx > /dev/null 2>&1 || true
    endscript
}
```

### Backup Configuration

```bash
sudo mkdir -p /backup
sudo tar -czf /backup/waf-config-$(date +%Y%m%d).tar.gz \\
    /etc/nginx/modsec/ \\
    /etc/nginx/conf.d/ \\
    /etc/nginx/nginx.conf
```

---

## 🎓 Summary & Key Takeaways

### ✅ What You've Accomplished

- Installed and configured OWASP WAF Control with ModSecurity
- Deployed OWASP CRS `v3.3.5`
- Set up NGINX as a reverse proxy with WAF capabilities
- Deployed OWASP Juice Shop in Docker
- Configured firewall rules to enforce traffic through the WAF
- Tested protection against SQL injection
- Learned troubleshooting techniques and best practices

### 🔑 Key Security Concepts

| Concept | Description |
| --- | --- |
| Defense in Depth | Multiple layers of security, such as WAF, firewall, and network segmentation |
| Reverse Proxy | NGINX sits between clients and the application, inspecting traffic |
| WAF | Web Application Firewall analyzes HTTP/HTTPS requests for attack patterns |
| Zero Trust | Application accepts connections only from trusted WAF sources |
| OWASP CRS | Community-maintained rule set protecting against common web attacks |

### 📈 Next Steps

- Study OWASP Top 10 vulnerabilities in depth
- Practice with Juice Shop challenges
- Customize CRS rules for your specific environment
- Set up log aggregation with ELK Stack, Splunk, or Graylog
- Configure alerting for blocked attacks
- Implement SSL/TLS with Let's Encrypt
- Monitor performance and tune WAF behavior
- Explore advanced ModSecurity features
- Learn Docker security best practices
- Deploy this setup to cloud platforms such as AWS, Azure, or GCP

---

## 📚 References & Additional Resources

### 🔗 Official Documentation

| Resource | URL |
| --- | --- |
| OWASP WAF Control | https://wafcontrol.org/ |
| OWASP Core Rule Set | https://coreruleset.org/ |
| ModSecurity | https://github.com/SpiderLabs/ModSecurity |
| OWASP Juice Shop | https://owasp.org/www-project-juice-shop/ |
| NGINX Documentation | https://nginx.org/en/docs/ |
| Docker Documentation | https://docs.docker.com/ |

### 📖 Learning Resources

- **OWASP Top 10:** https://owasp.org/www-project-top-ten/
- **WebGoat:** https://owasp.org/www-project-webgoat/
- **ModSecurity Handbook:** https://www.feistyduck.com/books/modsecurity-handbook/
- **PortSwigger Academy:** https://portswigger.net/web-security

### 💬 Community & Support

- OWASP Slack: https://owasp.org/slack/
- CRS GitHub Discussions: https://github.com/coreruleset/coreruleset/discussions
- NGINX Community Forum: https://community.nginx.org/
- Stack Overflow ModSecurity tag: https://stackoverflow.com/questions/tagged/modsecurity

---

## 🛡️ Document Information

**Title:** OWASP WAF Control & Juice Shop Setup Guide

**Purpose:** Educational and training documentation

**Technologies:** OWASP WAF Control, ModSecurity, NGINX, Docker, OWASP Juice Shop, OWASP CRS

**Version:** 1.0

**Year:** 2025

> This guide is provided as-is for learning purposes. Always follow security best practices in production environments.
>
