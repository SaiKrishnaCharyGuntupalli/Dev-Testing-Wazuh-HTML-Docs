# Complete Step-by-Step SOC Logstash Pipeline-to-Pipeline (P2P) Architecture Implementation Guide

---

## Step 1 — Objective of the Task

The main objective of this implementation was to build a production-style SOC log processing architecture using Logstash Pipeline-to-Pipeline (P2P) communication. Instead of using one large Logstash pipeline, the architecture was divided into multiple pipelines to improve scalability, troubleshooting, monitoring, and fault isolation.

**What is P2P architecture**

Pipeline-to-Pipeline architecture splits processing into multiple pipelines where each pipeline handles a specific task. This improves modularity, makes debugging easier, and allows independent scaling and monitoring.

**What pipelines we used**

I used four pipelines: Ingestion for parsing logs, Normalization for field standardization, Enrichment for detection logic and Routing for filtering and sending output.

**Where I configured pipelines**

I configured multiple pipelines in pipelines.yml, and each pipeline has its own configuration file inside `/etc/logstash/conf.d/`.

---

## Step 2 — Understanding Pipeline-to-Pipeline (P2P) Architecture

Pipeline-to-Pipeline (P2P) architecture means one Logstash pipeline internally forwards logs to another Logstash pipeline instead of processing everything inside one configuration file.

**Architecture flow used:**

```
Realtime Logs
↓
Filebeat
↓
Ingestion Pipeline
↓
Normalization Pipeline
↓
Enrichment Pipeline
↓
Routing Pipeline
↓
Grafana Visualization
```

**Purpose of using P2P:**
1. modular architecture
2. easier troubleshooting
3. better scalability
4. pipeline isolation
5. independent monitoring
6. easier failure identification (this is the main)

---

## Step 3 — Realtime Log Generation

Realtime logs were generated continuously to simulate endpoint activity. Different types of logs such as:
1. ERROR logs
2. INFO logs
3. DEBUG logs
4. failed authentication logs

were continuously written into:

```
/home/vahandjango/test_eps.log
```

**Command used:**

```bash
nohup bash -c '
while true; do
echo "ERROR: DB failed $RANDOM" >> /home/vahandjango/test_eps.log
echo "INFO: session started from 192.168.1.$((RANDOM%255))" >> /home/vahandjango/test_eps.log
echo "DEBUG: connection attempt from 192.168.1.$((RANDOM%255))" >> /home/vahandjango/test_eps.log
echo "ERROR: auth failed from 192.168.1.$((RANDOM%255))" >> /home/vahandjango/test_eps.log
sleep 0.5
done
' &
```

**Purpose:**
1. simulate realtime endpoint logs
2. continuously generate traffic
3. test pipeline flow under continuous ingestion

---

## Step 4 — Filebeat Configuration

Filebeat was configured to monitor the generated log file and send logs to Logstash.

**Configuration used:**

```yaml
filebeat.inputs:
- type: log
  enabled: true
  paths:
  - /home/vahandjango/test_eps.log

output.logstash:
  hosts: ["192.168.35.37:5044"]

logging.level: info
```

**Purpose of Filebeat:**
1. lightweight log shipper
2. continuous realtime forwarding
3. reliable delivery to Logstash

Filebeat continuously monitored the log file and pushed logs into Logstash ingestion pipeline.

---

## Step 5 — Creation of Multiple Pipelines

Four separate Logstash pipelines were created:
1. ingestion
2. normalization
3. enrichment
4. routing

These pipelines were configured inside:

```
/etc/logstash/pipelines.yml
```

**Configuration:**

```yaml
- pipeline.id: ingestion
  path.config: "/etc/logstash/conf.d/ingestion.conf"

- pipeline.id: normalization
  path.config: "/etc/logstash/conf.d/normalization.conf"

- pipeline.id: enrichment
  path.config: "/etc/logstash/conf.d/enrichment.conf"

- pipeline.id: routing
  path.config: "/etc/logstash/conf.d/routing.conf"
```

**Purpose:**
1. separate responsibilities
2. isolate failures
3. monitor pipelines independently

---

## Step 6 — Ingestion Pipeline Implementation

The ingestion pipeline acted as the entry point for all logs coming from Filebeat.

**Configuration:**

```ruby
input {
  beats {
    port => 5044
  }
}
filter {
  grok {
    match => {
      "message" => "%{GREEDYDATA:log_message}"
    }
  }
}
output {
  pipeline {
    send_to => "normalization"
  }
}
```

**Purpose of Grok Filter:**
1. parse incoming raw logs
2. extract message structure
3. prepare logs for downstream processing

**Why Grok was used:**
1. widely used in SOC parsing
2. useful for pattern extraction
3. helps normalize unstructured logs

---

## Step 7 — Normalization Pipeline

The normalization pipeline standardized log fields into a common schema format.

**Configuration:**

```ruby
input {
  pipeline {
    address => "normalization"
  }
}
filter {
  mutate {
    add_field => {
      "log_type" => "application"
    }
  }
}
output {
  pipeline {
    send_to => "enrichment"
  }
}
```

**Purpose:**
1. normalize field names
2. create consistent schema
3. prepare logs for enrichment

**Why Mutate Filter was used:**
1. It is lightweight
2. efficient field modification
3. field renaming and additions

---

## Step 8 — Enrichment Pipeline Implementation

The enrichment pipeline was used to add additional context and detection-related information into logs after normalization. This stage is very important in SOC architectures because raw logs alone are not enough for security analysis.

**Configuration:**

```ruby
input {
  pipeline { address => "enrichment" }
}
filter {
  # Example enrichment (tagging)
  if [log_type] == "auth" and "failed" in [message] {
    mutate {
      add_field => { "alert" => "FAILED_LOGIN" }
    }
  }
  # Example aggregation (simple)
  if [source][ip] {
    mutate {
      add_tag => ["has_ip"]
    }
  }
  # (Optional test failure --- REMOVE in real)
  # ruby {
  #   code => "event.set('test', 1/0)"
  # }
}
output {
  pipeline { send_to => "routing" }
}
```

**Severity Classification Configuration:**

```ruby
input {
  pipeline {
    address => "enrichment"
  }
}
filter {
  if "ERROR" in [message] {
    mutate {
      add_field => {
        "severity" => "high"
      }
    }
  }
  else {
    mutate {
      add_field => {
        "severity" => "low"
      }
    }
  }
}
output {
  pipeline {
    send_to => "routing"
  }
}
```

**Purpose of Enrichment Pipeline:**
1. add security context
2. identify severity
3. prepare logs for SOC analysis
4. classify important events
5. improve detection visibility

**Why enrichment is important:**
1. raw logs alone are difficult to analyze
2. enrichment improves threat visibility
3. security teams can prioritize alerts faster
4. helps SIEM correlation and filtering

**Explanation:**
If logs contained ERROR messages, the pipeline automatically tagged them with high severity. Other logs were marked as low severity. This simulates SOC detection logic where suspicious or failed events receive higher priority.

---

## Step 9 — Routing Pipeline Implementation

The routing pipeline was responsible for final filtering and sending logs to output destinations.

**Configuration:**

```ruby
input {
  pipeline { address => "routing" }
}
filter {
  if "DEBUG" in [message] {
    drop { }
  }
}
output {
  file {
    path => "/var/log/session_tracking.log"
    codec => rubydebug
  }
}
```

**Purpose:**
1. final output handling
2. centralized log storage
3. output isolation
4. separate processing from delivery layer

**Why routing pipeline is important:**
1. output failures stay isolated
2. easier troubleshooting
3. independent output scaling
4. prevents entire pipeline failure

**Explanation:**
Instead of sending logs directly from enrichment stage, a separate routing pipeline handled output delivery. This is useful in production SOC environments because output systems like Elasticsearch, Wazuh. But also here all logs are stored in `/var/log/session_tracking.log` which was created for in Logstash output so we can see properly logs are going out or not.

---

## Step 10 — Pipeline Verification and Monitoring

After creating all pipelines, Logstash pipeline statistics API was used to verify that all pipelines were running correctly.

**Command used:**

```bash
curl -s localhost:9600/_node/pipelines?pretty
```

**Purpose:**
1. verify all pipelines are active
2. monitor workers and queues
3. validate pipeline isolation
4. check pipeline health

**What was verified:**
1. pipeline IDs
2. pipeline workers
3. batch sizes
4. queue configuration
5. pipeline statistics

This confirmed that all pipelines were running independently in modular P2P architecture.

---

## Step 11 — Persistent Queue (PQ) Configuration

Persistent Queue (PQ) was enabled to prevent data loss during high EPS or output failures.

**Configuration added inside pipelines.yml:**

```yaml
queue.type: persisted
queue.max_bytes: 1gb
pipeline.workers: 2
pipeline.batch.size: 125
```

**Purpose of Persistent Queue:**
1. prevent event loss
2. store events temporarily on disk
3. handle output failures safely
4. improve reliability during spikes

**Why PQ is important:**
1. if destination becomes unavailable logs are buffered
2. protects realtime SOC pipelines
3. improves fault tolerance
4. supports recovery after restart

**Explanation:**
When downstream systems become slow or unreachable, Logstash stores events temporarily inside persistent queues instead of immediately dropping logs.

---

## Step 13 — Dead Letter Queue (DLQ) Reprocessing Pipeline

After configuring the Dead Letter Queue (DLQ), a dedicated DLQ Reprocessing Pipeline was created to automatically read failed events stored in the DLQ and send them back through Logstash for successful indexing.

This ensures that events which previously failed due to mapping conflicts or indexing errors are not permanently lost.

**Purpose:**
1. Reprocess failed events stored inside the Dead Letter Queue.
2. Recover logs that failed during indexing.
3. Prevent permanent log loss.
4. Improve overall pipeline reliability.

### Create DLQ Reprocessing Pipeline

Create a new Logstash pipeline configuration file.

```bash
sudo nano /etc/logstash/conf.d/dlq-reprocessing.conf
```

Add the following configuration:

```ruby
input {
  dead_letter_queue {
    path => "/var/lib/logstash/dead_letter_queue"
    pipeline_id => "main"
    commit_offsets => true
  }
}
filter {
  # Optional filter section
  # Modify or clean failed events before reprocessing if required.
}
output {
  elasticsearch {
    hosts => ["http://localhost:9200"]
    index => "reprocessed-logs-%{+YYYY.MM.dd}"
  }
  stdout {
    codec => rubydebug
  }
}
```

### Register the Pipeline

Open the Logstash pipelines configuration file.

```bash
sudo nano /etc/logstash/pipelines.yml
```

Add the following pipeline entry:

```yaml
- pipeline.id: dlq-reprocessing
  path.config: "/etc/logstash/conf.d/dlq-reprocessing.conf"
  queue.type: persisted
  queue.max_bytes: 1gb
  pipeline.workers: 2
  pipeline.batch.size: 125
```

### Validate the Configuration

Before restarting Logstash, verify that the configuration contains no syntax errors.

```bash
sudo /usr/share/logstash/bin/logstash --config.test_and_exit
```

**Expected Output:**

```
Configuration OK
```

### Restart Logstash

```bash
sudo systemctl restart logstash
```

Verify that the Logstash service is running successfully.

```bash
sudo systemctl status logstash
```

**Expected Output:**

```
Active: active (running)
```

### Verify Pipeline Creation

Check whether the DLQ Reprocessing Pipeline has been loaded successfully.

```bash
curl -s localhost:9600/_node/pipelines?pretty
```

**Expected Result:**

The output should include the following pipeline.

```
pipeline.id : dlq-reprocessing
```

### Working Flow

```
Elasticsearch Rejects Event
│
▼
Dead Letter Queue (DLQ)
│
▼
DLQ Reprocessing Pipeline
│
▼
Elasticsearch
```

### Validation

To validate the implementation:
1. Generate an event that causes an Elasticsearch indexing failure.
2. Confirm that the event is stored inside the Dead Letter Queue.
3. Start the DLQ Reprocessing Pipeline.
4. Verify that the failed event is successfully reprocessed.
5. Confirm that the event appears in Elasticsearch under the configured index.

This confirms that the DLQ Reprocessing Pipeline is functioning correctly and can recover failed events without data loss.

**Next Step:** After validating the Dead Letter Queue Reprocessing Pipeline, the next objective was to monitor the Logstash server and pipeline performance using Node Exporter, Prometheus, and Grafana.

---

## Step 14 — Infrastructure Monitoring Setup

After successfully implementing the Logstash Pipeline-to-Pipeline architecture along with Persistent Queue (PQ) and Dead Letter Queue (DLQ), the next requirement was to monitor the health and performance of the Logstash server.

To achieve this, a complete monitoring stack consisting of Node Exporter, Prometheus, and Grafana was implemented. This monitoring stack provides real-time visibility into system resources and Logstash performance.

### Components Used

1. Node Exporter
2. Prometheus
3. Grafana

### Monitoring Objectives

1. Monitor CPU utilization.
2. Monitor Memory utilization.
3. Monitor Disk usage.
4. Monitor Network statistics.
5. Monitor Logstash performance.
6. Monitor Pipeline throughput.
7. Monitor Events Per Second (EPS).
8. Monitor JVM utilization.

### Monitoring Architecture

```
Linux VM (Logstash Server)
│
▼
Node Exporter
│
▼
Exposes Metrics (:9100)
│
▼
Prometheus
│
▼
Scrapes Metrics Periodically
│
▼
Grafana
│
▼
Real-Time Monitoring Dashboard
```

During the setup, four separate terminal windows were used to independently run and verify each monitoring component.

| Terminal   | Purpose                              |
|------------|--------------------------------------|
| Terminal 1 | Node Exporter                        |
| Terminal 2 | Prometheus                           |
| Terminal 3 | Logstash Monitoring / Metrics Verification |
| Terminal 4 | Grafana Dashboard                    |

Next Step: The first component of the monitoring stack is Node Exporter, which collects infrastructure metrics from the Linux server.

---

## Step 15 — Node Exporter Installation and Configuration

Node Exporter is an open-source monitoring agent developed by Prometheus. It collects operating system metrics from the Linux server where Logstash is installed. These metrics are exposed through an HTTP endpoint and later collected by Prometheus.

### Why Node Exporter?

Node Exporter was installed to monitor the infrastructure resources of the Logstash server, including CPU utilization, memory utilization, disk usage, network statistics, filesystem usage, and system load.

Without Node Exporter, Prometheus cannot collect operating system metrics.

### Installation Commands

**Install Node Exporter**

```bash
cd /opt
wget https://github.com/prometheus/node_exporter/releases/download/v1.8.1/node_exporter-1.8.1.linux-amd64.tar.gz
```

**Extract**

```bash
tar -xvf node_exporter-1.8.1.linux-amd64.tar.gz
```

**Move Binary**

```bash
sudo mv node_exporter-1.8.1.linux-amd64/node_exporter /usr/local/bin/
```

**Create Service User**

```bash
sudo useradd --no-create-home --shell /bin/false node_exporter
```

**Create Service**

```bash
sudo vi /etc/systemd/system/node_exporter.service
```

```ini
[Unit]
Description=Node Exporter

[Service]
User=node_exporter
ExecStart=/usr/local/bin/node_exporter

[Install]
WantedBy=multi-user.target
```

**Reload**

```bash
sudo systemctl daemon-reload
```

**Enable**

```bash
sudo systemctl enable node_exporter
```

**Start**

```bash
sudo systemctl start node_exporter
```

**Verify**

```bash
sudo systemctl status node_exporter
```

**Open Browser**

```
http://<SERVER-IP>:9100/metrics
```

Thousands of metrics should be displayed, confirming that Node Exporter is successfully exposing server metrics.

**Next Step:** After successfully exposing infrastructure metrics using Node Exporter, the next step is to install Prometheus to collect and store these metrics.

---

## Step 16 — Install Prometheus

**Why Prometheus?**
Node Exporter only exposes metrics and does not store them. Prometheus periodically collects (scrapes) these metrics and stores them in its time-series database. These stored metrics are later visualized in Grafana.

**Download**

```bash
cd /opt
wget https://github.com/prometheus/prometheus/releases/download/v2.53.0/prometheus-2.53.0.linux-amd64.tar.gz
```

**Extract**

```bash
tar -xvf prometheus-2.53.0.linux-amd64.tar.gz
```

**Move**

```bash
sudo mv prometheus-2.53.0.linux-amd64 /opt/prometheus
```

**Open Configuration**

```bash
sudo nano /opt/prometheus/prometheus.yml
```

Add the Node Exporter target:

```yaml
scrape_configs:
  - job_name: "node_exporter"
    static_configs:
      - targets:
          - localhost:9100
```

**Start Prometheus**

```bash
cd /opt/prometheus
./prometheus \
  --config.file=prometheus.yml
```

**Open**

```
http://<SERVER-IP>:9090
```

**Verify**

```
Status → Targets
Expected: node_exporter UP
```

This confirms Prometheus is successfully collecting metrics from Node Exporter.

**Next Step:** After Prometheus begins collecting metrics, the next step is to install Grafana to visualize these metrics through dashboards.

---

## Step 17 — Grafana Installation and Configuration

## Why Grafana?

Although Prometheus stores metrics, it cannot provide rich visual dashboards. Grafana connects to Prometheus and displays the collected metrics through graphs, gauges, and monitoring panels. This enables administrators to monitor Logstash and overall server health in real time.

## Install Grafana

Updates the package repository.

```bash
sudo apt update
```

Installs the Grafana package from the Ubuntu repository.

```bash
sudo apt install grafana -y
```

**Enable Grafana Service**
Enables the Grafana service so it starts automatically after every system reboot.

```bash
sudo systemctl enable grafana-server
```

**Start Grafana Service**
Starts the Grafana service immediately without requiring a server restart.

```bash
sudo systemctl start grafana-server
```

## Verify Grafana Service

Checks the current status of the Grafana service and confirms that it is running successfully.

```bash
sudo systemctl status grafana-server
```

---

## Step 19 — Connect Grafana with Prometheus

After installing Grafana, it must be connected to Prometheus so that infrastructure metrics can be visualized through dashboards.

### Navigation

```
Connections
↓
Data Sources
↓
Add Data Source
↓
Prometheus
```

### Configuration

| Field      | Value                   |
|------------|-------------------------|
| Server URL | http://localhost:9090   |

Click **Save & Test**.

### Expected Result

```
Data source is working
```

This confirms that Grafana has successfully connected to Prometheus and can retrieve metrics.

---

## Step 20 — Logstash Pipeline Monitoring Using Grafana

**Objective**
After successfully installing Node Exporter, Prometheus, and Grafana, the next objective was to monitor the Logstash Pipeline-to-Pipeline (P2P) architecture in real time.

Instead of checking Logstash logs manually, Grafana was configured to visualize pipeline metrics collected by Prometheus. This allowed real-time monitoring of every Logstash pipeline, making it easier to verify pipeline health, identify failures, and analyze performance.

### Why We Used Grafana?

Grafana was implemented to provide a centralized dashboard for monitoring Logstash pipeline performance. The dashboard enables administrators to:
1. Monitor each pipeline independently.
2. Detect pipeline failures immediately.
3. Observe event flow through every processing stage.
4. Verify that all pipelines are processing logs correctly.
5. Analyze throughput and pipeline performance.

Without Grafana, pipeline statistics would have to be checked manually using the Logstash Metrics API.

### Monitoring Architecture

```
Linux Logstash Server
↓
Node Exporter
↓
Prometheus
↓
Collects Metrics
↓
Grafana
↓
Visualizes Pipeline Metrics
```

**Metrics Collected**

Prometheus continuously collected Logstash pipeline metrics such as:
- Events Received
- Events Processed
- Events Output
- Pipeline Throughput
- Pipeline Health
- Pipeline Status
- Worker Performance
- Queue Utilization

These metrics were visualized inside Grafana using Time Series graphs.

**Creating the Pipeline Monitoring Dashboard**
After connecting Grafana with Prometheus, a new dashboard named Logstash Pipeline Monitoring was created. Inside this dashboard, separate visualization panels were configured to monitor every Logstash pipeline.

### Creating the Panel

**Navigate to:**

```
Grafana
↓
Dashboards
↓
New Dashboard
↓
Add Visualization
↓
Select Prometheus Data Source
```

The visualization type selected was: **Time Series** because pipeline metrics continuously change over time.

### Configuring the Prometheus Query

To monitor the number of events entering every Logstash pipeline, the following PromQL query was configured.

**Query**

```
{__name__=~"logstash_.*_events_in"}
```

### Why This Query Was Used?

`logstash_` — Selects only Logstash metrics.

`.*` — Matches every Logstash pipeline.

**Examples:**
1. logstash_ingestion_events_in
2. logstash_normalization_events_in
3. logstash_enrichment_events_in
4. logstash_routing_events_in

`events_in` — Displays the number of events entering each pipeline. Therefore, one PromQL query automatically monitors every pipeline without creating separate queries for each stage.

## Result

Grafana automatically displayed four different metrics:
1. logstash_ingestion_events_in
2. logstash_normalization_events_in
3. logstash_enrichment_events_in
4. logstash_routing_events_in

Each metric appeared as a separate line on the graph. This allowed real-time comparison of pipeline activity.

**Understanding the Graph**
Each colored line represents one Logstash pipeline.

| Color  | Pipeline               |
|--------|------------------------|
| Blue   | Normalization Pipeline |
| Orange | Routing Pipeline       |
| Yellow | Ingestion Pipeline     |
| Green  | Enrichment Pipeline    |

As logs pass through Logstash, every pipeline processes events independently, causing each graph to increase over time. If all pipelines are functioning correctly, all graphs continue increasing.

### Pipeline Failure Testing

To verify monitoring accuracy, one pipeline was intentionally stopped.

**Pipeline Stopped:** Enrichment Pipeline

After stopping the pipeline:
1. Ingestion Pipeline continued processing logs.
2. Normalization Pipeline continued processing logs.
3. Routing Pipeline continued processing logs.
4. Enrichment Pipeline stopped receiving events.

## Observation

Inside Grafana, the Enrichment Pipeline graph immediately became a straight horizontal line. The remaining pipeline graphs continued increasing. This confirmed that Grafana was successfully monitoring each pipeline independently.

### Why This Test Was Performed?

The objective of this validation was to verify that:
1. Every pipeline is monitored separately.
2. Pipeline failures can be identified immediately.
3. Monitoring is independent for each processing stage.
4. Administrators can quickly locate failed pipelines.

## Verification

The following conditions were verified successfully:

1. Ingestion Pipeline receiving events.
2. Normalization Pipeline processing events.
3. Enrichment Pipeline stopped.
4. Routing Pipeline processing events.
5. Grafana immediately reflected pipeline failure.
6. Prometheus continuously updated metrics.

## Benefits

Using Grafana for pipeline monitoring provides several advantages:
1. Real-time monitoring.
2. Easy visualization.
3. Immediate failure detection.
4. Pipeline performance analysis.
5. Historical metric analysis.
6. Simplified troubleshooting.
7. Better operational visibility.

### Overall Monitoring Flow

```
Application Logs
↓
Filebeat
↓
Logstash Ingestion Pipeline
↓
Normalization Pipeline
↓
Enrichment Pipeline
↓
Routing Pipeline
↓
Elasticsearch
↓
Logstash Metrics API
↓
Prometheus
↓
Grafana Dashboard
```

---

![Grafana Dashboard - Pipeline Monitoring](<../../../assets/images/POC's/Yaseen/SOC Logstash Pipeline-to-Pipeline (P2P) Architecture Implementation Guide/grafana part.png>)

**Grafana dashboard displaying real-time event counts for the Ingestion, Normalization, Enrichment, and Routing pipelines using the `logstash_.*_events_in` Prometheus query.**

---

![Grafana Dashboard - Pipeline Failure Validation](<../../../assets/images/POC's/Yaseen/SOC Logstash Pipeline-to-Pipeline (P2P) Architecture Implementation Guide/grafana part.png>)

**Validation of pipeline monitoring by intentionally stopping the Enrichment Pipeline. Grafana immediately reflected the failure by displaying a flat line for the Enrichment Pipeline while the remaining pipelines continued processing events normally.**
