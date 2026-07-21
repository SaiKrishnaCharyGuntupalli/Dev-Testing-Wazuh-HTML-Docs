# PROMETHEUS 

## 1. WHAT IS PROMETHEUS?

**Prometheus** is an **open-source monitoring and alerting tool** mainly used to track system performance like:

- CPU usage
- RAM usage
- Disk usage
- Application metrics
- Server health

It is widely used in DevOps, cloud systems, and tools like **Kubernetes**.

### 1.1 Why Do We Need Prometheus?

Imagine your Wazuh setup:

- You send logs → system processes → dashboard shows data
- But how do you know:
  - CPU is overloaded?
  - RAM is getting full?
  - System is slow?

Prometheus helps you **monitor all this in real-time**.

#### Real-life Example:

- You deployed Wazuh on a VM
- You send heavy logs
- Suddenly system slows down

Prometheus shows:

- CPU = 95%
- RAM = 90%

So you understand your system capacity.

### 1.2 How Prometheus Works (Simple Flow)

1. Exporter exposes metrics
2. Prometheus **pulls data (scrapes)** periodically
3. Stores data in a **time-series database**
4. You query and visualize data

### 1.3 Visualization Tool

Prometheus UI is basic. For advanced dashboards use:

**Grafana**

- Beautiful dashboards
- Charts & panels
- Used with Prometheus

### 1.4 Is Prometheus Free?

Yes — **100% FREE & Open Source**

- No license cost
- Backed by the **Cloud Native Computing Foundation**

### 1.5 What You Get by Using Prometheus

- Real-time monitoring
- Performance analysis
- System capacity testing
- Alerts before failure
- Historical data tracking

### 1.6 Example for Wazuh Testing

1. Install Prometheus + Node Exporter
2. Start sending logs to Wazuh
3. Monitor:
   a. CPU usage
   b. Memory usage
4. Compare:
   a. Without load
   b. With heavy load

Then report:

- System capacity
- Breaking point
- Performance trend

### 1.7 Simple Summary

- Prometheus = Monitoring tool
- Pulls metrics from systems
- Stores time-based data
- Shows graphs & alerts
- Free and widely used

---

## 2. WHAT IS NODE EXPORTER?

##### Simple meaning:

It is a **small tool that exposes system data**.

##### Without Node Exporter:

Prometheus cannot see CPU, RAM.

##### With Node Exporter:

Prometheus can read:

```
CPU = 40%
RAM = 60%
Disk = 70%
```

---

## 3. STEP-BY-STEP INSTALLATION

### 3.1 Install Prometheus

#### First navigate to the /opt directory in the Linux VM

##### Step 1: Download

```bash
wget https://github.com/prometheus/prometheus/releases/latest/download/prometheus-2.51.2.linux-amd64.tar.gz
```

Downloads the Prometheus package.

##### Step 2: Extract

```bash
tar -xvf prometheus-2.51.2.linux-amd64.tar.gz
cd prometheus-2.51.2.linux-amd64
```

Extracts files and moves into the folder.

##### Step 3: Run Prometheus

```bash
./prometheus --config.file=prometheus.yml
```

Starts the Prometheus server.

##### Access UI:

```
http://<Manager_VM_IP>:9090
```

##### **Challenge:**

When I run the Step 1 command, I faced an issue related to the version.

##### **Solution:**

I found a stable version, then downloaded that stable version and followed the above process.

**Working link:**

```bash
wget https://github.com/prometheus/prometheus/releases/download/v2.47.2/prometheus-2.47.2.linux-amd64.tar.gz
```

### 3.2 Install Node Exporter

Node Exporter is used to expose the system data of a Linux VM only. For a Windows machine we must install `windows_exporter`.

- **Prometheus** → central server (stores & shows metrics)
- **Node Exporter** → installed on each Linux VM
- **windows_exporter** → installed on a Windows machine

Prometheus **pulls (scrapes)** data from all these exporters.

##### Step 1: Download

```bash
wget https://github.com/prometheus/node_exporter/releases/latest/download/node_exporter-1.8.1.linux-amd64.tar.gz
```

##### Step 2: Extract

```bash
tar -xvf node_exporter-1.8.1.linux-amd64.tar.gz
cd node_exporter-1.8.1.linux-amd64
```

##### Step 3: Run

```bash
./node_exporter
```

Runs on:

```
http://<IP>:9100/metrics
```

##### **Challenge:**

When I run the Step 1 command, I faced an issue related to the version.

##### **Solution:**

I found a stable version, then downloaded that stable version and followed the above process.

**Working link:**

```bash
wget https://github.com/prometheus/node_exporter/releases/download/v1.7.0/node_exporter-1.7.0.linux-amd64.tar.gz
```

### 3.3 Configure Prometheus

Edit file:

```bash
nano prometheus.yml
```

##### Add this:

```yaml
scrape_configs:
  - job_name: "manager"
    static_configs:
      - targets: ["localhost:9100"]

  - job_name: "agent_vm"
    static_configs:
      - targets: ["<Agent_VM_IP>:9100"]
```

##### Restart Prometheus

```bash
pkill prometheus
./prometheus --config.file=prometheus.yml
```

##### **Challenge:**

In my case, when I tried to configure Node Exporter with Prometheus in the `prometheus.yml` file, I faced a firewall-related issue because my Prometheus and Node Exporter are in different subnets under the same private network.

##### **Solution:**

I took the help of a DevOps engineer, and he implemented routing between them to make them communicate with each other. After that it worked fine, and I could see the data in my browser.

---

## 4. HOW DATA COLLECTION WORKS

##### Step-by-step:

1. Node Exporter exposes data:

```
CPU=30%, RAM=50%
```

2. Prometheus **pulls (scrapes)** data every few seconds

3. Stores it with a timestamp:

```
CPU=30% at 10:01
CPU=40% at 10:02
```

---

## 5. WHERE IS DATA STORED?

Inside Prometheus itself.

##### Location:

```
/prometheus/data/
```

---

## 6. DATA FORMAT

Stored as **Time-Series Data**.

Example:

```
node_cpu_seconds_total{cpu="0"} 12345
node_cpu_seconds_total{cpu="1"} 67890
```

Looks like key-value + timestamp.

---

## 7. HOW IT LOOKS IN UI

In the browser:

```
http://<IP>:9090
```

You can type:

```
node_memory_MemAvailable_bytes
```

Shows graph

---

## 8. REAL EXAMPLE (YOUR WAZUH TESTING)

##### Without Load:

- CPU = 20%
- RAM = 30%

##### With Heavy Logs:

- CPU = 85%
- RAM = 90%

---

## 9. HOW TO SEPARATE DATA COMING FROM MULTIPLE EXPORTERS?

This is a very important concept.

### Answer: Using labels

Prometheus automatically separates data using:

- **job name**
- **target (IP:port)**
- **labels**

#### Example Configuration

```yaml
scrape_configs:
  - job_name: "manager"
    static_configs:
      - targets: ["XXX.XXX.X.10:9100"]

  - job_name: "agent"
    static_configs:
      - targets: ["XXX.XXX.X.11:9100"]
```

#### How data looks internally

```
node_cpu_seconds_total{job="manager", instance="XXX.XXX.X.10:9100"}
node_cpu_seconds_total{job="agent", instance="XXX.XXX.X.11:9100"}
```

See the difference?

- `job="manager"` → Manager VM
- `job="agent"` → Agent VM

#### Query Examples

##### Manager CPU

```
node_cpu_seconds_total{job="manager"}
```

##### Agent CPU

```
node_cpu_seconds_total{job="agent"}
```

So Prometheus separates data **automatically using labels**.

---

## 10. WHAT YOU SHOULD MONITOR (FOR YOUR WAZUH SETUP)

Since you're testing performance, focus on:

##### System Metrics (from Node Exporter)

1. CPU usage
2. Memory usage
3. Disk usage
4. Load average
5. Network traffic

These help you analyze Wazuh performance under load.

---

## 11. WHERE TO RUN QUERIES?

Open:

```
http://<Prometheus_VM_IP>:9090
```

This is the **Prometheus UI**.

##### Go to:

- **Graph tab**
- Enter query
- Click **Execute**

---

## 12. IMPORTANT QUERIES (WITH EXPLANATION)

### 12.1 CPU Usage

##### Query:

```
rate(node_cpu_seconds_total{mode!="idle"}[1m])
```

##### Meaning:

- `node_cpu_seconds_total` → total CPU time
- `mode!="idle"` → ignore idle time
- `rate(...[1m])` → usage per second over the last 1 minute

##### Expected Output:

```
{instance="XXX.XXX.XX.41:9100", cpu="0"} 0.25
{instance="XXX.XXX.XX.41:9100", cpu="1"} 0.30
```

Meaning:

- CPU core 0 → 25% usage
- CPU core 1 → 30% usage

##### Total CPU (better query):

```
sum(rate(node_cpu_seconds_total{mode!="idle"}[1m])) by (instance)
```

### 12.2 Memory Usage

##### Query:

```
(node_memory_MemTotal_bytes - node_memory_MemAvailable_bytes)
```

##### Meaning:

- Total memory − available memory = used memory

##### Output:

```
8.5e+09
```

~8.5 GB used

##### Memory %:

```
(1 - (node_memory_MemAvailable_bytes / node_memory_MemTotal_bytes)) * 100
```

##### Output:

```
65
```

65% RAM used

### 12.3 Disk Usage

##### Query:

```
(1 - (node_filesystem_avail_bytes / node_filesystem_size_bytes)) * 100
```

##### Meaning:

- Used disk percentage

##### Output:

```
70
```

Disk is 70% full

### 12.4 Load Average

##### Query:

```
node_load1
```

##### Meaning:

- CPU load over the last 1 minute

##### Output:

```
1.5
```

If CPU cores = 2:

- Load 1.5 → normal
- Load > 2 → overloaded

### 12.5 Network Traffic

##### Query:

```
rate(node_network_receive_bytes_total[1m])
```

##### Meaning:

- Incoming network traffic

##### Output:

```
120000
```

~120 KB/sec incoming

---

## 13. PROMETHEUS DATA STORAGE PATH

Your scraped data is stored at:

```
/opt/prometheus-2.47.2.linux-amd64/data/
```

##### Directory Contents Explained

| File/Folder | What It Is |
|---|---|
| `01KN79PZ7A44PJX19WQQ1P5WV6` | TSDB (Time Series DB) block — older compressed data |
| `01KN7BB11DK2EP3XTMRSSBSF8P` | Another TSDB block |
| `01KN9G09SM4HV70ENP4MMCJ18S` | Another TSDB block |
| `01KNDBKCAXZ7AFY2Y1Y8E703VY` | Another TSDB block |
| ... (more blocks) | Each block = 2 hours of scraped metrics |
| `chunks_head/` | In-memory chunk data currently being written |
| `wal/` | Write-Ahead Log — recent uncompressed scrape data |
| `lock` | Lock file to prevent multiple Prometheus instances |
| `queries.active` | Tracks currently running queries |

---

## 14. COMPLETE PROMETHEUS DATA STORAGE GUIDE

### 14.1 Where Is Current/Recently Scraped Data Stored?

```
/opt/prometheus-2.47.2.linux-amd64/data/wal/
```

**WAL (Write-Ahead Log)** is where ALL fresh scraped data lands first. Then after ~2 hours, it gets compressed and moved into a TSDB block folder.

### 14.2 Can You See the Stored Data?

WAL and block files are in **binary format** — you cannot read them with `cat`. But you can query data using Prometheus's built-in tools.

**Option A — Using promtool (command line query):**

```bash
cd /opt/prometheus-2.47.2.linux-amd64

# Check all TSDB block info
./promtool tsdb analyze data/

# List all blocks with time ranges
./promtool tsdb list data/
```

**Option B — Using Prometheus Web UI (easiest):**

```
http://<your-server-ip>:9090
```

Go to the **Graph** tab and type a metric name like:

```
up
node_cpu_seconds_total
prometheus_http_requests_total
```

**Option C — Check WAL directory size and files:**

```bash
ls -lh /opt/prometheus-2.47.2.linux-amd64/data/wal/
du -sh /opt/prometheus-2.47.2.linux-amd64/data/
```

### 14.3 How Does Data Rotation Work? (Why Multiple Folders?)

```
Scrape happens every 15s
↓
Data written to WAL (wal/)
↓
After ~2 hours → WAL flushed → New TSDB Block created
↓
After some time → Small blocks merged into bigger blocks
↓
After 15 days (default) → Old blocks DELETED
```

### 14.4 Is This Data Permanent or Temporary?

**Temporary** — Prometheus is a **short-term storage** system by design.

| Storage Type | Default Retention |
|---|---|
| Default retention period | **15 days** |
| Your setup (no custom flag set) | **15 days** |

Since your Prometheus started on **Apr 2** and today is **Apr 7**, you have **~5 days of data** stored right now.

### 14.5 What Happens to Old Data After Retention?

```
Block older than 15 days
↓
Prometheus automatically DELETES it permanently
↓
Disk space is freed
↓
No backup — data is GONE forever (unless exported)
```

To change retention to 30 days, you would restart with:

```bash
./prometheus --config.file=prometheus.yml --storage.tsdb.retention.time=30d
```

### 14.6 Complete Rotation Flow — Step by Step

```
STEP 1: Scrape happens every 15 seconds
↓
Prometheus contacts localhost:9090 and XXX.XXX.XX.41:9100
Collects all metrics values at that moment

STEP 2: Data written immediately to WAL
Location: data/wal/00000066, 00000067, 00000068
↓
WAL file grows until it reaches ~128MB
Then a new WAL file is created (00000067, 00000068...)

STEP 3: After ~2 hours of WAL data accumulates
↓
Prometheus flushes WAL → Creates new TSDB Block folder
Example: 01KNKJQE56PFE4XWTP3ZBR22RJ (2hr block)

STEP 4: Compaction runs in background
↓
Prometheus looks at small blocks sitting next to each other
If they are adjacent in time → MERGES them into one big block
3 blocks of 6hr each → becomes 1 block of 18hr
This saves disk space and speeds up queries

STEP 5: After 15 days
↓
Old blocks are permanently deleted
```

---

## 15. PROMETHEUS PERMANENT STORAGE SOLUTION

### 15.1 The Problem With Prometheus Default Storage

```
Node Exporter → Prometheus → Local Disk (only 15 days!)
Data lost after 15 days
```

### 15.2 The Solution: Long-Term Storage

**Permanent storage is absolutely possible!** There are a few tools for this. Let me explain the **best and most widely used approach**.

### 15.3 The Best Tool: Thanos or VictoriaMetrics

| Tool | Difficulty | Storage | Best For |
|---|---|---|---|
| **VictoriaMetrics** | Easy | Local disk / S3 / GCS | Simple setup, best performance |
| **Thanos** | Hard | S3 / GCS / Azure | Large scale, multi-cluster |
| **Cortex** | Very Hard | S3 / GCS | Enterprise scale |

#### Recommended: VictoriaMetrics — Simple, Fast, Production-Ready

#### How the Full Architecture Works

```
Node Exporter (collects metrics)
↓
Prometheus (scrapes every 15s, stores 15 days locally)
↓ [remote_write]
VictoriaMetrics (stores PERMANENTLY on disk / forever)
↓
Grafana (reads from VictoriaMetrics for dashboards)
```

#### Role of Each Tool:

| Tool | Role |
|---|---|
| **Node Exporter** | Collects system metrics (CPU, RAM, Disk) |
| **Prometheus** | Scrapes Node Exporter, short-term buffer (15 days) |
| **remote_write** | The **bridge** — moves data from Prometheus → VictoriaMetrics in real-time |
| **VictoriaMetrics** | **Permanently stores** all metrics on disk |
| **Grafana** | Reads from VictoriaMetrics to show dashboards |

#### Where Is the Data Actually Stored?

```
/opt/victoriametrics/data/
├── data/
│   ├── big/   ← compressed old data (efficient storage)
│   └── small/ ← recent incoming data
├── snapshots/ ← manual backups
└── indexdb/   ← index for fast queries
```

VictoriaMetrics **compresses data up to 10x** better than Prometheus — so 1 year of data takes very little disk space!

#### In What Format Is Data Stored in VictoriaMetrics?

VictoriaMetrics stores data in its **own custom binary format** — NOT in plain text, NOT in JSON.

How it looks on disk:

```
/opt/victoriametrics/data/
├── small/   ← Recent data (last few hours) - raw binary
├── big/     ← Older data - heavily compressed binary
└── indexdb/ ← Index file (like a book's index) for fast search
```

##### Why Binary Format?

```
Plain Text JSON:
cpu_usage{host="vm1"} 45.23 1710000000 = 35 bytes per line
× 1 million entries = 35 MB

VictoriaMetrics Binary (compressed):
Same data = 3-4 MB only (10x smaller!)
```

**Simple Words:** Like how a ZIP file compresses your documents, VictoriaMetrics compresses metrics into a special binary format that saves 10x disk space.

---

## 16. WHAT IS remote_write? TOOL OR CONFIG TERM?

### remote_write Is a Configuration Term Inside prometheus.yml

It is **NOT** a separate tool. It is NOT software you install. It is a **built-in feature of Prometheus** that you activate by writing a few lines in the config file.

Think of it like this:

```
Prometheus is a WATER PUMP
Local storage (15 days) is a SMALL BUCKET beside the pump
remote_write is a PIPE connected from the pump to a BIG TANK
VictoriaMetrics is the BIG TANK (permanent storage)

Without remote_write: Pump → Small Bucket (overflows after 15 days)
With remote_write:    Pump → Small Bucket
                            ↘ Pipe → Big Tank (forever!)
```

#### What Does remote_write Do

```yaml
# This is remote_write in prometheus.yml — just 2 lines!
remote_write:
  - url: http://<victoriametrics-ip>:8428/api/v1/write
```

##### What Happens When You Add This?

```
BEFORE remote_write:
Prometheus scrapes metric → saves to local disk → deleted after 15 days

AFTER remote_write:
Prometheus scrapes metric → saves to local disk (15 days)
                           ↘ ALSO sends copy to VictoriaMetrics (FOREVER)
```

---
## 17. DEEP DIVE — COMPLETE FLOW EXPLANATION

### 17.1 Part 1: What Does Node Exporter Actually Do?

Node Exporter's job is to **"expose"** system metrics. But what does "expose" mean?

Your Linux System has:

- CPU usage → stored in `/proc/stat`
- Memory usage → stored in `/proc/meminfo`
- Disk usage → stored in `/proc/diskstats`
- Network usage → stored in `/proc/net/dev`

These are RAW kernel files — not readable by Prometheus directly!

Node Exporter's job:

```
READ these /proc files → CONVERT to Prometheus format → SERVE on HTTP port 9100
```

#### What Does "Expose" Mean?

Node Exporter opens an **HTTP web endpoint** at:

```
http://<your-vm-ip>:9100/metrics
```

If you open this URL in a browser or run:

```bash
curl http://localhost:9100/metrics
```

You will see output like this:

```
# HELP node_cpu_seconds_total CPU time spent in seconds
# TYPE node_cpu_seconds_total counter
node_cpu_seconds_total{cpu="0",mode="idle"} 5432.12
node_cpu_seconds_total{cpu="0",mode="user"} 234.56
node_memory_MemAvailable_bytes 2147483648
node_filesystem_avail_bytes{mountpoint="/"} 10737418240
node_network_receive_bytes_total{device="eth0"} 9876543
```

### 17.2 Part 2: How Does Prometheus Scrape Data?

#### "Scrape" Means Prometheus Sends an HTTP GET Request to Node Exporter

Every 15 seconds (default), Prometheus does this:

```
Prometheus thinks: "Time to collect data from Node Exporter!"

Prometheus sends: GET http://localhost:9100/metrics
↑
This is a normal HTTP request
Same as typing a URL in a browser!

Node Exporter gets the request and replies with all metrics as plain text

Prometheus receives the plain text response and saves it to local disk
```

#### Where Is This Configured?

In your `prometheus.yml`:

```yaml
scrape_configs:
  - job_name: 'node_exporter'
    scrape_interval: 15s   # scrape every 15 seconds
    static_configs:
      - targets: ['localhost:9100']   # address of Node Exporter
```

This tells Prometheus:

- **WHO** to scrape → `localhost:9100` (Node Exporter)
- **HOW OFTEN** → every 15 seconds
- **WHAT PATH** → `/metrics` (default path)

### 17.3 Part 3: How Data Moves Between Node Exporter and Prometheus?

#### Step by Step — Complete Journey

```
STEP 1: Timer triggers (every 15 seconds)
Prometheus internal clock says "15 seconds passed, time to scrape!"


STEP 2: Prometheus sends HTTP GET request
Prometheus ──── GET /metrics HTTP/1.1 ────▶ Node Exporter
Host: localhost:9100


STEP 3: Node Exporter reads /proc files at that moment
Node Exporter reads:
/proc/stat      → CPU data right now
/proc/meminfo   → Memory data right now
/proc/diskstats → Disk data right now


STEP 4: Node Exporter sends HTTP response back
Node Exporter ──── HTTP 200 OK ────────────▶ Prometheus
Content-Type: text/plain
Body:
node_cpu_seconds_total{cpu="0"} 5432.12
node_memory_MemFree_bytes 2147483648
node_disk_reads_completed 98765
... (hundreds of metrics)


STEP 5: Prometheus receives and parses
Prometheus reads the plain text
Adds a TIMESTAMP to each metric
Saves to local storage (TSDB - Time Series Database)

node_cpu_seconds_total{cpu="0"} 5432.12 @timestamp:1710000015
node_cpu_seconds_total{cpu="0"} 5435.67 @timestamp:1710000030
node_cpu_seconds_total{cpu="0"} 5438.91 @timestamp:1710000045
```

### 17.4 Part 4: Is the Data Encrypted Between Node Exporter and Prometheus?

#### By Default — NO, the Data Is NOT Encrypted

```
DEFAULT SETUP (Plain HTTP):
Prometheus ──── plain text HTTP ────▶ Node Exporter
(no encryption)

Data travels as RAW PLAIN TEXT like this:
node_cpu_seconds_total{cpu="0",mode="idle"} 5432.12
node_memory_MemFree_bytes 2147483648
```

---

## 18. DATA FORMAT AT EVERY STAGE

```
Node Exporter ──▶ Prometheus ──▶ WAL ──▶ TSDB Blocks ──▶ VictoriaMetrics ──▶ Grafana
```

### 18.1 Node Exporter → Prometheus

**Format: Plain Text (UTF-8) over HTTP**

```
node_cpu_seconds_total{cpu="0",mode="idle"} 5432.12
node_memory_MemFree_bytes 2147483648
```

- Human readable
- No compression
- No encryption (default)

### 18.2 Prometheus WAL Folder Format

**Format: Binary (raw, uncompressed)**

```
/prometheus/wal/
├── 00000001 ← raw binary file
├── 00000002 ← raw binary file
```

- NOT human readable
- NOT compressed yet
- Just raw binary writes for crash recovery
- Temporary — gets moved to TSDB blocks after 2 hours

### 18.3 Prometheus TSDB Blocks (data/ folder)

**Format: Compressed Binary**

```
/prometheus/data/
├── chunks/    ← compressed binary metric values
├── index      ← binary index for fast search
└── meta.json  ← plain JSON (block info only)
```

- NOT human readable
- Compressed binary
- Permanent local storage (until the 15-day limit)

### 18.4 Prometheus → VictoriaMetrics (remote_write)

**Format: Protobuf (compressed binary) over HTTP**

```
Prometheus compresses metrics → Protobuf binary → HTTP POST → VictoriaMetrics
```

- NOT human readable
- Compressed binary (Snappy compression)
- No encryption by default (plain HTTP)
- HTTPS only if you manually configure it

### 18.5 VictoriaMetrics Local Disk Storage

**Format: Custom Compressed Binary**

```
/opt/victoriametrics/data/
├── small/ ← recent data, compressed binary
└── big/   ← older data, heavily compressed binary
```

- NOT human readable
- More compressed than Prometheus (10x better)
- Custom format (not the same as Prometheus TSDB)

### 18.6 VictoriaMetrics → Grafana

**Format: JSON over HTTP**

```json
{
  "metric": {"__name__": "node_cpu_seconds_total", "cpu": "0"},
  "values": [[1710000000, "5432.12"], [1710000015, "5435.67"]]
}
```

- Human readable JSON
- No compression
- No encryption by default

### 18.7 Encryption & Format — All Stages at a Glance

| Stage | Format | Compressed? | Encrypted? |
|---|---|---|---|
| Node Exporter → Prometheus | Plain Text (HTTP) | No | No (HTTP) |
| Prometheus → WAL | Raw Binary | No | No |
| WAL → TSDB Blocks | Compressed Binary | Yes | No |
| Prometheus → VictoriaMetrics | Protobuf + Snappy (HTTP) | Yes | No (unless HTTPS) |
| VictoriaMetrics → Disk | Custom Compressed Binary | Yes | No |
| VictoriaMetrics → Grafana | JSON (HTTP) | No | No (unless HTTPS) |

---

## 19. WHAT IS ALERTING IN PROMETHEUS?

Alerting is the mechanism that **notifies you when something goes wrong** in your infrastructure — before your users or boss notices. Instead of you manually watching dashboards, Prometheus watches metrics for you and fires an alert when a condition is met (e.g., CPU > 80% for 5 minutes).

### 19.1 How It Works — The Full Flow

![Image 1](<../../../assets/images/POC's/Sai krishna/Prometheus/image1.png>)

### 19.2 The 4 Key Components

**1. Alert Rules** (on Prometheus VM) — You write conditions in PromQL. Example: "if CPU > 80% for 2 minutes → fire alert." Stored in a `.yml` file.

**2. Prometheus Server** (on Prometheus VM) — Evaluates your rules every `evaluation_interval` (default: 15s). If the condition is true, it moves the alert through 3 states: **Inactive → Pending → Firing**.

**3. Alertmanager** (also installed on Prometheus VM) — Receives the fired alert, handles deduplication (so you don't get 100 emails), grouping, silencing, and routes it to the right receiver.

**4. Receiver** — Email, Slack, PagerDuty, webhook, etc.

### 19.3 Is Alertmanager a Separate Tool?

Yes, 100%. Alertmanager is a completely separate binary (separate program), just like how Prometheus is a separate binary from Node Exporter. Each is its own tool:

- `node_exporter` — collects metrics from the Node VM
- `prometheus` — scrapes, stores, and evaluates rules
- `alertmanager` — receives fired alerts and sends notifications

They are three separate programs that talk to each other over HTTP. You install and run each one independently.

### 19.4 Install Alertmanager

Install Alertmanager the same way you run Prometheus (in `/opt`, run as an application).

Since you already have Prometheus installed in `/opt` and run it manually, do the exact same pattern for Alertmanager — on your **Prometheus VM**:

#### Step 1: Install Alertmanager (on Prometheus VM)

```bash
# Run on: Prometheus VM
cd /opt

# Download Alertmanager
wget https://github.com/prometheus/alertmanager/releases/download/v0.27.0/alertmanager-0.27.0.linux-amd64.tar.gz

# Extract it
tar -xvf alertmanager-0.27.0.linux-amd64.tar.gz

# Rename folder to a clean name (just like your prometheus folder)
mv alertmanager-0.27.0.linux-amd64 alertmanager

# Go inside
cd /opt/alertmanager

# See what's inside
ls
```

You will see: `alertmanager`, `amtool`, `alertmanager.yml`

Now run it exactly like you run Prometheus:

```bash
# Run on: Prometheus VM
cd /opt/alertmanager
./alertmanager --config.file=alertmanager.yml
```

That's it. Alertmanager runs on port 9093 by default. Open `http://<prometheus-vm-ip>:9093` to see its UI.

#### Step 2: Create Alertmanager Config

```bash
# Run on: Prometheus VM
sudo nano /opt/alertmanager/alertmanager.yml
```

Paste this (email example):

```yaml
global:
  resolve_timeout: 5m

route:
  receiver: 'email-alert'

receivers:
  - name: 'email-alert'
    email_configs:
      - to: 'you@example.com'
        from: 'alertmanager@example.com'
        smarthost: 'smtp.gmail.com:587'
        auth_username: 'you@example.com'
        auth_password: 'your-app-password'
```

#### Step 3: Create Alert Rules File (on Prometheus VM)

```bash
# Run on: Prometheus VM
sudo nano /opt/prometheus/alert_rules.yml
```

Paste this — a real useful rule:

```yaml
groups:
  - name: node_alerts
    rules:
      - alert: HighCPUUsage
        expr: 100 - (avg by(instance) (rate(node_cpu_seconds_total{mode="idle"}[2m])) * 100) > 80
        for: 2m
        labels:
          severity: warning
        annotations:
          summary: "High CPU on {{ $labels.instance }}"
          description: "CPU has been above 80% for 2 minutes."

      - alert: InstanceDown
        expr: up == 0
        for: 1m
        labels:
          severity: critical
        annotations:
          summary: "Instance {{ $labels.instance }} is DOWN"
```

#### Step 4: Update prometheus.yml to Link Rules + Alertmanager (on Prometheus VM)

```bash
# Run on: Prometheus VM
sudo nano /opt/prometheus/prometheus.yml
```

Add/edit these sections:

```yaml
rule_files:
  - "/opt/prometheus/alert_rules.yml"

alerting:
  alertmanagers:
    - static_configs:
        - targets:
            - "localhost:9093"   # Alertmanager default port
```

### 19.5 How Do the 3 Tools Communicate?

Who is first, second, third? Let me show you the exact data flow:

![Image 2](<../../../assets/images/POC's/Sai krishna/Prometheus/image2.png>)

**The flow in plain words:**

Node Exporter sits quietly on the Node VM and reads OS-level data (CPU, RAM, disk) from the Linux kernel. It exposes this data as plain text at `http://<node-vm-ip>:9100/metrics`. Node Exporter does NOT push data anywhere — it just sits and waits.

Prometheus (on Prometheus VM) comes to Node Exporter every 15 seconds and does an HTTP GET to `/metrics` and pulls the data. This is called "scraping." Prometheus stores this in its own time-series database. Then separately, every 15 seconds, Prometheus evaluates your alert rules against the stored data. If a rule condition becomes true, Prometheus sends an HTTP POST request to Alertmanager at port 9093 with the alert details in JSON format.

Alertmanager (on Prometheus VM) receives that POST, deduplicates (if the same alert comes 5 times, it sends only 1 email), groups them, and routes to the configured receiver like email or Slack.

### 19.6 What Is Email, Slack, PagerDuty in the Notify Section?

These are the 3 different ways Alertmanager can send you a notification when an alert fires:

- **Email** — Alertmanager sends an email to your Gmail/Outlook/any email inbox. Just like a normal email you receive.
- **Slack** — Alertmanager sends a message to a Slack channel in your workspace. Your team sees it there.
- **PagerDuty** — A paid on-call incident management platform. Used by big companies. When an alert fires, it calls/texts the on-call engineer's phone.

You don't need all three. For learning and personal use, just Email is enough. You only configure what you need.

### 19.7 Alertmanager Sends a Real Email to Your Inbox

Yes, when an alert fires (e.g., CPU > 80%), Prometheus sends the alert to Alertmanager, and Alertmanager connects to Gmail's SMTP server and sends a real email to whatever address you put in the `to:` field. You will literally see it arrive in your Gmail inbox.

### 19.8 Is It Mandatory to Install Alertmanager on the Same Prometheus VM?

No, it is NOT mandatory. You have 3 options:

| Option | Where Alertmanager Runs | Works? |
|---|---|---|
| Same VM as Prometheus | Prometheus VM | Yes (most common for learning) |
| Separate dedicated VM | Any other VM | Yes (production setups) |
| Any reachable server | Anywhere with a network | Yes, as long as Prometheus can reach it on port 9093 |

For your current setup (learning/lab), installing it on the same Prometheus VM is perfectly fine and the simplest approach.

### 19.9 Is Alertmanager Free and Open Source?

Yes, 100% free and open source. Alertmanager is an official part of the Prometheus project, maintained by the CNCF (Cloud Native Computing Foundation). The license is Apache 2.0 — free to use, modify, and distribute. No cost, no license key, no trial period.

### 19.10 What Is SMTP?

SMTP stands for **Simple Mail Transfer Protocol**. It is the standard rule/language that computers use to **send emails** across the internet.

Think of it this way — when you send a letter, you need a post office. The post office accepts your letter, puts it in the right delivery route, and delivers it to the recipient. SMTP is exactly that post office — but for emails.

Every email you send from Gmail, Outlook, Yahoo — behind the scenes, it goes through an SMTP server. Without SMTP, no email can be sent from any application.

### 19.11 Why Does Alertmanager Need SMTP?

Alertmanager is a program running on your Linux server. It has no built-in email account. It cannot magically send emails on its own. It needs to use an existing email service (like Gmail) to deliver the alert email to your inbox.

So Alertmanager says: "I need to send an alert email — let me connect to Gmail's SMTP server, log in with the credentials you gave me, and ask Gmail to deliver this email on my behalf."

### 19.12 How to Get Your 16-Character App Password

**Follow these exact steps:**

**Step 1:** Open this link in your browser (must be logged into your Google account): `https://myaccount.google.com/security`

**Step 2:** Enable 2-Step Verification first if not already done. App Passwords only work when 2-Step Verification is ON. Click "2-Step Verification" and complete the setup.

**Step 3:** After enabling 2-Step Verification, go back to the Security page and search for "App passwords" — click it. If you don't see it, go directly to: `https://myaccount.google.com/apppasswords`

**Step 4:** In the "App name" box, type anything — for example, type `alertmanager` — then click "Create".

**Step 5:** Google shows you a 16-character password like this:

```
Ex: abcd efgh ijkl mnop
```

Copy it immediately. Google will NEVER show it again after you close that popup. The spaces are just for readability — when you paste it into your yml file, remove the spaces:

```yaml
auth_password: 'abcdefghijklmnop'
```

### 19.13 How to Test

#### Test 1: Check Everything Is Running (Prometheus VM)

```bash
# Run on: Prometheus VM
sudo systemctl status alertmanager   # should be active
sudo systemctl status prometheus     # should be active
```

#### Test 2: View Alerts in Prometheus UI

Open in browser: `http://<prometheus-vm-ip>:9090/alerts`

You should see your `HighCPUUsage` and `InstanceDown` rules listed as **Inactive** (green — normal state).

#### Test 3: Trigger a Fake Alert (Stop Node Exporter on Node VM)

```bash
# Run on: Node VM
sudo systemctl stop node_exporter
```

Now wait ~1 minute, then go back to `http://<prometheus-vm-ip>:9090/alerts` — you'll see `InstanceDown` move from **Inactive → Pending → Firing** (red). Alertmanager will then send the notification.

![Image 3](<../../../assets/images/POC's/Sai krishna/Prometheus/image3.png>)

![Image 4](<../../../assets/images/POC's/Sai krishna/Prometheus/image4.png>)

```bash
# Start it back after testing
# Run on: Node VM
sudo systemctl start node_exporter
```

#### Test 4: Check Alertmanager UI

Open: `http://<prometheus-vm-ip>:9093`

You'll see all active/resolved alerts here.

### **Challenge:**

I faced a challenge in the notification part — in the UI I could see the generated alerts, but I couldn't receive any email notification in my Gmail inbox.

### **Solution:**

Then I modified the `alertmanager.yml` file with the below new configuration:

```yaml
global:
  resolve_timeout: 5m
  smtp_require_tls: true

route:
  group_by: ['alertname']
  group_wait: 30s
  group_interval: 5m
  repeat_interval: 1h
  receiver: 'email-alert'

receivers:
  - name: 'email-alert'
    email_configs:
      - to: 'you@example.com'
        from: 'alertmanager@example.com'
        smarthost: 'smtp.gmail.com:587'
        auth_username: 'you@example.com'
        auth_password: 'your-app-password'
        send_resolved: true
```

Then I followed the same running process, and I tested the implementation, and I successfully received the email notification when the instance was down:

![Image 5](<../../../assets/images/POC's/Sai krishna/Prometheus/image5.png>)

And I received another email notification after resolving that fired alert.

![Image 6](<../../../assets/images/POC's/Sai krishna/Prometheus/image6.png>)

---

## 20. WHAT IS GRAFANA?

**Grafana is an open-source visualization and monitoring tool (application).**

In simple words: it **converts raw data into meaningful dashboards (graphs, charts, alerts)**.

### 20.1 So Is It a Tool or Application?

It is **both**:

- A **tool** → used by engineers for monitoring
- A **web-based application** → runs on server/browser

### 20.2 Why Do We Need Grafana?

Imagine you have thousands of logs/metrics like:

- CPU usage = 75%
- Memory usage = 60%
- Disk = 90%

Raw numbers are hard to understand.

#### Grafana Helps By:

- Turning data into **graphs & dashboards**
- Providing **real-time monitoring**

### 20.3 Is Grafana Free?

Yes:

- **Grafana OSS (Open Source)** → Free
- Enterprise version → Paid features

Most companies use the **free version + plugins**.

### 20.4 How Grafana Works With Prometheus

#### Roles:

| Tool | Role |
|---|---|
| Prometheus | Collects & stores metrics |
| Grafana | Visualizes metrics |

Grafana **does NOT collect data**. It **reads data from Prometheus**.

### 20.5 How Prometheus Works

1. Prometheus **scrapes metrics** from targets
2. Stores data in a **time-series database**
3. Query using **PromQL**

### 20.6 Data Flow (End-to-End Architecture)

Let's break it down clearly:

```
Application / Server (VM)
↓
Exporter (Node Exporter)
↓
Prometheus (scrapes metrics)
↓
Grafana (queries Prometheus)
↓
Dashboard / Alerts
```

### 20.7 How to Install Grafana (Linux)

#### Step 1: Download Grafana

```bash
cd /opt
wget https://dl.grafana.com/oss/release/grafana-10.x.x.linux-amd64.tar.gz
tar -xvf grafana-10.x.x.linux-amd64.tar.gz
mv grafana-10.x.x grafana
cd grafana
```

#### Step 2: Run Grafana

```bash
cd /opt/grafana/bin
./grafana-server
```

#### Step 3: Access UI

```
http://<your-ip>:3000
```

Default:

- user: `admin`
- password: `admin`

### **Challenge:**

While following the above process, I faced an issue while downloading Grafana, and the issue was related to the version.

### **Solution:**

I used the below approach to install the correct working version of Grafana:

#### Correct Working Versions

As of now:

- Latest versions are **12.x / 11.x**
- Stable OSS version example: **11.6.0**

**Important:** Grafana version **does NOT need to match the Prometheus version** — they are independent tools.

#### Recommended Command

Use this (OSS version):

```bash
wget https://dl.grafana.com/oss/release/grafana-11.6.0.linux-amd64.tar.gz
```

**Then extract:**

```bash
tar -xvf grafana-11.6.0.linux-amd64.tar.gz
mv grafana-11.6.0 grafana
cd grafana
```

#### Run Grafana

```bash
cd /opt/grafana/bin
./grafana-server
```

### 20.8 How to Connect Grafana With Prometheus

#### Step 1: Open Grafana UI

Go to: Configuration → Data Sources

#### Step 2: Add Data Source

- Select **Prometheus**

#### Step 3: Enter URL

```
http://localhost:9090
```

Your Prometheus server URL.

#### Step 4: Save & Test

If successful → Connected

---

## 21. HOW TOOLS COMMUNICATE

Let's go deep but simple.

### 21.1 Components Involved

- Node Exporter
- Prometheus
- Grafana

### 21.2 Step-by-Step Communication Flow

#### Step 1: Exporter Exposes Metrics

Node Exporter runs:

```
http://localhost:9100/metrics
```

It gives data like:

```
node_cpu_seconds_total 12345
```

#### Step 2: Prometheus Pulls Data

Prometheus config:

```
targets: ['localhost:9100']
```

Prometheus **calls (HTTP GET)**:

```
http://localhost:9100/metrics
```

Stores data in the **time-series DB**.

#### Step 3: Grafana Connects to Prometheus

In Grafana:

- Add data source → Prometheus
- URL: `http://localhost:9090`

#### Step 4: How Grafana Gets Data

When you open a dashboard:

1. Grafana sends a query to Prometheus:

```
GET /api/v1/query?query=rate(node_cpu_seconds_total[1m])
```

2. Prometheus executes **PromQL**

3. Returns a JSON response:

```json
{
  "status": "success",
  "data": {
    "result": [...]
  }
}
```

4. Grafana converts JSON → Graph

- Grafana **never stores data**
- It only **reads from Prometheus**

---

## 22. END-TO-END TEST SCENARIO

**Goal:** Verify full data flow.

### Step 1: Start Node Exporter

```bash
./node_exporter
```

Check: `http://localhost:9100/metrics`

### Step 2: Configure Prometheus

Add in `prometheus.yml`:

```yaml
scrape_configs:
  - job_name: 'node'
    static_configs:
      - targets: ['localhost:9100']
```

Restart Prometheus.

### Step 3: Verify in Prometheus

Open: `http://localhost:9090`

Query:

```
up
```

Output should be:

```
up{job="node"} 1
```

### Step 4: Connect Grafana

- Add Prometheus data source
- Create dashboard

### Step 5: Visualize Metrics

Use query:

```
rate(node_cpu_seconds_total[1m])
```

You should see a graph.

![Image 7](<../../../assets/images/POC's/Sai krishna/Prometheus/image7.png>)

---
