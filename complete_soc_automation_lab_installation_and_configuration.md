## Wazuh + VPN Sensor + Zeek + Suricata + n8n + DFIR-IRIS + Cortex + MISP + OpenCTI + AI Triage

**Document version:** 2.0

**Prepared for:** SOC automation / lab deployment

**Scope:** Full installation and configuration documentation for the lab architecture discussed earlier.

> **Important:** Replace every placeholder such as `<WAZUH_SERVER_IP>`, `<VPN_SERVER_IP>`, `<OPENCTI_TOKEN>`, `<IRIS_API_KEY>`, `<MISP_API_KEY>`, and `<DOMAIN>` with values from your own environment. Do not paste production secrets into documentation or Git repositories.
> 

---

# 1. Architecture Overview

```
                          ┌──────────────────────────┐
                          │          MISP VM          │
                          │  Threat Intel Platform    │
                          │  MISP API Key             │
                          └────────────▲─────────────┘
                                       │
                                       │ Cortex MISP Analyzer
                                       │
┌──────────────────────────────────────┴──────────────────────────────────────┐
│                          Wazuh / SOAR Server VM                              │
│                                                                              │
│  ┌──────────────┐   ┌──────────────┐   ┌──────────────────┐                 │
│  │ Wazuh Server │   │ Wazuh Indexer│   │ Wazuh Dashboard  │                 │
│  └──────▲───────┘   └──────────────┘   └──────────────────┘                 │
│         │                                                                    │
│         │ Wazuh custom integration                                           │
│         │ POST selected alerts                                               │
│         ▼                                                                    │
│  ┌──────────────────────────────────────────────────────────┐                │
│  │                         n8n                              │                │
│  │  Webhook → Normalize → Extract IOCs → OpenCTI → AI → IRIS│                │
│  └──────────▲───────────────────────────────┬───────────────┘                │
│             │                               │                                │
│             │                               ▼                                │
│             │                       ┌──────────────┐                         │
│             │                       │   OpenCTI    │                         │
│             │                       │ IOC Enrich   │                         │
│             │                       └──────────────┘                         │
│             │                                                                │
│             ▼                                                                │
│       ┌──────────────┐           ┌──────────────┐                            │
│       │ DFIR-IRIS    │──────────▶│   Cortex     │                            │
│       │ Alerts/Cases │           │ Analyzers    │                            │
│       └──────────────┘           └──────────────┘                            │
└──────────────────────────────────────────────────────────────────────────────┘
                                       ▲
                                       │ Wazuh Agent
                                       │
                          ┌────────────┴────────────┐
                          │        VPN Server        │
                          │  Wazuh Agent             │
                          │  Zeek                    │
                          │  Suricata                │
                          │  /var/log/auth.log       │
                          └─────────────────────────┘
```

---

# 2. Lab Hosts and Services

## 2.1 Wazuh / SOAR Server VM

Installed services:

```
Wazuh Server / Manager
Wazuh Indexer
Wazuh Dashboard
n8n
DFIR-IRIS
Cortex
OpenCTI
Docker / Docker Compose
```

## 2.2 VPN Server

Installed services:

```
Wazuh Agent
Zeek
Suricata
OpenSSH server logs monitored through auth.log
```

## 2.3 MISP VM

Installed services:

```
MISP
MISP API
MISP threat intelligence data
```

---

# 3. Base System Preparation

Run this on all Ubuntu-based VMs unless already completed.

```bash
sudo apt update
sudo apt upgrade -y
sudo apt install -y curl wget git vim nano jq net-tools ca-certificates gnupg lsb-release software-properties-common unzip tar
```

Set timezone if required:

```bash
sudo timedatectl set-timezone Asia/Dhaka
```

Check IP addresses:

```bash
ip -br a
hostname -I
```

---

# 4. Docker and Docker Compose Installation

Several services in this lab are easiest to run with Docker Compose: n8n, DFIR-IRIS, OpenCTI, and optionally Cortex.

```bash
sudo apt update
sudo apt install -y ca-certificates curl gnupg
sudo install -m 0755 -d /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
sudo chmod a+r /etc/apt/keyrings/docker.gpg

. /etc/os-release

echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu\
${VERSION_CODENAME} stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

sudo apt update
sudo apt install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
```

Enable Docker:

```bash
sudo systemctl enable docker
sudo systemctl start docker
```

Allow the current user to run Docker without sudo:

```bash
sudo usermod -aG docker $USER
newgrp docker
```

Verify:

```bash
docker --version
docker compose version
```

---

# 5. Wazuh Server Installation

> Use the official Wazuh documentation as the authoritative installation reference. The commands below show the common all-in-one quickstart pattern for a lab. For production, follow the official step-by-step guide.
> 

## 5.1 All-in-One Quickstart

On the Wazuh / SOAR server VM:

```bash
curl -sO https://packages.wazuh.com/4.x/wazuh-install.sh
sudo bash ./wazuh-install.sh -a
```

After installation, Wazuh prints dashboard credentials. Save them securely.

Access Wazuh Dashboard:

```
https://<WAZUH_SERVER_IP>
```

## 5.2 Useful Wazuh Commands

```bash
sudo systemctl status wazuh-manager
sudo systemctl status wazuh-indexer
sudo systemctl status wazuh-dashboard
sudo systemctl status filebeat
```

Restart Wazuh Manager:

```bash
sudo systemctl restart wazuh-manager
```

Test Wazuh rule/decoder configuration:

```bash
sudo /var/ossec/bin/wazuh-analysisd -t
```

Test logs manually:

```bash
sudo /var/ossec/bin/wazuh-logtest
```

---

# 6. Wazuh Agent Installation on VPN Server

Install the Wazuh Agent on the VPN server using the deployment command generated from the Wazuh Dashboard:

```
Wazuh Dashboard → Agents management → Deploy new agent
```

Example Linux agent installation pattern:

```bash
wget https://packages.wazuh.com/4.x/apt/pool/main/w/wazuh-agent/wazuh-agent_<VERSION>_amd64.deb
sudo WAZUH_MANAGER='<WAZUH_SERVER_IP>' WAZUH_AGENT_NAME='Internal_vpn' dpkg -i ./wazuh-agent_<VERSION>_amd64.deb
sudo systemctl daemon-reload
sudo systemctl enable wazuh-agent
sudo systemctl start wazuh-agent
```

Check agent status:

```bash
sudo systemctl status wazuh-agent
sudo tail -f /var/ossec/logs/ossec.log
```

---

# 7. Zeek Installation and Configuration on VPN Server

## 7.1 Install Zeek on Ubuntu

Example for Ubuntu 24.04 using Zeek packages:

```bash
echo 'deb https://download.opensuse.org/repositories/security:/zeek/xUbuntu_24.04/ /' | sudo tee /etc/apt/sources.list.d/security:zeek.list
curl -fsSL https://download.opensuse.org/repositories/security:zeek/xUbuntu_24.04/Release.key | gpg --dearmor | sudo tee /etc/apt/trusted.gpg.d/security_zeek.gpg > /dev/null
sudo apt update
sudo apt install -y zeek-8.0
```

For Ubuntu 22.04, use the matching Zeek repository path for `xUbuntu_22.04`.

Add Zeek to PATH:

```bash
echo 'export PATH=$PATH:/opt/zeek/bin' >> ~/.bashrc
source ~/.bashrc
```

Check Zeek:

```bash
zeek --version
zeekctl --version
```

## 7.2 Configure Zeek Interface

Find the VPN server network interface:

```bash
ip -br a
```

Edit Zeek node configuration:

```bash
sudo nano /opt/zeek/etc/node.cfg
```

Example standalone configuration:

```
[zeek]
type=standalone
host=localhost
interface=<VPN_INTERFACE>
```

## 7.3 Enable Zeek JSON Logs

Edit Zeek local policy:

```bash
sudo nano /opt/zeek/share/zeek/site/local.zeek
```

Add:

```
@load policy/tuning/json-logs.zeek
```

If required by Zeek version, also use:

```
redef LogAscii::use_json = T;
```

Deploy Zeek:

```bash
sudo /opt/zeek/bin/zeekctl check
sudo /opt/zeek/bin/zeekctl deploy
sudo /opt/zeek/bin/zeekctl status
```

Check logs:

```bash
ls -lah /opt/zeek/logs/current/
tail -f /opt/zeek/logs/current/conn.log
```

## 7.4 Zeek Logs Selected for Wazuh

Forward these logs first:

```
conn.log
ssl.log
weird.log
```

Do not forward all operational Zeek logs unless troubleshooting.

---

# 8. Suricata Installation and Configuration on VPN Server

## 8.1 Install Suricata

```bash
sudo apt-get install -y software-properties-common
sudo add-apt-repository -y ppa:oisf/suricata-stable
sudo apt-get update
sudo apt-get install -y suricata jq
```

Check Suricata:

```bash
suricata -V
sudo systemctl status suricata
```

## 8.2 Configure Suricata Interface

Find interface:

```bash
ip -br a
```

Edit Suricata configuration:

```bash
sudo nano /etc/suricata/suricata.yaml
```

Set `HOME_NET` and interface under `af-packet`:

```yaml
vars:
address-groups:
HOME_NET:"[192.168.18.0/24]"
EXTERNAL_NET:"!$HOME_NET"

af-packet:
-interface: <VPN_INTERFACE>
cluster-id:99
cluster-type: cluster_flow
defrag:yes
```

Make sure EVE JSON is enabled:

```yaml
outputs:
-eve-log:
enabled:yes
filetype: regular
filename: eve.json
```

## 8.3 Update Suricata Rules

```bash
sudo suricata-update
sudo systemctl restart suricata
sudo systemctl status suricata
```

Check Suricata log:

```bash
sudo tail -f /var/log/suricata/eve.json | jq
```

---

# 9. Wazuh Agent Log Collection on VPN Server

Edit Wazuh Agent config:

```bash
sudo nano /var/ossec/etc/ossec.conf
```

Add Suricata:

```xml
<localfile>
  <location>/var/log/suricata/eve.json</location>
  <log_format>json</log_format>
</localfile>
```

Add Zeek logs:

```xml
<localfile>
  <location>/opt/zeek/logs/current/conn.log</location>
  <log_format>json</log_format>
</localfile>

<localfile>
  <location>/opt/zeek/logs/current/ssl.log</location>
  <log_format>json</log_format>
</localfile>

<localfile>
  <location>/opt/zeek/logs/current/weird.log</location>
  <log_format>json</log_format>
</localfile>
```

Add auth log if not already monitored:

```xml
<localfile>
  <location>/var/log/auth.log</location>
  <log_format>syslog</log_format>
</localfile>
```

Restart agent:

```bash
sudo systemctl restart wazuh-agent
sudo tail -f /var/ossec/logs/ossec.log
```

---

# 10. Wazuh Custom Rules and Decoders

## 10.1 SSH Pre-Auth Decoder

Create or edit:

```bash
sudo nano /var/ossec/etc/decoders/local_decoder.xml
```

Add:

```xml
<decoder name="sshd-closed-authenticating-user">
  <parent>sshd</parent>
  <prematch>Connection closed by authenticating user</prematch>
  <regex type="pcre2">^Connection closed by authenticating user (\S+) (\d{1,3}(?:\.\d{1,3}){3}) port (\d+) \[preauth\]</regex>
  <order>user,srcip,srcport</order>
</decoder>
```

If PCRE2 fails, use the simpler decoder:

```xml
<decoder name="sshd-closed-authenticating-user">
  <parent>sshd</parent>
  <prematch>Connection closed by authenticating user</prematch>
  <regex>^Connection closed by authenticating user ([^ ]+) ([0-9]+.[0-9]+.[0-9]+.[0-9]+) port ([0-9]+) \[preauth\]</regex>
  <order>user,srcip,srcport</order>
</decoder>
```

## 10.2 SSH Pre-Auth Rules

Create:

```bash
sudo nano /var/ossec/etc/rules/100710-ssh-preauth_rules.xml
```

Add:

```xml
<group name="syslog,sshd,authentication_failed,ssh_pre_auth,">

  <rule id="100710" level="5">
    <if_sid>5722</if_sid>
    <match>Connection closed by authenticating user root</match>
    <description>sshd: Pre-authentication SSH connection closed for root user</description>
    <group>sshd,ssh_pre_auth,root_login_attempt,</group>
  </rule>

  <rule id="100711" level="10" frequency="5" timeframe="180">
    <if_matched_sid>100710</if_matched_sid>
    <same_srcip />
    <description>sshd: Multiple pre-authentication root SSH attempts from same source IP</description>
    <mitre>
      <id>T1110</id>
    </mitre>
    <group>sshd,authentication_failed,brute_force,root_login_attempt,</group>
  </rule>

</group>
```

Validate and restart:

```bash
sudo /var/ossec/bin/wazuh-analysisd -t
sudo systemctl restart wazuh-manager
```

Test:

```bash
sudo /var/ossec/bin/wazuh-logtest
```

## 10.3 Zeek Rule File

Create:

```bash
sudo nano /var/ossec/etc/rules/zeek_rules.xml
```

Recommended starter rules:

```xml
<group name="zeek,">
  <rule id="100900" level="0">
    <decoded_as>json</decoded_as>
    <description>Zeek alerts</description>
  </rule>

  <rule id="100901" level="5">
    <if_sid>100900</if_sid>
    <field name="dnsquery">\w+.</field>
    <dstport type="pcre2">^53$</dstport>
    <field name="resolved_by">\d+.\d+.\d+.\d+</field>
    <description>Zeek: DNS Query $(dnsquery) attempted from source ip $(srcip) source port $(srcport) resolved to IP(s) $(resolved_by)</description>
  </rule>

  <rule id="100902" level="0">
    <if_sid>100900</if_sid>
    <srcport>5353</srcport>
    <description>Zeek: Normal mDNS traffic - $(dnsquery) from source ip $(srcip). Generally benign. Investigate only if the source is unexpected. </description>
  </rule>

  <rule id="100903" level="7">
    <if_sid>100900</if_sid>
    <field name="connection_state">REJ</field>
    <description>Zeek: Connection from $(srcip) to $(dstip) $(dstport) rejected</description>
  </rule>

  <rule id="100904" level="10" frequency="5" timeframe="20">
    <if_matched_sid>100903</if_matched_sid>
    <description>Zeek: Multiple rejected connections from $(srcip) (5+ in 20s - possible port scan activity)</description>
  </rule>

<!-- Suppresses alerts for DNS queries to the Wazuh CTI domain. -->
  <rule id="100905" level="0">
    <if_sid>100900</if_sid>
    <field name="dnsquery">cti.wazuh.com</field>
    <description>Zeek: DNS Query $(dnsquery) exclude</description>
  </rule>

  <rule id="100906" level="8">
    <if_sid>100900</if_sid>
    <field name="ssl_validation_status">self signed certificate</field>
    <description>Zeek: Client $(srcip) connected to a server with self-signed certificate $(dstip)</description>
  </rule>

  <rule id="100907" level="12">
    <if_sid>100900</if_sid>
    <field name="ssl_validation_status">certificate has expired</field>
    <description>Zeek: Client $(srcip) connected to a server with expired certificate $(dstip)</description>
  </rule>
</group>
```

Validate:

```bash
sudo /var/ossec/bin/wazuh-analysisd -t
sudo systemctl restart wazuh-manager
```

---

## **Zeek decoder File**

```jsx
zeek_decoders.xml
```

```xml
<!-- DNS Query -->
<decoder name="zeek_decoder">
  <parent>json</parent>
  <regex>"ts":\d+</regex>
  <order>timestamp</order>
</decoder>

<decoder name="zeek_decoder">
  <parent>json</parent>
  <regex>"uid":"(\w+)"</regex>
  <order>uid</order>
</decoder>

<decoder name="zeek_decoder">
  <parent>json</parent>
  <regex>"id.orig_h":"(\d+.\d+.\d+.\d+)"</regex>
  <order>srcip</order>
</decoder>

<decoder name="zeek_decoder">
  <parent>json</parent>
  <regex>"id.orig_p":(\d+)</regex>
  <order>srcport</order>
</decoder>

<decoder name="zeek_decoder">
  <parent>json</parent>
  <regex>"id.resp_h":"(\d+.\d+.\d+.\d+)"</regex>
  <order>dstip</order>
</decoder>

<decoder name="zeek_decoder">
  <parent>json</parent>
  <regex>"id.resp_p":(\d+)</regex>
  <order>dstport</order>
</decoder>

<decoder name="zeek_decoder">
  <parent>json</parent>
  <regex>"proto":"(\w+)"</regex>
  <order>protocol</order>
</decoder>

<decoder name="zeek_decoder">
  <parent>json</parent>
  <regex>"trans_id":(\d+)</regex>
  <order>DNS_transaction_id</order>
</decoder>

<decoder name="zeek_decoder">
  <parent>json</parent>
  <regex>"query":"(\.+)"</regex>
  <order>dnsquery</order>
</decoder>

<decoder name="zeek_decoder">
  <parent>json</parent>
  <regex>"rcode_name":"(\w+)"</regex>
  <order>dns_response_code</order>
</decoder>

<decoder name="zeek_decoder">
  <parent>json</parent>
  <regex>"AA":(\w+)</regex>
  <order>authoritative_answer</order>
</decoder>

<decoder name="zeek_decoder">
  <parent>json</parent>
  <regex>"TC":(\w+)</regex>
  <order>truncate_flag</order>
</decoder>

<decoder name="zeek_decoder">
  <parent>json</parent>
  <regex>"RD":(\w+)</regex>
  <order>recursion_desired_flag</order>
</decoder>

<decoder name="zeek_decoder">
  <parent>json</parent>
  <regex>"RA":(\w+)</regex>
  <order>recursion_avalable_flag</order>
</decoder>

<decoder name="zeek_decoder">
  <parent>json</parent>
  <regex>"Z":(\d+)</regex>
  <order>reserved_for_future_use</order>
</decoder>

<decoder name="zeek_decoder">
  <parent>json</parent>
  <regex>"answers":(["\.+"])</regex>
  <order>resolved_by</order>
</decoder>

<decoder name="zeek_decoder">
  <parent>json</parent>
  <regex>"TTLs":([\d+])</regex>
  <order>time_to_live</order>
</decoder>

<decoder name="zeek_decoder">
  <parent>json</parent>
  <regex>"rejected":(\w+)</regex>
  <order>query_rejected</order>
</decoder>

<!-- Additional DNS metadata -->
<decoder name="zeek_decoder">
  <parent>json</parent>
  <regex>"service":"(\w+)"</regex>
  <order>application_layer_protocol</order>
</decoder>

<decoder name="zeek_decoder">
  <parent>json</parent>
  <regex>"duration":(\d+.\d+)</regex>
  <order>duration_of_the_connection</order>
</decoder>

<decoder name="zeek_decoder">
  <parent>json</parent>
  <regex>"orig_bytes":(\d+)</regex>
  <order>byte_send_by_originator</order>
</decoder>

<decoder name="zeek_decoder">
  <parent>json</parent>
  <regex>"resp_bytes":(\d+)</regex>
  <order>byte_sent_by_responder</order>
</decoder>

<decoder name="zeek_decoder">
  <parent>json</parent>
  <regex>"conn_state":"(\w+)"</regex>
  <order>connection_state</order>
</decoder>

<decoder name="zeek_decoder">
  <parent>json</parent>
  <regex>"local_orig":(\w+)</regex>
  <order>local_origin</order>
</decoder>

<decoder name="zeek_decoder">
  <parent>json</parent>
  <regex>"local_resp":(\w+)</regex>
  <order>local_response</order>
</decoder>

<decoder name="zeek_decoder">
  <parent>json</parent>
  <regex>"missed_bytes":(\d+)</regex>
  <order>missed_bytes_might_packet_loss</order>
</decoder>

<decoder name="zeek_decoder">
  <parent>json</parent>
  <regex>"history":\w+</regex>
  <order>history</order>
</decoder>

<decoder name="zeek_decoder">
  <parent>json</parent>
  <regex>"orig_pkts":(\d+)</regex>
  <order>packet_sent_by_origin</order>
</decoder>

<decoder name="zeek_decoder">
  <parent>json</parent>
  <regex>"orig_ip_bytes":(\d+)</regex>
  <order>ip_layer_bytes_from_origin</order>
</decoder>

<decoder name="zeek_decoder">
  <parent>json</parent>
  <regex>"resp_pkts":(\d+)</regex>
  <order>packet_sent_by_responder</order>
</decoder>

<decoder name="zeek_decoder">
  <parent>json</parent>
  <regex>"resp_ip_bytes":(\d+)</regex>
  <order>ip_layer_bytes_sent_by_responder</order>
</decoder>

<decoder name="zeek_decoder">
  <parent>json</parent>
  <regex>"ip_proto":(\d+)</regex>
  <order>protocol_number_ip_header</order>
</decoder>

<!-- Software related -->
<decoder name="zeek_decoder">
  <parent>json</parent>
  <regex>"host":"(\d+.\d+.\d+.\d+)"</regex>
  <order>host_ip</order>
</decoder>

<decoder name="zeek_decoder">
  <parent>json</parent>
  <regex>"software_type":"(\.+)"</regex>
  <order>software_type</order>
</decoder>

<decoder name="zeek_decoder">
  <parent>json</parent>
  <regex>"name":"(\w+)"</regex>
  <order>software_name</order>
</decoder>

<decoder name="zeek_decoder">
  <parent>json</parent>
  <regex>"unparsed_version":"(\.+)"</regex>
  <order>software_version</order>
</decoder>

<decoder name="zeek_decoder">
  <parent>json</parent>
  <regex>"host_p":(\d+)</regex>
  <order>host_port</order>
</decoder>

<!-- SSL/TLS Connection Decoders -->
<decoder name="zeek_decoder">
  <parent>json</parent>
  <regex>"version":"(\.+)"</regex>
  <order>ssl_version</order>
</decoder>

<decoder name="zeek_decoder">
  <parent>json</parent>
  <regex>"cipher":"(\.+)"</regex>
  <order>ssl_cipher</order>
</decoder>

<decoder name="zeek_decoder">
  <parent>json</parent>
  <regex>"curve":"(\.+)"</regex>
  <order>ssl_curve</order>
</decoder>

<decoder name="zeek_decoder">
  <parent>json</parent>
  <regex>"validation_status":"(\.+)"</regex>
  <order>ssl_validation_status</order>
</decoder>

<decoder name="zeek_decoder">
  <parent>json</parent>
  <regex>"server_name":"(\.+)"</regex>
  <order>ssl_server_name</order>
</decoder>

<decoder name="zeek_decoder">
  <parent>json</parent>
  <regex>"established":(\w+)</regex>
  <order>ssl_established</order>
</decoder>

<decoder name="zeek_decoder">
  <parent>json</parent>
  <regex>"resumed":(\w+)</regex>
  <order>ssl_resumed</order>
</decoder>

<decoder name="zeek_decoder">
  <parent>json</parent>
  <regex>"ssl_history":"(\.+)"</regex>
  <order>ssl_history</order>
</decoder>

<decoder name="zeek_decoder">
  <parent>json</parent>
  <regex>"cert_chain_fps":"(\.+)"</regex>
  <order>ssl_cert_fingerprint</order>
</decoder>

<decoder name="zeek_decoder">
  <parent>json</parent>
  <regex>"next_protocol":"(\.+)"</regex>
  <order>ssl_next_protocol</order>
</decoder>
```

---

# 11. Wazuh to n8n Custom Integration

## 11.1 n8n Webhook

In n8n:

```
Node: Webhook
Method: POST
Path: wazuh-alert
Authentication: Basic Auth or Header Auth
Response: Immediately
```

Production URL:

```
http://<N8N_SERVER_IP>:5678/webhook/wazuh-alert
```

## 11.2 Wazuh ossec.conf Integration Block

On the Wazuh server:

```bash
sudo nano /var/ossec/etc/ossec.conf
```

Add:

```xml
<integration>
  <name>custom-n8n</name>
  <hook_url>http://<N8N_SERVER_IP>:5678/webhook/wazuh-alert</hook_url>
  <level>7</level>
  <alert_format>json</alert_format>
</integration>
```

## 11.3 Wazuh custom-n8n Script

Create:

```bash
sudo nano /var/ossec/integrations/custom-n8n
```

Script:

```python
#!/bin/sh
# Copyright (C) 2015, Wazuh Inc.
# Created by Wazuh, Inc. <info@wazuh.com>.
# This program is free software; you can redistribute it and/or modify it under the terms of GPLv2
# this is a copy of the slack integration file renamed to custom-n8n 

WPYTHON_BIN="framework/python/bin/python3"

SCRIPT_PATH_NAME="$0"

DIR_NAME="$(cd $(dirname ${SCRIPT_PATH_NAME}); pwd -P)"
SCRIPT_NAME="$(basename ${SCRIPT_PATH_NAME})"

case ${DIR_NAME} in
    */active-response/bin | */wodles*)
        if [ -z "${WAZUH_PATH}" ]; then
            WAZUH_PATH="$(cd ${DIR_NAME}/../..; pwd)"
        fi

        PYTHON_SCRIPT="${DIR_NAME}/${SCRIPT_NAME}.py"
    ;;
    */bin)
        if [ -z "${WAZUH_PATH}" ]; then
            WAZUH_PATH="$(cd ${DIR_NAME}/..; pwd)"
        fi

        PYTHON_SCRIPT="${WAZUH_PATH}/framework/scripts/$(echo ${SCRIPT_NAME} | sed 's/\-/_/g').py"
    ;;
     */integrations)
        if [ -z "${WAZUH_PATH}" ]; then
            WAZUH_PATH="$(cd ${DIR_NAME}/..; pwd)"
        fi

        PYTHON_SCRIPT="${DIR_NAME}/${SCRIPT_NAME}.py"
    ;;
esac

${WAZUH_PATH}/${WPYTHON_BIN} ${PYTHON_SCRIPT} "$@"
```

Create:

```bash
sudo nano /var/ossec/integrations/custom-n8n.py
```

Script:

```
#!/usr/bin/env python3
import sys
import requests
import json
import logging

logging.basicConfig(
    filename="/var/ossec/logs/integrations.log",
    level=logging.INFO,
    format="%(asctime)s - custom-n8n - %(levelname)s - %(message)s"
)

def main():
    if len(sys.argv) < 4:
        logging.error("Missing arguments")
        sys.exit(1)

    alert_file = sys.argv[1]
    hook_url   = sys.argv[3]

    try:
        with open(alert_file) as f:
            alert_json = json.load(f)
    except Exception as e:
        logging.error(f"Failed to read alert: {e}")
        sys.exit(1)

    logging.info(f"Sending | rule={alert_json.get('rule',{}).get('id')} | agent={alert_json.get('agent',{}).get('name')}")

    try:
        response = requests.post(
            hook_url,
            json=alert_json,
            timeout=10
        )
        logging.info(f"HTTP {response.status_code}")
    except Exception as e:
        logging.error(f"Error: {e}")
        sys.exit(1)

if __name__ == "__main__":
    main()
```

Set permissions:

```bash
sudo chown root:wazuh /var/ossec/integrations/custom-n8n
sudo chmod 750 /var/ossec/integrations/custom-n8n
sudo systemctl restart wazuh-manager
```

Check logs:

```bash
sudo tail -f /var/ossec/logs/integrations.log
```

---

# 12. n8n Installation and Configuration

## 12.1 n8n Docker Compose Install

Create directory:

```bash
mkdir -p /opt/n8n
cd /opt/n8n
```

Create `.env`:

```bash
nano .env
```

Example:

```
GENERIC_TIMEZONE=Asia/Dhaka
TZ=Asia/Dhaka
N8N_HOST=<N8N_SERVER_IP>
N8N_PORT=5678
N8N_PROTOCOL=http
WEBHOOK_URL=http://<N8N_SERVER_IP>:5678/
N8N_SECURE_COOKIE=false
```

Create `docker-compose.yml`:

```bash
nano docker-compose.yml
```

```yaml
services:
n8n:
image: docker.n8n.io/n8nio/n8n:latest
restart: unless-stopped
ports:
-"5678:5678"
env_file:
- .env
volumes:
- n8n_data:/home/node/.n8n

volumes:
n8n_data:
```

Start n8n:

```bash
docker compose up -d
```

Check:

```bash
docker compose ps
docker compose logs -f n8n
```

Access:

```
http://<N8N_SERVER_IP>:5678
```

---

# 13. DFIR-IRIS Installation and Configuration

## 13.1 Install DFIR-IRIS with Docker Compose

```bash
cd /opt
sudo git clone https://github.com/dfir-iris/iris-web.git
cd iris-web
```

Checkout stable version if needed:

```bash
git tag
sudo git checkout <STABLE_VERSION>
```

Create environment file:

```bash
sudo cp .env.model .env
sudo nano .env
```

Recommended `.env` changes:

```
POSTGRES_PASSWORD=<STRONG_POSTGRES_PASSWORD>
POSTGRES_ADMIN_PASSWORD=<STRONG_POSTGRES_ADMIN_PASSWORD>
IRIS_SECRET_KEY=<LONG_RANDOM_SECRET>
IRIS_SECURITY_PASSWORD_SALT=<LONG_RANDOM_SALT>
IRIS_ADM_EMAIL=admin@example.local
IRIS_ADM_USERNAME=administrator
IRIS_ADM_PASSWORD=<STRONG_ADMIN_PASSWORD>
```

Pull and start:

```bash
sudo docker compose pull
sudo docker compose up -d
```

Check services:

```bash
sudo docker compose ps
sudo docker compose logs -f app
```

Access:

```
https://<IRIS_SERVER_IP>
```

## 13.2 DFIR-IRIS Initial Setup

Inside the IRIS UI:

```
1. Log in as administrator.
2. Change default password if necessary.
3. Create or identify the customer.
4. Note the customer ID, commonly 1 in a lab.
5. Create a service/API user for n8n.
6. Assign the user to the correct customer.
7. Give the user alerts_read and alerts_write permissions.
8. Generate API key from My Settings → API Key.
```

## 13.3 DFIR-IRIS API Test

```bash
curl -k -X POST "https://<IRIS_SERVER>/alerts/add" \
  -H "Authorization: Bearer <IRIS_API_KEY>" \
  -H "Content-Type: application/json" \
  -d '{
    "alert_title": "n8n test alert",
    "alert_description": "Test alert from n8n",
    "alert_source": "n8n",
    "alert_source_ref": "n8n-test-001",
    "alert_severity_id": 4,
    "alert_status_id": 2,
    "alert_customer_id": 1,
    "alert_tags": "wazuh,n8n,test",
    "alert_source_content": {"test": true}
  }'
```

If datetime validation fails, omit `alert_source_event_time`.

---

# 14. Cortex Installation and Configuration

## 14.1 Run Cortex with Docker

Create directory:

```bash
mkdir -p /opt/cortex
cd /opt/cortex
```

Create `docker-compose.yml`:

```yaml
services:
elasticsearch:
image: docker.elastic.co/elasticsearch/elasticsearch:7.17.9
restart: unless-stopped
environment:
- discovery.type=single-node
- xpack.security.enabled=false
- ES_JAVA_OPTS=-Xms512m -Xmx512m
volumes:
- esdata:/usr/share/elasticsearch/data
ports:
-"9200:9200"

cortex:
image: thehiveproject/cortex:latest
restart: unless-stopped
depends_on:
- elasticsearch
ports:
-"9001:9001"
volumes:
- ./application.conf:/etc/cortex/application.conf
- /var/run/docker.sock:/var/run/docker.sock
- cortex-jobs:/tmp/cortex-jobs

volumes:
esdata:
cortex-jobs:
```

Create `application.conf`:

```bash
nano application.conf
```

Example:

```
play.http.secret.key="CHANGE_THIS_TO_A_LONG_RANDOM_SECRET"

search {
  index = cortex
  uri = "http://elasticsearch:9200"
}

analyzer {
  urls = [
    "https://catalogs.download.strangebee.com/latest/json/analyzers.json"
  ]
}

responder {
  urls = [
    "https://catalogs.download.strangebee.com/latest/json/responders.json"
  ]
}

job {
  directory = "/tmp/cortex-jobs"
  dockerDirectory = "/tmp/cortex-jobs"
}
```

Start Cortex:

```bash
docker compose up -d
```

Access:

```
http://<CORTEX_SERVER_IP>:9001
```

## 14.2 Cortex Initial Setup

In Cortex UI:

```
1. Create admin user.
2. Create organization.
3. Create analyst/service user.
4. Generate API key for DFIR-IRIS integration.
5. Refresh analyzers catalog.
6. Enable MISP analyzer.
```

---

# 15. MISP Installation and Configuration on MISP VM

> For production, use the official MISP install guides and secure the instance properly.
> 

## 15.1 Recommended Ubuntu Version

Use Ubuntu 24.04 for MISP 2.5.

## 15.2 Install MISP Using Official Installer

```bash
sudo apt update
sudo apt install -y git curl wget
cd /opt
sudo git clone https://github.com/MISP/MISP.git
cd MISP
sudo git checkout 2.5
```

Run Ubuntu 24.04 installer:

```bash
sudo bash INSTALL/INSTALL.ubuntu2404.sh
```

During installation, configure:

```
MISP domain
Base URL
Admin password
Database password
SSL settings
```

## 15.3 MISP Post-Install

Access:

```
https://<MISP_SERVER>
```

Create MISP API key:

```
User profile → Auth keys → Add authentication key
```

Save:

```
MISP URL
MISP API Key
```

---

# 16. Cortex MISP Analyzer Configuration

In Cortex:

```
Organization → Analyzers → MISP → Enable
```

Configure:

```
url: https://<MISP_SERVER>
key: <MISP_API_KEY>
cert_check: false if using self-signed certificate
```

Supported observable types include:

```
ip
domain
url
fqdn
hash
mail
filename
other
```

Test analyzer by submitting an IP/domain/hash.

---

# 17. Install DFIR-IRIS Cortex Analyzer Module

On the DFIR-IRIS server:

```bash
cd /opt
git clone https://github.com/socfortress/iris-cortexanalyzer-module.git
cd iris-cortexanalyzer-module
sudo ./buildnpush2iris.sh -a
```

In DFIR-IRIS UI:

```
Advanced → Modules → Add module
```

Module name:

```
iris_cortexanalyzer_module
```

Configure:

```
Cortex API Endpoint: http://<CORTEX_SERVER_IP>:9001
Cortex API Key: <CORTEX_API_KEY>
Cortex Analyzer Name: MISP or exact MISP analyzer flavor
```

Supported IOC types:

```
IP address
domain
hash
```

Workflow:

```
DFIR-IRIS Alert/Case → IOC → Run Cortex Analyzer → Cortex MISP Analyzer → MISP result
```

---

# 18. OpenCTI Installation and Configuration

## 18.1 Install OpenCTI with Docker Compose

```bash
mkdir -p /opt/opencti
cd /opt/opencti
git clone https://github.com/OpenCTI-Platform/docker.git
cd docker
cp .env.sample .env
```

Edit `.env`:

```bash
nano .env
```

Important values to configure:

```
OPENCTI_ADMIN_EMAIL=admin@example.local
OPENCTI_ADMIN_PASSWORD=<STRONG_PASSWORD>
OPENCTI_ADMIN_TOKEN=<UUID_OR_RANDOM_TOKEN>
OPENCTI_BASE_URL=http://<OPENCTI_SERVER_IP>:8080
MINIO_ROOT_USER=<MINIO_USER>
MINIO_ROOT_PASSWORD=<MINIO_PASSWORD>
RABBITMQ_DEFAULT_USER=<RABBITMQ_USER>
RABBITMQ_DEFAULT_PASS=<RABBITMQ_PASSWORD>
ELASTIC_MEMORY_SIZE=4G
```

Generate UUID token if needed:

```bash
uuidgen
```

Start OpenCTI:

```bash
docker compose up -d
```

Check:

```bash
docker compose ps
docker compose logs -f opencti
```

Access:

```
http://<OPENCTI_SERVER_IP>:8080
```

## 18.2 OpenCTI API Token

In OpenCTI:

```
Profile → API Access → Copy token
```

n8n uses:

```
Authorization: Bearer <OPENCTI_TOKEN>
Content-Type: application/json
```

GraphQL endpoint:

```
http://<OPENCTI_SERVER_IP>:8080/graphql
```

---

# 19. n8n Workflow Full Configuration

## 19.1 Node 1: Webhook

```
Name: Webhook
Method: POST
Path: wazuh-alert
Authentication: Basic Auth or Header Auth
Response Mode: Immediately
```

## 19.2 Node 2: Normalize Wazuh Alert

Code node:

## 19.3 Node: Extract IOC’s

```jsx
const normalized = $json;
const observables = normalized.observables || [];
const indicators = normalized.indicators || {};
const network = normalized.network || {};

function isPrivateIp(ip) {
  if (!ip) return false;
  return (
    ip.startsWith("10.") ||
    ip.startsWith("192.168.") ||
    /^172\.(1[6-9]|2[0-9]|3[0-1])\./.test(ip) ||
    ip.startsWith("127.") ||
    ip.startsWith("169.254.")
  );
}

function addIOC(arr, type, value, source) {
  if (!value) return;
  const clean = String(value).trim();
  if (!clean || clean === "null" || clean === "undefined") return;
  arr.push({ type, value: clean, source });
}

let iocs = [];

for (const obs of observables) {
  if (!obs || !obs.data) continue;
  let type = obs.dataType || "other";
  if (type === "ip") type = "ipv4-addr";
  if (type === "domain") type = "domain-name";
  if (type === "hash") type = "file";
  addIOC(iocs, type, obs.data, obs.message || "observable");
}

addIOC(iocs, "ipv4-addr", network.src_ip, "network.src_ip");
addIOC(iocs, "ipv4-addr", network.dst_ip, "network.dst_ip");
addIOC(iocs, "domain-name", indicators.domain, "indicators.domain");
addIOC(iocs, "url", indicators.url, "indicators.url");
addIOC(iocs, "file", indicators.file_hash, "indicators.file_hash");

iocs = iocs.filter(ioc => !(ioc.type === "ipv4-addr" && isPrivateIp(ioc.value)));

const seen = new Set();
iocs = iocs.filter(ioc => {
  const key = `${ioc.type}|${ioc.value}`.toLowerCase();
  if (seen.has(key)) return false;
  seen.add(key);
  return true;
});

return [{ json: { normalized, iocs } }];
```

## 19.4 Node: If IOC Exists

```jsx
{{ $json.iocs && $json.iocs.length > 0 }}
```

## 19.5 Node: Split Out

Split field:

```
iocs
```

If needed, use Code node to emit one item per IOC:

```jsx
const normalized = $json.normalized;
const iocs = $json.iocs || [];

return iocs.map(ioc => ({
  json: { normalized, ioc }
}));
```

## 19.6 Node: OpenCTI Query

HTTP Request:

```
Method: POST
URL: http://<OPENCTI_SERVER_IP>:8080/graphql
Headers:
  Authorization: Bearer <OPENCTI_TOKEN>
  Content-Type: application/json
```

Body:

```json
{
  "query": "query SearchOpenCTI($search: String!) { stixCyberObservables(search: $search, first: 5) { edges { node { id entity_type observable_value x_opencti_score created_at updated_at objectLabel { id value color } createdBy { ... on Identity { name } } } } } indicators(search: $search, first: 5) { edges { node { id name pattern pattern_type x_opencti_score valid_from valid_until created_at updated_at objectLabel { id value color } createdBy { ... on Identity { name } } } } } }",
  "variables": {
    "search": "{{ $json.ioc.value }}"
  }
}
```

## 19.7 Node: Normalize OpenCTI Result

```jsx
const response = $json;
const original = $('Split Out').item.json;
const ioc = original.ioc;
const normalized = original.normalized;
const data = response.data || {};

function labels(labelObj) {
  if (!labelObj) return [];
  if (Array.isArray(labelObj)) return labelObj.map(l => l?.value).filter(Boolean);
  if (labelObj.value) return [labelObj.value];
  if (labelObj.edges && Array.isArray(labelObj.edges)) return labelObj.edges.map(e => e.node?.value).filter(Boolean);
  return [];
}

function creator(createdBy) {
  return createdBy?.name || null;
}

const observableMatches = [];

for (const edge of data.stixCyberObservables?.edges || []) {
  const node = edge.node || {};
  observableMatches.push({
    match_type: "observable",
    id: node.id || null,
    entity_type: node.entity_type || null,
    value: node.observable_value || null,
    score: node.x_opencti_score ?? null,
    labels: labels(node.objectLabel),
    created_by: creator(node.createdBy),
    created_at: node.created_at || null,
    updated_at: node.updated_at || null
  });
}

const indicatorMatches = [];

for (const edge of data.indicators?.edges || []) {
  const node = edge.node || {};
  indicatorMatches.push({
    match_type: "indicator",
    id: node.id || null,
    name: node.name || null,
    pattern: node.pattern || null,
    pattern_type: node.pattern_type || null,
    score: node.x_opencti_score ?? null,
    valid_from: node.valid_from || null,
    valid_until: node.valid_until || null,
    labels: labels(node.objectLabel),
    created_by: creator(node.createdBy),
    created_at: node.created_at || null,
    updated_at: node.updated_at || null
  });
}

const allMatches = [...observableMatches, ...indicatorMatches];
let maxScore = 0;

for (const m of allMatches) {
  if (typeof m.score === "number" && m.score > maxScore) maxScore = m.score;
}

let verdict = "not_found";
if (allMatches.length > 0 && maxScore >= 80) verdict = "malicious";
else if (allMatches.length > 0 && maxScore >= 50) verdict = "suspicious";
else if (allMatches.length > 0) verdict = "known_low_score";

return [{
  json: {
    normalized,
    opencti_result: {
      ioc,
      found: allMatches.length > 0,
      verdict,
      max_score: maxScore,
      matches: allMatches
    }
  }
}];
```

## 19.8 Node: Aggregate OpenCTI Results

Set Code node to **Run Once for All Items**:

```jsx
const items = $input.all();
const normalized = items[0]?.json?.normalized || $('Normalize Wazuh Alert').item.json;
const opencti_enrichment = items.map(item => item.json.opencti_result);

const malicious = opencti_enrichment.filter(x => x.verdict === "malicious");
const suspicious = opencti_enrichment.filter(x => x.verdict === "suspicious");
const found = opencti_enrichment.filter(x => x.found);
const highestScore = Math.max(0, ...opencti_enrichment.map(x => x.max_score || 0));

const opencti_summary = {
  total_iocs_checked: opencti_enrichment.length,
  total_found: found.length,
  malicious_count: malicious.length,
  suspicious_count: suspicious.length,
  highest_score: highestScore,
  verdict: malicious.length > 0
    ? "malicious_ioc_found"
    : suspicious.length > 0
      ? "suspicious_ioc_found"
      : found.length > 0
        ? "known_ioc_found"
        : "no_ioc_found"
};

return [{ json: { normalized, opencti_summary, opencti_enrichment } }];
```

## 19.9 Node: No OpenCTI Context

```jsx
return [{
  json: {
    normalized: $json.normalized || $('Normalize Wazuh Alert').item.json || {},
    opencti_summary: {
      total_iocs_checked: 0,
      total_found: 0,
      malicious_count: 0,
      suspicious_count: 0,
      highest_score: 0,
      verdict: "no_ioc_available"
    },
    opencti_enrichment: []
  }
}];
```

## 19.10 Node: Message a Model

### System Prompt

```
You are a SOC Tier-2 security analyst assisting an automated Wazuh → n8n → OpenCTI → DFIR-IRIS workflow.

The alert may originate from Zeek, Suricata, Linux authentication logs, Wazuh FIM/syscheck, firewall logs, malware detection, UEBA detections, or generic Wazuh rules.

Your task is to decide whether the alert is suspicious enough to create an alert in DFIR-IRIS and send a notification.

Rules:
1. Return ONLY valid JSON.
2. Do not use markdown outside JSON.
3. Do not include explanations outside the JSON.
4. Do not invent facts.
5. Base your assessment only on the provided normalized alert and OpenCTI enrichment.
6. If evidence is weak or incomplete, lower the confidence.
7. Do not recommend automatic blocking, deletion, isolation, or destructive response unless human approval is required.
8. Prefer analyst-safe recommendations.
9. Treat Suricata malware/exploit/C2 alerts, Zeek repeated anomalies, port scans, brute-force, suspicious TLS, privilege escalation, malware, UEBA high-risk anomalies, and critical FIM events as potentially suspicious.
10. Treat routine successful connections, low-level informational events, expected internal traffic, and incomplete evidence as lower priority.
11. If the alert should create a DFIR-IRIS alert, set create_iris_alert to true.
12. If a human should be notified, set send_notification to true.
13. If the alert appears benign/noisy, set create_iris_alert to false and explain why in reason.
14. If OpenCTI enrichment exists, use it as supporting evidence.
15. If OpenCTI has no match, do not assume the alert is benign. Continue evaluating based on alert behavior.
16. If OpenCTI shows malicious or suspicious IOC context, increase confidence and severity when appropriate.
17. If OpenCTI only matches internal/private IPs or low-score benign observables, do not escalate based on OpenCTI alone.
18. Include OpenCTI findings in opencti_assessment, key_observations, iris_description, and iris_note when relevant.

OpenCTI decision rules:
- If opencti_summary.verdict is "malicious_ioc_found", normally set suspicious=true and create_iris_alert=true unless the IOC is clearly unrelated.
- If any OpenCTI match has max_score >= 80, treat the IOC as high-risk.
- If any OpenCTI match has max_score between 50 and 79, treat the IOC as suspicious but consider alert context.
- If OpenCTI labels include malware, c2, botnet, phishing, ransomware, exploit, scanner, trojan, or command-and-control, raise confidence.
- If OpenCTI found only known low-score items, mention this but do not escalate solely from that.
- If OpenCTI found nothing, set opencti_assessment.verdict to "no_ioc_found".

Severity mapping:
- CRITICAL: active compromise, malware execution, confirmed C2, privilege escalation, high-confidence exploitation, ransomware-like behavior.
- HIGH: strong suspicious behavior, IDS malware/exploit alert, brute-force with success, scanning against sensitive assets, malicious OpenCTI IOC match.
- MEDIUM: suspicious but not confirmed, anomalies, failed brute-force, weak TLS issue, unusual connection pattern, suspicious OpenCTI IOC match.
- LOW: weak signal, likely benign, informational but worth logging.

DFIR-IRIS severity mapping:
- CRITICAL = 6
- HIGH = 5
- MEDIUM = 4
- LOW = 3
- INFORMATIONAL = 2
- UNSPECIFIED = 1

Return JSON using this exact schema:
{
  "suspicious": false,
  "create_iris_alert": false,
  "send_notification": false,
  "severity": "LOW",
  "iris_severity_id": 3,
  "confidence": "LOW",
  "alert_title": "",
  "summary": "",
  "reason": "",
  "opencti_assessment": {
    "used": false,
    "verdict": "no_ioc_found",
    "highest_score": 0,
    "matched_iocs": [],
    "impact_on_decision": ""
  },
  "likely_attack": "unknown",
  "mitre_attack": [],
  "key_observations": [],
  "recommended_investigation_steps": [],
  "recommended_response_actions": [],
  "false_positive_likelihood": "MEDIUM",
  "notification_message": "",
  "iris_tags": [],
  "iris_description": "",
  "iris_note": ""
}
```

### User Message

```
Analyze this normalized Wazuh alert with OpenCTI enrichment and decide whether to create a DFIR-IRIS alert.

Input JSON:
{{ JSON.stringify({
  normalized: $json.normalized || $json,
  opencti_summary: $json.opencti_summary || {},
  opencti_enrichment: $json.opencti_enrichment || []
}) }}
```

## 19.11 Node: Parse AI Triage JSON

```jsx
const item = $input.first().json;
let text = null;

if (item.output && Array.isArray(item.output)) {
  for (const outputItem of item.output) {
    if (outputItem.content && Array.isArray(outputItem.content)) {
      for (const contentItem of outputItem.content) {
        if (contentItem.type === 'output_text' && contentItem.text) {
          text = contentItem.text;
          break;
        }
      }
    }
    if (text) break;
  }
}

if (!text && item.text) text = item.text;
if (!text && item.message && item.message.content) text = item.message.content;
if (!text) throw new Error('Could not find AI output text');

text = text.trim()
  .replace(/^```json/i, '')
  .replace(/^```/i, '')
  .replace(/```$/i, '')
  .trim();

const triage = JSON.parse(text);

return [{ json: triage }];
```

## 19.12 Node: IF Create IRIS Alert

```jsx
{{
  $json.create_iris_alert === true &&
  ["HIGH", "CRITICAL"].includes($json.severity) &&
  ["MEDIUM", "HIGH"].includes($json.confidence)
}}
```

## 19.13 Node: Build DFIR-IRIS Payload

```jsx
const ai = $json.ai_triage || $json;
const normalized = $('Aggregate OpenCTI Results').item.json.normalized || $('Normalize Wazuh Alert').item.json;
const opencti_summary = $('Aggregate OpenCTI Results').item.json.opencti_summary || {};
const opencti_enrichment = $('Aggregate OpenCTI Results').item.json.opencti_enrichment || [];

const irisPayload = {
  alert_title: ai.alert_title || normalized.rule?.description || "Wazuh AI triage alert",
  alert_description: ai.iris_description || ai.summary || "AI triage generated alert",
  alert_source: "Wazuh-n8n-OpenAI-OpenCTI",
  alert_source_ref: normalized.event?.fingerprint || normalized.wazuh?.alert_id || `${Date.now()}`,
  alert_source_link: "https://<WAZUH_DASHBOARD>",
  alert_severity_id: ai.iris_severity_id || 4,
  alert_status_id: 2,
  alert_note: ai.iris_note || ai.reason || "",
  alert_tags: Array.isArray(ai.iris_tags)
    ? ai.iris_tags.join(",")
    : "wazuh,n8n,openai,opencti",
  alert_customer_id: 1,
  alert_source_content: {
    ai_triage: ai,
    normalized_alert: normalized,
    opencti_summary,
    opencti_enrichment
  }
};

return [{ json: irisPayload }];
```

If the workflow has a no-IOC branch, use a safer version that references the current item:

```jsx
const ai = $json.ai_triage || $json;
const source = $input.first().json;
const normalized = source.normalized || $('Normalize Wazuh Alert').item.json;
const opencti_summary = source.opencti_summary || {};
const opencti_enrichment = source.opencti_enrichment || [];

return [{
  json: {
    alert_title: ai.alert_title || normalized.rule?.description || "Wazuh AI triage alert",
    alert_description: ai.iris_description || ai.summary || "AI triage generated alert",
    alert_source: "Wazuh-n8n-OpenAI-OpenCTI",
    alert_source_ref: normalized.event?.fingerprint || normalized.wazuh?.alert_id || `${Date.now()}`,
    alert_source_link: "https://<WAZUH_DASHBOARD>",
    alert_severity_id: ai.iris_severity_id || 4,
    alert_status_id: 2,
    alert_note: ai.iris_note || ai.reason || "",
    alert_tags: Array.isArray(ai.iris_tags) ? ai.iris_tags.join(",") : "wazuh,n8n,openai,opencti",
    alert_customer_id: 1,
    alert_source_content: {
      ai_triage: ai,
      normalized_alert: normalized,
      opencti_summary,
      opencti_enrichment
    }
  }
}];
```

## 19.14 Node: DFIR-IRIS HTTP Request

```
Method: POST
URL: https://<IRIS_SERVER>/alerts/add
Authentication: None
Headers:
  Authorization: Bearer <IRIS_API_KEY>
  Content-Type: application/json
Body: JSON from previous node
```

If using self-signed certificate:

```
Ignore SSL Issues: true
```

## 19.15 Node: IF Send Notification

```jsx
{{
  $json.send_notification === true &&
  ["MEDIUM", "HIGH", "CRITICAL"].includes($json.severity)
}}
```

## 19.16 Node: Telegram Notification

Example message:

```
🚨 DFIR-IRIS Alert Created

Severity: {{ $json.severity }}
Confidence: {{ $json.confidence }}

Title:
{{ $json.alert_title }}

Summary:
{{ $json.summary }}

Reason:
{{ $json.reason }}
```

---

# 20. UEBA Prototype Notes

A small Isolation Forest script was reviewed for UEBA-style authentication anomaly detection. It can be used as a prototype, but it does not provide complete UEBA coverage by itself.

Minimum UEBA pipeline recommendation:

```
1. Collect Wazuh authentication alerts.
2. Aggregate per user/entity per time window.
3. Train baseline model.
4. Score new login windows.
5. Emit UEBA anomaly event into n8n.
6. Query OpenCTI where relevant.
7. Send to AI triage.
8. Create DFIR-IRIS alert if high risk.
```

Recommended UEBA features:

```
login_count_1h
failed_count_1h
failed_count_24h
unique_src_ips_24h
new_ip_for_user
new_country_for_user
after_hours
success_after_failures
impossible_travel_flag
admin_login_from_unusual_source
```

---

# 21. Troubleshooting

## 21.1 n8n Webhook 404

Cause:

```
Workflow inactive
Wrong webhook path
Using /webhook/ while workflow is inactive
Using /webhook-test/ outside test mode
Wrong HTTP method
```

Fix:

```
Activate workflow.
Use /webhook/wazuh-alert for production.
Use /webhook-test/... only when testing.
Confirm method is POST.
```

Test:

```bash
curl -i -X POST \
  "http://<N8N_SERVER_IP>:5678/webhook/wazuh-alert" \
  -H "Content-Type: application/json" \
  -d '{"test":"wazuh webhook"}'
```

## 21.2 DFIR-IRIS Credentials Not Found in n8n

Cause:

```
HTTP Request node selected a missing n8n credential.
```

Fix:

```
Set Authentication = None.
Use manual headers.
Authorization: Bearer <IRIS_API_KEY>
Content-Type: application/json
```

## 21.3 DFIR-IRIS Datetime Error

Error:

```
{'alert_source_event_time': ['Not a valid datetime.']}
```

Fix:

```
Remove alert_source_event_time from payload.
```

or convert:

```jsx
new Date(timestamp).toISOString()
```

## 21.4 OpenCTI objectLabel Error

Error:

```
Cannot query field "edges" on type "Label".
```

Fix query:

```graphql
objectLabel {
  id
  value
  color
}
```

Do not use:

```graphql
objectLabel {
  edges {
    node {
      value
    }
  }
}
```

## 21.5 AI Prompt Shows undefined

Use safe input:

```jsx
{{ JSON.stringify({
  normalized: $json.normalized || $json,
  opencti_summary: $json.opencti_summary || {},
  opencti_enrichment: $json.opencti_enrichment || []
}) }}
```

## 21.6 Duplicate DFIR-IRIS Alerts

Cause:

```
AI runs once per IOC after Split Out.
```

Fix:

```
Aggregate OpenCTI results before AI.
```

## 21.7 Wazuh Rule Syntax Errors

Common fixes:

- Use `<field name="field.name">regex</field>` instead of `<field.name>regex</field.name>`.
- Use `type="pcre2"` when regex contains `|`, groups, `\d`, `\S`, or advanced syntax.
- Validate using:

```bash
sudo /var/ossec/bin/wazuh-analysisd -t
```

---

# 22. Security Hardening Recommendations

## 22.1 SSH Hardening on VPN Server

```bash
sudo nano /etc/ssh/sshd_config
```

Recommended:

```
PermitRootLogin no
PasswordAuthentication no
PubkeyAuthentication yes
```

Restart:

```bash
sudo systemctl restart ssh
```

Restrict SSH with firewall/security group to trusted IPs only.

## 22.2 API Key Handling

```
Do not hardcode API keys in exported workflows.
Use n8n credentials where possible.
Use environment variables or secret storage.
Rotate keys if exposed.
Restrict API users to minimum permissions.
```

## 22.3 n8n Security

```
Use HTTPS behind reverse proxy.
Enable authentication.
Limit webhook exposure.
Use Basic/Header/JWT auth for Wazuh webhook.
Keep n8n updated.
Back up n8n volume.
```

## 22.4 DFIR-IRIS Security

```
Do not expose IRIS directly to the Internet.
Use VPN/private network.
Assign API user to correct customer.
Use least-privilege permissions.
Back up database.
```

## 22.5 OpenCTI Security

```
Use dedicated API token for n8n.
Do not expose GraphQL publicly.
Limit API user permissions.
Back up OpenCTI stack.
```

---

# 23. Final End-to-End Data Flow

```
1. Zeek / Suricata / Linux auth logs are generated on VPN server.
2. Wazuh Agent forwards logs to Wazuh Manager.
3. Wazuh decoders and rules generate alerts.
4. Wazuh custom-n8n integration sends selected alerts to n8n.
5. n8n normalizes the alert.
6. n8n extracts IOCs.
7. n8n filters private/internal IOCs.
8. n8n queries OpenCTI for IOC intelligence.
9. n8n normalizes and aggregates OpenCTI results.
10. AI model performs SOC Tier-2 triage.
11. n8n parses AI JSON output.
12. If suspicious, n8n builds DFIR-IRIS payload.
13. n8n creates DFIR-IRIS alert through /alerts/add.
14. n8n sends Telegram notification.
15. Analyst reviews DFIR-IRIS alert.
16. Analyst can run Cortex analyzer from DFIR-IRIS IOC view.
17. Cortex MISP Analyzer queries MISP VM.
18. Analyst uses results to investigate and respond.
```

---

# 24. Key Endpoints

```
Wazuh Dashboard:
https://<WAZUH_SERVER_IP>

n8n:
http://<WAZUH_SERVER_IP>:5678

n8n Wazuh Webhook:
http://<WAZUH_SERVER_IP>:5678/webhook/wazuh-alert

DFIR-IRIS:
https://<IRIS_SERVER_IP>

DFIR-IRIS Alert API:
https://<IRIS_SERVER_IP>/alerts/add

Cortex:
http://<CORTEX_SERVER_IP>:9001

OpenCTI:
http://<OPENCTI_SERVER_IP>:8080

OpenCTI GraphQL:
http://<OPENCTI_SERVER_IP>:8080/graphql

MISP:
https://<MISP_SERVER_IP>
```

---

# 25. Reference Documentation

Use these official or project references for authoritative installation and configuration details:

- Wazuh official documentation: https://documentation.wazuh.com/
- Wazuh agent installation: https://documentation.wazuh.com/current/installation-guide/wazuh-agent/index.html
- Wazuh Suricata integration: https://documentation.wazuh.com/current/proof-of-concept-guide/integrate-network-ids-suricata.html
- Zeek official documentation: https://docs.zeek.org/
- Suricata official documentation: https://docs.suricata.io/
- n8n Docker documentation: https://docs.n8n.io/hosting/installation/docker/
- n8n Docker Compose documentation: https://docs.n8n.io/hosting/installation/server-setups/docker-compose/
- DFIR-IRIS documentation: https://docs.dfir-iris.org/
- DFIR-IRIS GitHub: https://github.com/dfir-iris/iris-web
- iris-cortexanalyzer-module: https://github.com/socfortress/iris-cortexanalyzer-module
- Cortex documentation: https://docs.strangebee.com/cortex/
- Cortex analyzers: https://thehive-project.github.io/Cortex-Analyzers/
- MISP project: https://www.misp-project.org/
- MISP install guides: https://misp.github.io/MISP/
- OpenCTI documentation: https://docs.opencti.io/
- OpenCTI Docker deployment: https://github.com/OpenCTI-Platform/docker

---

# 26. Appendix: Quick Validation Commands

## Wazuh

```bash
sudo /var/ossec/bin/wazuh-analysisd -t
sudo systemctl restart wazuh-manager
sudo tail -f /var/ossec/logs/ossec.log
sudo tail -f /var/ossec/logs/integrations.log
```

## Zeek

```bash
sudo /opt/zeek/bin/zeekctl status
sudo /opt/zeek/bin/zeekctl check
sudo /opt/zeek/bin/zeekctl deploy
ls -lah /opt/zeek/logs/current/
```

## Suricata

```bash
sudo systemctl status suricata
sudo suricata-update
sudo tail -f /var/log/suricata/eve.json | jq
```

## n8n

```bash
cd /opt/n8n
docker compose ps
docker compose logs -f n8n
```

## DFIR-IRIS

```bash
cd /opt/iris-web
sudo docker compose ps
sudo docker compose logs -f app
```

## Cortex

```bash
cd /opt/cortex
docker compose ps
docker compose logs -f cortex
```

## OpenCTI

```bash
cd /opt/opencti/docker
docker compose ps
docker compose logs -f opencti
```

## MISP

```bash
sudo systemctl status apache2
sudo systemctl status mariadb
sudo systemctl status redis-server
```
