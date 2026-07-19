## Wazuh + Zeek + Suricata + n8n + DFIR-IRIS + Cortex + MISP + OpenCTI + AI Triage

> **Purpose:** This document describes the SOC/SOAR lab architecture and workflow built during the project. Wazuh receives endpoint and network telemetry from a VPN server running Zeek and Suricata, forwards selected alerts to n8n, enriches IOCs with OpenCTI, uses an AI model for triage, creates DFIR-IRIS alerts for suspicious events, sends notifications, and supports analyst enrichment through DFIR-IRIS → Cortex → MISP Analyzer.
> 

---

## 1. High-Level Architecture

```
                ┌──────────────────────────────┐
                │          MISP VM              │
                │  Threat Intel Platform        │
                │  Provides IOC intelligence    │
                └──────────────▲───────────────┘
                               │
                               │ Cortex MISP Analyzer
                               │
┌──────────────────────────────┴──────────────────────────────┐
│                  Wazuh / SOAR Server VM                      │
│                                                              │
│  ┌──────────────┐   ┌──────────────┐   ┌──────────────────┐ │
│  │ Wazuh Server │   │ Wazuh Indexer│   │ Wazuh Dashboard  │ │
│  └──────▲───────┘   └──────────────┘   └──────────────────┘ │
│         │                                                    │
│         │ Custom integration webhook                         │
│         ▼                                                    │
│  ┌──────────────┐       ┌──────────────┐                     │
│  │     n8n      │──────▶│   OpenCTI    │                     │
│  │ SOAR logic   │◀──────│ IOC lookup   │                     │
│  └──────▲───────┘       └──────────────┘                     │
│         │                                                    │
│         │ AI triage + suspicious decision                    │
│         ▼                                                    │
│  ┌──────────────┐       ┌──────────────┐                     │
│  │ DFIR-IRIS    │──────▶│   Cortex     │                     │
│  │ Alerts/Cases │       │ Analyzers    │                     │
│  └──────────────┘       └──────────────┘                     │
└──────────────────────────────────────────────────────────────┘
                               ▲
                               │ Wazuh Agent log forwarding
                               │
                ┌──────────────┴───────────────┐
                │          VPN Server           │
                │  Wazuh Agent                  │
                │  Zeek                         │
                │  Suricata                     │
                └──────────────────────────────┘
```

---

## 2. Environment Summary

### 2.1 Wazuh / SOAR Server VM

The Wazuh/SOAR VM hosts the main SOC backend:

- Wazuh Server / Manager
- Wazuh Indexer
- Wazuh Dashboard
- n8n
- DFIR-IRIS
- Cortex
- OpenCTI

Wazuh was installed step by step using the official Wazuh documentation. This document does **not** repeat the full Wazuh installation procedure. Use the official Wazuh documentation for installation, upgrade, package, and deployment details.

### 2.2 VPN Server

The VPN server acts as a monitored endpoint and network sensor:

- Wazuh Agent
- Zeek
- Suricata
- Linux system logs, including SSH authentication logs

The Wazuh Agent forwards Zeek, Suricata, and Linux authentication logs to the Wazuh Manager.

### 2.3 MISP VM

MISP runs on a separate VM and provides threat intelligence. Cortex uses the MISP Analyzer to query MISP for observable enrichment.

---

## 3. Component Roles

## 3.1 Wazuh

Wazuh provides SIEM/XDR capabilities:

- Receives logs from Wazuh Agents.
- Applies decoders and rules.
- Generates alerts.
- Sends selected alerts to n8n through a custom integration.
- Provides alert visibility in the Wazuh Dashboard.

## 3.2 Zeek

Zeek provides network metadata and protocol context. The important Zeek logs selected for Wazuh forwarding were:

```
conn.log
ssl.log
weird.log
```

Not every Zeek log was forwarded to Wazuh. Operational Zeek logs such as `stdout.log`, `stderr.log`, `packet_filter.log`, `loaded_scripts.log`, and similar internal runtime logs were considered low-value for SIEM alerting.

## 3.3 Suricata

Suricata provides network IDS alerts. Suricata writes JSON alerts to:

```
/var/log/suricata/eve.json
```

The Wazuh Agent monitors this file and forwards IDS alerts to Wazuh.

## 3.4 n8n

n8n is the SOAR/orchestration layer. It receives Wazuh alerts, normalizes data, extracts IOCs, queries OpenCTI, sends context to the AI model, creates DFIR-IRIS alerts, and sends notifications.

## 3.5 OpenCTI

OpenCTI provides threat intelligence enrichment before AI triage. n8n queries OpenCTI for extracted IOCs such as:

```
IPv4 addresses
domains
URLs
hashes
```

## 3.6 DFIR-IRIS

DFIR-IRIS is used for alert and case management. n8n creates an IRIS alert only when AI triage decides that an alert is suspicious enough.

## 3.7 Cortex

Cortex is used as an observable analysis engine. DFIR-IRIS connects to Cortex through the `iris-cortexanalyzer-module` so analysts can run Cortex analyzers from IRIS.

## 3.8 MISP

MISP is used as a threat intelligence source. Cortex is configured with a MISP Analyzer that queries the MISP VM.

---

## 4. VPN Server Log Forwarding

### 4.1 Suricata Log Forwarding

Example Wazuh Agent configuration:

```xml
<localfile>
  <location>/var/log/suricata/eve.json</location>
  <log_format>json</log_format>
</localfile>
```

### 4.2 Zeek Log Forwarding

Example Wazuh Agent configuration:

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

### 4.3 SSH Authentication Logs

Linux authentication logs are monitored to detect SSH brute force, invalid user attempts, and repeated pre-authentication root attempts.

Example suspicious log observed:

```
Connection closed by authenticating user root 103.238.155.149 port 32478 [preauth]
```

Wazuh default rule `5722` classified this as level `0`, so a custom decoder and rule were added to detect repeated suspicious activity.

---

## 5. Wazuh Custom Detection Work

## 5.1 Zeek Rules

Custom Zeek rules were created for:

- Base Zeek JSON detection.
- Zeek connection events.
- Rejected connections.
- Repeated rejected connections from the same source.
- Administrative port access.
- SSH/RDP observations.
- SSL/TLS weak or suspicious events.
- Zeek `weird.log` anomalies.

## 5.2 SSH Pre-Authentication Rules

Custom Wazuh rules were created to detect repeated SSH pre-authentication root attempts.

Example rule logic:

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

---

## 6. Wazuh to n8n Integration

Wazuh forwards selected alerts to n8n using a custom integration script.

Example Wazuh integration block:

```xml
<integration>
  <name>custom-n8n</name>
  <hook_url>http://N8N_SERVER_IP:5678/webhook/wazuh-alert</hook_url>
  <level>7</level>
  <alert_format>json</alert_format>
</integration>
```

### Important Webhook Notes

- `/webhook-test/...` is for testing only.
- `/webhook/...` is for active production workflows.
- If n8n returns `404`, usually the workflow is inactive or the webhook path is wrong.
- The n8n workflow must be activated for production webhook URLs.

---

## 7. Final n8n Workflow

## 7.1 Workflow Overview

```
Webhook
  ↓
Normalize Wazuh Alert
  ↓
Extract IOC's
  ↓
If IOC exists?
  ├── True:
  │     ↓
  │   Split Out
  │     ↓
  │   OpenCTI Query
  │     ↓
  │   Normalize OpenCTI Result
  │     ↓
  │   Aggregate OpenCTI Results
  │     ↓
  │   Message a model
  │
  └── False:
        ↓
      No OpenCTI Context
        ↓
      Message a model

Message a model
  ↓
Parse AI Triage JSON
  ↓
If create_iris_alert == true
  ↓
Build DFIR-IRIS Payload
  ↓
DFIR-IRIS HTTP Request

Parse AI Triage JSON
  ↓
If send_notification == true
  ↓
Telegram Notification
```

---

## 8. n8n Node Details

## 8.1 Webhook

Receives Wazuh alerts.

Recommended settings:

```
Method: POST
Path: wazuh-alert
Authentication: Basic Auth or Header Auth
Response: Immediately
```

## 8.2 Normalize Wazuh Alert

Normalizes different Wazuh alert structures into a canonical schema.

Canonical output:

```json
{
  "schema_version": "1.0",
  "event": {},
  "wazuh": {},
  "rule": {},
  "agent": {},
  "network": {},
  "identity": {},
  "indicators": {},
  "source_specific": {},
  "observables": [],
  "normalization_notes": [],
  "raw": {}
}
```

## 8.3 Extract IOC’s

Extracts:

```
ip
domain
url
hash
user
file
process
port
```

Private/internal IPs are filtered before OpenCTI lookup.

## 8.4 If IOC Exists

Condition:

```jsx
{{ $json.iocs && $json.iocs.length > 0 }}
```

## 8.5 Split Out

Creates one n8n item per IOC.

## 8.6 OpenCTI Query

Queries OpenCTI GraphQL API.

HTTP Request configuration:

```
Method: POST
URL: https://OPENCTI_SERVER/graphql
Headers:
  Authorization: Bearer <OPENCTI_TOKEN>
  Content-Type: application/json
```

GraphQL query example:

```json
{
  "query": "query SearchOpenCTI($search: String!) { stixCyberObservables(search: $search, first: 5) { edges { node { id entity_type observable_value x_opencti_score created_at updated_at objectLabel { id value color } createdBy { ... on Identity { name } } } } } indicators(search: $search, first: 5) { edges { node { id name pattern pattern_type x_opencti_score valid_from valid_until created_at updated_at objectLabel { id value color } createdBy { ... on Identity { name } } } } } }",
  "variables": {
    "search": "{{ $json.ioc.value }}"
  }
}
```

## 8.7 Normalize OpenCTI Result

Normalizes each OpenCTI response into:

```json
{
  "opencti_result": {
    "ioc": {},
    "found": false,
    "verdict": "not_found",
    "max_score": 0,
    "matches": []
  }
}
```

Verdict logic:

```
max_score >= 80  → malicious
max_score >= 50  → suspicious
found low score   → known_low_score
no match          → not_found
```

## 8.8 Aggregate OpenCTI Results

Combines all IOC lookups into one alert-level object.

Output:

```json
{
  "normalized": {},
  "opencti_summary": {
    "total_iocs_checked": 1,
    "total_found": 0,
    "malicious_count": 0,
    "suspicious_count": 0,
    "highest_score": 0,
    "verdict": "no_ioc_found"
  },
  "opencti_enrichment": []
}
```

This prevents duplicate AI decisions and duplicate DFIR-IRIS alerts.

## 8.9 No OpenCTI Context

Used when no IOC exists.

Output:

```json
{
  "normalized": {},
  "opencti_summary": {
    "total_iocs_checked": 0,
    "total_found": 0,
    "malicious_count": 0,
    "suspicious_count": 0,
    "highest_score": 0,
    "verdict": "no_ioc_available"
  },
  "opencti_enrichment": []
}
```

## 8.10 Message a Model

The AI model receives normalized alert data plus OpenCTI enrichment.

Input format:

```json
{
  "normalized": {},
  "opencti_summary": {},
  "opencti_enrichment": []
}
```

The AI decides:

```
suspicious
create_iris_alert
send_notification
severity
confidence
summary
reason
recommended investigation steps
recommended response actions
```

## 8.11 Parse AI Triage JSON

The AI response is wrapped inside the model output object. The parse node extracts the actual JSON string and converts it into structured JSON.

Expected parsed output:

```json
{
  "suspicious": true,
  "create_iris_alert": true,
  "send_notification": true,
  "severity": "HIGH",
  "iris_severity_id": 5,
  "confidence": "HIGH",
  "alert_title": "Suspicious activity detected",
  "summary": "...",
  "reason": "..."
}
```

## 8.12 If create_iris_alert

Condition:

```jsx
{{
  $json.create_iris_alert === true &&
  ["HIGH", "CRITICAL"].includes($json.severity) &&
  ["MEDIUM", "HIGH"].includes($json.confidence)
}}
```

## 8.13 Build DFIR-IRIS Payload

Builds the payload for `/alerts/add`.

Example payload:

```json
{
  "alert_title": "Suspicious SSH Brute Force Activity",
  "alert_description": "AI-generated investigation summary",
  "alert_source": "Wazuh-n8n-OpenAI-OpenCTI",
  "alert_source_ref": "linux_auth|5710|Internal_vpn|118.179.197.121|unknown|unknown",
  "alert_source_link": "https://WAZUH_DASHBOARD",
  "alert_severity_id": 5,
  "alert_status_id": 2,
  "alert_note": "Investigate repeated authentication failures.",
  "alert_tags": "wazuh,n8n,openai,opencti,ssh,brute-force",
  "alert_customer_id": 1,
  "alert_source_content": {
    "ai_triage": {},
    "normalized_alert": {},
    "opencti_summary": {},
    "opencti_enrichment": []
  }
}
```

## 8.14 DFIR-IRIS HTTP Request

Configuration:

```
Method: POST
URL: https://DFIR_IRIS_SERVER/alerts/add
Authentication: None
Headers:
  Authorization: Bearer <DFIR_IRIS_API_KEY>
  Content-Type: application/json
Body:
  JSON payload from Build DFIR-IRIS Payload
```

## 8.15 Notification

Telegram notification is sent when:

```jsx
{{
  $json.send_notification === true &&
  ["MEDIUM", "HIGH", "CRITICAL"].includes($json.severity)
}}
```

---

## 9. AI Triage Prompt Summary

The AI prompt was designed to act as a SOC Tier-2 analyst.

Key instructions:

- Return only valid JSON.
- Do not invent facts.
- Use normalized alert and OpenCTI enrichment only.
- Treat Suricata malware/exploit/C2 alerts as suspicious.
- Treat Zeek repeated anomalies, scan behavior, rejected connection bursts, suspicious TLS, and weird.log events as suspicious.
- Treat brute-force, privilege escalation, malware, suspicious FIM, and critical auth events as suspicious.
- Use OpenCTI as supporting evidence.
- If OpenCTI has no match, do not assume benign.
- If OpenCTI shows malicious IOC context, increase confidence and severity when appropriate.
- Recommend safe analyst actions only.
- Do not recommend automatic destructive response without human approval.

AI output schema:

```json
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

---

## 10. DFIR-IRIS and Cortex Analyzer Integration

## 10.1 iris-cortexanalyzer-module

Installed module:

```
iris-cortexanalyzer-module
```

Purpose:

```
Allow DFIR-IRIS analysts to run Cortex analyzers directly from DFIR-IRIS on IOCs.
```

Module configuration values:

```
Cortex API endpoint
Cortex API key
Cortex Analyzer name
```

Supported IOC types include:

```
IP address
domain
hash
```

## 10.2 Cortex MISP Analyzer

Cortex was configured with the MISP Analyzer.

Typical MISP analyzer settings:

```
name: MISP
url: https://MISP_SERVER
key: <MISP_API_KEY>
cert_check: false or true depending on certificate setup
```

This allows Cortex to query MISP for observables during analyst investigation.

---

## 11. Example End-to-End Scenarios

## 11.1 SSH Brute Force / Invalid User

Flow:

```
1. SSH attempts hit VPN server.
2. Linux auth.log records attempts.
3. Wazuh Agent forwards auth logs.
4. Wazuh rules generate alert.
5. Wazuh forwards alert to n8n.
6. n8n normalizes alert.
7. n8n extracts source IP.
8. n8n queries OpenCTI.
9. AI triages based on Wazuh + OpenCTI.
10. DFIR-IRIS alert is created if suspicious.
11. Telegram notification is sent if required.
```

## 11.2 Suricata C2 / Malware Alert

Flow:

```
1. Suricata detects suspicious network traffic.
2. Suricata writes alert to eve.json.
3. Wazuh Agent forwards eve.json event.
4. Wazuh generates Suricata IDS alert.
5. n8n extracts IP/domain/URL.
6. OpenCTI checks IOC reputation.
7. AI evaluates IDS signature + OpenCTI context.
8. DFIR-IRIS alert is created if suspicious.
```

## 11.3 Zeek Internal Noise Suppression

Example:

```
Single internal Zeek REJ connection
```

AI should classify as:

```json
{
  "suspicious": false,
  "create_iris_alert": false,
  "send_notification": false,
  "severity": "LOW",
  "false_positive_likelihood": "HIGH"
}
```

---

## 12. Troubleshooting

## 12.1 n8n Webhook 404

Cause:

```
Workflow inactive
Wrong webhook path
Using production URL while workflow is not active
Using test URL outside test mode
```

Fix:

```
Activate the workflow.
Use /webhook/... for production.
Use /webhook-test/... only while testing.
Verify method is POST.
```

## 12.2 DFIR-IRIS Credentials Not Found

Cause:

```
n8n HTTP Request node is trying to use a missing credential.
```

Fix:

```
Set Authentication = None.
Use manual headers:
Authorization: Bearer <DFIR_IRIS_API_KEY>
Content-Type: application/json
```

## 12.3 DFIR-IRIS Datetime Error

Error:

```
{'alert_source_event_time': ['Not a valid datetime.']}
```

Fix:

```
Remove alert_source_event_time from the payload.
```

or convert timestamp to ISO UTC:

```jsx
new Date(timestamp).toISOString()
```

## 12.4 OpenCTI GraphQL Label Error

Error:

```
Cannot query field "edges" on type "Label".
```

Fix:

Use:

```graphql
objectLabel {
  id
  value
  color
}
```

instead of:

```graphql
objectLabel {
  edges {
    node {
      value
    }
  }
}
```

## 12.5 AI Prompt Shows undefined

Cause:

```
The AI prompt references fields that do not exist in the current n8n item.
```

Fix:

```jsx
{{ JSON.stringify({
  normalized: $json.normalized || $json,
  opencti_summary: $json.opencti_summary || {},
  opencti_enrichment: $json.opencti_enrichment || []
}) }}
```

## 12.6 Duplicate DFIR-IRIS Alerts

Cause:

```
AI runs once per IOC after Split Out.
```

Fix:

```
Aggregate OpenCTI results before AI.
```

Correct flow:

```
Split Out → OpenCTI Query → Normalize OpenCTI Result → Aggregate OpenCTI Results → AI
```

---

## 13. Operational Recommendations

## 13.1 Do Not Send Every Alert to AI

Send:

```
Wazuh level >= 7 or 8
Suricata IDS alerts
Zeek weird/recon/TLS anomaly alerts
SSH brute-force alerts
FIM high-severity events
UEBA high-risk anomalies
```

Avoid:

```
Every Zeek conn.log event
Every successful login
Low-level system noise
Routine internal traffic
```

## 13.2 Keep AI as Analyst Assistant

AI should:

```
Summarize
Prioritize
Recommend investigation steps
Recommend safe response actions
Create IRIS alert only when appropriate
```

AI should not:

```
Automatically block IPs
Delete files
Isolate hosts
Make destructive changes without human approval
```

## 13.3 Keep Evidence in DFIR-IRIS

Store this in `alert_source_content`:

```
AI triage output
Normalized Wazuh alert
OpenCTI summary
OpenCTI enrichment
IOC list
Rule metadata
Network metadata
MITRE mapping
```

## 13.4 Use Cortex for Analyst-Driven Enrichment

DFIR-IRIS alert/case analysts can run Cortex analyzers from IRIS using the installed Cortex Analyzer module.

Recommended enrichment path:

```
DFIR-IRIS IOC → Cortex Analyzer → MISP Analyzer → MISP intelligence
```

---

## 14. Final Data Flow

```
1. Zeek / Suricata / Linux auth logs are generated on VPN server.
2. Wazuh Agent forwards logs to Wazuh Manager.
3. Wazuh decoders and rules generate alerts.
4. Wazuh custom-n8n integration sends selected alerts to n8n.
5. n8n normalizes the alert.
6. n8n extracts IOCs.
7. n8n queries OpenCTI for IOC intelligence.
8. n8n aggregates OpenCTI results.
9. AI model performs SOC Tier-2 triage.
10. n8n parses AI JSON output.
11. If suspicious, n8n creates DFIR-IRIS alert.
12. n8n sends notification.
13. Analyst reviews DFIR-IRIS alert.
14. Analyst can use IRIS Cortex Analyzer module to run Cortex analyzers.
15. Cortex MISP analyzer queries the MISP VM for additional IOC context.
```

---

## 15. Key Endpoints

Replace placeholders with actual lab values.

```
Wazuh Dashboard:
https://<WAZUH_SERVER_IP>

n8n:
http://<WAZUH_SERVER_IP>:5678

n8n Wazuh Webhook:
http://<WAZUH_SERVER_IP>:5678/webhook/wazuh-alert

DFIR-IRIS:
https://<IRIS_SERVER>:<IRIS_PORT>

DFIR-IRIS Alert API:
https://<IRIS_SERVER>/alerts/add

Cortex:
http://<CORTEX_SERVER>:9001

OpenCTI:
https://<OPENCTI_SERVER>

OpenCTI GraphQL:
https://<OPENCTI_SERVER>/graphql

MISP:
https://<MISP_VM_IP>
```

---

## 16. References

Use the official documentation for installation and authoritative configuration details:

- Wazuh official documentation: installation, Wazuh agent, log collection, custom integrations, Suricata/NIDS integration.
- DFIR-IRIS documentation: Alerts API, API authentication, modules, customers, access control.
- SOCFortress `iris-cortexanalyzer-module`: DFIR-IRIS to Cortex Analyzer integration.
- Cortex documentation: analyzers, observable analysis, API.
- Cortex MISP Analyzer documentation: MISP analyzer configuration and supported observable types.
- OpenCTI documentation: GraphQL API, API token authentication, GraphQL playground.
- n8n documentation: Webhook node, HTTP Request node, workflow activation and production/test webhook behavior.
