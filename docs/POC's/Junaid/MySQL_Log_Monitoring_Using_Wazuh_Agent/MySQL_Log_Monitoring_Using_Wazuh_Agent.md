# MySQL Log Monitoring Using Wazuh Agent

## 1. Objective

This document explains how to deploy a Wazuh agent on the MySQL database server and collect MySQL log files.

The MySQL server is:

```text
<IP-address>
```

The Wazuh agent reads MySQL logs locally and sends them to the Wazuh Manager. The Wazuh Manager analyzes the events, generates alerts where applicable, and sends those alerts to the Wazuh Indexer. The events can then be viewed in Wazuh Dashboard Discover.

---

## 2. Architecture

```text
MySQL Server: <IP-address>
        |
        | MySQL error, general, and slow-query logs
        v
Wazuh Agent
        |
        | TCP 1514 / TCP 1515
        v
Wazuh Manager
        |
        | Wazuh alert processing
        v
Wazuh Indexer
        |
        v
Wazuh Dashboard → Discover
```

---

## 3. Prerequisites

Before starting, ensure the following requirements are met.

| Requirement           | Details                                                                        |
| --------------------- | ------------------------------------------------------------------------------ |
| MySQL server          | Running on `<IP-address>`                                                     |
| Wazuh Manager         | Reachable from the MySQL server                                                |
| Network access        | TCP port `1514` and TCP port `1515` allowed from MySQL server to Wazuh Manager |
| Administrative access | `sudo` access on the MySQL server                                              |
| Wazuh Dashboard       | Available for validating alerts in Discover                                    |

---

# Part A — Configure MySQL Logging

## Step 1: Connect to the MySQL Server

```bash
ssh <username>@<IP-address>
```

---

## Step 2: Verify MySQL Service Status

```bash
sudo systemctl status mysql
```

If the service is not running, start and enable it:

```bash
sudo systemctl start mysql
sudo systemctl enable mysql
```

---

## Step 3: Open MySQL Configuration File

For Ubuntu-based MySQL installations, the configuration file is commonly:

```bash
sudo nano /etc/mysql/mysql.conf.d/mysqld.cnf
```

---

## Step 4: Enable MySQL Logs

Under the `[mysqld]` section, add or update the following settings:

```ini
[mysqld]

log_error = /var/log/mysql/error.log

general_log = 1
general_log_file = /var/log/mysql/mysql-general.log

slow_query_log = 1
slow_query_log_file = /var/log/mysql/mysql-slow.log

long_query_time = 2
log_queries_not_using_indexes = 1
```

### Log Description

| MySQL Log      | Path                               | Purpose                                                         |
| -------------- | ---------------------------------- | --------------------------------------------------------------- |
| Error log      | `/var/log/mysql/error.log`         | MySQL startup, shutdown, crash, and authentication errors       |
| General log    | `/var/log/mysql/mysql-general.log` | Connections, disconnects, SQL statements, and database activity |
| Slow query log | `/var/log/mysql/mysql-slow.log`    | Queries exceeding the configured execution threshold            |

Save the file and exit.

---

## Step 5: Restart MySQL

```bash
sudo systemctl restart mysql
sudo systemctl status mysql
```

---

## Step 6: Verify MySQL Log Files

```bash
sudo ls -lh /var/log/mysql/
```

Expected files:

```text
error.log
mysql-general.log
mysql-slow.log
```

Verify that MySQL is writing events:

```bash
sudo tail -f /var/log/mysql/error.log
```

In a separate terminal, connect and disconnect from MySQL:

```bash
mysql -u root -p
```

Then run:

```sql
exit;
```

Verify the general log:

```bash
sudo tail -f /var/log/mysql/mysql-general.log
```

---

# Part B — Install Wazuh Agent on the MySQL Server

## Step 7: Log In to Wazuh Dashboard

Open the Wazuh Dashboard URL in a web browser:

```
https://<WAZUH_DASHBOARD_URL>
```

Log in using an authorized Wazuh Dashboard account.

---

## Open the Agent Deployment Page

From the left navigation menu, go to:

```
Sidebar-agent management - agent summary - Deploy agent
```

Click:

```
select OS Type (linux) - enter ServerIP(manager ip) & agent name 
```

---

## Step 8: Follow Installation commands shown in agent deploy page


curl -o wazuh-agent-4.14.4-1.x86_64.rpm https://packages.wazuh.com/4.x/yum/wazuh-agent-4.14.4-1.x86_64.rpm && sudo WAZUH_MANAGER='<IP-address>' WAZUH_AGENT_NAME='mysql_collector' rpm -ihv wazuh-agent-4.14.4-1.x86_64.rpm

```bash
sudo systemctl daemon-reload
sudo systemctl enable wazuh-agent
sudo systemctl start wazuh-agent
```

At this stage, the agent may be inactive until its configuration is completed.

---

# Part C — Configure Wazuh Agent to Read MySQL Logs

## Step 9: Back Up Wazuh Agent Configuration

```bash
sudo cp /var/ossec/etc/ossec.conf /var/ossec/etc/ossec.conf.backup
```

---

## Step 10: Open Wazuh Agent Configuration File

```bash
sudo nano /var/ossec/etc/ossec.conf
```

##

---

## Step 11: Add MySQL Log Collection Entries

Add the following entries inside the `<ossec_config>` section of `/var/ossec/etc/ossec.conf`.

```xml
<localfile>
  <location>/var/log/mysql/error.log</location>
  <log_format>syslog</log_format>
</localfile>

<localfile>
  <location>/var/log/mysql/mysql-general.log</location>
  <log_format>syslog</log_format>
</localfile>

<localfile>
  <location>/var/log/mysql/mysql-slow.log</location>
  <log_format>syslog</log_format>
</localfile>
```

These entries instruct the Wazuh agent to monitor the MySQL log files continuously.

---

## Step 12: Confirm the Complete MySQL Log Collection Configuration

The relevant configuration should look similar to the following:

```xml
<ossec_config>

  <client>
    <server>
      <address><WAZUH_MANAGER_IP></address>
      <port>1514</port>
      <protocol>tcp</protocol>
    </server>
  </client>

  <localfile>
    <location>/var/log/mysql/error.log</location>
    <log_format>syslog</log_format>
  </localfile>

  <localfile>
    <location>/var/log/mysql/mysql-general.log</location>
    <log_format>syslog</log_format>
  </localfile>

  <localfile>
    <location>/var/log/mysql/mysql-slow.log</location>
    <log_format>syslog</log_format>
  </localfile>

</ossec_config>
```

Save the file and exit.

---

# Part D — Configure Required Permissions

## Step 13: Check MySQL Log File Permissions

```bash
sudo ls -l /var/log/mysql/
```

The Wazuh agent runs as the `wazuh` user and must be able to read the MySQL log files.

Add the `wazuh` user to the `adm` group:

```bash
sudo usermod -aG adm wazuh
```

Restart the Wazuh agent after changing group membership:

```bash
sudo systemctl restart wazuh-agent
```

Verify that the Wazuh user can read the error log:

```bash
sudo -u wazuh head -n 5 /var/log/mysql/error.log
```

If this command returns log content without a permission error, access is configured correctly.

---

# Part E — Start and Register the Wazuh Agent

## Step 14: Enable and Start Wazuh Agent

```bash
sudo systemctl daemon-reload
sudo systemctl enable wazuh-agent
sudo systemctl restart wazuh-agent
```

Check status:

```bash
sudo systemctl status wazuh-agent
```

Expected status:

```text
Active: active (running)
```

---

## Step 15: Verify Agent Connection Logs

```bash
sudo tail -f /var/ossec/logs/ossec.log
```

A successful connection typically contains messages indicating that the agent connected to the manager.

---

## Step 16: Verify Agent Registration on Wazuh Manager

Connect to the Wazuh Manager host.

If the Wazuh Manager is deployed using Docker, run:

```bash
docker exec -it single-node-wazuh.manager-1 /var/ossec/bin/manage_agents -l
```

If the Wazuh Manager is installed directly on the operating system, run:

```bash
sudo /var/ossec/bin/manage_agents -l
```

Confirm that the MySQL server appears in the agent list.

Example expected result:

```text
ID: 008, Name: mysql-server, IP: <IP-address>, Active
```

---

# Part F — Validate MySQL Log Collection

## Step 17: Generate a Failed MySQL Login Event

On the MySQL server, run:

```bash
mysql -u invalid_user -p
```

Enter an invalid password.

This should generate an authentication failure in the MySQL error log.

Check the local error log:

```bash
sudo tail -n 20 /var/log/mysql/error.log
```

Expected event example:

```text
Access denied for user 'invalid_user'@'localhost'
```

---

## Step 18: Generate a Slow Query Event

Connect to MySQL:

```bash
mysql -u root -p
```

Run the following query:

```sql
SELECT SLEEP(5);
```

Exit MySQL:

```sql
exit;
```

Verify the slow query log:

```bash
sudo tail -n 30 /var/log/mysql/mysql-slow.log
```

---

## Step 19: Verify Agent Is Reading the MySQL Logs

On the MySQL server, check the Wazuh agent log:

```bash
sudo tail -f /var/ossec/logs/ossec.log
```

The agent should show that it is monitoring the configured MySQL log files.

---

# Part G — Verify Alerts in Wazuh Dashboard

## Step 20: Open Wazuh Dashboard

Open the Wazuh Dashboard:

```text
https://<WAZUH_DASHBOARD_URL>
```

Sign in using an authorized Wazuh Dashboard account.

---

## Step 21: Open Discover

Navigate to:

```text
Discover
```

Select the Wazuh alert index pattern:

```text
wazuh-alerts-*
```

Set the time range to:

```text
Last 15 minutes
```

---

## Step 22: Filter MySQL Server Events

Use the following Discover filter to display events from the MySQL server:

```text
agent.ip: "<IP-address>"
```

You can also filter by the agent name:

```text
agent.name: "<MYSQL_AGENT_NAME>"
```

Example:

```text
agent.name: "mysql-server"
```

---

## Step 23: Search for MySQL Authentication Failures

Use the Discover search query:

```text
full_log: "Access denied"
```

Or combine it with the agent filter:

```text
agent.ip: "<IP-address>" AND full_log: "Access denied"
```

---

# Part H — Verify Alerts Directly on Wazuh Manager

## Step 24: Check Alerts File on Wazuh Manager

If Wazuh Manager is running in Docker:

```bash
docker exec -it single-node-wazuh.manager-1 sh -c 'tail -f /var/ossec/logs/alerts/alerts.json'
```

To search for MySQL server events:

```bash
docker exec -it single-node-wazuh.manager-1 sh -c 'grep -i "<IP-address>" /var/ossec/logs/alerts/alerts.json | tail -n 20'
```

To search for MySQL authentication failures:

```bash
docker exec -it single-node-wazuh.manager-1 sh -c 'grep -i "Access denied" /var/ossec/logs/alerts/alerts.json | tail -n 20'
```

---

# Part I — Firewall Requirements

Ensure the following ports are allowed between the MySQL server and Wazuh Manager.

| Source          | Destination   | Port   | Protocol | Purpose                           |
| --------------- | ------------- | ------ | -------- | --------------------------------- |
| `<IP-address>` | Wazuh Manager | `1514` | TCP      | Agent event communication         |
| `<IP-address>` | Wazuh Manager | `1515` | TCP      | Agent enrollment and registration |

Example UFW rules on the Wazuh Manager host:

```bash
sudo ufw allow from <IP-address> to any port 1514 proto tcp
sudo ufw allow from <IP-address> to any port 1515 proto tcp
```

---

# Part J — Troubleshooting

## Wazuh Agent Is Not Running

```bash
sudo systemctl status wazuh-agent
sudo journalctl -u wazuh-agent -n 100 --no-pager
```

Restart the agent:

```bash
sudo systemctl restart wazuh-agent
```

---

## MySQL Log Permission Denied

Check permissions:

```bash
sudo ls -l /var/log/mysql/
```

Add Wazuh user to the `adm` group:

```bash
sudo usermod -aG adm wazuh
sudo systemctl restart wazuh-agent
```

Test log access:

```bash
sudo -u wazuh head -n 5 /var/log/mysql/error.log
```

---

## Agent Cannot Connect to Wazuh Manager

Check connectivity from the MySQL server:

```bash
nc -vz <WAZUH_MANAGER_IP> 1514
```

```bash
nc -vz <WAZUH_MANAGER_IP> 1515
```

If `nc` is not installed:

```bash
sudo apt install netcat-openbsd -y
```

---

## Events Are Not Visible in Discover

Check the sequence in this order:

```text
1. Confirm MySQL log files contain new events.
2. Confirm Wazuh agent is running.
3. Confirm Wazuh agent can read MySQL log files.
4. Confirm agent is active in Wazuh Manager.
5. Confirm alerts are present in alerts.json on Wazuh Manager.
6. Confirm the Wazuh Indexer and Filebeat are running.
7. Check Discover with the wazuh-alerts-* index pattern and correct time range.
```

---

# Part K — Final Data Flow

```text
MySQL Error Log
MySQL General Log
MySQL Slow Query Log
        |
        v
Wazuh Agent on <IP-address>
        |
        v
Wazuh Manager
        |
        v
Wazuh Alerts Index
        |
        v
Wazuh Dashboard Discover
```

---

# Important Implementation Notes

1. Do not use Filebeat on the MySQL server for these same MySQL log files when Wazuh Agent collection is enabled, unless duplicate ingestion is intended.

2. The Wazuh agent forwards log events to the Wazuh Manager. It does not directly send MySQL logs to the Indexer.

3. Only events matching Wazuh rules at the configured alert level are written to `alerts.json` and appear in the `wazuh-alerts-*` indices.

4. If a MySQL log line is collected but does not trigger a Wazuh alert rule, it may be available only in Wazuh archives when archive logging is enabled.

5. For production monitoring, create custom Wazuh rules for:

   * Repeated MySQL authentication failures
   * MySQL access-denied events
   * Privilege changes
   * New MySQL user creation
   * Database deletion commands
   * MySQL service crashes or unexpected shutdowns
