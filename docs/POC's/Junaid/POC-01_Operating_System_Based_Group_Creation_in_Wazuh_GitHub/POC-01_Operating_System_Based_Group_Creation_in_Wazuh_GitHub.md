# POC-01: Operating System Based Group Creation in Wazuh

## 1. Objective

Demonstrate how Linux endpoints can be logically grouped based on their operating system to simplify centralized configuration management, rule deployment, and policy administration.

For this POC, the following agents are used:

| Agent ID | Agent Name | Operating System |
|----------|------------|------------------|
| 001 | Kibana | Ubuntu 22.04.2 LTS |
| 002 | Jenkins | Ubuntu 22.04.5 LTS |

Both agents will be assigned to a common group:

| Group | Purpose |
|-------|---------|
| linux-servers | Common monitoring policy for Linux endpoints |

The final group structure will be:

```text
Linux Servers Group

linux-servers
│
├── Kibana (001)
│
└── Jenkins (002)
```

## 2. Expected Outcome

After completing this POC:

- A Linux group named `linux-servers` is created.
- Kibana and Jenkins agents belong to the Linux group.
- A common monitoring configuration is deployed to both agents.
- Both agents begin monitoring the same log file.
- A common Wazuh rule detects events from both agents.
- Alerts are visible in the Wazuh Dashboard.

# Part A – Wazuh Dashboard Steps

## Step 1 – Create the Linux Group

Log in to the Wazuh Dashboard.

Navigate to:

```text
Sidebar
  ↓
Agents
  ↓
Management
  ↓
Groups
```

Click **Create Group**.

Enter:

```text
linux-servers
```

Click **Create**.

This group represents the common monitoring policy for Linux endpoints.

## Step 2 – Verify Group Creation

Navigate to:

```text
Agents
  ↓
Groups
```

**Expected Result**

```text
default
linux-servers
```

Take a screenshot.

## Step 3 – Assign Linux Agents

Open:

```text
linux-servers
```

Click:

```text
Manage Agents
```

From **Available Agents** select:

- Kibana
- Jenkins

Click:

```text
Add Selected Items
Apply Changes
```

## Step 4 – Verify Group Membership

Open **Agents**.

Verify:

```text
Kibana
↓
Group
linux-servers
```

Similarly:

```text
Jenkins
↓
Group
linux-servers
```

Take a screenshot.

# Part B – Wazuh Manager Steps

## Step 5 – Access the Wazuh Manager

**Docker Deployment**

```bash
docker exec -it single-node-wazuh.manager-1 bash
```

**VM Deployment**

```bash
sudo su -
```

## Step 6 – Verify Agent Membership

Run:

```bash
/var/ossec/bin/agent_groups -S -g linux-servers
```

Expected:

```text
001
002
```

If an agent is missing:

```bash
/var/ossec/bin/agent_groups -a -i 001 -g linux-servers
/var/ossec/bin/agent_groups -a -i 002 -g linux-servers
```

Verify again:

```bash
/var/ossec/bin/agent_groups -S -g linux-servers
```

## Step 7 – Create the Group Configuration

Create the directory:

```bash
mkdir -p /var/ossec/etc/shared/linux-servers
```

Create:

```bash
vi /var/ossec/etc/shared/linux-servers/agent.conf
```

Add:

```xml
<agent_config>
    <localfile>
        <location>/var/log/wazuh-linux-group-poc.log</location>
        <log_format>syslog</log_format>
    </localfile>
</agent_config>
```

Save:

```text
Esc
:wq
Enter
```

## Step 8 – Validate XML

```bash
xmllint --noout /var/ossec/etc/shared/linux-servers/agent.conf
```

Expected:

```text
No output.
```

## Step 9 – Restart the Wazuh Manager

```bash
/var/ossec/bin/wazuh-control restart
```

Wait approximately one minute for the configuration to synchronize.

# Part C – Linux Agent Steps

## Step 10 – Create the Common Log File

Perform the following steps on both Kibana and Jenkins agents.

Create the log file:

```bash
sudo touch /var/log/wazuh-linux-group-poc.log
```

Set permissions:

```bash
sudo chmod 644 /var/log/wazuh-linux-group-poc.log
```

Restart the Wazuh Agent:

```bash
sudo systemctl restart wazuh-agent
```

## Step 11 – Verify Configuration Synchronization

On both agents, run:

```bash
sudo grep -i "wazuh-linux-group-poc.log" /var/ossec/logs/ossec.log
```

Expected output:

```text
Analyzing file:

'/var/log/wazuh-linux-group-poc.log'
```

This confirms that the common Linux group configuration has been synchronized successfully.

# Part D – Detection Rule Configuration

## Step 12 – Configure the Detection Rule

On the Wazuh Manager:

Backup the local rules file:

```bash
cp /var/ossec/etc/rules/local_rules.xml \
/var/ossec/etc/rules/local_rules.xml.backup-linux-group
```

Edit:

```bash
vi /var/ossec/etc/rules/local_rules.xml
```

Add:

```xml
<rule id="100510" level="8">
    <match>LINUX_GROUP_POC_EVENT</match>
    <description>POC Linux Group: Common monitoring event detected</description>
    <group>group_poc,linux_group,</group>
</rule>
```

## Step 13 – Validate the Rules

```bash
/var/ossec/bin/wazuh-analysisd -t
```

Expected:

```text
Configuration Passed
```

Restart:

```bash
/var/ossec/bin/wazuh-control restart
```

# Part E – Generate Test Events

## Step 14 – Generate Events on Both Agents

**On Kibana**

```bash
echo "$(date '+%b %d %H:%M:%S') $(hostname) LINUX_GROUP_POC_EVENT generated from Kibana" \
| sudo tee -a /var/log/wazuh-linux-group-poc.log
```

**On Jenkins**

```bash
echo "$(date '+%b %d %H:%M:%S') $(hostname) LINUX_GROUP_POC_EVENT generated from Jenkins" \
| sudo tee -a /var/log/wazuh-linux-group-poc.log
```

# Part F – Validation

## Step 15 – Validate Alerts on the Manager

Run:

```bash
grep "LINUX_GROUP_POC_EVENT" \
/var/ossec/logs/alerts/alerts.json | tail -10
```

Expected:

| Rule ID | Agent |
|---------|-------|
| 100510 | Kibana |
| 100510 | Jenkins |

Both alerts should contain:

- `agent.id`
- `agent.name`
- `linux-servers`

## Step 16 – Validate Alerts in Dashboard

Navigate to:

```text
Threat Hunting
↓
Discover
```

Select:

```text
wazuh-alerts-*
```

Time Range:

```text
Last 15 Minutes
```

Search:

```text
rule.id:100510
```

Expected:

- Kibana
- Jenkins
