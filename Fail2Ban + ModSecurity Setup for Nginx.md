## Install First:

```
sudo apt update
sudo apt install fail2ban
```

You can verify this by using the `systemctl` command:
`systemctl status fail2ban.service`

Output:

```
○ fail2ban.service - Fail2Ban Service
     Loaded: loaded (/lib/systemd/system/fail2ban.service; disabled; vendor preset: enabled
     Active: inactive (dead)
       Docs: man:fail2ban(1) ....
```

---

## STEP 1: Create Fail2Ban Filter for ModSecurity

Create a new filter file:

```bash
sudo nano /etc/fail2ban/filter.d/nginx-modsecurity.conf
```

Paste exactly this:

```
[Definition]
failregex = ^.*\[client <HOST>\] ModSecurity: Access denied with code 403.*
ignoreregex =
```

Your log line looks like:
`[client 192.168.18.247] ModSecurity: Access denied with code 403`

`<HOST>` automatically extracts the IP for banning. Save and exit.

---

## STEP 2: Create a Fail2Ban Jail for ModSecurity

Edit the jail file:
`sudo nano /etc/fail2ban/jail.local`

Add this block (do not remove the sshd jail):

```
[nginx-modsecurity]
enabled  = true
filter   = nginx-modsecurity
logpath  = /var/log/nginx/wafcontrol_error.log
backend  = auto

maxretry = 3
findtime = 60s
bantime  = 12h

action   = iptables-multiport[name=nginx-modsec, port="80,443", protocol=tcp]
```

Meaning of key settings:

| Setting | Meaning |
| --- | --- |
| maxretry | 3 WAF hits |
| findtime | within 1 minute |
| bantime | ban for 12 hours |
| ports | block HTTP/HTTPS |

---

## STEP 3: Reload Fail2Ban

`sudo systemctl restart fail2ban`

Verify jail is active:
`sudo fail2ban-client status`

Expected output:
`Jail list: sshd, nginx-modsecurity`

---

## STEP 4: Test the Jail

Use a test IP or be ready to unban yourself. From a client:

```
curl "http://192.168.18.120/?file=../../../../etc/passwd"
curl "http://192.168.18.120/?file=../../../../etc/passwd"
curl "http://192.168.18.120/?file=../../../../etc/passwd"
```

Check banned IPs:
`sudo fail2ban-client status nginx-modsecurity`

Expected output:
`Banned IP list: 192.168.18.247`

## STEP 5: Unban Your IP

Unban specific IP:
`sudo fail2ban-client unban 192.168.18.247`

Or clear all bans:
`sudo fail2ban-client unban --all`

---

# Important Security Note

Your WAF log may include your own IP (e.g., 192.168.18.247).

If that IP is a shared gateway/NAT, banning it will block everyone.

Recommended approach: whitelist internal/NAT IPs:
`ignoreip = 127.0.0.1/8 ::1 192.168.18.247`
