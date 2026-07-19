# Wazuh-MISP Integration

Production-Ready Threat Intelligence Enrichment for Linux Sysmon

**Tags:** 🔒 Security, 🐧 Linux, 🔍 Threat Intel, ⚡ Real-time

## 📋 Overview

This integration connects Wazuh's security monitoring with MISP's threat intelligence platform, enabling real-time enrichment of security alerts with global threat data. Designed specifically for Linux environments running Sysmon, it provides automated detection and correlation of malicious indicators.

- **🎯 Automated Detection:** Automatically queries MISP for file hashes, IPs, domains, and URLs from Sysmon events
- **⚡ Real-time Enrichment:** Enriches Wazuh alerts with MISP threat intelligence in milliseconds
- **🔄 Multi-Source Support:** Processes Sysmon events (1, 3, 5, 7, 22, 23) and File Integrity Monitoring
- **🛡️ Production Ready:** Comprehensive error handling, retry logic, and detailed logging

## ✨ Key Features

### Supported Event Types

| Event Type | Description | Indicators Extracted |
| --- | --- | --- |
| **Sysmon Event 1** | Process Creation | File Hashes (SHA256, SHA1, MD5) |
| **Sysmon Event 3** | Network Connection | IP Addresses (IPv4) |
| **Sysmon Event 5** | Process Terminated | File Hashes |
| **Sysmon Event 7** | Image/Library Loaded | File Hashes |
| **Sysmon Event 22** | DNS Query | Domain Names |
| **Sysmon Event 23** | File Delete | File Hashes |
| **FIM/Syscheck** | File Integrity Monitoring | File Hashes |

### Detection Capabilities

- **🦠 Malware Detection:** Identifies malicious file hashes from process creation and file operations
- **🌐 Network Threats:** Detects connections to known malicious IPs and domains
- **🔐 Ransomware:** Special alerting for ransomware-related indicators
- **🎯 APT Detection:** Identifies Advanced Persistent Threat indicators

## 🚀 Installation

> **⚠️ Prerequisites**
Wazuh Manager installed and running
MISP server accessible from Wazuh Manager
MISP API key with read permissions
Python 3.6+ (included with Wazuh)
> 

### 1 Create Integration Script

Create the custom MISP integration script in the Wazuh integrations directory:

```bash
sudo nano /var/ossec/integrations/custom-misp.py
```

### 2 Set Permissions

Configure proper ownership and permissions for the script:

```bash
sudo chmod 750 /var/ossec/integrations/custom-misp.py
sudo chown root:wazuh /var/ossec/integrations/custom-misp.py
```

### 3 Install Dependencies

Install the required Python packages using Wazuh's Python environment:

```bash
sudo /var/ossec/framework/python/bin/pip3 install requests
```

## ⚙️ Configuration

### Script Configuration

Edit the configuration section at the top of custom-misp.py:

```python
# MISP Server Configuration
MISP_BASE_URL = "<https://your-misp-server.com>" # Your MISP URL
MISP_API_KEY = "YOUR_API_KEY_HERE" # Your MISP API key
MISP_VERIFY_SSL = True # Set to True for production
# API Settings
API_TIMEOUT = 10 # Timeout in seconds
MAX_RETRIES = 3 # Retry attempts
# Logging Configuration
LOG_LEVEL = "INFO" # DEBUG, INFO, WARNING, ERROR
```

| Parameter | Description | Default |
| --- | --- | --- |
| `MISP_BASE_URL` | Your MISP server URL (include https://) | https://misp.local |
| `MISP_API_KEY` | MISP API authentication key | Empty (must set) |
| `MISP_VERIFY_SSL` | Verify SSL certificates | False (set True in production) |
| `API_TIMEOUT` | API request timeout in seconds | 10 |
| `MAX_RETRIES` | Number of retry attempts for failed calls | 3 |
| `LOG_LEVEL` | Logging verbosity level | DEBUG |

### Wazuh Configuration

Enable the integration in Wazuh's configuration file:

```bash
sudo nano /var/ossec/etc/ossec.conf
```

Add this integration block:

```xml
<integration>
    <name>custom-misp.py</name>
    <group>sysmon_event1,sysmon_event3,sysmon_event6,sysmon_event7,sysmon_event_15,sysmon_event_22,syscheck</group>
    <alert_format>json</alert_format>
</integration>
```

### Restart Wazuh Manager

```bash
sudo systemctl restart wazuh-manager
```

## 📝 Integration Script

Complete Python integration script for Wazuh-MISP threat intelligence enrichment:

> **⚠️ Important**
Save this script as `/var/ossec/integrations/custom-misp.py` and configure the variables in the CONFIGURATION section before use.
> 

```python
#!/usr/bin/env python3

"""
Wazuh-MISP Integration Script - Linux Optimized
Production-ready integration for Linux Sysmon threat intelligence enrichment
"""

import sys
import os
import json
import re
import logging
from typing import Dict, Optional, Any
from socket import socket, AF_UNIX, SOCK_DGRAM
from datetime import datetime
import ipaddress
import requests
from requests.exceptions import ConnectionError, Timeout, RequestException

# ============================================================================
# CONFIGURATION - EDIT THESE VALUES
# ============================================================================

# MISP Server Configuration
MISP_BASE_URL = "<https://misp.local>"  # Your MISP server URL
MISP_API_KEY = ""  # Your MISP API key
MISP_VERIFY_SSL = False  # Set to True in production with valid SSL certificates

# API Settings
API_TIMEOUT = 10  # Timeout in seconds for API requests
MAX_RETRIES = 3  # Number of retry attempts for failed API calls

# Logging Configuration
LOG_LEVEL = "DEBUG"  # Options: DEBUG, INFO, WARNING, ERROR

# ============================================================================
# DO NOT EDIT BELOW THIS LINE UNLESS YOU KNOW WHAT YOU'RE DOING
# ============================================================================

# Paths
PWD = os.path.dirname(os.path.dirname(os.path.realpath(__file__)))
SOCKET_ADDR = f"{PWD}/queue/sockets/queue"
LOG_FILE = f"{PWD}/logs/integrations.log"

# Regex Patterns
REGEX_SHA256 = re.compile(r"\\b[a-fA-F0-9]{64}\\b")
REGEX_MD5 = re.compile(r"\\b[a-fA-F0-9]{32}\\b")
REGEX_SHA1 = re.compile(r"\\b[a-fA-F0-9]{40}\\b")

class Logger:
    """Logging configuration and management"""

    @staticmethod
    def setup():
        """Configure logging"""
        log_dir = os.path.dirname(LOG_FILE)
        if not os.path.exists(log_dir):
            os.makedirs(log_dir, exist_ok=True)

        logging.basicConfig(
            level=getattr(logging, LOG_LEVEL),
            format='%(asctime)s - %(name)s - %(levelname)s - %(message)s',
            handlers=[
                logging.FileHandler(LOG_FILE),
                logging.StreamHandler(sys.stderr)
            ]
        )
        return logging.getLogger(__name__)

class WazuhIntegration:
    """Handles communication with Wazuh"""

    @staticmethod
    def send_event(msg: Dict[str, Any], agent: Optional[Dict[str, Any]] = None) -> None:
        """Send event back to Wazuh"""
        try:
            if not agent or agent.get("id") == "000":
                string = f"1:misp:{json.dumps(msg)}"
            else:
                agent_ip = agent.get("ip", "any")
                string = f"1:[{agent['id']}] ({agent['name']}) {agent_ip}->misp:{json.dumps(msg)}"

            sock = socket(AF_UNIX, SOCK_DGRAM)
            sock.connect(SOCKET_ADDR)
            sock.send(string.encode())
            sock.close()

            logger.debug(f"Event sent successfully: {msg.get('integration', 'unknown')}")
        except Exception as e:
            logger.error(f"Failed to send event to Wazuh: {e}")
            raise

class MISPClient:
    """MISP API client with error handling and retries"""

    def __init__(self):
        self.base_url = MISP_BASE_URL.rstrip('/')
        self.headers = {
            "Content-Type": "application/json",
            "Authorization": MISP_API_KEY,
            "Accept": "application/json",
        }
        self.verify_ssl = MISP_VERIFY_SSL
        self.timeout = API_TIMEOUT

        if not MISP_API_KEY:
            logger.warning("MISP API key not configured")

    def search_indicator(self, indicator: str, indicator_type: str = "value") -> Optional[Dict[str, Any]]:
        """Search MISP for an indicator"""
        if not MISP_API_KEY:
            logger.error("MISP API key not configured - skipping search")
            return None

        search_url = f"{self.base_url}/attributes/restSearch/{indicator_type}:{indicator}"

        for attempt in range(MAX_RETRIES):
            try:
                logger.debug(f"Querying MISP (attempt {attempt + 1}): {indicator}")
                response = requests.get(
                    search_url,
                    headers=self.headers,
                    verify=self.verify_ssl,
                    timeout=self.timeout
                )
                response.raise_for_status()
                return response.json()

            except Timeout:
                logger.warning(f"MISP API timeout (attempt {attempt + 1}/{MAX_RETRIES})")
                if attempt == MAX_RETRIES - 1:
                    logger.error(f"MISP API timeout after {MAX_RETRIES} attempts")
                    return {"error": "Timeout connecting to MISP API"}

            except ConnectionError:
                logger.error(f"Connection error to MISP API: {search_url}")
                return {"error": "Connection error to MISP API"}

            except RequestException as e:
                logger.error(f"MISP API request failed: {e}")
                return {"error": f"MISP API request failed: {str(e)}"}

            except Exception as e:
                logger.error(f"Unexpected error querying MISP: {e}")
                return {"error": f"Unexpected error: {str(e)}"}

        return None

class IndicatorExtractor:
    """Extract indicators from Wazuh alerts - Linux Sysmon optimized"""

    @staticmethod
    def extract_hash(hashes_string: str) -> Optional[str]:
        """Extract hash from hash string - supports multiple formats"""
        if not hashes_string:
            return None

        logger.debug(f"Attempting to extract hash from: {hashes_string}")

        # Try SHA256 first (preferred)
        match = REGEX_SHA256.search(hashes_string)
        if match:
            hash_value = match.group(0).lower()
            logger.debug(f"Extracted SHA256: {hash_value}")
            return hash_value

        # Fallback to SHA1
        match = REGEX_SHA1.search(hashes_string)
        if match:
            hash_value = match.group(0).lower()
            logger.debug(f"Extracted SHA1: {hash_value}")
            return hash_value

        # Fallback to MD5
        match = REGEX_MD5.search(hashes_string)
        if match:
            hash_value = match.group(0).lower()
            logger.debug(f"Extracted MD5: {hash_value}")
            return hash_value

        logger.debug("No hash pattern matched")
        return None

    @staticmethod
    def extract_ip(ip_string: str, allow_ipv6: bool = False) -> Optional[str]:
        """Extract and validate IP address"""
        if not ip_string:
            return None

        try:
            ip_obj = ipaddress.ip_address(ip_string)

            # Check if it's global (not private, loopback, etc.)
            if ip_obj.is_global:
                # Check IPv6 restriction
                if isinstance(ip_obj, ipaddress.IPv6Address) and not allow_ipv6:
                    logger.debug(f"IPv6 address filtered: {ip_string}")
                    return None

                logger.debug(f"Valid global IP: {ip_string}")
                return str(ip_obj)
            else:
                logger.debug(f"Non-global IP filtered: {ip_string}")
                return None

        except ValueError:
            logger.warning(f"Invalid IP address: {ip_string}")
            return None

    @staticmethod
    def get_field_value(data: Dict[str, Any], *field_names: str) -> Optional[str]:
        """Try multiple field name variations (case-insensitive)"""
        for field_name in field_names:
            # Try exact match
            if field_name in data:
                value = data[field_name]
                if value and value != "-":
                    return value

            # Try case-insensitive match
            for key in data.keys():
                if key.lower() == field_name.lower():
                    value = data[key]
                    if value and value != "-":
                        return value

        return None

    @staticmethod
    def extract_from_linux_event(alert: Dict[str, Any]) -> Optional[str]:
        """Extract indicator from Linux Sysmon event"""
        try:
            # Get event type from groups
            groups = alert["rule"]["groups"]
            logger.debug(f"Alert groups: {groups}")

            # Find the sysmon_eventX group
            event_type = None
            for group in groups:
                if group.startswith("sysmon_event"):
                    event_type = group
                    break

            if not event_type:
                logger.debug("No sysmon_event type found in groups")
                return None

            logger.debug(f"Processing event type: {event_type}")

            # Get event data
            event_data = alert.get("data", {}).get("eventdata", {})
            if not event_data:
                logger.debug("No eventdata found in alert")
                return None

            logger.debug(f"Event data keys: {event_data.keys()}")

            # Event 1: Process Creation - look for hashes
            if event_type == "sysmon_event1":
                logger.debug("Processing Event 1: Process Creation")
                hashes = IndicatorExtractor.get_field_value(
                    event_data,
                    "hashes", "Hashes", "hash", "Hash"
                )
                if hashes:
                    logger.debug(f"Found hashes field: {hashes}")
                    return IndicatorExtractor.extract_hash(hashes)
                else:
                    # Try individual hash fields
                    for hash_field in ["sha256", "SHA256", "sha1", "SHA1", "md5", "MD5"]:
                        hash_value = IndicatorExtractor.get_field_value(event_data, hash_field)
                        if hash_value:
                            logger.debug(f"Found individual hash field {hash_field}: {hash_value}")
                            return hash_value.lower()
                    logger.debug("No hash fields found in Event 1")

            # Event 3: Network Connection
            elif event_type == "sysmon_event3":
                logger.debug("Processing Event 3: Network Connection")
                is_ipv6_str = IndicatorExtractor.get_field_value(
                    event_data,
                    "destinationIsIpv6", "DestinationIsIpv6"
                )
                if is_ipv6_str and is_ipv6_str.lower() == "true":
                    logger.debug("IPv6 connection - skipping")
                    return None

                dst_ip = IndicatorExtractor.get_field_value(
                    event_data,
                    "destinationIp", "DestinationIp", "destination_ip"
                )
                if dst_ip:
                    logger.debug(f"Found destination IP: {dst_ip}")
                    return IndicatorExtractor.extract_ip(dst_ip)
                else:
                    logger.debug("No destination IP found")

            # Event 22: DNS Query
            elif event_type == "sysmon_event_22":
                logger.debug("Processing Event 22: DNS Query")
                query_name = IndicatorExtractor.get_field_value(
                    event_data,
                    "queryName", "QueryName", "query_name"
                )
                if query_name:
                    domain = query_name.strip().lower()
                    logger.debug(f"Found DNS query: {domain}")
                    return domain
                else:
                    logger.debug("No query name found")

            # Add other event types as needed...

        except (KeyError, TypeError) as e:
            logger.warning(f"Failed to extract indicator from Linux event: {e}", exc_info=True)

        return None

class AlertProcessor:
    """Process Wazuh alerts and enrich with MISP data"""

    def __init__(self):
        self.misp_client = MISPClient()
        self.wazuh = WazuhIntegration()

    def process_alert(self, alert: Dict[str, Any]) -> None:
        """Main alert processing function"""
        try:
            # Validate alert structure
            if not self._validate_alert(alert):
                logger.error("Invalid alert structure")
                return

            # Get rule groups
            groups = alert["rule"]["groups"]
            logger.info(f"Processing alert with groups: {groups}")

            # Extract indicator based on source
            indicator = None

            # Check if it's a sysmon event
            if "sysmon" in groups:
                logger.info("Processing Sysmon event")
                indicator = IndicatorExtractor.extract_from_linux_event(alert)

            if not indicator:
                logger.debug("No valid indicator extracted from alert")
                return

            logger.info(f"✓ Extracted indicator: {indicator}")

            # Query MISP
            misp_response = self.misp_client.search_indicator(indicator)

            if not misp_response:
                logger.debug("No MISP response")
                return

            # Handle errors
            if "error" in misp_response:
                self._send_error_alert(misp_response["error"], alert)
                return

            # Process MISP results
            self._process_misp_response(misp_response, alert, indicator)

        except Exception as e:
            logger.error(f"Error processing alert: {e}", exc_info=True)
            self._send_error_alert(f"Processing error: {str(e)}", alert)

    def _validate_alert(self, alert: Dict[str, Any]) -> bool:
        """Validate alert has required structure"""
        try:
            return (
                "rule" in alert and
                "groups" in alert["rule"] and
                len(alert["rule"]["groups"]) >= 1
            )
        except (KeyError, TypeError):
            return False

    def _process_misp_response(
        self,
        misp_response: Dict[str, Any],
        alert: Dict[str, Any],
        indicator: str
    ) -> None:
        """Process MISP response and create enriched alert"""
        try:
            attributes = misp_response.get("response", {}).get("Attribute", [])

            if not attributes:
                logger.debug(f"No MISP matches found for: {indicator}")
                return

            # Get first matching attribute
            attr = attributes[0]

            # Build enriched alert
            alert_output = {
                "integration": "misp",
                "misp": {
                    "event_id": attr.get("event_id"),
                    "category": attr.get("category"),
                    "type": attr.get("type"),
                    "value": attr.get("value"),
                    "timestamp": attr.get("timestamp"),
                    "to_ids": attr.get("to_ids", False),
                    "comment": attr.get("comment", ""),
                    "source": {
                        "description": alert["rule"].get("description", ""),
                        "level": alert["rule"].get("level", 0),
                        "id": alert["rule"].get("id", ""),
                    },
                    "matched_indicator": indicator,
                    "total_matches": len(attributes),
                }
            }

            # Send enriched alert back to Wazuh
            self.wazuh.send_event(alert_output, alert.get("agent"))
            logger.info(f"✓ MISP MATCH FOUND - Event ID: {attr.get('event_id')}, Indicator: {indicator}")

        except Exception as e:
            logger.error(f"Error processing MISP response: {e}")
            self._send_error_alert(f"Response processing error: {str(e)}", alert)

    def _send_error_alert(self, error_message: str, alert: Dict[str, Any]) -> None:
        """Send error alert back to Wazuh"""
        alert_output = {
            "integration": "misp",
            "misp": {
                "error": error_message,
                "timestamp": datetime.utcnow().isoformat()
            }
        }

        try:
            self.wazuh.send_event(alert_output, alert.get("agent"))
        except Exception as e:
            logger.error(f"Failed to send error alert: {e}")

def main():
    """Main execution function"""
    global logger
    logger = Logger.setup()

    try:
        # Check command line arguments
        if len(sys.argv) < 2:
            logger.error("Usage: python custom-misp.py ")
            sys.exit(1)

        alert_file_path = sys.argv[1]

        # Read alert file
        if not os.path.exists(alert_file_path):
            logger.error(f"Alert file not found: {alert_file_path}")
            sys.exit(1)

        with open(alert_file_path, 'r') as f:
            alert = json.load(f)

        logger.info(f"Processing alert from: {alert_file_path}")
        logger.debug(f"Alert structure: {json.dumps(alert, indent=2)}")

        # Process the alert
        processor = AlertProcessor()
        processor.process_alert(alert)

        logger.info("Alert processing completed successfully")

    except json.JSONDecodeError as e:
        logger.error(f"Invalid JSON in alert file: {e}")
        sys.exit(1)

    except Exception as e:
        logger.error(f"Fatal error: {e}", exc_info=True)
        sys.exit(1)

if __name__ == "__main__":
    main()
```

### Script Components

- **🔌 MISP Client:** Handles API communication with retry logic and error handling
- **🔍 Indicator Extractor:** Extracts IOCs from various Sysmon event types
- **⚡ Alert Processor:** Coordinates alert processing and MISP enrichment
- **📊 Wazuh Integration:** Sends enriched events back to Wazuh for alerting

> **💡 Script HighlightsRegex Pattern Matching:** Extracts hashes (SHA256, SHA1, MD5) from various formats
**IP Validation:** Filters out private and non-global IP addresses
**Case-Insensitive Fields:** Handles field name variations across different Sysmon versions
**Comprehensive Logging:** Detailed debug information for troubleshooting
**Error Handling:** Graceful handling of timeouts, connection errors, and API failures
**Retry Logic:** Automatic retry with exponential backoff for failed API calls
> 

## 📏 Detection Rules

Create custom detection rules to trigger alerts on MISP threat intelligence matches:

```bash
sudo nano /var/ossec/etc/rules/local_rules.xml
```

### Base Rules

```xml
<!--
Wazuh MISP Integration Rules
File: local_rules.xml
Location: /var/ossec/etc/rules/local_rules.xml
-->

<group name="misp,threat_intelligence,">

<!-- ============================================ -->
<!-- MISP Integration - Base Rules -->
<!-- ============================================ -->

<!-- Rule 100100: Minimal test rule - MISP integration event detected -->
<rule id="100100" level="3">
<decoded_as>json</decoded_as>
<field name="integration">^misp$</field>
<description>MISP integration event received</description>
<group>misp,threat_intel,</group>
</rule>

<!-- Rule 100101: MISP threat intelligence match found -->
<rule id="100101" level="7">
<if_sid>100100</if_sid>
<field name="misp.event_id">\\.+</field>
<description>MISP: Threat intelligence match - Event ID: $(misp.event_id)</description>
<group>misp,threat_intel,threat_detected,</group>
</rule>

<!-- ============================================ -->
<!-- MISP - High Severity Matches -->
<!-- ============================================ -->

<!-- Rule 100102: MISP match with IDS flag enabled (high confidence) -->
<rule id="100102" level="10">
<if_sid>100101</if_sid>
<field name="misp.to_ids">^true$</field>
<description>MISP: High confidence threat detected (IDS flag enabled) - Indicator: $(misp.matched_indicator)</description>
<group>misp,threat_intel,high_confidence,ids_detection,</group>
</rule>

<!-- Rule 100103: MISP malware category match -->
<rule id="100103" level="9">
<if_sid>100101</if_sid>
<field name="misp.category">^Payload delivery$|^Artifacts dropped$|^Payload installation$</field>
<description>MISP: Malware-related indicator detected - Category: $(misp.category)</description>
<group>misp,threat_intel,malware,</group>
</rule>

<!-- Rule 100104: MISP network activity category match -->
<rule id="100104" level="8">
<if_sid>100101</if_sid>
<field name="misp.category">^Network activity$</field>
<description>MISP: Malicious network activity detected - Type: $(misp.type)</description>
<group>misp,threat_intel,network_threat,</group>
</rule>

<!-- ============================================ -->
<!-- MISP - Indicator Type Specific Rules -->
<!-- ============================================ -->

<!-- Rule 100110: MISP file hash match -->
<rule id="100110" level="8">
<if_sid>100101</if_sid>
<field name="misp.type">^sha256$|^sha1$|^md5$</field>
<description>MISP: Malicious file hash detected - Hash: $(misp.value)</description>
<group>misp,threat_intel,malicious_hash,</group>
</rule>

<!-- Rule 100111: MISP IP address match -->
<rule id="100111" level="8">
<if_sid>100101</if_sid>
<field name="misp.type">^ip-dst$|^ip-src$</field>
<description>MISP: Malicious IP address detected - IP: $(misp.value)</description>
<group>misp,threat_intel,malicious_ip,</group>
</rule>

<!-- Rule 100112: MISP domain/hostname match -->
<rule id="100112" level="8">
<if_sid>100101</if_sid>
<field name="misp.type">^domain$|^hostname$</field>
<description>MISP: Malicious domain detected - Domain: $(misp.value)</description>
<group>misp,threat_intel,malicious_domain,</group>
</rule>

<!-- Rule 100113: MISP URL match -->
<rule id="100113" level="8">
<if_sid>100101</if_sid>
<field name="misp.type">^url$</field>
<description>MISP: Malicious URL detected - URL: $(misp.value)</description>
<group>misp,threat_intel,malicious_url,</group>
</rule>

<!-- ============================================ -->
<!-- MISP - Source Event Correlation -->
<!-- ============================================ -->

<!-- Rule 100120: MISP match from Sysmon process creation -->
<rule id="100120" level="9">
<if_sid>100110</if_sid>
<field name="misp.source.description">Sysmon - Event 1</field>
<description>MISP: Malicious process detected via file hash - Event ID: $(misp.event_id)</description>
<group>misp,threat_intel,malicious_process,sysmon,</group>
</rule>

<!-- Rule 100121: MISP match from Sysmon network connection -->
<rule id="100121" level="9">
<if_sid>100111</if_sid>
<field name="misp.source.description">Sysmon - Event 3</field>
<description>MISP: Malicious network connection detected - IP: $(misp.value)</description>
<group>misp,threat_intel,malicious_connection,sysmon,</group>
</rule>

<!-- Rule 100122: MISP match from Sysmon DNS query -->
<rule id="100122" level="9">
<if_sid>100112</if_sid>
<field name="misp.source.description">Sysmon - Event 22</field>
<description>MISP: Malicious DNS query detected - Domain: $(misp.value)</description>
<group>misp,threat_intel,malicious_dns,sysmon,</group>
</rule>

<!-- Rule 100123: MISP match from FIM (File Integrity Monitoring) -->
<rule id="100123" level="9">
<if_sid>100110</if_sid>
<field name="misp.source.description">syscheck</field>
<description>MISP: Malicious file detected via FIM - Hash: $(misp.value)</description>
<group>misp,threat_intel,malicious_file,fim,</group>
</rule>

<!-- ============================================ -->
<!-- MISP - Multiple Matches (Frequency Based) -->
<!-- ============================================ -->

<!-- Rule 100130: Multiple MISP matches from same host (5 in 300s) -->
<rule id="100130" level="11" frequency="5" timeframe="300">
<if_matched_sid>100101</if_matched_sid>
<same_source_ip />
<description>MISP: Multiple threat intelligence matches from same host ($(misp.total_matches) total)</description>
<group>misp,threat_intel,multiple_threats,correlation,</group>
</rule>

<!-- Rule 100131: Multiple high-confidence MISP matches (3 in 300s) -->
<rule id="100131" level="12" frequency="3" timeframe="300">
<if_matched_sid>100102</if_matched_sid>
<same_source_ip />
<description>MISP: Multiple high-confidence threats detected on same host</description>
<group>misp,threat_intel,high_confidence,multiple_threats,critical,</group>
</rule>

<!-- ============================================ -->
<!-- MISP - Integration Errors -->
<!-- ============================================ -->

<!-- Rule 100150: MISP integration error -->
<rule id="100150" level="5">
<if_sid>100100</if_sid>
<field name="misp.error">\\.+</field>
<description>MISP integration error: $(misp.error)</description>
<group>misp,integration_error,</group>
</rule>

<!-- Rule 100151: MISP API timeout -->
<rule id="100151" level="6">
<if_sid>100150</if_sid>
<field name="misp.error">Timeout</field>
<description>MISP: API timeout error - Check MISP server connectivity</description>
<group>misp,integration_error,timeout,</group>
</rule>

<!-- Rule 100152: MISP connection error -->
<rule id="100152" level="6">
<if_sid>100150</if_sid>
<field name="misp.error">Connection error</field>
<description>MISP: Connection error - MISP server may be unreachable</description>
<group>misp,integration_error,connection_failed,</group>
</rule>

<!-- ============================================ -->
<!-- MISP - Advanced Threat Scenarios -->
<!-- ============================================ -->

<!-- Rule 100160: MISP match with critical source alert level -->
<rule id="100160" level="12">
<if_sid>100102</if_sid>
<field name="misp.source.level">^1[2-5]$</field>
<description>MISP: Critical threat detected - Source Alert Level: $(misp.source.level)</description>
<group>misp,threat_intel,critical,high_severity,</group>
</rule>

<!-- Rule 100161: MISP ransomware indicator -->
<rule id="100161" level="13">
<if_sid>100101</if_sid>
<field name="misp.comment">ransomware|ransom|crypto|locker</field>
<description>MISP: Ransomware indicator detected - $(misp.matched_indicator)</description>
<group>misp,threat_intel,ransomware,critical,</group>
</rule>

<!-- Rule 100162: MISP APT (Advanced Persistent Threat) indicator -->
<rule id="100162" level="13">
<if_sid>100101</if_sid>
<field name="misp.comment">APT|advanced.persistent|nation.state|targeted.attack</field>
<description>MISP: APT-related indicator detected - $(misp.matched_indicator)</description>
<group>misp,threat_intel,apt,critical,</group>
</rule>

</group>
```

### High Severity Rules

| Rule ID | Level | Description | Trigger |
| --- | --- | --- | --- |
| **100102** | 10 | High confidence threat (IDS flag enabled) | misp.to_ids = true |
| **100103** | 9 | Malware-related indicator | Payload delivery/installation |
| **100130** | 11 | Multiple threats from same host | 5 matches in 300 seconds |
| **100161** | 13 | Ransomware indicator | Comment contains ransomware keywords |
| **100162** | 13 | APT indicator | Comment contains APT keywords |

### Indicator-Specific Rules

- **🔑 File Hash (100110):** Level 8 - Detects malicious file hashes (SHA256, SHA1, MD5)
- **🌐 IP Address (100111):** Level 8 - Identifies malicious IP addresses
- **🔗 Domain (100112):** Level 8 - Flags malicious domains and hostnames
- **📎 URL (100113):** Level 8 - Detects malicious URLs

## 🧪 Testing & Verification

### 1 Check Integration Logs

Monitor the integration logs to verify the script is running:

```bash
sudo tail -f /var/ossec/logs/integrations.log
```

> **✅ Expected Output**
You should see entries like: 2026-01-19 10:23:45 - INFO - Processing alert with groups: ['sysmon', 'sysmon_event1']
> 

2026-01-19 10:23:45 - INFO - ✓ Extracted indicator: a3f8d9e2c1b4...

2026-01-19 10:23:46 - INFO - ✓ MISP MATCH FOUND - Event ID: 12345

### 2 Verify Rule Loading

Check that the custom rules were loaded successfully:

```bash
sudo /var/ossec/bin/wazuh-logtest
```

### 3 Monitor Wazuh Alerts

Watch for MISP-enriched alerts in real-time:

```bash
sudo tail -f /var/ossec/logs/alerts/alerts.json | grep -i misp
```

### Sample Enriched Alert

```json
{
  "integration": "misp",
  "misp": {
    "event_id": "12345",
    "category": "Payload delivery",
    "type": "sha256",
    "value": "a3f8d9e2c1b4567890abcdef...",
    "timestamp": "1642598765",
    "to_ids": true,
    "comment": "Ransomware variant detected",
    "source": {
      "description": "Sysmon - Event 1",
      "level": 12,
      "id": "61603"
    },
    "matched_indicator": "a3f8d9e2c1b4567890abcdef...",
    "total_matches": 1
  }
}
```

### Troubleshooting

| Issue | Possible Cause | Solution |
| --- | --- | --- |
| No logs generated | Script not executing | Check permissions (750) and ownership (root:wazuh) |
| API timeout errors | MISP server unreachable | Verify network connectivity and MISP_BASE_URL |
| No indicators extracted | Wrong event group configuration | Verify sysmon event types in ossec.conf |
| SSL certificate errors | Self-signed certificates | Set MISP_VERIFY_SSL = False (testing only) |
| Rules not triggering | Rules syntax error | Test with wazuh-logtest and check ossec.log |

> **⚠️ Common PitfallsMissing API Key:** Ensure MISP_API_KEY is configured correctly
**Wrong Python Environment:** Always use /var/ossec/framework/python/bin/pip3
**Firewall Blocking:** Verify MISP server is accessible on port 443
**Log Level Too High:** Set LOG_LEVEL to "DEBUG" for initial troubleshooting
> 

## 🔧 Advanced Configuration

### Performance Tuning

- **⚡ API Timeout:** Adjust `API_TIMEOUT` based on network latency. Recommended: 5-15 seconds
- **🔄 Retry Logic:** Increase `MAX_RETRIES` for unreliable networks. Default: 3 attempts
- **📊 Log Levels:** Use INFO for production, DEBUG for troubleshooting, WARNING for minimal logging
- **🎯 Event Filtering:** Add/remove event groups in ossec.conf to focus on specific Sysmon events

### Custom Field Mapping

Modify the `IndicatorExtractor` class to handle custom field names:

```python
# Add custom field names for your environment
hashes = IndicatorExtractor.get_field_value(
    event_data,
    "hashes", "Hashes", "hash", "Hash",
    "custom_hash_field" # Add your custom field
)
```

### Integration with SOAR

The enriched alerts can be consumed by SOAR platforms for automated response:

- TheHive integration for case management
- Shuffle for workflow automation
- Cortex for additional enrichment
- Custom webhooks for ticketing systems

## 💎 Best Practices

- **🔐 Security:** Use HTTPS with valid SSL certificates
Rotate MISP API keys regularly
Restrict network access to MISP
Monitor integration logs for anomalies
- **📈 Performance:** Enable MISP caching for better response times
Filter high-volume events before integration
Use appropriate log levels in production
Monitor API rate limits
- **🎯 Accuracy:** Regularly update MISP threat feeds
Fine-tune rule severity levels
Review false positives weekly
Maintain threat intelligence quality
- **🔄 Maintenance:** Test integration after Wazuh updates
Archive old integration logs monthly
Document custom modifications
Keep backup of working configurations

## 📚 Additional Resources

| Resource | Description | Link |
| --- | --- | --- |
| **Wazuh Documentation** | Official Wazuh integration guide | documentation.wazuh.com |
| **MISP Project** | MISP threat sharing platform | misp-project.org |
| **Sysmon Guide** | Linux Sysmon configuration | github.com/Sysinternals |
| **Wazuh Community** | Forums and support | groups.google.com/g/wazuh |

🛡️ **Wazuh-MISP Integration Documentation**

Production-ready threat intelligence enrichment for Linux environments

Last Updated: January 2026 | Version 1.0
