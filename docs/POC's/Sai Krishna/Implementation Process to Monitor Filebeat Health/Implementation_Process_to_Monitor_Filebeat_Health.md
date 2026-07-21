# IMPLEMENTATION PROCESS TO MONITOR FILEBEAT HEALTH

## 1. EXISTING SETUP

##### Meaning:

I already installed Filebeat on the XXX.XXX.XX.12 VM, installed Logstash on the XXX.XXX.XX.87 VM, and installed Wazuh on the XXX.XXX.XX.43 VM.

I already have a pipeline to collect application logs by using Filebeat. My pipeline is: Filebeat → Logstash → Wazuh Manager.

---

## 2. WHAT CHANGES ARE NEEDED?

##### Meaning:

You need to: install Metricbeat on the Beat VM (where Filebeat is running), install OpenSearch + OpenSearch Dashboards on a new or existing VM, and add one new pipeline config file to your existing Logstash.

Now here is the full picture of both pipelines side by side:

![Image 1](<../../../assets/images/POC's/Sai krishna/Implementation Process to Monitor Filebeat Health/image1.png>)

##### Explanation:

The key insight from the diagram: Pipeline 1 and Pipeline 2 use different input ports (5044 vs 5045) so they never interfere. Filebeat sends logs to port 5044 as before. Metricbeat sends health data to port 5045, which is the new pipeline only.

Now here are the exact steps and changes needed:

---

## 3. IMPLEMENTATION PROCESS

### 3.1 Step 1: On the Beat VM — Install Metricbeat and Enable Filebeat Stats

```bash
# Add Elastic repo (same as before)
curl -fsSL https://artifacts.elastic.co/GPG-KEY-elasticsearch \
  | sudo gpg --dearmor -o /usr/share/keyrings/elastic-keyring.gpg
# Downloads and saves the GPG trust key for Elastic packages

echo "deb [signed-by=/usr/share/keyrings/elastic-keyring.gpg] \
https://artifacts.elastic.co/packages/oss-7.x/apt stable main" \
  | sudo tee /etc/apt/sources.list.d/elastic-7.x.list
# Adds the Elastic package repository to apt

sudo apt update && sudo apt install metricbeat -y
# Downloads and installs Metricbeat
```

**Note:** while installing Metricbeat, try to install Metricbeat with the same version as Filebeat to avoid issues.

Enable Filebeat's internal stats port:

```bash
sudo nano /etc/filebeat/filebeat.yml
# Add these 3 lines at the bottom of the file:
```

```yaml
http.enabled: true      # turns on Filebeat's internal health API
http.host: localhost    # only accessible from this machine — safe
http.port: 5066         # Metricbeat will read from here every 10s
```

```bash
sudo systemctl restart filebeat
curl http://localhost:5066/stats   # test — you should see JSON data
```

Configure Metricbeat to watch Filebeat and send to your Logstash on the NEW port:

```bash
sudo nano /etc/metricbeat/metricbeat.yml
```

```yaml
# Send to your EXISTING Logstash VM, but on NEW port 5045
output.logstash:
  hosts: ["YOUR-LOGSTASH-VM-IP:5045"]
  # use port 5045 so it goes to the new pipeline, not the existing one
```

Enable the beat monitoring module:

```bash
sudo metricbeat modules enable beat
sudo nano /etc/metricbeat/modules.d/beat.yml
```

```yaml
- module: beat
  metricsets: ["stats", "state"]
  period: 10s
  hosts: ["http://localhost:5066"]  # reads Filebeat health from here
```

```bash
sudo systemctl enable metricbeat && sudo systemctl start metricbeat
sudo systemctl status metricbeat   # verify running
```

---

### 3.2 Step 2: On the Logstash VM — Add ONE New Pipeline Config File

##### Meaning:

Your existing config file stays completely untouched. You only add a new file.

First, install the OpenSearch output plugin on Logstash:

```bash
sudo /usr/share/logstash/bin/logstash-plugin install logstash-output-opensearch
# This adds the ability to write to OpenSearch
# It does NOT affect any existing plugins or pipelines
```

Create the new pipeline config file:

```bash
sudo nano /etc/logstash/conf.d/opensearch-pipeline.conf
```

```ruby
input {
  beats {
    port => 5045   # NEW port — Metricbeat sends here
    # completely separate from port 5044 used by Filebeat
  }
}

filter {
  if [agent][type] == "metricbeat" {
    mutate {
      add_field => { "pipeline" => "filebeat-health-monitor" }
      # adds a label so you can filter in OpenSearch Dashboards
    }
  }
}

output {
  opensearch {
    hosts => ["https://YOUR-OPENSEARCH-VM-IP:9200"]
    user => "admin"
    password => "YOUR-OPENSEARCH-PASSWORD"
    index => "filebeat-health-%{+YYYY.MM.dd}"
    # creates daily index like filebeat-health-2026.05.18
    ssl_certificate_verification => false
    # for lab/dev — set to true with proper certs in production
  }
}
```

Now register this new pipeline in Logstash's pipeline manager:

```bash
sudo nano /etc/logstash/pipelines.yml
```

```yaml
# --- EXISTING pipeline — DO NOT CHANGE ---
- pipeline.id: wazuh-pipeline
  path.config: "/etc/logstash/conf.d/wazuh.conf"
  # whatever your existing config file is named

# --- NEW pipeline — ADD THIS BLOCK ---
- pipeline.id: opensearch-pipeline
  path.config: "/etc/logstash/conf.d/opensearch-pipeline.conf"
```

Note: If your `pipelines.yml` file is configured to a root folder, then no need to add again as above.

```bash
sudo systemctl restart logstash
# Restarts Logstash to load the new pipeline
# The existing pipeline reloads too but its config is unchanged — safe

sudo systemctl status logstash
# Should show active (running)

sudo tail -f /var/log/logstash/logstash-plain.log
# Watch for: "Successfully started Logstash API endpoint"
# Watch for any ERROR lines — there should be none
```

---

### 3.3 Step 3: On the OpenSearch VM — Install OpenSearch and Dashboards

This is a fresh install on a new VM (or any VM that has free resources):

```bash
# System prereq — OpenSearch needs this memory setting
sudo sysctl -w vm.max_map_count=262144
echo "vm.max_map_count=262144" | sudo tee -a /etc/sysctl.conf
# vm.max_map_count controls how many memory segments a process can use
# OpenSearch needs a large number for its search indexes

# Add OpenSearch GPG key
curl -o- https://artifacts.opensearch.org/publickeys/opensearch.pgp \
  | sudo gpg --dearmor --batch --yes -o /usr/share/keyrings/opensearch-keyring

# Add repository
echo "deb [signed-by=/usr/share/keyrings/opensearch-keyring] \
https://artifacts.opensearch.org/releases/bundle/opensearch/2.x/apt stable main" \
  | sudo tee /etc/apt/sources.list.d/opensearch.list

# Install (sets your admin password during install)
sudo env OPENSEARCH_INITIAL_ADMIN_PASSWORD=XXXXXXXXXXXX apt install opensearch -y
```

##### **Challenge:**

When I am trying to install OpenSearch, I faced an error: `Unable to locate package opensearch`.

##### **Solution:**

###### Run these commands in order

```bash
# Step 1 — refresh apt's package list from ALL repositories including the new OpenSearch one
sudo apt update
# This goes to opensearch.org and downloads the package catalog
# You should see a line like: "Get:X https://artifacts.opensearch.org ..."
# If you see any errors here, paste them — that tells us if the repo is reachable

# Step 2 — verify opensearch is now findable
apt-cache show opensearch | grep Version
# This should print the available version like: Version: 2.x.x
# If this shows nothing, the repo is still not loading correctly

# Step 3 — now install with the password
sudo env OPENSEARCH_INITIAL_ADMIN_PASSWORD=XXXXXXXXXXXX apt install opensearch -y
```

Then continue the implementation process as follows:

```bash
# Allow OpenSearch to accept connections from other VMs
sudo nano /etc/opensearch/opensearch.yml
```

```yaml
network.host: 0.0.0.0        # listens on all network interfaces
discovery.type: single-node  # single server, not a cluster
```

```bash
sudo systemctl enable opensearch && sudo systemctl start opensearch

# Verify it is working (run from OpenSearch VM itself)
curl -k -u admin:XXXXXXXXXXXX https://localhost:9200
# You should see JSON with cluster_name and version
```

Install OpenSearch Dashboards on the same VM:

```bash
echo "deb https://artifacts.opensearch.org/releases/bundle/opensearch-dashboards/2.x/apt stable main" \
  | sudo tee -a /etc/apt/sources.list.d/opensearch-2.x.list

sudo apt update && sudo apt install opensearch-dashboards -y

sudo nano /etc/opensearch-dashboards/opensearch_dashboards.yml
```

```yaml
server.host: "0.0.0.0"
opensearch.hosts: ["https://localhost:9200"]
opensearch.username: "kibanaserver"
opensearch.password: "XXXXXXXXXXXX"
opensearch.ssl.verificationMode: none
```

```bash
sudo systemctl enable opensearch-dashboards && sudo systemctl start opensearch-dashboards
# Access at http://YOUR-OPENSEARCH-VM-IP:5601
```

**Note:**

Your existing Filebeat → Logstash → Wazuh pipeline continues running exactly as before. The new Metricbeat → Logstash → OpenSearch pipeline runs in parallel without touching it.

##### **Challenge:**

After doing the changes at the Logstash level, I am trying to restart Logstash, but it is taking a long time to restart and is not restarting. When I investigate, I find an error as follows:

**Logstash is stuck in `deactivating (stop-sigterm)` for 1 hour 36 minutes**

Look at this line:

```
Active: deactivating (stop-sigterm) since Tue 2026-05-19 06:50:28 UTC; 1h 36min ago
```

This means you ran `systemctl restart` or `systemctl stop`, but Logstash is refusing to die. The Java process (PID 2449678) is stuck in a retry loop trying to reach the dead OpenSearch host, and it is ignoring the stop signal. Systemd is waiting politely, but the process will not stop on its own.

##### **Solution:**

###### Force kill the stuck Logstash process

```bash
# The normal stop is not working, so we force kill it
sudo kill -9 2449678
# kill -9 = SIGKILL — immediately terminates the process, no questions asked
# 2449678 is the PID shown in your status output

# Verify it is gone
sudo systemctl status logstash
# Should now show: inactive (dead)

# If it still shows the same PID, wait 5 seconds and check again
sleep 5 && sudo systemctl status logstash
```

###### Test the config syntax before starting

```bash
sudo /usr/share/logstash/bin/logstash \
  --config.test_and_exit \
  -f /etc/logstash/conf.d/opensearch-pipeline.conf
# This checks for syntax errors WITHOUT actually starting Logstash
# Wait for: "Configuration OK"
```

###### Start Logstash cleanly

```bash
sudo systemctl start logstash

# Watch it start in real time — do this immediately after the above
sudo tail -f /var/log/logstash/logstash-plain.log
# Wait about 60-90 seconds
# You are looking for this line:
# "Successfully started Logstash API endpoint"
# You should NOT see "YOUR-OPENSEARCH-VM-IP" anywhere
# You should NOT see "ResolutionFailure" anywhere
```

###### Confirm both pipelines are running

```bash
sudo systemctl status logstash
# Should show: active (running)

# Confirm both ports are listening
sudo ss -tlnp | grep -E "5044|5045"
# Should show TWO lines — one for 5044 (wazuh pipeline) and one for 5045 (opensearch pipeline)
```

##### **Challenge:**

After fixing the previous challenge, when I am trying to access OpenSearch Dashboards from the browser to display the data in the UI, I faced another challenge: when opening the OpenSearch dashboard in the browser using the `http://XXX.XXX.XX.87:5601` URL, it displays "OpenSearch Dashboards server is not ready yet" in the browser.

###### Exact cause of the problem

Look at this output you got:

```bash
curl -k -u kibanaserver:XXXXXXXXXXXX https://localhost:9200
Unauthorized
```

The `kibanaserver` user is being **rejected** by OpenSearch. This is the root cause of everything.

Here is what is happening step by step:

OpenSearch Dashboards needs a dedicated internal user called `kibanaserver` to talk to OpenSearch. This user is built into OpenSearch's security plugin by default, but it has its **own separate default password**, not the admin password you set during installation. You set a custom password as the admin password, but `kibanaserver` still has its old default password. So when Dashboards tries to connect to OpenSearch using `kibanaserver:<admin password>`, OpenSearch says Unauthorized, and Dashboards cannot start properly — hence the "server is not ready yet" message in the browser.

Think of it like this: you changed the front door key but the internal staff door still uses the old key. The staff (Dashboards) cannot get in.

##### **Solution:**

###### Simpler but less secure — use only for lab

```bash
sudo nano /etc/opensearch-dashboards/opensearch_dashboards.yml
```

Find these two lines and change them:

```yaml
# Change FROM:
opensearch.username: "kibanaserver"
opensearch.password: "XXXXXXXXXXXX"

# Change TO:
opensearch.username: "admin"
opensearch.password: "XXXXXXXXXXXX"
```

```bash
sudo systemctl restart opensearch-dashboards
sudo journalctl -u opensearch-dashboards -f
# Watch for: Server running at http://0.0.0.0:5601
```

---

### 3.4 Step 4: Verify the Full Chain Is Working

```bash
# On Beat VM: is Metricbeat sending data?
sudo journalctl -u metricbeat -n 30 | grep -i "events published"
# "events published" confirms data is flowing out

# On Logstash VM: is the new pipeline receiving on port 5045?
sudo ss -tlnp | grep 5045
# ss = socket status tool, -tlnp = show TCP listening ports
# you should see logstash listening on 0.0.0.0:5045

# On Logstash VM: is existing pipeline still working?
sudo ss -tlnp | grep 5044
# should ALSO still show — confirms existing pipeline untouched

# On OpenSearch VM: did data arrive?
curl -k -u admin:XXXXXXXXXXXX \
  "https://localhost:9200/_cat/indices?v"
# After 1-2 minutes you should see: filebeat-health-2026.05.18

# Count documents to confirm data is flowing
curl -k -u admin:XXXXXXXXXXXX \
  "https://localhost:9200/filebeat-health-*/_count"
# Should return { "count": some_number_greater_than_0 }
```

And check by accessing OpenSearch Dashboards from the browser using `admin` / `<your admin password>` as username and password, and it will be based on your setup and configured values. When we open OpenSearch Dashboards for the first time it will ask a few setup questions. So first we need to complete those steps before seeing the data in the UI. This setup will look like the following:

![Image 2](<../../../assets/images/POC's/Sai krishna/Implementation Process to Monitor Filebeat Health/image2.png>)

After completing the initial setup, you can now see the incoming metric data in the Discover section in OpenSearch Dashboards, and it will look something like the following image.

![Image 3](<../../../assets/images/POC's/Sai krishna/Implementation Process to Monitor Filebeat Health/image3.png>)

Now data is reaching and being stored in OpenSearch, and we can verify data using the Discover section — that is fine, but for monitoring this is not the recommended way. To monitor agent health, we must create a dashboard to display all the agents on one screen, so that we can monitor multiple agents in one place — this is the recommended way.

---

## 4. FOR VISUALIZATIONS

### 4.1 First Confirm Your Data Has the Right Fields

| Your Requirement | Exact Field in Your Document |
|---|---|
| Is Filebeat running | `beat.stats.uptime.ms` — if growing = running |
| IP address | `host.ip` |
| Hostname | `host.hostname` |
| OS name | `host.os.name` |
| OS version | `host.os.version` |
| Filebeat type confirmation | `beat.type = "filebeat"` |
| Output errors | `beat.stats.libbeat.output.events.failed` |

---

### 4.2 Important: First Create the Index Pattern

Before creating any visualization you must do this once:

1. Click the top-left hamburger menu (☰)
2. Click "Management"
3. Click "Dashboards Management"
4. Click "Index Patterns"
5. Click "Create index pattern" button
6. In the text box type exactly: `filebeat-health-*`
7. Click "Next step"
8. In "Time field" dropdown select: `@timestamp`
9. Click "Create index pattern"

Now you are ready to build visualizations. If you already set this at initial setup, check and confirm once again.

---

### 4.3 Creating the Data Table — Exact Step by Step

Exact steps to create the Filebeat Health Data Table in OpenSearch Dashboards:

**Part A — Navigate to Visualize**

1. Click the ☰ hamburger menu on the top left of the page
2. In the left sidebar find and click Visualize
3. Click the Create visualization button (top right area)
4. A popup appears showing visualization types — scroll down and click Data Table
5. Another popup asks "Choose a source" — click `filebeat-health-*` (the index pattern you created)
6. You are now in the Data Table editor. You see a left panel (config) and right panel (preview table)

**Part B — Set the Time Range to See Your Data**

7. On the TOP RIGHT of the page click the time picker showing something like "Last 15 minutes"
8. Change it to "Last 24 hours" or "Today" — click Apply

**Part C — Add Filter to Show ONLY Filebeat Beat Data**

9. Click "+ Add filter" (below the search bar at the top of the page)
10. In "Field" dropdown — type and select `event.module`
11. In "Operator" dropdown — select `is`
12. In "Value" field — type `beat`
13. Click Save — you will see a blue filter tag appear: "event.module: beat"

**Part D — Configure the Metrics Section (Left Panel)**

14. Under Metrics you see "Count" by default. Click on it to expand it.
15. Change Aggregation from Count to Max
16. In "Field" that appears — type and select `beat.stats.uptime.ms`
17. In "Custom label" type: Uptime (ms)
18. Click "+ Add metrics" to add a second column
19. Select Aggregation: Max — Field: `beat.stats.libbeat.output.events.failed` — Label: Output Errors

**Part E — Configure Buckets (the Rows — One Row per Machine)**

20. Scroll down in the left panel to the Buckets section — click Add then click Split rows
21. Aggregation: select Terms
22. Field: type and select `host.hostname.keyword` — this creates one row per machine
23. Size: type 100 (so it shows up to 100 machines)
24. Custom label: type Hostname
25. Click the ▶ Play / Apply changes button at the top of the left panel — the table now shows one row per machine

**Part F — What You Will See in the Table Now**

26. Column 1 — auto — Hostname — e.g. dashboard, beat-vm
27. Column 2 — you added — Uptime (ms) — e.g. 1096256 = ~18 minutes running
28. Column 3 — you added — Output Errors — 0 = healthy, any number = problem

**Part G — Save the Visualization**

29. Click Save in the top right
30. Enter title: Filebeat Health Status — click Save

**Part H — Add IP, OS Columns Using Discover Instead (Important Note)**

**Note:** OpenSearch Dashboards Data Table in Visualize only supports metric aggregations (count, max, sum) — it cannot display raw text fields like IP address or OS name directly as columns. For those fields, use the Discover view, which shows raw documents in a table with any column you choose.

31. Go to ☰ menu → Discover
32. Select index `filebeat-health-*` from the dropdown top left
33. Add filter: `event.module is beat` (same as before)
34. On the left panel you see all field names — find and click `host.hostname` → click Add
35. Also add: `host.ip` → Add
36. Also add: `host.os.name` → Add
37. Also add: `host.os.version` → Add
38. Also add: `beat.stats.uptime.ms` → Add
39. Now your Discover view shows a clean table with all columns your seniors need
40. Click Save top right → name it "Filebeat Machine Info" → click Save

---

### 4.4 How to Interpret "Is Filebeat Running or Not" From the Uptime Field

##### Meaning:

This is the most important thing for your seniors. The `beat.stats.uptime.ms` field tells you exactly:

- If the number is **increasing** every time you refresh → Filebeat is running continuously
- If the number **resets to a small value** → Filebeat restarted recently
- If the machine **disappears from the table completely** → Filebeat has stopped and Metricbeat can no longer reach it on port 5066

A simple way to convert milliseconds to readable time for your seniors — uptime of 1096256 ms means 1096256 ÷ 1000 ÷ 60 = 18 minutes running.

---

### 4.5 Final Step: Create One Combined Dashboard

Once you have saved both the visualization and the Discover search:

1. Click ☰ menu → Dashboards
2. Click "Create dashboard"
3. Click "Add" in the toolbar
4. Search for "Filebeat Health Status" → click it to add
5. Search for "Filebeat Machine Info" → click it to add
6. Set time range to "Last 24 hours" top right
7. Click "Save" → name it "Filebeat Health Monitor"

After adding these two visualizations to the dashboard, the dashboard will look something like the image below:

![Image 4](<../../../assets/images/POC's/Sai krishna/Implementation Process to Monitor Filebeat Health/image4.png>)

This dashboard shows Filebeat health based on uptime, but if you observe Wazuh, it displays agent health as "active" for running agents and "disconnected" for agents that are not running. The correct tool for this is **Vega visualization** — it lets you write a custom visual using your OpenSearch data and display exactly what you want, including colored status indicators like Wazuh does.

---

### 4.6 How It Will Work (Vega Visualization)

##### Meaning:

Vega queries your OpenSearch index directly, finds the latest document per machine, checks when it last sent data, and shows green if data arrived in the last 2 minutes, red if not. This is exactly how Wazuh works — based on last-seen time, not counter values.

---

### 4.7 Exact Steps

#### Step 1: Create New Vega Visualization

☰ menu → Visualize → Create visualization
Scroll down and select: Vega

You will see a text editor with sample Vega code. **Delete everything** in that editor and paste the exact code below.

#### Step 2: Paste This Complete Vega Spec

```json
{
  "$schema": "https://vega.github.io/schema/vega/v5.json",
  "width": 700,
  "height": 300,
  "padding": 20,

  "data": [
    {
      "name": "beats",
      "url": {
        "%context%": true,
        "%timefield%": "@timestamp",
        "index": "filebeat-health-*",
        "body": {
          "size": 0,
          "query": {
            "bool": {
              "must": [
                {"term": {"event.module": "beat"}},
                {"term": {"beat.type": "filebeat"}}
              ]
            }
          },
          "aggs": {
            "by_host": {
              "terms": {
                "field": "host.hostname.keyword",
                "size": 50
              },
              "aggs": {
                "latest_doc": {
                  "top_hits": {
                    "size": 1,
                    "sort": [{"@timestamp": {"order": "desc"}}],
                    "_source": {
                      "includes": [
                        "host.hostname",
                        "host.ip",
                        "host.os.name",
                        "host.os.version",
                        "beat.stats.uptime.ms",
                        "@timestamp"
                      ]
                    }
                  }
                }
              }
            }
          }
        }
      },
      "format": {"property": "aggregations.by_host.buckets"},
      "transform": [
        {
          "type": "formula",
          "as": "hostname",
          "expr": "datum.key"
        },
        {
          "type": "formula",
          "as": "last_seen",
          "expr": "datum.latest_doc.hits.hits[0]._source['@timestamp']"
        },
        {
          "type": "formula",
          "as": "uptime_ms",
          "expr": "datum.latest_doc.hits.hits[0]._source.beat.stats.uptime.ms"
        },
        {
          "type": "formula",
          "as": "ip",
          "expr": "isArray(datum.latest_doc.hits.hits[0]._source.host.ip) ? datum.latest_doc.hits.hits[0]._source.host.ip[0] : datum.latest_doc.hits.hits[0]._source.host.ip"
        },
        {
          "type": "formula",
          "as": "os_name",
          "expr": "datum.latest_doc.hits.hits[0]._source.host.os.name"
        },
        {
          "type": "formula",
          "as": "os_version",
          "expr": "datum.latest_doc.hits.hits[0]._source.host.os.version"
        },
        {
          "type": "formula",
          "as": "uptime_min",
          "expr": "round(datum.uptime_ms / 60000)"
        },
        {
          "type": "formula",
          "as": "last_seen_ms",
          "expr": "toDate(datum.last_seen)"
        },
        {
          "type": "formula",
          "as": "seconds_ago",
          "expr": "(now() - toDate(datum.last_seen)) / 1000"
        },
        {
          "type": "formula",
          "as": "status",
          "expr": "datum.seconds_ago < 120 ? 'Active' : 'Disconnected'"
        },
        {
          "type": "formula",
          "as": "status_color",
          "expr": "datum.seconds_ago < 120 ? '#1aab61' : '#bd271e'"
        }
      ]
    }
  ],

  "scales": [
    {
      "name": "y",
      "type": "band",
      "domain": {"data": "beats", "field": "hostname"},
      "range": "height",
      "padding": 0.3
    }
  ],

  "marks": [
    {
      "type": "rect",
      "from": {"data": "beats"},
      "encode": {
        "enter": {
          "x": {"value": 0},
          "width": {"signal": "width"},
          "y": {"scale": "y", "field": "hostname"},
          "height": {"scale": "y", "band": 1},
          "fill": {"value": "#f5f7fa"},
          "stroke": {"value": "#dde3ed"},
          "strokeWidth": {"value": 1},
          "cornerRadius": {"value": 6}
        }
      }
    },
    {
      "type": "symbol",
      "from": {"data": "beats"},
      "encode": {
        "enter": {
          "shape": {"value": "circle"},
          "size": {"value": 120},
          "x": {"value": 24},
          "y": {
            "scale": "y",
            "field": "hostname",
            "band": 0.5
          },
          "fill": {"field": "status_color"},
          "stroke": {"value": "#ffffff"},
          "strokeWidth": {"value": 2}
        }
      }
    },
    {
      "type": "text",
      "from": {"data": "beats"},
      "encode": {
        "enter": {
          "x": {"value": 44},
          "y": {"scale": "y", "field": "hostname", "band": 0.5},
          "baseline": {"value": "middle"},
          "text": {"field": "status"},
          "fill": {"field": "status_color"},
          "font": {"value": "Inter, sans-serif"},
          "fontSize": {"value": 12},
          "fontWeight": {"value": "bold"}
        }
      }
    },
    {
      "type": "text",
      "from": {"data": "beats"},
      "encode": {
        "enter": {
          "x": {"value": 120},
          "y": {"scale": "y", "field": "hostname", "band": 0.5},
          "baseline": {"value": "middle"},
          "text": {"field": "hostname"},
          "fill": {"value": "#1a1c21"},
          "font": {"value": "Inter, sans-serif"},
          "fontSize": {"value": 13},
          "fontWeight": {"value": "600"}
        }
      }
    },
    {
      "type": "text",
      "from": {"data": "beats"},
      "encode": {
        "enter": {
          "x": {"value": 280},
          "y": {"scale": "y", "field": "hostname", "band": 0.5},
          "baseline": {"value": "middle"},
          "text": {"signal": "'IP: ' + datum.ip"},
          "fill": {"value": "#535966"},
          "font": {"value": "Inter, sans-serif"},
          "fontSize": {"value": 12}
        }
      }
    },
    {
      "type": "text",
      "from": {"data": "beats"},
      "encode": {
        "enter": {
          "x": {"value": 440},
          "y": {"scale": "y", "field": "hostname", "band": 0.5},
          "baseline": {"value": "middle"},
          "text": {"signal": "datum.os_name + ' ' + datum.os_version"},
          "fill": {"value": "#535966"},
          "font": {"value": "Inter, sans-serif"},
          "fontSize": {"value": 12}
        }
      }
    },
    {
      "type": "text",
      "from": {"data": "beats"},
      "encode": {
        "enter": {
          "x": {"value": 630},
          "y": {"scale": "y", "field": "hostname", "band": 0.5},
          "baseline": {"value": "middle"},
          "text": {"signal": "datum.uptime_min + ' min'"},
          "fill": {"value": "#535966"},
          "font": {"value": "Inter, sans-serif"},
          "fontSize": {"value": 12}
        }
      }
    },
    {
      "type": "text",
      "encode": {
        "enter": {
          "x": {"value": 44},
          "y": {"value": -8},
          "text": {"value": "Status"},
          "fill": {"value": "#98a2b3"},
          "fontSize": {"value": 11},
          "fontWeight": {"value": "600"}
        }
      }
    },
    {
      "type": "text",
      "encode": {
        "enter": {
          "x": {"value": 120},
          "y": {"value": -8},
          "text": {"value": "Hostname"},
          "fill": {"value": "#98a2b3"},
          "fontSize": {"value": 11},
          "fontWeight": {"value": "600"}
        }
      }
    },
    {
      "type": "text",
      "encode": {
        "enter": {
          "x": {"value": 280},
          "y": {"value": -8},
          "text": {"value": "IP Address"},
          "fill": {"value": "#98a2b3"},
          "fontSize": {"value": 11},
          "fontWeight": {"value": "600"}
        }
      }
    },
    {
      "type": "text",
      "encode": {
        "enter": {
          "x": {"value": 440},
          "y": {"value": -8},
          "text": {"value": "OS"},
          "fill": {"value": "#98a2b3"},
          "fontSize": {"value": 11},
          "fontWeight": {"value": "600"}
        }
      }
    },
    {
      "type": "text",
      "encode": {
        "enter": {
          "x": {"value": 630},
          "y": {"value": -8},
          "text": {"value": "Uptime"},
          "fill": {"value": "#98a2b3"},
          "fontSize": {"value": 11},
          "fontWeight": {"value": "600"}
        }
      }
    }
  ]
}
```

#### Step 3: Click Update

Click the blue "Update" button at the top of the editor.

You will immediately see a table-style panel with one row per machine showing a green circle + "Active" or red circle + "Disconnected", plus hostname, IP, OS, and uptime in minutes.

#### Step 4: Save

Click Save top right
Name: Filebeat Status Panel
Click Save

Then add it to your existing dashboard:

☰ → Dashboards → Filebeat Health Monitor → Edit
→ Add → select "Filebeat Status Panel"
→ Save dashboard

---

### 4.8 How the Active/Disconnected Logic Works

##### Meaning:

The Vega spec calculates exactly this:

```
seconds_ago = current time - last document timestamp

if seconds_ago < 120 seconds (2 minutes):
    status = Active → green circle
else:
    status = Disconnected → red circle
```

You can change the 120 to any number of seconds you want. For example, change it to 300 for 5 minutes tolerance. Find this line in the code:

```
"expr": "datum.seconds_ago < 120 ? 'Active' : 'Disconnected'"
```

Change 120 to 300 for 5 minutes. Change the same number in the color line below it too.

##### **Challenge:**

When I run the above custom script in OpenSearch Dashboards in Vega Visualize, it actually gives errors while running the script. The errors are: `"width" and "height" params are ignored because "autosize" is enabled. Set "autosize": "none" to disable` and `url.%context% and url.%timefield% must not be used when url.body.query is set`.

**Error 1** — width and height are ignored because OpenSearch Dashboards enables autosize automatically. The fix is to add `"autosize": "none"`.

**Error 2** — In OpenSearch Dashboards Vega, you cannot use `%context%` and `%timefield%` together with a custom `body.query`. You must choose one approach. Since we need a custom query to filter by `event.module` and `beat.type`, we remove `%context%` and `%timefield%` and write the time filter manually inside the query.

##### **Solution:**

I updated the code, and now the errors are resolved. But visually it was not good, so I again updated the code to look better, and the condition was also changed to 30 seconds instead of 120 seconds, and I updated the code to display 0 minutes when Filebeat is not running. After making all these modifications, the final code is as follows:

```json
{
  "$schema": "https://vega.github.io/schema/vega/v5.json",
  "autosize": "none",
  "width": 800,
  "height": 600,
  "padding": {"top": 40, "left": 10, "right": 10, "bottom": 10},
  "data": [
    {
      "name": "beats",
      "url": {
        "index": "filebeat-health-*",
        "body": {
          "size": 0,
          "query": {
            "bool": {
              "must": [
                {"term": {"event.module": "beat"}},
                {"term": {"beat.type": "filebeat"}}
              ],
              "filter": [
                {
                  "range": {
                    "@timestamp": {
                      "gte": "now-24h",
                      "lte": "now"
                    }
                  }
                }
              ]
            }
          },
          "aggs": {
            "by_host": {
              "terms": {
                "field": "host.hostname.keyword",
                "size": 50,
                "order": {"_key": "asc"}
              },
              "aggs": {
                "latest_doc": {
                  "top_hits": {
                    "size": 1,
                    "sort": [{"@timestamp": {"order": "desc"}}],
                    "_source": {
                      "includes": [
                        "host.hostname",
                        "host.ip",
                        "host.os.name",
                        "host.os.version",
                        "beat.stats.uptime.ms",
                        "@timestamp"
                      ]
                    }
                  }
                }
              }
            }
          }
        }
      },
      "format": {"property": "aggregations.by_host.buckets"},
      "transform": [
        {
          "type": "formula",
          "as": "hostname",
          "expr": "datum.key"
        },
        {
          "type": "formula",
          "as": "last_seen",
          "expr": "datum.latest_doc.hits.hits[0]._source['@timestamp']"
        },
        {
          "type": "formula",
          "as": "uptime_ms",
          "expr": "datum.latest_doc.hits.hits[0]._source.beat.stats.uptime.ms"
        },
        {
          "type": "formula",
          "as": "ip",
          "expr": "isArray(datum.latest_doc.hits.hits[0]._source.host.ip) ? datum.latest_doc.hits.hits[0]._source.host.ip[0] : datum.latest_doc.hits.hits[0]._source.host.ip"
        },
        {
          "type": "formula",
          "as": "os_name",
          "expr": "datum.latest_doc.hits.hits[0]._source.host.os.name"
        },
        {
          "type": "formula",
          "as": "os_version",
          "expr": "datum.latest_doc.hits.hits[0]._source.host.os.version"
        },
        {
          "type": "formula",
          "as": "seconds_ago",
          "expr": "(now() - toDate(datum.last_seen)) / 1000"
        },
        {
          "type": "formula",
          "as": "is_active",
          "expr": "datum.seconds_ago < 30"
        },
        {
          "type": "formula",
          "as": "uptime_min",
          "expr": "datum.is_active ? round(datum.uptime_ms / 60000) : 0"
        },
        {
          "type": "formula",
          "as": "status",
          "expr": "datum.is_active ? 'Active' : 'Disconnected'"
        },
        {
          "type": "formula",
          "as": "status_color",
          "expr": "datum.is_active ? '#1aab61' : '#bd271e'"
        }
      ]
    },
    {
      "name": "beats_indexed",
      "source": "beats",
      "transform": [
        {
          "type": "window",
          "ops": ["row_number"],
          "as": ["row_num"]
        },
        {
          "type": "formula",
          "as": "row_num",
          "expr": "datum.row_num - 1"
        }
      ]
    }
  ],
  "signals": [
    {"name": "rowHeight", "value": 48},
    {"name": "totalRows", "update": "length(data('beats_indexed'))"},
    {"name": "dynamicHeight", "update": "max(totalRows * rowHeight, rowHeight)"},
    {"name": "dotX", "value": 18},
    {"name": "statusX", "value": 32},
    {"name": "hostnameX", "value": 160},
    {"name": "ipX", "value": 310},
    {"name": "osX", "value": 450},
    {"name": "uptimeX", "value": 610}
  ],
  "scales": [
    {
      "name": "y",
      "type": "band",
      "domain": {"data": "beats_indexed", "field": "row_num"},
      "range": [0, {"signal": "dynamicHeight"}],
      "padding": 0.15
    }
  ],
  "marks": [
    {
      "type": "text",
      "encode": {
        "enter": {
          "x": {"signal": "dotX"},
          "y": {"value": -16},
          "text": {"value": "Status"},
          "fill": {"value": "#888780"},
          "fontSize": {"value": 11},
          "fontWeight": {"value": "600"},
          "baseline": {"value": "middle"}
        }
      }
    },
    {
      "type": "text",
      "encode": {
        "enter": {
          "x": {"signal": "hostnameX"},
          "y": {"value": -16},
          "text": {"value": "Hostname"},
          "fill": {"value": "#888780"},
          "fontSize": {"value": 11},
          "fontWeight": {"value": "600"},
          "baseline": {"value": "middle"}
        }
      }
    },
    {
      "type": "text",
      "encode": {
        "enter": {
          "x": {"signal": "ipX"},
          "y": {"value": -16},
          "text": {"value": "IP Address"},
          "fill": {"value": "#888780"},
          "fontSize": {"value": 11},
          "fontWeight": {"value": "600"},
          "baseline": {"value": "middle"}
        }
      }
    },
    {
      "type": "text",
      "encode": {
        "enter": {
          "x": {"signal": "osX"},
          "y": {"value": -16},
          "text": {"value": "OS"},
          "fill": {"value": "#888780"},
          "fontSize": {"value": 11},
          "fontWeight": {"value": "600"},
          "baseline": {"value": "middle"}
        }
      }
    },
    {
      "type": "text",
      "encode": {
        "enter": {
          "x": {"signal": "uptimeX"},
          "y": {"value": -16},
          "text": {"value": "Uptime"},
          "fill": {"value": "#888780"},
          "fontSize": {"value": 11},
          "fontWeight": {"value": "600"},
          "baseline": {"value": "middle"}
        }
      }
    },
    {
      "type": "rect",
      "from": {"data": "beats_indexed"},
      "encode": {
        "enter": {
          "x": {"value": 0},
          "width": {"value": 700},
          "y": {"scale": "y", "field": "row_num"},
          "height": {"scale": "y", "band": 1},
          "fill": [
            {"test": "datum.row_num % 2 === 0", "value": "#f5f7fa"},
            {"value": "#ffffff"}
          ],
          "stroke": {"value": "#e3e6eb"},
          "strokeWidth": {"value": 0.5},
          "cornerRadius": {"value": 5}
        }
      }
    },
    {
      "type": "symbol",
      "from": {"data": "beats_indexed"},
      "encode": {
        "enter": {
          "shape": {"value": "circle"},
          "size": {"value": 90},
          "x": {"signal": "dotX"},
          "y": {"scale": "y", "field": "row_num", "band": 0.5},
          "fill": {"field": "status_color"},
          "stroke": {"value": "#ffffff"},
          "strokeWidth": {"value": 1.5}
        }
      }
    },
    {
      "type": "text",
      "from": {"data": "beats_indexed"},
      "encode": {
        "enter": {
          "x": {"signal": "statusX"},
          "y": {"scale": "y", "field": "row_num", "band": 0.5},
          "baseline": {"value": "middle"},
          "text": {"field": "status"},
          "fill": {"field": "status_color"},
          "fontSize": {"value": 12},
          "fontWeight": {"value": "bold"},
          "limit": {"value": 110}
        }
      }
    },
    {
      "type": "text",
      "from": {"data": "beats_indexed"},
      "encode": {
        "enter": {
          "x": {"signal": "hostnameX"},
          "y": {"scale": "y", "field": "row_num", "band": 0.5},
          "baseline": {"value": "middle"},
          "text": {"field": "hostname"},
          "fill": {"value": "#1a1c21"},
          "fontSize": {"value": 12},
          "fontWeight": {"value": "600"},
          "limit": {"value": 130}
        }
      }
    },
    {
      "type": "text",
      "from": {"data": "beats_indexed"},
      "encode": {
        "enter": {
          "x": {"signal": "ipX"},
          "y": {"scale": "y", "field": "row_num", "band": 0.5},
          "baseline": {"value": "middle"},
          "text": {"field": "ip"},
          "fill": {"value": "#535966"},
          "fontSize": {"value": 11},
          "limit": {"value": 125}
        }
      }
    },
    {
      "type": "text",
      "from": {"data": "beats_indexed"},
      "encode": {
        "enter": {
          "x": {"signal": "osX"},
          "y": {"scale": "y", "field": "row_num", "band": 0.5},
          "baseline": {"value": "middle"},
          "text": {"signal": "datum.os_name + ' ' + datum.os_version"},
          "fill": {"value": "#535966"},
          "fontSize": {"value": 11},
          "limit": {"value": 140}
        }
      }
    },
    {
      "type": "text",
      "from": {"data": "beats_indexed"},
      "encode": {
        "enter": {
          "x": {"signal": "uptimeX"},
          "y": {"scale": "y", "field": "row_num", "band": 0.5},
          "baseline": {"value": "middle"},
          "text": {"signal": "datum.uptime_min + ' min'"},
          "fill": {"value": "#535966"},
          "fontSize": {"value": 11},
          "limit": {"value": 70}
        }
      }
    }
  ]
}
```

For the above code, it will look the same as the image below, with each Filebeat agent as one row:

![Image 5](<../../../assets/images/POC's/Sai krishna/Implementation Process to Monitor Filebeat Health/image5.png>)

---