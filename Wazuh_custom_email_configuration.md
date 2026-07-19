# 🛡️ Wazuh Custom Email Alert Configuration

Complete Guide with Custom Python Script and HTML Template

## 📋 Table of Contents

- Step 1: Prerequisites and Package Installation
- Step 2: Configure Postfix for Gmail Relay
- Step 3: Create HTML Email Template
- Step 4: Create Python Integration Script
- Step 5: Configure Wazuh Integration
- Step 6: Set Permissions and Restart Services
- Step 7: Testing the Configuration
- Troubleshooting Guide

## 1. Prerequisites and Package Installation📦

### Install Required Packages

We need Postfix for mail relay and Python for our custom script

### Verify Python Installation

Check if Python 3 is installed (usually pre-installed on modern Linux systems):

Verification Command

```bash
python3 --version
```

### Expected Output

You should see something like:

```
Python 3.8.10
```

or higher

## 2. Configure Postfix for Gmail Relay📧

### Set Up Gmail SMTP Relay

Configure Postfix to send emails through Gmail

### Step 2.1: Generate Gmail App Password

### Important: Use App Password, Not Regular Password

Regular Gmail passwords will not work. You must create an App Password.

1. Go to **Google Account Settings**: https://myaccount.google.com/
2. Navigate to **Security** → **2-Step Verification** (enable if not enabled)
3. Go to **App Passwords**: https://myaccount.google.com/apppasswords
4. Select app: **Mail**
5. Select device: **Other (Custom name)** → Enter "Wazuh Server"
6. Click **Generate**
7. Copy the 16-character password (you won't see it again)

### Step 2.2: Install Postfix and Required Packages

Terminal Command

```bash
# Update package repository and install Postfix with required packages
apt-get update && apt-get install postfix mailutils libsasl2-2 ca-certificates libsasl2-modules
```

### Installation Prompt

During Postfix installation, you'll be prompted to select a configuration type:

- Select: **"no configuration"**

### Step 2.3: Configure Postfix Main Configuration

Edit /etc/postfix/main.cf

```bash
sudo nano /etc/postfix/main.cf
```

Append these lines to the file. Create the file if missing.

Configuration Content

```
# Gmail SMTP Relay Configuration
relayhost = [smtp.gmail.com]:587
smtp_sasl_auth_enable = yes
smtp_sasl_password_maps = hash:/etc/postfix/sasl_passwd
smtp_sasl_security_options = noanonymous
smtp_tls_CAfile = /etc/ssl/certs/ca-certificates.crt
smtp_use_tls = yes
smtpd_relay_restrictions = permit_mynetworks, permit_sasl_authenticated, defer_unauth_destination
```

### Step 2.4: Create SASL Password File

Set the credentials of the sender in the /etc/postfix/sasl_passwd file and create a database file for Postfix. Replace the <USERNAME> and <PASSWORD> variables with sender’s email address username and password respectively.

Create Password File and Database

```bash
echo "[smtp.gmail.com]:587 <USERNAME>@gmail.com:<PASSWORD>" > /etc/postfix/sasl_passwd
postmap /etc/postfix/sasl_passwd
```

### Example

```bash
echo "[smtp.gmail.com]:587 alert.meghnacloud@gmail.com:your-16-char-app-password" > /etc/postfix/sasl_passwd
```

Note: Replace the placeholders with your actual credentials.

### Step 2.5: Hash and Secure the Password File

Secure Configuration

```bash
# Set secure permissions
sudo chmod 600 /etc/postfix/sasl_passwd
sudo chmod 600 /etc/postfix/sasl_passwd.db

# Restart Postfix
sudo systemctl restart postfix

# Enable Postfix on boot
sudo systemctl enable postfix
```

### Step 2.6: Test Postfix Configuration

Send Test Email

```bash
echo "This is a test email from Postfix" | mail -s "Postfix Test" user@mail.com
```

Check if email was sent by viewing mail logs:

Check Logs

```bash
sudo tail -50 /var/log/mail.log
```

### Success Indicators

Look for lines like:

```
status=sent (250 2.0.0 OK)
relay=smtp.gmail.com
```

## 3. Create HTML Email Template🎨

### Design Eye-Catching Email Template

Create a non-technical, visually appealing HTML template

### Create Template File

Create Template

```bash
sudo nano /var/ossec/integrations/wazuh_email_template.html
```

Copy and paste the following complete HTML template into the file:

HTML Template Content

```html
<!DOCTYPE html>
<html>
<head>
  <meta charset="UTF-8">
  <meta name="viewport" content="width=device-width, initial-scale=1.0">
</head>
<body style="margin: 0; padding: 0; background-color: #f4f4f4; font-family: 'Segoe UI', Tahoma, Geneva, Verdana, sans-serif;">
  <!-- Email Container -->
  <table width="100%" cellpadding="0" cellspacing="0" style="background-color: #f4f4f4; padding: 20px;">
    <tr>
      <td align="center">
        <table width="600" cellpadding="0" cellspacing="0" style="background-color: #ffffff; border-radius: 10px; overflow: hidden; box-shadow: 0 4px 6px rgba(0,0,0,0.1);">
          <!-- Header with Alert Icon -->
          <tr>
            <td style="background: linear-gradient(135deg, {alert_color} 0%, {alert_color_dark} 100%); padding: 40px 30px; text-align: center;">
              <div style="font-size: 60px; margin-bottom: 10px;">{alert_icon}</div>
              <h1 style="color: #ffffff; margin: 0; font-size: 32px; font-weight: 600;">SIEM - Security Alert Detected</h1>
              <div style="background-color: rgba(255,255,255,0.2); display: inline-block; padding: 8px 20px; border-radius: 20px; margin-top: 15px;">
                <span style="color: #ffffff; font-size: 18px; font-weight: 600;">{severity_text}</span>
              </div>
            </td>
          </tr>
          <!-- Quick Summary Box -->
          <tr>
            <td style="padding: 30px;">
              <div style="background-color: #fff3cd; border-left: 4px solid #ffc107; padding: 20px; border-radius: 5px; margin-bottom: 25px;">
                <h2 style="margin: 0 0 10px 0; color: #856404; font-size: 20px;">⚠️ What Happened?</h2>
                <p style="margin: 0; color: #856404; font-size: 16px; line-height: 1.6;">{description}</p>
              </div>
              <!-- Attack Type Card -->
              <table width="100%" cellpadding="0" cellspacing="0" style="margin-bottom: 20px;">
                <tr>
                  <td style="background: linear-gradient(135deg, #667eea 0%, #764ba2 100%); padding: 20px; border-radius: 8px; color: white;">
                    <table width="100%">
                      <tr>
                        <td width="60" valign="top">
                          <div style="font-size: 40px;">🎯</div>
                        </td>
                        <td>
                          <h3 style="margin: 0 0 5px 0; font-size: 16px; opacity: 0.9;">Attack Type</h3>
                          <p style="margin: 0; font-size: 20px; font-weight: 600;">{attack_type}</p>
                        </td>
                      </tr>
                    </table>
                  </td>
                </tr>
              </table>
              <!-- Attack Status -->
              <table width="100%" cellpadding="0" cellspacing="0" style="margin-bottom: 25px;">
                <tr>
                  <td style="background-color: {status_bg_color}; padding: 20px; border-radius: 8px; border-left: 5px solid {status_border_color};">
                    <table width="100%">
                      <tr>
                        <td width="60" valign="top">
                          <div style="font-size: 40px;">{status_icon}</div>
                        </td>
                        <td>
                          <h3 style="margin: 0 0 5px 0; font-size: 16px; color: {status_text_color};">Attack Status</h3>
                          <p style="margin: 0; font-size: 20px; font-weight: 600; color: {status_text_color};">{attack_status}</p>
                          <p style="margin: 5px 0 0 0; font-size: 14px; color: {status_text_color}; opacity: 0.8;">{status_message}</p>
                        </td>
                      </tr>
                    </table>
                  </td>
                </tr>
              </table>
              <!-- Key Information Grid -->
              <h3 style="color: #333; font-size: 18px; margin: 0 0 15px 0; border-bottom: 2px solid #e0e0e0; padding-bottom: 10px;">📋 Key Information</h3>
              <table width="100%" cellpadding="0" cellspacing="0" style="margin-bottom: 20px;">
                <tr>
                  <td width="50%" style="padding: 15px; background-color: #f8f9fa; border-radius: 5px; vertical-align: top;" valign="top">
                    <div style="color: #6c757d; font-size: 13px; margin-bottom: 5px;">🖥️ AFFECTED SYSTEM</div>
                    <div style="color: #212529; font-size: 16px; font-weight: 600;">{agent_name}</div>
                    <div style="color: #6c757d; font-size: 13px; margin-top: 3px;">{agent_ip}</div>
                  </td>
                  <td width="10"></td>
                  <td width="50%" style="padding: 15px; background-color: #f8f9fa; border-radius: 5px; vertical-align: top;" valign="top">
                    <div style="color: #6c757d; font-size: 13px; margin-bottom: 5px;">🕐 WHEN</div>
                    <div style="color: #212529; font-size: 16px; font-weight: 600;">{time_only}</div>
                    <div style="color: #6c757d; font-size: 13px; margin-top: 3px;">{date_only}</div>
                  </td>
                </tr>
              </table>
              <table width="100%" cellpadding="0" cellspacing="0" style="margin-bottom: 20px;">
                <tr>
                  <td width="50%" style="padding: 15px; background-color: #f8f9fa; border-radius: 5px; vertical-align: top;" valign="top">
                    <div style="color: #6c757d; font-size: 13px; margin-bottom: 5px;">⚠️ SEVERITY LEVEL</div>
                    <div style="color: #212529; font-size: 16px; font-weight: 600;">Level {rule_level} - {severity_text}</div>
                  </td>
                  <td width="10"></td>
                  <td width="50%" style="padding: 15px; background-color: #f8f9fa; border-radius: 5px; vertical-align: top;" valign="top">
                    <div style="color: #6c757d; font-size: 13px; margin-bottom: 5px;">🔢 RULE ID</div>
                    <div style="color: #212529; font-size: 16px; font-weight: 600;">{rule_id}</div>
                  </td>
                </tr>
              </table>
              <!-- Attack Details (if available) -->
              {attack_details_section}
              <!-- Action Required Box -->
              <div style="background: linear-gradient(135deg, #f093fb 0%, #f5576c 100%); padding: 25px; border-radius: 8px; margin-top: 25px;">
                <h3 style="color: #ffffff; margin: 0 0 10px 0; font-size: 18px;">💡 What Should You Do?</h3>
                <ul style="color: #ffffff; margin: 0; padding-left: 20px; line-height: 1.8;">
                  <li>Review the alert details above</li>
                  <li>Check if this activity was authorized</li>
                  <li>Contact your IT security team if suspicious</li>
                  <li>Monitor the affected system: <strong>{agent_name}</strong></li>
                </ul>
              </div>
              <!-- Technical Details (Collapsible) -->
              <details style="margin-top: 25px; padding: 20px; background-color: #f8f9fa; border-radius: 5px;">
                <summary style="cursor: pointer; font-weight: 600; color: #495057; font-size: 16px; margin-bottom: 15px;">🔧 Technical Details</summary>
                <div style="margin-top: 15px; padding: 15px; background-color: #ffffff; border-radius: 5px; border: 1px solid #dee2e6;">
                  <table width="100%" cellpadding="8" cellspacing="0" style="font-size: 13px;">
                    <tr>
                      <td style="color: #6c757d; width: 150px;"><strong>Agent ID:</strong></td>
                      <td style="color: #212529;">{agent_id}</td>
                    </tr>
                    <tr>
                      <td style="color: #6c757d;"><strong>Location:</strong></td>
                      <td style="color: #212529;">{location}</td>
                    </tr>
                    <tr>
                      <td style="color: #6c757d;"><strong>Full Timestamp:</strong></td>
                      <td style="color: #212529;">{timestamp}</td>
                    </tr>
                    {technical_details}
                  </table>
                </div>
              </details>
            </td>
          </tr>
          <!-- Footer -->
          <tr>
            <td style="background-color: #2c3e50; padding: 25px; text-align: center;">
              <p style="color: #ecf0f1; margin: 0; font-size: 14px;">🛡️ <strong>MeghnaCloud Security Platform</strong></p>
              <p style="color: #95a5a6; margin: 10px 0 0 0; font-size: 12px;">This is an automated security alert. Do not reply to this email.</p>
              <p style="color: #95a5a6; margin: 5px 0 0 0; font-size: 12px;">For support, contact us on support@meghnacloud.com</p>
            </td>
          </tr>
        </table>
      </td>
    </tr>
  </table>
</body>
</html>
```

### Template Features

The template includes:

- 🎯 Attack Type Detection
- ✅ Attack Status
- 📊 Severity Levels
- 🖥️ Agent Information
- 💡 Action Items
- 🔧 Technical Details

### Set Template Permissions

Set Permissions

```bash
sudo chmod 640 /var/ossec/integrations/wazuh_email_template.html
sudo chown root:wazuh /var/ossec/integrations/wazuh_email_template.html
```

## 4. Create Python Integration Script🐍

### Build Custom Alert Handler

Python script to process alerts and send formatted emails

### Create Python Script

Create Script

```bash
sudo nano /var/ossec/integrations/custom-email.py
```

### Complete Python Script

Copy and paste the following complete Python script into the file:

Python Script Content

```python
#!/usr/bin/env python3
import sys
import json
import smtplib
from email.mime.multipart import MIMEMultipart
from email.mime.text import MIMEText
from datetime import datetime
import logging

# Configuration
SMTP_SERVER = '127.0.0.1'
SMTP_PORT = 25
SENDER_EMAIL = 'user@gmail.com'
RECEIVER_EMAIL = 'user@gmail.com'

logging.basicConfig(
    filename='/var/ossec/logs/custom-email_integration.log',
    filemode='a',
    format='%(asctime)s,%(msecs)d %(name)s %(levelname)s %(message)s',
    datefmt='%Y-%m-%dT%H:%M:%S',
    level=logging.DEBUG
)

def determine_attack_type(rule_id, description, data):
    """Determine attack type from rule and data"""
    desc_lower = description.lower()
    if 'brute' in desc_lower or 'authentication' in desc_lower:
        return "Brute Force Attack"
    elif 'sql' in desc_lower or 'injection' in desc_lower:
        return "SQL Injection Attempt"
    elif 'malware' in desc_lower or 'virus' in desc_lower:
        return "Malware Detection"
    elif 'intrusion' in desc_lower or 'ids' in desc_lower or 'suricata' in desc_lower:
        return "Intrusion Detection"
    elif 'vulnerability' in desc_lower:
        return "Vulnerability Detected"
    elif 'file integrity' in desc_lower or 'syscheck' in desc_lower:
        return "File Integrity Alert"
    elif 'firewall' in desc_lower:
        return "Firewall Alert"
    elif 'web attack' in desc_lower or 'http' in desc_lower:
        return "Web Attack"
    elif 'unauthorized' in desc_lower:
        return "Unauthorized Access"
    else:
        return "Security Event"

def determine_attack_status(description, data):
    """Determine if attack was successful or blocked"""
    desc_lower = description.lower()

    # Check for success indicators
    if any(word in desc_lower for word in ['successful', 'succeeded', 'logged in', 'accessed']):
        return "SUCCESSFUL", "🚨", "#dc3545", "#f8d7da", "#721c24", "The attack appears to have succeeded. Immediate action required!"

    # Check for blocked indicators
    elif any(word in desc_lower for word in ['blocked', 'denied', 'prevented', 'failed', 'detected', 'alert']):
        return "BLOCKED", "✅", "#28a745", "#d4edda", "#155724", "The attack was detected and blocked by your security system."

    # Check action field in data
    if data.get('action') == 'allowed':
        return "ALLOWED", "⚠️", "#ffc107", "#fff3cd", "#856404", "This activity was allowed. Review if this was authorized."
    elif data.get('action') == 'blocked' or data.get('action') == 'denied':
        return "BLOCKED", "✅", "#28a745", "#d4edda", "#155724", "The attack was blocked successfully."

    # Default - Under Investigation
    return "UNDER INVESTIGATION", "🔍", "#17a2b8", "#d1ecf1", "#0c5460", "This event is being analyzed. Monitor for further activity."

try:
    # Read alert file
    logging.info("Reading alert file")
    alert_file = open(sys.argv[1])
    alert_json = json.loads(alert_file.read())
    alert_file.close()

    # Extract basic fields
    logging.info("Extracting fields")
    timestamp = alert_json.get('timestamp', 'N/A')
    location = alert_json.get('location', 'N/A')
    rule_id = alert_json.get('rule', {}).get('id', 'N/A')
    rule_level = alert_json.get('rule', {}).get('level', 0)
    description = alert_json.get('rule', {}).get('description', 'Security Alert')
    agent_id = alert_json.get('agent', {}).get('id', 'N/A')
    agent_name = alert_json.get('agent', {}).get('name', 'Unknown')
    agent_ip = alert_json.get('agent', {}).get('ip', 'N/A')
    data_fields = alert_json.get('data', {})

    # Parse timestamp for better display
    try:
        dt = datetime.fromisoformat(timestamp.replace('Z', '+00:00'))
        time_only = dt.strftime('%I:%M %p')
        date_only = dt.strftime('%B %d, %Y')
    except:
        time_only = timestamp
        date_only = ""

    # Determine severity and colors
    if rule_level >= 12:
        alert_color = '#dc3545'
        alert_color_dark = '#c82333'
        severity_text = 'CRITICAL'
        alert_icon = '🔴'
    elif rule_level >= 7:
        alert_color = '#fd7e14'
        alert_color_dark = '#e8590c'
        severity_text = 'HIGH'
        alert_icon = '🟠'
    elif rule_level >= 5:
        alert_color = '#ffc107'
        alert_color_dark = '#e0a800'
        severity_text = 'MEDIUM'
        alert_icon = '🟡'
    else:
        alert_color = '#28a745'
        alert_color_dark = '#218838'
        severity_text = 'LOW'
        alert_icon = '🟢'

    # Determine attack type and status
    attack_type = determine_attack_type(rule_id, description, data_fields)
    attack_status, status_icon, status_border_color, status_bg_color, status_text_color, status_message = determine_attack_status(description, data_fields)

    # Build attack details section
    attack_details_html = ""
    if data_fields:
        details_rows = ""
        if 'src_ip' in data_fields:
            details_rows += f"<tr><td style='padding: 10px; border-bottom: 1px solid #dee2e6; color: #6c757d;'><strong>Source IP:</strong></td><td style='padding: 10px; border-bottom: 1px solid #dee2e6; color: #212529;'>{data_fields['src_ip']}</td></tr>"
        if 'dest_ip' in data_fields:
            details_rows += f"<tr><td style='padding: 10px; border-bottom: 1px solid #dee2e6; color: #6c757d;'><strong>Destination IP:</strong></td><td style='padding: 10px; border-bottom: 1px solid #dee2e6; color: #212529;'>{data_fields['dest_ip']}</td></tr>"
        if 'proto' in data_fields:
            details_rows += f"<tr><td style='padding: 10px; border-bottom: 1px solid #dee2e6; color: #6c757d;'><strong>Protocol:</strong></td><td style='padding: 10px; border-bottom: 1px solid #dee2e6; color: #212529;'>{data_fields['proto']}</td></tr>"

        if 'alert' in data_fields:
            alert_info = data_fields['alert']
            if 'signature' in alert_info:
                details_rows += f"<tr><td style='padding: 10px; border-bottom: 1px solid #dee2e6; color: #6c757d;'><strong>Signature:</strong></td><td style='padding: 10px; border-bottom: 1px solid #dee2e6; color: #212529;'>{alert_info['signature']}</td></tr>"
            if 'category' in alert_info:
                details_rows += f"<tr><td style='padding: 10px; border-bottom: 1px solid #dee2e6; color: #6c757d;'><strong>Category:</strong></td><td style='padding: 10px; border-bottom: 1px solid #dee2e6; color: #212529;'>{alert_info['category']}</td></tr>"

        if details_rows:
            attack_details_html = f"""
            <h3 style="color: #333; font-size: 18px; margin: 25px 0 15px 0; border-bottom: 2px solid #e0e0e0; padding-bottom: 10px;">🎯 Attack Details</h3>
            <table width="100%" cellpadding="0" cellspacing="0" style="background-color: #ffffff; border: 1px solid #dee2e6; border-radius: 5px;">
              {details_rows}
            </table>
            """

    # Build technical details
    technical_details = ""
    if data_fields:
        for key, value in data_fields.items():
            if isinstance(value, dict):
                continue
            technical_details += f"<tr><td style='color: #6c757d;'><strong>{key}:</strong></td><td style='color: #212529;'>{value}</td></tr>"

    # Load HTML template
    with open('/var/ossec/integrations/wazuh_email_template.html', 'r') as f:
        html_template = f.read()

    # Replace all placeholders
    html_body = html_template.format(
        alert_color=alert_color,
        alert_color_dark=alert_color_dark,
        alert_icon=alert_icon,
        severity_text=severity_text,
        description=description,
        attack_type=attack_type,
        status_bg_color=status_bg_color,
        status_border_color=status_border_color,
        status_icon=status_icon,
        status_text_color=status_text_color,
        attack_status=attack_status,
        status_message=status_message,
        agent_name=agent_name,
        agent_ip=agent_ip,
        time_only=time_only,
        date_only=date_only,
        rule_level=rule_level,
        rule_id=rule_id,
        attack_details_section=attack_details_html,
        agent_id=agent_id,
        location=location,
        timestamp=timestamp,
        technical_details=technical_details
    )

    # Create and send email
    logging.info("Creating email message")
    message = MIMEMultipart('alternative')
    message['From'] = SENDER_EMAIL
    message['To'] = RECEIVER_EMAIL
    message['Subject'] = f'{alert_icon} [{severity_text}] Security Alert - {attack_type} on {agent_name}'

    message.attach(MIMEText(html_body, 'html'))

    with smtplib.SMTP(SMTP_SERVER, SMTP_PORT) as server:
        server.send_message(message)

    logging.info("Email sent successfully!")

except Exception as e:
    logging.error(f"Failed to process alert: {str(e)}")
    import traceback
    logging.error(traceback.format_exc())
    sys.exit(1)

sys.exit(0)
```

### Script Configuration Variables

At the top of the script, configure these variables:

| Variable | Description | Example |
| --- | --- | --- |
| `SMTP_SERVER` | SMTP server address | `'127.0.0.1'` |
| `SMTP_PORT` | SMTP port | `25` |
| `SENDER_EMAIL` | From email address | `'alert.meghnacloud@gmail.com'` |
| `RECEIVER_EMAIL` | To email address(es) | `'support@meghnacloud.com, admin@meghnacloud.com'` |
| `SERVER_NAME` | Wazuh server identifier | `'SIEM'` |

Configuration Example

```python
# SMTP Configuration
SMTP_SERVER = '127.0.0.1'
SMTP_PORT = 25
SENDER_EMAIL = 'user@gmail.com'

# Multiple recipients separated by comma
RECEIVER_EMAIL = 'user@gmail.com, admin@gmail.com'

# Server identifier for email subject
SERVER_NAME = 'Name SIEM'
```

### Multiple Recipients

To send alerts to multiple email addresses, separate them with commas:

```python
RECEIVER_EMAIL = 'email1@domain.com, email2@domain.com, email3@domain.com'
```

### Key Script Features

- **Attack Type Detection:** Automatically identifies attack types from alert descriptions
- **Status Determination:** Detects if attack was successful, blocked, or needs investigation
- **Severity Color Coding:** Visual indicators based on alert level
- **Dynamic HTML Generation:** Populates template with alert data
- **Comprehensive Logging:** All activities logged to `/var/ossec/logs/custom-email_integration.log`

### Set Script Permissions

Make Executable

```bash
sudo chmod 750 /var/ossec/integrations/custom-email.py
sudo chown root:wazuh /var/ossec/integrations/custom-email.py
```

### Verify Permissions

Check file permissions:

```bash
ls -l /var/ossec/integrations/custom-email.py
```

Should show:

```
-rwxr-x--- 1 root wazuh
```

## 5. Configure Wazuh Integration⚙️

### Enable Custom Integration

Configure Wazuh to trigger our custom script

### Step 5.1: Disable Built-in Email (Recommended)

Edit Wazuh Configuration

```bash
sudo nano /var/ossec/etc/ossec.conf
```

Find the `<global>` section and modify:

Global Configuration

```xml
<global>
  <jsonout_output>yes</jsonout_output>
  <alerts_log>yes</alerts_log>
  <logall>no</logall>
  <logall_json>no</logall_json>
  <email_notification>no</email_notification> <!-- Disabled -->
  <agents_disconnection_time>10m</agents_disconnection_time>
  <agents_disconnection_alert_time>0</agents_disconnection_alert_time>
  <update_check>yes</update_check>
</global>
```

### Step 5.2: Configure Alert Levels

Alert Configuration

```xml
<alerts>
  <log_alert_level>3</log_alert_level>
</alerts>
```

### Step 5.3: Add Custom Integration

Add this section anywhere in `ossec.conf` (typically after `<alerts>`):

Integration Configuration

```xml
<integration>
  <name>custom-email.py</name>
  <level>7</level> <!-- Only alerts level 7 and above -->
  <alert_format>json</alert_format>
</integration>
```

### Alert Level Configuration

Adjust the `<level>` based on your needs:

- `<level>3</level>` - All alerts (for testing)
- `<level>5</level>` - Medium and above
- `<level>7</level>` - High and above (recommended)
- `<level>12</level>` - Only critical alerts

### Step 5.4: Optional - Filter by Specific Rules or Groups

You can also filter alerts by rule ID or group:

Filter by Rule ID

```xml
<integration>
  <name>custom-email.py</name>
  <rule_id>100005,100006,100007</rule_id> <!-- Specific rules -->
  <alert_format>json</alert_format>
</integration>
```

Filter by Group

```xml
<integration>
  <name>custom-email.py</name>
  <group>ids,vulnerability-detector,malware</group> <!-- Specific groups -->
  <level>5</level>
  <alert_format>json</alert_format>
</integration>
```

### Step 5.5: Validate Configuration

Test Configuration

```bash
sudo /var/ossec/bin/wazuh-control configtest
```

### Expected Output

```
wazuh-control: Configuration is OK
```

### Configuration Error?

If you see errors, check:

- XML tags are properly closed
- No typos in file paths or script names
- Proper indentation (use spaces, not tabs)

## 6. Set Permissions and Restart Services🔐

### Apply Configuration

Finalize setup and restart Wazuh services

### Verify File Structure

📁 /var/ossec/

📁 integrations/

📄 custom-email.py (rwxr-x--- root:wazuh)

📄 wazuh_email_template.html (rw-r----- root:wazuh)

📁 etc/

📄 ossec.conf

📁 logs/

📄 custom-email_integration.log (created automatically)

### Final Permission Check

Verify All Permissions

```bash
# Check Python script
ls -l /var/ossec/integrations/custom-email.py

# Check HTML template
ls -l /var/ossec/integrations/wazuh_email_template.html

# Check configuration
ls -l /var/ossec/etc/ossec.conf
```

### Restart Wazuh Manager

Restart Services

```bash
# Stop Wazuh
sudo systemctl stop wazuh-manager

# Verify it stopped
sudo systemctl status wazuh-manager

# Start Wazuh
sudo systemctl start wazuh-manager

# Verify it's running
sudo systemctl status wazuh-manager

# Check if integration is loaded
ps aux | grep integratord
```

### Service Should Be Active

You should see:

```
Active: active (running)
```

### Monitor Logs in Real-Time

Watch Logs

```bash
# Monitor custom integration log
sudo tail -f /var/ossec/logs/custom-email_integration.log

# Monitor Wazuh main log
sudo tail -f /var/ossec/logs/ossec.log

# Monitor mail log
sudo tail -f /var/log/mail.log
```

## 7. Testing the Configuration🧪

### Test Email Alerts

Verify everything works correctly

### Method 1: Test with Real Alert JSON

Create a test alert file:

Create Test Alert

```bash
sudo cat <<EOF > /tmp/test_alert.json
{
  "timestamp": "2025-11-24T10:30:00+0000",
  "rule": {
    "level": 10,
    "description": "Test Security Alert - Intrusion Detection",
    "id": "100001"
  },
  "agent": {
    "id": "001",
    "name": "WebServer-01",
    "ip": "192.168.1.100"
  },
  "location": "/var/log/test.log",
  "data": {
    "src_ip": "10.0.0.1",
    "dest_ip": "192.168.1.100",
    "proto": "TCP"
  }
}
EOF
```

Run Test

```bash
sudo python3 /var/ossec/integrations/custom-email.py /tmp/test_alert.json
```

### Expected Behavior

- No errors in terminal output
- Email received in inbox within 1-2 minutes
- Log entry in `/var/ossec/logs/custom-email_integration.log`

### Method 2: Test with Wazuh Log Test Tool

Generate Test Alert

```bash
# Start log test tool
sudo /var/ossec/bin/wazuh-logtest

# Paste a sample log entry (e.g., failed SSH login)
Nov 24 10:00:00 server sshd[1234]: Failed password for root from 1.1.1.1 port 22 ssh2

# Press Ctrl+D to exit
```

### Method 3: Trigger Real Alert

Generate a real security event on a monitored agent:

Example: Trigger Failed SSH Attempt

```bash
# From another machine, try to SSH with wrong password
ssh wronguser@your-wazuh-agent-ip

# This should trigger an alert if SSH monitoring is enabled
```

### Verify Email Delivery

Check your email inbox for:

| Element | What to Check |
| --- | --- |
| **Subject Line** | Should show: `SIEM [SEVERITY] Security Alert - Attack Type on Hostname` |
| **Header** | Color-coded based on severity with emoji icon |
| **Attack Type** | Should show plain English description |
| **Attack Status** | Blocked/Successful/Under Investigation with appropriate icon |
| **Agent Info** | Correct hostname and IP address |
| **Formatting** | Professional layout with colors and sections |

### Check Logs for Success

Review Integration Log

```bash
sudo tail -20 /var/ossec/logs/custom-email_integration.log
```

### Success Log Example

You should see entries like:

```
2025-11-24T10:30:15 INFO Reading alert file
```

```
2025-11-24T10:30:15 INFO Extracting fields
```

```
2025-11-24T10:30:15 INFO Creating email message
```

```
2025-11-24T10:30:16 INFO Email sent successfully!
```

## ❓Troubleshooting Guide🔧

### Common Issues and Solutions

Fix problems quickly with these solutions

### Issue 1: Script Runs Manually But Not Automatically

### Symptoms

- Test with JSON file works fine
- Real alerts don't trigger emails
- No errors in logs

**Solutions:**

1. **Check alert level threshold:** Your alerts might be below the configured level

```bash
# Check recent alert levels
sudo grep '"level":' /var/ossec/logs/alerts/alerts.json | tail -20

# If alerts are level 5 but config is level 7, lower it
sudo nano /var/ossec/etc/ossec.conf

# Change:
#   <level>7</level>
# to:
#   <level>3</level>
```

1. **Verify integration is loaded:**

```bash
sudo grep -i integration /var/ossec/logs/ossec.log | tail -10
```

1. **Check script permissions:**

```bash
ls -l /var/ossec/integrations/custom-email.py

# Should be: -rwxr-x--- root wazuh
# If not, fix it:
sudo chmod 750 /var/ossec/integrations/custom-email.py
sudo chown root:wazuh /var/ossec/integrations/custom-email.py
```

1. **Restart Wazuh completely:**

```bash
sudo systemctl stop wazuh-manager
sleep 5
sudo systemctl start wazuh-manager
```

### Issue 2: Postfix Authentication Failed

### Error in mail.log

```
SASL authentication failed
Authentication failed
```

**Solutions:**

1. **Verify Gmail App Password:** Ensure you're using an App Password, not your regular Gmail password
2. **Check sasl_passwd file:**

```bash
sudo cat /etc/postfix/sasl_passwd

# Should show:
# [smtp.gmail.com]:587 your-email@gmail.com:16-char-password

# Recreate hash if needed
sudo postmap /etc/postfix/sasl_passwd
sudo systemctl restart postfix
```

1. **Enable 2-Step Verification:** Gmail requires 2FA to generate App Passwords

### Issue 3: Python Script Errors

### Error Messages

Syntax errors, import errors, or template not found

**Solutions:**

1. **Check Python version:**

```bash
python3 --version

# Should be 3.6 or higher
```

1. **Verify template exists:**

```bash
ls -l /var/ossec/integrations/wazuh_email_template.html
```

1. **Check script syntax:**

```bash
python3 -m py_compile /var/ossec/integrations/custom-email.py
```

1. **Review error logs:**

```bash
sudo tail -50 /var/ossec/logs/custom-email_integration.log
```

### Issue 4: Emails Not Received

### Symptoms

Script runs successfully but no email arrives

**Solutions:**

1. **Check spam folder:** First check your spam/junk folder
2. **Verify Postfix is running:**

```bash
sudo systemctl status postfix
```

1. **Check mail queue:**

```bash
mailq

# If emails are stuck, flush queue
sudo postfix flush
```

1. **Review mail logs:**

```bash
sudo tail -100 /var/log/mail.log | grep -i error
```

1. **Test Postfix directly:**

```bash
echo "Direct test" | mail -s "Test" your-email@example.com
```

### Issue 5: Multiple Duplicate Emails

### Symptoms

Receiving same alert multiple times

**Solutions:**

1. **Check if built-in email is still enabled:**

```bash
sudo grep "email_notification" /var/ossec/etc/ossec.conf

# Should be:
# <email_notification>no</email_notification>
```

1. **Verify only one integration configured:**

```bash
sudo grep -A 4 "<integration>" /var/ossec/etc/ossec.conf
```

### Quick Diagnostic Commands

Run All Diagnostics

```bash
# 1. Check Wazuh status
sudo systemctl status wazuh-manager

# 2. Check Postfix status
sudo systemctl status postfix

# 3. Verify files exist
ls -l /var/ossec/integrations/custom-email.py
ls -l /var/ossec/integrations/wazuh_email_template.html

# 4. Check recent alerts
sudo tail -20 /var/ossec/logs/alerts/alerts.json

# 5. Check integration log
sudo tail -50 /var/ossec/logs/custom-email_integration.log

# 6. Check mail log
sudo tail -50 /var/log/mail.log

# 7. Test configuration
sudo /var/ossec/bin/wazuh-control configtest
```

## 📊 Configuration Summary

| Component | File/Location | Status |
| --- | --- | --- |
| Postfix SMTP | /etc/postfix/main.cf | ✅ Configured for Gmail relay |
| Gmail Credentials | /etc/postfix/sasl_passwd | ✅ App Password configured |
| HTML Template | /var/ossec/integrations/wazuh_email_template.html | ✅ Created and permissions set |
| Python Script | /var/ossec/integrations/custom-email.py | ✅ Created and configured |
| Wazuh Integration | /var/ossec/etc/ossec.conf | ✅ Custom integration enabled |
| Services | systemctl | ✅ Postfix and Wazuh restarted |

© Rabiul Hasan
