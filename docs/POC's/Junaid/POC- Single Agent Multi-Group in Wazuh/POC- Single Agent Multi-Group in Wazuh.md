# POC: Single Agent Multi-Group in Wazuh

## 1\. Objective

Demonstrate that one Wazuh agent can belong to multiple groups and receive layered monitoring policies.

For this POC, the following agent is used:

| Agent ID | Agent Name | Tenant  | Operating System   |
| -------- | ---------- | ------- | ------------------ |
| 001      | Kibana     | tenantA | Ubuntu 22.04.2 LTS |

The agent will be assigned to two groups:

| Group              | Purpose                                         |
| ------------------ | ----------------------------------------------- |
| baseline-linux     | Applies common Linux baseline monitoring        |
| tenantA-compliance | Applies Tenant A compliance-specific monitoring |

The final policy model is:

Kibana Agent (001)  
|  
+-- baseline-linux  
| +-- Common Linux monitoring  
|  
+-- tenantA-compliance  
+-- Tenant-specific compliance monitoring

## 2\. Expected Outcome

After completing this POC:

- The Kibana agent belongs to both groups.
- The baseline-linux group configures monitoring of one log file.
- The tenantA-compliance group configures monitoring of a second log file.
- The same agent collects both log sources.
- Two different Wazuh rules generate two alerts from the same agent.
- The alerts can be viewed in the Wazuh Dashboard.

## Part A: Wazuh Dashboard Steps

## Step 1: Create the baseline-linux group

- Log in to the Wazuh Dashboard.
- Navigate to:

Sidebar -> Agents Management → Groups

- Click **Create group**.
- Enter the group name:

baseline-linux

- Click **Create**.

This group represents the standard monitoring policy for Linux endpoints.

## Step 2: Create the tenantA-compliance group

- Stay on:

Agents Management → Groups

- Click **Create group**.
- Enter the group name:

tenantA-compliance

- Click **Create**.

This group represents additional monitoring required for Tenant A compliance requirements.

## Step 3: Assign the Kibana agent to both groups

- Navigate to:

Agents Management (From sidebar) → groups

- Search for the groups:

baseline-linux & tenantA - compliance

- Open the baseline-linux group :

- \- click on manage agents (on top right side)  
   \- Under the available agent section on left side,click on "Kibana" agent than click on "add selected items"

- Follow the same for the "tenantA-compliance" group  
   and click on "apply changes"

## Step 4: Verify multi-group membership in the Dashboard

Open the Kibana agent details page and confirm that the agent is assigned to:

default  
baseline-linux  
tenantA-compliance

Take a screenshot. This is the primary UI evidence that one agent belongs to multiple groups.

## Part B: Wazuh Manager Server Steps

## Step 5: Access the Wazuh Manager container

Run the following command on the Docker host:

docker exec -it single-node-wazuh.manager-1 bash

## Step 6: Verify that agent 001 belongs to both groups

Inside the Wazuh Manager container, run:

/var/ossec/bin/agent_groups -s -i 001

Expected output should include:

baseline-linux  
tenantA-compliance

If either group is missing, add it using:

/var/ossec/bin/agent_groups -a -i 001 -g baseline-linux  
/var/ossec/bin/agent_groups -a -i 001 -g tenantA-compliance

Verify again:

/var/ossec/bin/agent_groups -s -i 001

## Step 7: Create the baseline Linux group configuration

Create the group configuration directory:

mkdir -p /var/ossec/etc/shared/baseline-linux

Create the group configuration file:

vi /var/ossec/etc/shared/baseline-linux/agent.conf

Add the following configuration:

&lt;agent_config&gt;  
&lt;localfile&gt;  
&lt;location&gt;/var/log/wazuh-linux-baseline-poc.log&lt;/location&gt;  
&lt;log_format&gt;syslog&lt;/log_format&gt;  
&lt;/localfile&gt;  
&lt;/agent_config&gt;

Save and exit Vim:

Esc  
:wq  
Enter

Validate the XML:

xmllint --noout /var/ossec/etc/shared/baseline-linux/agent.conf

If the command returns no output, the XML is valid.

This configuration represents the common Linux monitoring layer.

## Step 8: Create the Tenant A compliance group configuration - The below is basic config,you can add additional paths for additional log collection

Create the group configuration directory:

mkdir -p /var/ossec/etc/shared/tenantA-compliance

Create the group configuration file:

vi /var/ossec/etc/shared/tenantA-compliance/agent.conf

Add the following configuration:

&lt;agent_config&gt;  
&lt;localfile&gt;  
&lt;location&gt;/var/log/wazuh-tenanta-compliance-poc.log&lt;/location&gt;  
&lt;log_format&gt;syslog&lt;/log_format&gt;  
&lt;/localfile&gt;  
&lt;/agent_config&gt;

Save and exit Vim:

Esc  
:wq  
Enter

Validate the XML:

xmllint --noout /var/ossec/etc/shared/tenantA-compliance/agent.conf

If the command returns no output, the XML is valid.

This configuration represents the Tenant A compliance monitoring layer.

## Step 9: Restart the Wazuh Manager

Restart the Wazuh Manager so the group configurations are distributed:

/var/ossec/bin/wazuh-control restart

Wait approximately one minute for the agent to receive the updated group configurations.

## Part C: Kibana Agent Steps

## Step 10: Create the two test log files on the Kibana agent

Connect to the Kibana agent server:

ssh &lt;username&gt;@<IP-address>

Create both log files:

sudo touch /var/log/wazuh-linux-baseline-poc.log  
sudo touch /var/log/wazuh-tenanta-compliance-poc.log

Set readable permissions:

sudo chmod 644 /var/log/wazuh-linux-baseline-poc.log  
sudo chmod 644 /var/log/wazuh-tenanta-compliance-poc.log

Restart the Wazuh agent:

sudo systemctl restart wazuh-agent

## Step 11: Verify that both group configurations reached the agent

Run:

sudo grep -Ei "wazuh-linux-baseline-poc.log|wazuh-tenanta-compliance-poc.log" /var/ossec/logs/ossec.log

Expected output:

Analyzing file: '/var/log/wazuh-linux-baseline-poc.log'  
Analyzing file: '/var/log/wazuh-tenanta-compliance-poc.log'

This proves that the same agent received monitoring configuration from both groups.

## Part D: Detection Rule Configuration

## Step 12: Return to the Wazuh Manager container

Run this on the Docker host:

docker exec -it single-node-wazuh.manager-1 bash

Back up the existing local rules file:

cp /var/ossec/etc/rules/local_rules.xml /var/ossec/etc/rules/local_rules.xml.backup-multigroup-poc

Edit the local rules file:

vi /var/ossec/etc/rules/local_rules.xml

Add the following two rules inside the existing &lt;group&gt; section:

&lt;rule id="100503" level="6"&gt;  
&lt;match&gt;LINUX_BASELINE_POC_EVENT&lt;/match&gt;  
&lt;description&gt;POC Layer 1: Linux baseline monitoring event detected&lt;/description&gt;  
&lt;group&gt;group_poc,linux_baseline,&lt;/group&gt;  
&lt;/rule&gt;  
<br/>&lt;rule id="100504" level="9"&gt;  
&lt;match&gt;TENANTA_COMPLIANCE_POC_EVENT&lt;/match&gt;  
&lt;description&gt;POC Layer 2: Tenant A compliance monitoring event detected&lt;/description&gt;  
&lt;group&gt;group_poc,tenanta_compliance,&lt;/group&gt;  
&lt;/rule&gt;

Do not create a second &lt;group&gt; root element if one already exists in local_rules.xml. Add the rules inside the existing group block.

## Step 13: Validate and activate the rules

Validate the Wazuh ruleset:

/var/ossec/bin/wazuh-analysisd -t

Expected result:

Configuration passed

Restart the Wazuh Manager:

/var/ossec/bin/wazuh-control restart

## Part E: Generate Test Events

## Step 14: Generate the Linux baseline event

On the Kibana agent, run:

echo "\$(date '+%b %d %H:%M:%S') \$(hostname) LINUX_BASELINE_POC_EVENT common-linux-baseline-test" | sudo tee -a /var/log/wazuh-linux-baseline-poc.log

## Step 15: Generate the Tenant A compliance event

On the same Kibana agent, run:

echo "\$(date '+%b %d %H:%M:%S') \$(hostname) TENANTA_COMPLIANCE_POC_EVENT tenantA-compliance-test" | sudo tee -a /var/log/wazuh-tenanta-compliance-poc.log

## Part F: Validation

## Step 16: Validate alerts on the Wazuh Manager

Inside the Wazuh Manager container, run:

grep -E "LINUX_BASELINE_POC_EVENT|TENANTA_COMPLIANCE_POC_EVENT" /var/ossec/logs/alerts/alerts.json | tail -10

Expected result:

| Rule ID | Alert Description                                          | Agent  |
| ------- | ---------------------------------------------------------- | ------ |
| 100503  | POC Layer 1: Linux baseline monitoring event detected      | Kibana |
| 100504  | POC Layer 2: Tenant A compliance monitoring event detected | Kibana |

Both alerts should contain:

agent.id: 001  
agent.name: Kibana  
agent.labels.tenant: tenantA

## Step 17: Validate alerts in the Wazuh Dashboard

Open:

Threat Hunting → Discover

Select the Tenant A data view:

wazuh-tenanta-alerts-\*

Set the time range to:

Last 15 minutes

Search for:

rule.id: (100503 OR 100504)

Expected result: two alerts from the same Kibana agent.

## POC Success Criteria

The POC is successful when all the following are true:

| Validation               | Expected Result                                              |
| ------------------------ | ------------------------------------------------------------ |
| Group assignment         | Kibana belongs to both baseline-linux and tenantA-compliance |
| Baseline configuration   | Kibana monitors /var/log/wazuh-linux-baseline-poc.log        |
| Compliance configuration | Kibana monitors /var/log/wazuh-tenanta-compliance-poc.log    |
| Baseline alert           | Rule 100503 is triggered                                     |
| Compliance alert         | Rule 100504 is triggered                                     |
| Dashboard evidence       | Both alerts are visible for Kibana in Tenant A's alert index |

## Business Use Case

This POC represents a real MSSP deployment model where a single endpoint requires multiple policy layers.

For example, a production Linux payment server can receive:

Linux baseline policy  
\+ Tenant-specific policy  
\+ PCI compliance policy  
\+ Production policy  
\+ Critical asset policy

Instead of creating one large group for every possible combination, reusable policy layers can be applied through multiple groups. This reduces duplicate configuration, improves maintainability, and allows targeted policy changes without affecting unrelated endpoints.