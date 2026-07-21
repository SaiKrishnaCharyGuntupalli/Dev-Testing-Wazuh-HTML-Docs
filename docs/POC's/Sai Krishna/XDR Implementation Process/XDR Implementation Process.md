# XDR Implementation Process

## First Understand: What is XDR in Wazuh?

XDR = Correlating security data from:

- Endpoint (agent logs)
- System activity
- File changes
- Rootkits
- Vulnerabilities
- Authentication events
- Threat intelligence

Then:

- Detect threats
- Generate alerts
- Correlate events
- Trigger active response

Wazuh already supports XDR features. You just need to enable and configure them properly.

---

## Configuration Files You Must Edit

---

## On Agent VM

File: `/var/ossec/etc/ossec.conf`

---

### Enable Log Collection

Inside `<localfile>`:

```xml

<localfile>
  <log_format>syslog</log_format>
  <location>/var/log/auth.log</location>
</localfile>

<localfile>
  <log_format>syslog</log_format>
  <location>/var/log/syslog</location>
</localfile>

```

**Why?** Collect authentication logs for brute force detection.

---

### Enable File Integrity Monitoring (FIM)

```xml

<syscheck> 
  <frequency>3600</frequency> 
  <directories check_all="yes">/etc,/usr/bin</directories> 
</syscheck> 

```

**Why?** Detect file modifications caused by malware or persistence mechanisms.

---

### Enable Rootcheck

```xml

<rootcheck> 
  <disabled>no</disabled> 
</rootcheck> 

```

**Why?** Detect rootkits and suspicious binaries.

---

### Enable SCA

```xml

<sca> 
  <enabled>yes</enabled> 
  <scan_on_start>yes</scan_on_start> 
</sca> 

```

**Why?** Detect insecure configurations.

---

!!! note "Before Making Changes"
    First verify whether the above changes are already configured. If not configured, configure based on your requirement. If configured but you need a custom directory monitored, add the required directory path under the File Integrity Monitoring section.

### Apply Changes on Agent

Restart the Wazuh agent to apply changes:

```bash
sudo systemctl restart wazuh-agent
```

Verify the agent is running:

```bash
sudo systemctl status wazuh-agent
```

Expected result: `active (running)`

---

## On Wazuh Manager VM

File: `/var/ossec/etc/ossec.conf`

---

### Enable Vulnerability Detection

```xml

<vulnerability-detector> 
  <enabled>yes</enabled> 
  <interval>5m</interval> 
  <provider name="canonical"> 
    <enabled>yes</enabled> 
  </provider> 
</vulnerability-detector> 
  

```

**Why?** Detect installed packages with known CVEs.

---

### Custom Rules

If you want to write a custom rule, add it to:
```

/var/ossec/etc/rules/local_rules.xml

```
Example — block connections from blacklisted IPs:

```xml
<group name="threat_intel">
  <rule id="100100" level="10">
    <if_sid>5710</if_sid>
    <list field="srcip" lookup="address_match_key">etc/lists/bad-ip-list</list>
    <description>Connection from blacklisted IP</description>
  </rule>
</group>
```

### Adding Custom Rules via the UI

Custom rules can also be added directly from the Wazuh Dashboard UI:

**Step 1** — Login to Wazuh in the browser and navigate to the **Rules** section.

**Step 2** — Click the **Add new rule** button.

**Step 3** — An editor window opens. Write your custom rule logic in XML format, then click **Save**.

**Step 4** — After saving, navigate back one step and click the **Custom rules** button.

**Step 5** — Your custom rule will now appear in the custom rules list.

---

### Enable Active Response

In `/var/ossec/etc/ossec.conf` on the manager:

```xml
<active-response>
  <command>firewall-drop</command>
  <location>local</location>
  <rules_id>5710</rules_id>
  <timeout>60</timeout>
</active-response>
```

**Why?** Automatically block a brute force attacker's IP address.

!!! note
    Use `<location>local</location>` to run the response on the manager itself, or `<location>localhost</location>` if running in a local container context. Choose based on your deployment.

---

!!! note "Before Making Changes"
    First verify whether the above changes are already configured. If not configured, configure based on your requirement.

### Apply Changes on Manager

Restart the Wazuh manager to apply changes:

```bash
sudo systemctl restart wazuh-manager
```

Verify the manager is running:

```bash
sudo systemctl status wazuh-manager
```

Expected result: `active (running)`

---

## Problem Observed During Implementation

When verifying the Wazuh manager configuration, a Wazuh manager server-related issue was encountered. The Wazuh Dashboard became inaccessible in the browser, showing a connection error page.

!!! warning "If You See This Error"
    If the Wazuh Dashboard becomes unreachable after editing `ossec.conf`, the most likely cause is a syntax error in the configuration file. Validate the XML before restarting:

```bash
    /var/ossec/bin/wazuh-control check-config
```

    Fix any reported errors, then restart:

```bash
    sudo systemctl restart wazuh-manager
```
