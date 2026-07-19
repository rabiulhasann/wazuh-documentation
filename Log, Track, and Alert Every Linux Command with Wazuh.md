## Table of Contents

- 4.1. Linux Configuration
- 4.2. Wazuh Configuration
    1. Checking Results in Wazuh Dashboard
    2. Creating Alarm Rule

## 4.1. Linux Configuration

There are several ways to log all user commands to Wazuh. In this guide, we'll use rsyslog, which is already included by default in most Linux distributions (Ubuntu, Debian, CentOS, Red Hat).

### Add Logging Logic to Bash

Append the following line to the end of:

### Ubuntu/Debian: /etc/bash.bashrc

```bash
export PROMPT_COMMAND='RETRN_VAL=$?;logger -t LinuxCommandsWazuh -p local6.debug "User $(whoami) [$$]: $(history 1 | sed "s/^[ ]*[0-9]\\+[ ]*//")"'
```

### Red Hat/CentOS: /etc/bashrc

```bash
export PROMPT_COMMAND='RETRN_VAL=$?;logger -t LinuxCommandsWazuh -p local6.debug "User $(whoami) [$$]: $(history 1 | sed "s/^[ ]*[0-9]\\+[ ]*//")"'
```

This ensures every executed command is automatically sent to the local syslog service under the tag **LinuxCommandsWazuh**.

### Create a Custom Rsyslog Config

Create a new file:

```bash
sudo nano /etc/rsyslog.d/bash.conf
```

Add the following line:

```bash
local6.* /var/log/commands.log
```

### Exclude Local6 from Default Syslog

Find this line inside /etc/rsyslog.d/50-default.conf:

```bash
*.*;auth,authpriv.none -/var/log/syslog
```

Replace it with:

```bash
*.*;auth,authpriv.none,local6.none -/var/log/syslog
```

> This prevents duplication of command logs in both /var/log/syslog and /var/log/commands.log.
> 

### Restart and Rotate Logs

Restart the rsyslog service:

```bash
sudo systemctl restart rsyslog
```

Then ensure log rotation is configured for /var/log/commands.log inside:

### Ubuntu/Debian: /etc/logrotate.d/rsyslog

Add the following configuration:

```bash
/var/log/commands.log {
    rotate 4
    weekly
    missingok
    notifempty
    compress
    delaycompress
    sharedscripts
    postrotate
        /usr/lib/rsyslog/rsyslog-rotate
    endscript
}
```

### RedHat/CentOS: /etc/logrotate.d/syslog

Add the following configuration:

```bash
/var/log/commands.log {
    rotate 4
    weekly
    missingok
    notifempty
    compress
    delaycompress
    sharedscripts
    postrotate
        /bin/kill -HUP `cat /var/run/syslogd.pid 2> /dev/null` 2> /dev/null || true
    endscript
}
```

Finally, log out and back in to apply the new configuration.

> **✅ You now have all executed commands logged to:** /var/log/commands.log
> 

## 4.2. Wazuh Configuration

Now it's time to bring those logs into Wazuh for monitoring and alerting.

### Configure the Wazuh Agent

Add the following snippet inside your agent configuration, ideally through a group configuration (e.g. linux-ubuntu):

```xml
<agent_config>
    <localfile>
        <location>/var/log/commands.log</location>
        <log_format>syslog</log_format>
    </localfile>
</agent_config>
```

This tells the Wazuh Agent to continuously read the /var/log/commands.log file.

### Create Custom Decoders

Inside the Wazuh Manager, create or edit your decoder file (e.g. /var/ossec/etc/decoders/local_decoder.xml):

```xml
<decoder name="Linux-commands">
    <program_name>^LinuxCommandsWazuh</program_name>
</decoder>

<decoder name="Linux-commands1">
    <parent>Linux-commands</parent>
    <regex>User (\\w+) [\\d+]: (.+)</regex>
    <order>User, Command</order>
</decoder>
```

These decoders capture the username and command executed from your syslog entries.

> You can also view and manage decoders through the Decoders page in the Wazuh web interface.
> 

### Create a Custom Rule

In Wazuh VM, add a new rule in /var/ossec/etc/rules/local_rules.xml:

```xml
<group name="Linux-commands,">
    <rule id="100002" level="3">
        <program_name>LinuxCommandsWazuh</program_name>
        <description>Command: "$(Command)" executed by $(User) in $(hostname)</description>
        <group>syslog, local</group>
    </rule>
</group>
```

> This rule triggers an alert each time a command is executed, showing the username, command, and hostname.
> 

## 5. Checking Results in Wazuh Dashboard

Once everything is configured, restart Wazuh, and then open your Wazuh Dashboard and search for the rule ID 100002 or search for the command you use in Linux Ubuntu.

You'll see real-time alerts such as:

> Command: "docker container ls" executed by root on prod-app-server
or
Command: "docker container ls" executed by cumhur.akkaya on prod-app-server
> 

If you click on the magnifying glass, it will show the top 5:

## 6. Creating Alarm Rule

You can then visualize these events, correlate suspicious activity, or even set up notifications via email, Slack, etc.

This rule captures any command coming from the decoder and generates an alarm. Open the /var/ossec/etc/rules/local_rules.xml file on your Wazuh Manager server and add the following rule:

```xml
<group name="linux-commands,">
    <rule id="100050" level="5">
        <program_name>LinuxCommandsWazuh</program_name>
        <description>Command executed by $(User) on $(hostname): $(Command)</description>
        <group>syslog, local, audit</group>
        <options>alert_by_email</options>
    </rule>
</group>
```

> The level="5" value here is suitable for sending emails (exceeds the email threshold).
> 

If you want, you can add a "High-risk command detection" section and create additional rules that will give different levels of alarm (level 10+) for commands such as rm -rf, systemctl stop, useradd.

---

© 2025 Rabiul Hasan
