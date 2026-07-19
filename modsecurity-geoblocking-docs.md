**🔒 Security Guide**

# Block Traffic *Outside Bangladesh* with ModSecurity

A complete step-by-step guide to configure ModSecurity GeoIP-based blocking on Ubuntu, restricting access exclusively to Bangladeshi IP addresses.

**Environment:**

- Ubuntu LTS
- Apache / Nginx
- MaxMind GeoLite2
- Country Code: BD

## Overview & Prerequisites

What you'll need before starting

This guide walks you through configuring **ModSecurity** (Web Application Firewall) to allow only traffic from **Bangladesh (BD)** and block all other countries using MaxMind's GeoLite2 database.

- **Web Server:** Apache 2.4+ or Nginx
- **ModSecurity:** v2.9+ or v3.x
- **GeoIP Database:** MaxMind GeoLite2
- **Country Code:** BD — Bangladesh

> **Before You Begin**
Make sure ModSecurity is installed and active on your server. Run `sudo apachectl -M | grep security` to confirm. You'll also need a free MaxMind account.
> 
- ✅ Ubuntu 20.04 / 22.04 LTS with root or sudo access
- ✅ Apache2 or Nginx already installed
- ✅ ModSecurity installed and enabled
- ✅ Internet access to download MaxMind GeoLite2 database
- ✅ Free MaxMind account for database download

## Install GeoIP Packages

Install MaxMind database libraries and tools

Install the required GeoIP library packages using `apt`. These provide the backend MaxMind database reader that ModSecurity will use to resolve IP addresses to country codes.

```bash
# Update package lists
sudo apt-get update

# Install MaxMind GeoIP libraries
sudo apt-get install -y \\
  libmaxminddb0 \\
  libmaxminddb-dev \\
  mmdb-bin \\
  geoipupdate
```

```bash
# Install on CentOS/RHEL
sudo yum install -y \\
  libmaxminddb \\
  libmaxminddb-devel \\
  GeoIP \\
  GeoIP-update
```

> **Verify Installation**
After installing, run `mmdblookup --version` to confirm the MaxMind DB lookup tool is available.
> 

## Download GeoLite2 Database

Get the MaxMind country database file

You need a free MaxMind account to download the GeoLite2 database. Sign up at **maxmind.com**, then generate a license key from your account dashboard.

### Option A — Using geoipupdate (Recommended)

```bash
# Edit the geoipupdate config
sudo nano /etc/GeoIP.conf

# Add your MaxMind account details:
AccountID YOUR_ACCOUNT_ID
LicenseKey YOUR_LICENSE_KEY
EditionIDs GeoLite2-Country
# Run the updater
sudo geoipupdate

# The database will be saved to:
# /usr/share/GeoIP/GeoLite2-Country.mmdb
```

### Option B — Manual Download

```bash
# Create the GeoIP directory
sudo mkdir -p /usr/share/GeoIP

# Download manually using your license key
curl -o /tmp/GeoLite2-Country.tar.gz \\
  "<https://download.maxmind.com/app/geoip_download?edition_id=GeoLite2-Country&license_key=YOUR_KEY&suffix=tar.gz>"
# Extract and move to the GeoIP directory
tar -xzf /tmp/GeoLite2-Country.tar.gz -C /tmp/
sudo mv /tmp/GeoLite2-Country_*/GeoLite2-Country.mmdb /usr/share/GeoIP/

# Verify the file is in place
ls -la /usr/share/GeoIP/GeoLite2-Country.mmdb
```

> **Test the Database**
Run `mmdblookup --file /usr/share/GeoIP/GeoLite2-Country.mmdb --ip 1.1.1.1` to test the lookup. You should see country data returned.
> 

## Configure ModSecurity

Enable GeoIP lookups in modsecurity.conf

Edit the main ModSecurity configuration file to register the GeoLite2 database path. This enables the `GEO` variable collection used in blocking rules.

```bash
# Open the ModSecurity configuration file
sudo nano /etc/modsecurity/modsecurity.conf
```

Add the following directive to register the GeoIP database:

```
# ── GeoIP Database Configuration ───────────────────────
# Tell ModSecurity where to find the MaxMind GeoLite2 DB
SecGeoLookupDB /usr/share/GeoIP/GeoLite2-Country.mmdb
# Also ensure SecRuleEngine is set to On (not DetectionOnly)
SecRuleEngine On
```

> **Important:** Make sure `SecRuleEngine` is set to `On` and not `DetectionOnly`. In DetectionOnly mode, rules will log but not block traffic.
> 

## Create the Geo-Blocking Rule

Write the ModSecurity rule to block non-BD traffic

Create a dedicated custom rules file to keep your geo-blocking rules organized and separate from the core CRS rules.

```bash
# Create the custom rules file
sudo nano /etc/modsecurity/rules/custom-geoblocking.conf
```

Paste the following rules into the file. Two approaches are provided — use either one:

### Approach A — Chain Rule (Recommended)

```
# ─────────────────────────────────────────────────────────────
# GeoIP Blocking — Allow Bangladesh (BD) only
# Rule ID: 1000
# ─────────────────────────────────────────────────────────────
# First: Perform the GeoIP lookup on the remote IP
SecRule REMOTE_ADDR "@geoLookup" \\
    "id:1000,\\
    phase:1,\\
    chain,\\
    drop,\\
    msg:'GeoIP Block - Traffic outside Bangladesh',\\
    logdata:'Blocked Country: %{GEO.COUNTRY_CODE}',\\
    tag:'geo-blocking'"
SecRule GEO:COUNTRY_CODE "!@streq BD" \\
        "t:none"
```

### Approach B — Simple Single Rule

```
# Simpler version — deny with 403 status
SecRule GEO:COUNTRY_CODE "!@streq BD" \\
    "id:1000,\\
    phase:1,\\
    deny,\\
    status:403,\\
    msg:'Access Denied - Country not permitted',\\
    logdata:'Blocked Country Code: %{GEO.COUNTRY_CODE}',\\
    log,\\
    auditlog"
```

> **Rule Breakdown**`!@streq BD` means "not equal to BD". `phase:1` runs the check early in the request cycle. `drop` silently drops the connection (better than `deny` for blocking bots).
> 

## Include Custom Rules

Load your geo-blocking rule file in the server config

Include the custom rules file in your web server's ModSecurity configuration. Choose the tab for your server:

```
# Edit your Apache site config or modsec include file
sudo nano /etc/apache2/conf-available/modsecurity.conf

# Add this line at the end of the file
Include /etc/modsecurity/rules/custom-geoblocking.conf
# Or include the entire rules directory
IncludeOptional /etc/modsecurity/rules/*.conf
```

```
# Edit your Nginx server block
sudo nano /etc/nginx/sites-available/your-site.conf

# Inside the server {} block, add:
modsecurity on;
modsecurity_rules_file /etc/modsecurity/modsecurity.conf;
modsecurity_rules_file /etc/modsecurity/rules/custom-geoblocking.conf;
```

## Test Config & Restart

Validate and apply the new configuration

Always test your configuration before restarting to avoid taking the server offline with a syntax error.

```bash
# Test Apache configuration for syntax errors
sudo apachectl configtest

# Expected output: Syntax OK
# Restart Apache to apply changes
sudo systemctl restart apache2

# Verify Apache is running
sudo systemctl status apache2
```

```bash
# Test Nginx configuration for syntax errors
sudo nginx -t

# Expected output: nginx: configuration file test is successful
# Restart Nginx to apply changes
sudo systemctl restart nginx

# Verify Nginx is running
sudo systemctl status nginx
```

> **Warning:** If you're connected via SSH from outside Bangladesh, this rule will block your SSH/web access too. Whitelist your IP first (Step 7) before enabling the block rule!
> 

## Whitelist Specific IPs

Allow trusted IPs regardless of location

If you or your team access the server from outside Bangladesh, whitelist those IPs **before** the geo-blocking rule. ModSecurity processes rules in order, so whitelist rules must have a *lower* rule ID or appear earlier in the file.

```
# ─────────────────────────────────────────────────────────────
# Whitelist — Place this BEFORE the geo-blocking rule (ID < 1000)
# ─────────────────────────────────────────────────────────────
# Allow a single IP address
SecRule REMOTE_ADDR "@ipMatch 203.0.113.45" \\
    "id:999,\\
    phase:1,\\
    allow,\\
    msg:'Whitelisted IP - Trusted Admin'"
# Allow multiple IPs
SecRule REMOTE_ADDR "@ipMatch 203.0.113.45,198.51.100.10,192.0.2.5" \\
    "id:998,\\
    phase:1,\\
    allow,\\
    msg:'Whitelisted IPs - Admin Team'"
# Allow an IP range / subnet (CIDR notation)
SecRule REMOTE_ADDR "@ipMatch 10.0.0.0/8" \\
    "id:997,\\
    phase:1,\\
    allow,\\
    msg:'Whitelisted Subnet - Internal Network'"
```

> **Find Your Public IP**
Run `curl ifconfig.me` or `curl icanhazip.com` from your local machine to get your current public IP address.
> 

## Testing & Monitoring Logs

Verify the rules are working correctly

After restarting, monitor the ModSecurity audit log to confirm that blocks are being triggered correctly.

### Monitor Audit Logs in Real-Time

```bash
# Follow ModSecurity audit log live
sudo tail -f /var/log/apache2/modsec_audit.log

# Or for Nginx
sudo tail -f /var/log/nginx/modsec_audit.log

# Filter for GeoIP blocks only
sudo grep "GeoIP Block" /var/log/apache2/modsec_audit.log

# See error log for denied requests
sudo tail -f /var/log/apache2/error.log | grep "ModSecurity"
```

### Test GeoIP Lookup Manually

```bash
# Test a Bangladeshi IP (should return BD)
mmdblookup --file /usr/share/GeoIP/GeoLite2-Country.mmdb \\
  --ip 103.231.175.1 country iso_code

# Test a non-BD IP (should be blocked)
mmdblookup --file /usr/share/GeoIP/GeoLite2-Country.mmdb \\
  --ip 8.8.8.8 country iso_code

# Expected output format:
#   "US" <utf8_string>   ← This IP would be blocked
#   "BD" <utf8_string>   ← This IP would be allowed
```

### Set Up Automatic Database Updates

```bash
# Add weekly auto-update cron job for the GeoIP database
sudo crontab -e

# Add this line (runs every Sunday at 3:00 AM):
0 3 * * 0 /usr/bin/geoipupdate && systemctl reload apache2
```

> **You're Done!**
Your server will now only accept traffic from Bangladesh (country code `BD`). All other countries will be silently dropped or receive a 403 Forbidden response depending on which rule approach you used.
> 
- **Blocked Action:** drop / deny (403)
- **Allowed Country:** Bangladesh (BD)
- **Rule Phase:** Phase 1 (Request Headers)
- **Database:** GeoLite2-Country.mmdb
