# POC: NetFlow Pipeline 2 to Logstash and Wazuh Indexer

## 1. Objective

Demonstrate collection of NetFlow telemetry, conversion of collected flow records to NDJSON, ingestion through Logstash, storage in Wazuh Indexer, alert generation through OpenSearch Alerting, and storage of alert history in a separate index.

This POC is designed for high-volume network telemetry, where raw flow records and actionable alerts are stored separately.

```text
Network Traffic
        ↓
NetFlow Exporter
        ↓
nfcapd Collector
        ↓
nfdump Flow Files
        ↓
JSON / NDJSON Conversion
        ↓
Logstash Pipeline 1
        ↓
Wazuh Indexer — netflow-logs-*
        ↓
OpenSearch Alerting Monitor
        ↓
Trigger
        ↓
Custom Webhook Channel
        ↓
Logstash Pipeline 2 — Port 8082 -> Indexer -netflow alerts
```

---

## 2. Expected Outcome

After completing this POC:

1. NetFlow records are collected by `nfcapd`.
2. A collected flow file is converted to NDJSON format.
3. Logstash ingests the NetFlow records and enriches them with common metadata.
4. Raw flow telemetry is indexed in `netflow-logs-*`.
5. An OpenSearch monitor detects NetFlow activity.
6. The monitor sends an alert payload to Logstash Pipeline 2.
7. Logstash stores alert records in `netflow-alerts-history`.
8. Raw flow records and alert history are visible in Wazuh Dashboard.

---

## 3. Environment Details

| Component                    | Details                  |
| ---------------------------- | ------------------------ |
| NetFlow Exporter / Collector | `<IP-address>`          |
| Logstash Server              | `<IP-address>`          |
| Wazuh Indexer                | `<IP-address>:9200`     |
| NetFlow Collector Port       | UDP `2055`               |
| Alert Webhook Receiver       | TCP `8082`               |
| Raw Flow Index               | `netflow-logs-*`         |
| Alert History Index          | `netflow-alerts-history` |
| Indexer User                 | `admin`                  |

> Replace IP addresses, usernames, passwords, and index names according to the client environment. Do not expose credentials in screenshots or client-facing documents.

---

# Part A: NetFlow Collector and Exporter Setup

## Step 1: Install NetFlow collection tools

Connect to the NetFlow collector server:

```bash
ssh <username>@<IP-address>
```

Update package metadata and install the required tools:

```bash
sudo apt update
sudo apt install -y nfdump jq curl
```

Verify installation:

```bash
nfdump -V
nfcapd -V
jq --version
```

---

## Step 2: Verify the NetFlow collector is listening

Check whether UDP port `2055` is already in use:

```bash
sudo ss -ulpn | grep 2055
```

Expected example:

```text
udp   UNCONN 0 0 0.0.0.0:2055 0.0.0.0:* users:(("nfcapd",pid=<PID>,fd=<FD>))
```

If `nfcapd` is already listening, do not start a second collector. Identify the existing process:

```bash
ps -fp <PID>
```

If UDP port `2055` is free, create a storage directory and start the collector:

```bash
sudo mkdir -p /data/netflow
sudo nfcapd -D -l /data/netflow -p 2055
```

Verify that the collector is now listening:

```bash
sudo ss -ulpn | grep 2055
```

> The NetFlow exporter must be configured separately to send NetFlow records to `<IP-address>:2055/UDP`. In this POC, the collector receives those exported records.

---

## Step 3: Generate safe test traffic

Generate basic outbound traffic from the exporter/collector environment:

```bash
curl -I https://github.com
curl -I https://www.google.com
ping -c 10 8.8.8.8
```

Wait for the collector to rotate and write a flow file.

Locate the latest flow files. Use the directory where `nfcapd` is actually writing data:

```bash
sudo ls -ltr /data/netflow | tail
```

If an existing collector is writing to the default directory, use:

```bash
sudo ls -ltr /var/cache/nfdump | tail
```

Expected filename format:

```text
nfcapd.YYYYMMDDHHMM
```

---

# Part B: Convert NetFlow Data to NDJSON

## Step 4: Convert the latest flow file to JSON

Set the actual collected flow file path:

```bash
FLOW_FILE="/data/netflow/nfcapd.YYYYMMDDHHMM"
```

If your collector uses the default directory:

```bash
FLOW_FILE="/var/cache/nfdump/nfcapd.YYYYMMDDHHMM"
```

Convert the flow file to JSON:

```bash
sudo nfdump -r "$FLOW_FILE" -o json > /tmp/netflow.json
```

Verify that JSON was generated:

```bash
head -20 /tmp/netflow.json
```

---

## Step 5: Convert JSON to NDJSON

Logstash file input processes one JSON event per line. Convert the JSON array into NDJSON:

```bash
jq -c '.[]' /tmp/netflow.json > /tmp/netflow.ndjson
```

Verify the output:

```bash
head -1 /tmp/netflow.ndjson
```

Expected result: one complete JSON object per line.

Check the number of flow records:

```bash
wc -l /tmp/netflow.ndjson
```

---

## Step 6: Transfer the NDJSON file to Logstash

Copy the NDJSON file to the Logstash server:

```bash
scp /tmp/netflow.ndjson <logstash-user>@<IP-address>:/tmp/netflow.ndjson
```

On the Logstash server, verify the file:

```bash
ssh <logstash-user>@<IP-address>
wc -l /tmp/netflow.ndjson
head -1 /tmp/netflow.ndjson
```

---

# Part C: Configure Logstash Pipeline 1 — NetFlow Ingestion

## Step 7: Verify required Logstash plugins

On the Logstash server, verify the File input and OpenSearch output plugins:

```bash
sudo /usr/share/logstash/bin/logstash-plugin list --verbose | grep -E "logstash-input-file|logstash-output-opensearch|logstash-filter-mutate"
```

Expected plugins:

```text
logstash-input-file
logstash-output-opensearch
logstash-filter-mutate
```

If the OpenSearch output plugin is missing, install it:

```bash
sudo /usr/share/logstash/bin/logstash-plugin install logstash-output-opensearch
```

Restart Logstash after plugin installation:

```bash
sudo systemctl restart logstash
```

---

## Step 8: Create the NetFlow ingestion pipeline

Create the pipeline configuration file:

```bash
sudo nano /etc/logstash/conf.d/netflow.conf
```

Paste the following configuration:

```conf
input {
  file {
    path => "/tmp/netflow.ndjson"
    start_position => "beginning"
    sincedb_path => "/dev/null"
    codec => json
  }
}

filter {
  mutate {
    add_field => {
      "source_platform" => "netflow"
      "event_category"  => "network_telemetry"
      "environment"     => "poc"
      "tenant"          => "NON-AGENT-TENANT"
    }
  }

  mutate {
    replace => {
      "message" => "NetFlow src=%{src4_addr}:%{src_port} dst=%{dst4_addr}:%{dst_port} bytes=%{in_bytes} packets=%{in_packets}"
    }
  }
}

output {
  file {
    path => "/tmp/netflow-events.log"
    codec => rubydebug
  }

  opensearch {
    hosts => ["https://<IP-address>:9200"]
    user => "admin"
    password => "<INDEXER_PASSWORD>"
    index => "netflow-logs-%{+YYYY.MM.dd}"
    ssl => true
    ssl_certificate_verification => false
  }
}
```

Save and exit:

```text
Ctrl + O → Enter → Ctrl + X
```

> `sincedb_path => "/dev/null"` is appropriate for this POC because it forces Logstash to read the copied test file from the beginning each time the pipeline starts. Do not use this setting for a production continuous-ingestion design.

---

# Part D: Configure Logstash Pipeline 2 — Alert Webhook Receiver

## Step 9: Create the alert receiver pipeline

Create the pipeline configuration file:

```bash
sudo nano /etc/logstash/conf.d/netflow-alerts.conf
```

Paste the following configuration:

```conf
input {
  http {
    host => "0.0.0.0"
    port => 8082
    codec => json
  }
}

filter {
  mutate {
    add_field => {
      "alert_source"    => "opensearch-alerting"
      "source_platform" => "netflow"
      "event_category"  => "network_alert"
      "environment"     => "poc"
      "tenant"          => "NON-AGENT-TENANT"
    }
  }

  mutate {
    replace => {
      "message" => "NetFlow Alert: %{[monitor_name]} - %{[trigger_name]}"
    }
  }
}

output {
  file {
    path => "/tmp/netflow-alerts.log"
    codec => rubydebug
  }

  opensearch {
    hosts => ["https://<IP-address>:9200"]
    user => "admin"
    password => "<INDEXER_PASSWORD>"
    index => "netflow-alerts-history"
    ssl => true
    ssl_certificate_verification => false
  }
}
```

Save and exit.

---

## Step 10: Register both Logstash pipelines

Back up the existing pipeline configuration:

```bash
sudo cp /etc/logstash/pipelines.yml /etc/logstash/pipelines.yml.backup-netflow-poc
```

Edit the Logstash pipeline registration file:

```bash
sudo nano /etc/logstash/pipelines.yml
```

Add the following entries:

```yaml
- pipeline.id: netflow-ingestion
  path.config: "/etc/logstash/conf.d/netflow.conf"

- pipeline.id: netflow-alerts
  path.config: "/etc/logstash/conf.d/netflow-alerts.conf"
```

Save the file.

---

## Step 11: Validate and start the Logstash pipelines

Validate the configuration:

```bash
sudo /usr/share/logstash/bin/logstash --path.settings /etc/logstash -t
```

Expected result:

```text
Configuration OK
```

Restart Logstash:

```bash
sudo systemctl restart logstash
```

Verify service status:

```bash
sudo systemctl status logstash --no-pager
```

Verify that the alert webhook receiver is listening:

```bash
sudo ss -tlnp | grep 8082
```

Expected result:

```text
LISTEN ... *:8082 ... java
```

Check that both pipelines started:

```bash
sudo grep -E "netflow-ingestion|netflow-alerts" /var/log/logstash/logstash-plain.log | tail -20
```

---

# Part E: Validate Raw NetFlow Ingestion

## Step 12: Verify local Logstash event output

On the Logstash server:

```bash
sudo tail -n 20 /tmp/netflow-events.log
```

Expected fields include:

```text
src4_addr
src_port
dst4_addr
dst_port
in_bytes
in_packets
source_platform
event_category
```

---

## Step 13: Verify the NetFlow index

From the Logstash server or an authorized administration host:

```bash
curl -k -u admin:<INDEXER_PASSWORD> \
"https://<IP_address>:9200/_cat/indices?v" | grep netflow
```

Expected index:

```text
netflow-logs-YYYY.MM.DD
```

Verify document count:

```bash
curl -k -u admin:<INDEXER_PASSWORD> \
"https://<IP-address>:9200/netflow-logs-*/_count?pretty"
```

---

# Part F: Wazuh Dashboard UI — Data View and Raw Flow Validation

## Step 14: Create the raw NetFlow data view

1. Log in to Wazuh Dashboard.
2. Navigate to:

```text
Dashboard Management → Index patterns - create index pattern
```

3. Define the index pattern
4. Enter the data view name:

```text
netflow-logs-*
```

5. Select:

```text
@timestamp
```

as the time field.
6. Click **create index pattern**.

---

## Step 15: View raw NetFlow telemetry

1. Navigate to:

```text
Discover
```

2. Select the data view:

```text
netflow-logs-*
```

3. Set the time range to:

```text
Last 24 hours
```

4. search these fields in the table:

```text
src4_addr
src_port
dst4_addr
dst_port
in_bytes
in_packets
message
```

This confirms that raw NetFlow telemetry is indexed and searchable.

---

# Part G: Wazuh Dashboard UI — Create Monitor and Trigger

## Step 16: Create the NetFlow activity monitor

1. Navigate to:

```text
Explore->Alerting → Monitors
```

2. Click **Create monitor**.
3. Select:

```text
Per query monitor
```

4. Configure the monitor:

| Field           | Value                      |
| --------------- | -------------------------- |
| Monitor name    | `NetFlow Activity Monitor` |
| Defining method | Visual editor              |
| Schedule        | Every 1 minute             |
| Index           | `netflow-logs-*`           |
| Time field      | `@timestamp`               |
| Query           | Match all                  |

5. Save the monitor configuration.

---

## Step 17: Create the monitor trigger

1. Click **Add trigger**.
2. Configure:

| Field             | Value                  |
| ----------------- | ---------------------- |
| Trigger name      | `NetFlow Test Trigger` |
| Severity          | `3`                    |
| Trigger condition | Count is above         |
| Value             | `0`                    |
| Metric            | Count of documents     |

3. Save the trigger.

This POC trigger activates when one or more NetFlow records are found during the monitor evaluation period.

---

# Part H: Wazuh Dashboard UI — Webhook Channel and Action

## Step 18: Create the webhook notification channel

1. Navigate to:

```text
Notifications → Channels
```

2. Click **Create channel**.
3. Select:

```text
Custom webhook
```

4. Configure:

| Field        | Value                            |
| ------------ | -------------------------------- |
| Channel name | `netflow-alert-webhook`          |
| Host         | `<IP-address>`                  |
| Port         | `8082`                           |
| Path         | `/`                              |
| Method       | `POST`                           |
| Header       | `Content-Type: application/json` |

5. Save the channel.

> Confirm that the Wazuh Indexer can reach `<IP-address>:8082`. For production, secure this endpoint with TLS, authentication, and network restrictions.

---

## Step 19: Add the monitor action

1. Open the `NetFlow Activity Monitor`.
2. Open the `NetFlow Test Trigger`.
3. Under **Actions**, click **Add action**.
4. Configure:

| Field       | Value                            |
| ----------- | -------------------------------- |
| Action name | `Send NetFlow Alert to Logstash` |
| Channel     | `netflow-alert-webhook`          |

5. Use this message body:

```json
{
  "monitor_name": "{{ctx.monitor.name}}",
  "trigger_name": "{{ctx.trigger.name}}",
  "severity": "{{ctx.trigger.severity}}",
  "period_start": "{{ctx.periodStart}}",
  "period_end": "{{ctx.periodEnd}}",
  "message": "NetFlow monitor triggered"
}
```

6. Save the action and trigger.

---

# Part I: Validate Alert Receiver and Alert History

## Step 20: Test the alert receiver manually

On the Logstash server, run:

```bash
curl -X POST http://localhost:8082 \
-H "Content-Type: application/json" \
-d '{
  "monitor_name": "NetFlow Test",
  "trigger_name": "Manual Test",
  "severity": "3"
}'
```

Expected response:

```text
ok
```

Verify that Logstash received the alert:

```bash
sudo tail -n 20 /tmp/netflow-alerts.log
```

---

## Step 21: Verify alert-history indexing

Verify the document count:

```bash
curl -k -u admin:<INDEXER_PASSWORD> \
"https://<IP-address>:9200/netflow-alerts-history/_count?pretty"
```

View recent alert documents:

```bash
curl -k -u admin:<INDEXER_PASSWORD> \
"https://<IP-address>:9200/netflow-alerts-history/_search?pretty" \
-H 'Content-Type: application/json' \
-d '{
  "size": 10,
  "sort": [
    {
      "@timestamp": {
        "order": "desc"
      }
    }
  ]
}'
```

---

## Step 22: Create the alert-history data view

1. Navigate to:

```text
Dashboard Management → Index Patterns->create index pattern
```

2. Click **Create index pattern**.
3. Define index pattern:Enter:

```text
netflow-alerts-history
```

4. Select:

```text
@timestamp
```

as the time field.
5. Click **Create index pattern**.

Open **Discover**, select `netflow-alerts-history`, and verify the alert event.

---

# Part J: Final End-to-End Demonstration

## Step 23: Generate new traffic and ingest a fresh flow file

1. Generate test traffic again on the exporter or collector:

```bash
curl -I https://github.com
ping -c 10 8.8.8.8
```

2. Wait for a new flow file.
3. Convert the new flow file to JSON and NDJSON.
4. Copy the new NDJSON file to `/tmp/netflow.ndjson` on the Logstash server.
5. Restart Logstash to force the POC file input pipeline to read the file again:

```bash
sudo systemctl restart logstash
```

6. Wait for the monitor schedule to run.

---

## Step 24: Verify the complete flow

Verify raw events:

```bash
sudo tail -n 20 /tmp/netflow-events.log
```

Verify alert webhook events:

```bash
sudo tail -n 20 /tmp/netflow-alerts.log
```

Verify raw NetFlow records in Dashboard:

```text
Discover → netflow-logs-*
```

Verify alert records in Dashboard:

```text
Discover → netflow-alerts-history
```

---

# POC Success Criteria

| Validation   | Expected Result                                                   |
| ------------ | ----------------------------------------------------------------- |
| Collector    | `nfcapd` listens on UDP port `2055`                               |
| Flow capture | A flow file is created by the collector                           |
| Conversion   | The flow file is converted to valid NDJSON                        |
| Pipeline 1   | Logstash reads `/tmp/netflow.ndjson`                              |
| Raw index    | NetFlow records are stored in `netflow-logs-*`                    |
| Dashboard    | Raw flows are visible in `netflow-logs-*` data view               |
| Monitor      | NetFlow monitor triggers when flow records exist                  |
| Pipeline 2   | Alert payload reaches Logstash on port `8082`                     |
| Alert index  | Alert records are stored in `netflow-alerts-history`              |
| Final proof  | Raw flows and alerts are visible in separate Dashboard data views |

---

# Troubleshooting

| Issue                                   | Validation and Resolution                                                                                                                                                              |
| --------------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| UDP port `2055` is already in use       | Run `sudo ss -ulpn \| grep 2055`; reuse the existing `nfcapd` process instead of starting another collector.                                                                           |
| No flow file is created                 | Confirm that the exporter sends NetFlow to `<IP-address>:2055/UDP`; generate traffic and wait for flow-file rotation.                                                                 |
| Logstash configuration validation fails | Run `sudo /usr/share/logstash/bin/logstash --path.settings /etc/logstash -t` and correct the reported pipeline syntax error.                                                           |
| Logstash `path.data` is locked          | Do not run a second Logstash process. Stop manual processes, (sudo systemctl stop logstash)use validation mode only, then restart through `systemctl`.                                 |
| Dead-letter queue permission error      | Run `sudo chown -R logstash:logstash /var/lib/logstash/dead_letter_queue/netflow-alerts`, remove stale lock files only after confirming Logstash is stopped, then restart the service. |
| Port `8082` is not listening            | Run `sudo systemctl status logstash --no-pager` and `sudo grep netflow-alerts /var/log/logstash/logstash-plain.log`.                                                                   |
| Monitor count is zero                   | Confirm new documents exist in `netflow-logs-*`; widen the monitor lookback period or ingest fresh NetFlow records.                                                                    |
| Alert history is empty                  | Test the receiver manually with `curl`, then verify the webhook channel host, port, path, and Indexer-to-Logstash network connectivity.                                                |

---

# Business Use Case

This POC demonstrates a scalable architecture for handling high-volume network telemetry separately from actionable alerts.

```text
Network Traffic
        ↓
NetFlow Exporter
        ↓
nfcapd Collector
        ↓
nfdump Flow Files
        ↓
JSON / NDJSON Conversion
        ↓
Logstash Pipeline 1
        ↓
Wazuh Indexer — netflow-logs-*
        ↓
OpenSearch Alerting Monitor
        ↓
Trigger
        ↓
Custom Webhook Channel
        ↓
Logstash Pipeline 2 — Port 8082 -> Indexer -netflow alerts
```

Typical use cases include:

```text
Unexpected outbound traffic
Large data-transfer detection
Unusual source-to-destination communication
New external destination monitoring
High packet-volume detection
Potential data-exfiltration investigation
Network traffic baselining
```
