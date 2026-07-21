# POC: Zero-Downtime Group Configuration Deployment in Wazuh High Availability

## 1. Objective

Validate that updating the configuration for one Wazuh agent group (**Group-A**) does not interrupt log collection for agents belonging to another group (**Group-B**) in a High Availability deployment.

The POC demonstrates:

- Group-level configuration isolation
- Master-to-Worker configuration synchronization
- Continuous event processing during configuration deployment
- Zero downtime for unaffected groups

---

## 2. Test Environment

| Component | Purpose |
|-----------|---------|
| HAProxy | Agent connection endpoint |
| Master Manager | Configuration management |
| Worker Manager | Event processing |
| Group-A Agent | Configuration deployment validation |
| Group-B Agent | Continuous log generation |

---

## 3. Architecture

```text
                     Agents
                        │
                        ▼
                HAProxy :1514
                        │
          ┌─────────────┴─────────────┐
          │                           │
          ▼                           ▼
   Worker Manager 1           Worker Manager 2
          │                           │
          └─────────────┬─────────────┘
                        │
                        ▼
                 Master Manager
```

---

# Step 1 – Verify Cluster Health

Before starting the test, verify that:

- Master Manager is running.
- Worker Managers are running.
- HAProxy is operational.
- Group-A agent is connected.
- Group-B agent is connected.

### Expected Result

```text
Cluster Status : Healthy

Agents Connected : Yes
```

---

# Step 2 – Verify Agent Connectivity Through HAProxy

On both agents:

```bash
cat /var/ossec/etc/ossec.conf
```

Verify:

```xml
<server>
    <address>wazuh.company.com</address>
    <port>1514</port>
</server>
```

### Expected Result

All agents communicate only with HAProxy.

---

# Step 3 – Verify Group Membership

On the Master Manager:

```bash
/var/ossec/bin/agent_groups -S -g Group-A
```

### Expected

```text
001
```

Verify Group-B:

```bash
/var/ossec/bin/agent_groups -S -g Group-B
```

### Expected

```text
002
```

---

# Step 4 – Start Continuous Log Generation

On the **Group-B agent**, continuously generate logs.

```bash
while true
do
logger "GROUP_B_CONTINUOUS_TEST $(date)"
sleep 2
done
```

This process must remain running throughout the POC.

### Purpose

Group-B represents a production workload that must never stop sending logs.

---

# Step 5 – Monitor Incoming Events

On the active Worker Manager:

```bash
tail -f /var/ossec/logs/archives/archives.log
```

### Expected Output

```text
GROUP_B_CONTINUOUS_TEST

GROUP_B_CONTINUOUS_TEST

GROUP_B_CONTINUOUS_TEST
```

Observe that logs are continuously arriving before any maintenance activity.

---

# Step 6 – Modify Group-A Configuration

On the Master Manager:

```bash
vi /var/ossec/etc/shared/Group-A/agent.conf
```

Example:

```xml
<agent_config>

    <localfile>

        <location>/tmp/group-a.log</location>

        <log_format>syslog</log_format>

    </localfile>

</agent_config>
```

Save the file.

This simulates deploying a new monitoring configuration for only Group-A.

---

# Step 7 – Restart the Master Manager

Restart the Master Manager to synchronize the updated group configuration to all Worker Managers.

Example:

```bash
systemctl restart wazuh-manager
```

or

```bash
docker restart wazuh.manager
```

depending on the deployment model.

---

# Step 8 – Observe Group-B Log Flow During Deployment

Do **not** stop monitoring `archives.log`.

Continue observing:

```bash
tail -f /var/ossec/logs/archives/archives.log
```

### Expected Output

```text
GROUP_B_CONTINUOUS_TEST

GROUP_B_CONTINUOUS_TEST

GROUP_B_CONTINUOUS_TEST

GROUP_B_CONTINUOUS_TEST
```

### Expected Behaviour

- No interruption
- No missing events
- No pause in log collection

This is the primary validation of the POC.

---

# Step 9 – Verify Configuration Synchronization

After the Master Manager is available again, verify that the new configuration has reached the Group-A agent.

On **Agent 001**:

```bash
grep group-a.log /var/ossec/etc/shared/merged.mg
```

### Expected

```text
/tmp/group-a.log
```

---

# Step 10 – Verify Group Isolation

On **Agent 002**:

```bash
grep group-a.log /var/ossec/etc/shared/merged.mg
```

### Expected

```text
No output
```

This confirms that the Group-A configuration was **not** applied to Group-B.

---

# Step 11 – Validate Continuous Event Processing

Review the timestamps in the Worker Manager archive logs.

Example:

```text
15:30:01

15:30:03

15:30:05

15:30:07

15:30:09
```

There should be **no significant gap** corresponding to the Master Manager restart.

---

# Step 12 – Validate Cluster Synchronization

Verify that the Worker Managers have synchronized the updated shared configuration.

Check cluster status:

```bash
/var/ossec/bin/cluster_control -l
```

### Expected

```text
Master : Connected

Worker-1 : Connected

Worker-2 : Connected
```
