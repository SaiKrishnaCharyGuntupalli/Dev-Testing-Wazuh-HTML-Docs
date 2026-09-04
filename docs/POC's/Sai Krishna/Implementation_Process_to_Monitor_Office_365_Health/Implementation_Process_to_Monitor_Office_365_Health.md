# IMPLEMENTATION PROCESS TO MONITOR OFFICE 365 HEALTH


## 1. OVERVIEW

This document describes the implementation of a health monitoring solution for the Office 365 (Microsoft 365) log integration in Wazuh, following the same Active / Idle / Inactive monitoring pattern already implemented for AWS CloudTrail and Azure Activity Log.

---

## 2. PREREQUISITES

- Office 365 logs are expected to already be reaching the dedicated Wazuh instance and being stored in `wazuh-alerts-*`, tagged with `data.integration:office365`. (Log forwarding setup is out of scope for this document.)
- Wazuh Indexer (OpenSearch) API access with a user that has read/write permissions.
- Python 3 installed on the monitoring VM (Logstash VM), with the `requests` library available.
- Access to the Wazuh Dashboard to create a Vega visualization.

---

## 3. WHAT IS ALREADY DONE

- Office 365 logs are confirmed flowing into `wazuh-alerts-*` with field `data.integration: office365` (verified as a keyword field via index mapping check).
- Health status pattern (Active / Idle / Inactive) already implemented and proven for AWS CloudTrail and Azure Activity Log — this implementation reuses that same structure for consistency across the dashboard.

---

## 4. MONITORING APPROACH

- A scheduled Python script queries the Wazuh Indexer for the most recent Office 365 log (`data.integration:office365`), calculates the time gap since that log arrived, and classifies the pipeline status.
- The result is written to a dedicated health-status index (`office365-health-status`) as a single, continuously overwritten document.
- A Vega visualization in the Wazuh Dashboard reads that document and displays it as a colored status row.

### Status Thresholds

| Status | Condition |
|---|---|
| Active | Last Office 365 log received ≤ 15 minutes ago |
| Idle | Last Office 365 log received 15–60 minutes ago |
| Inactive | Last Office 365 log received > 60 minutes ago (or no log found) |

---

## 5. SETUP COMMANDS

Run this commands inside the VM where you can access the Wazuh VM from this VM (Logstash VM)

**Create directory and file:**

```bash
mkdir -p /opt/wazuh-scripts
nano /opt/wazuh-scripts/office365_health_monitor.py
```

(Paste script content, save, exit.)

---

## 6. PYTHON MONITORING SCRIPT

**Location:** `/opt/wazuh-scripts/office365_health_monitor.py`

```python
#!/usr/bin/env python3
import requests, json, urllib3
from datetime import datetime, timezone

urllib3.disable_warnings()

INDEXER_HOST = "https://<INDEXER_IP>:9200"
INDEXER_USER = "<INDEXER_USERNAME>"
INDEXER_PASS = "<INDEXER_PASSWORD>"
ALERTS_INDEX = "wazuh-alerts-*"
HEALTH_INDEX = "office365-health-status"

ACTIVE_MIN = 15  # minutes
IDLE_MIN = 60    # minutes

def get_last_office365_event():
    query = {
        "size": 1,
        "sort": [{"@timestamp": {"order": "desc"}}],
        "query": {"term": {"data.integration": "office365"}}
    }
    r = requests.post(f"{INDEXER_HOST}/{ALERTS_INDEX}/_search",
                       auth=(INDEXER_USER, INDEXER_PASS),
                       json=query, verify=False, timeout=10)
    r.raise_for_status()
    hits = r.json()["hits"]["hits"]
    return hits[0]["_source"]["@timestamp"] if hits else None

def classify(last_event_time_str):
    if last_event_time_str is None:
        return "inactive", None
    last_time = datetime.fromisoformat(last_event_time_str.replace("Z", "+00:00"))
    gap_min = (datetime.now(timezone.utc) - last_time).total_seconds() / 60
    if gap_min <= ACTIVE_MIN:
        status = "active"
    elif gap_min <= IDLE_MIN:
        status = "idle"
    else:
        status = "inactive"
    return status, round(gap_min)

def write_status(status, gap_min):
    doc = {
        "status": status.capitalize(),
        "service": "office365",
        "reason": f"Last log delivered {gap_min} min ago" if gap_min is not None else "No logs received",
        "@timestamp": datetime.now(timezone.utc).isoformat()
    }
    requests.put(f"{INDEXER_HOST}/{HEALTH_INDEX}/_doc/office365_health",
                 auth=(INDEXER_USER, INDEXER_PASS),
                 json=doc, verify=False, timeout=10)

if __name__ == "__main__":
    last_time = get_last_office365_event()
    status, gap = classify(last_time)
    write_status(status, gap)
    print(f"[{datetime.now()}] office365 status={status} gap={gap}min")
```

**Install dependency:**

```bash
pip install requests
```

**Expected output:** confirmation that requests is installed or already satisfied.

**Make executable:**

```bash
chmod +x /opt/wazuh-scripts/office365_health_monitor.py
```

---

## 7. TESTING THE SCRIPT

**Check the Office 365 log count before running (baseline):**

```bash
curl -k -u <INDEXER_USERNAME>:<INDEXER_PASSWORD> \
  "https://<INDEXER_IP>:9200/wazuh-alerts-*/_count" \
  -H 'Content-Type: application/json' \
  -d '{"query": {"term": {"data.integration": "office365"}}}'
```

**Expected output:** a JSON response containing a "count" field, e.g. `{"count": 128, ...}`. Note this value.

**Run the script manually:**

```bash
python3 /opt/wazuh-scripts/office365_health_monitor.py
```

**Expected output:**

```
[2026-09-03 10:05:12] office365 status=active gap=2min
```

**Verify the health document was written:**

```bash
curl -k -u <INDEXER_USERNAME>:<INDEXER_PASSWORD> \
  "https://<INDEXER_IP>:9200/office365-health-status/_doc/office365_health"
```

**Expected output:** a JSON document showing `"status": "Active"`, `"service": "office365"`, `"reason": "Last log delivered 2 min ago"`, and a recent `"@timestamp"`.

**Re-check the log count after a few minutes (to confirm logs are still increasing):**

```bash
curl -k -u <INDEXER_USERNAME>:<INDEXER_PASSWORD> \
  "https://<INDEXER_IP>:9200/wazuh-alerts-*/_count" \
  -H 'Content-Type: application/json' \
  -d '{"query": {"term": {"data.integration": "office365"}}}'
```

**Expected output:** a "count" value higher than the baseline recorded earlier, confirming new Office 365 logs are actively arriving.

---

## 8. SCHEDULING (CRON)

```bash
crontab -e
```

Add the following line, save and exit:

```
*/5 * * * * /usr/bin/python3 /opt/wazuh-scripts/office365_health_monitor.py >> /var/log/office365_health.log 2>&1
```

**Verify cron is registered:**

```bash
crontab -l
```

**Expected output:** the cron line listed above.

**Monitor execution over time:**

```bash
tail -f /var/log/office365_health.log
```

**Expected output:** a new log line every 5 minutes, e.g.:

```
[2026-09-03 10:10:03] office365 status=active gap=1min
[2026-09-03 10:15:04] office365 status=active gap=0min
```

**Confirm the health document is being updated (not just created once):**

```bash
curl -k -u <INDEXER_USERNAME>:<INDEXER_PASSWORD> \
  "https://<INDEXER_IP>:9200/office365-health-status/_doc/office365_health"
```

**Expected output:** the "@timestamp" field advances forward on each check, confirming the cron job is actively refreshing the status.

---

## 9. VEGA VISUALIZATION (WAZUH DASHBOARD)

**Location:** Wazuh Dashboard → Visualize → Create Visualization → Vega

```json
{
  "$schema": "https://vega.github.io/schema/vega/v5.json",
  "autosize": "none",
  "width": 1000,
  "height": 300,
  "title": {
    "text": "Office 365 Health",
    "fontSize": 20,
    "anchor": "start"
  },
  "data": [
    {
      "name": "health",
      "url": {
        "%context%": true,
        "index": "office365-health-status",
        "body": {
          "size": 1,
          "sort": [
            { "@timestamp": { "order": "desc" } }
          ]
        }
      },
      "format": {
        "property": "hits.hits"
      },
      "transform": [
        {
          "type": "window",
          "ops": ["row_number"],
          "as": ["row"]
        }
      ]
    }
  ],
  "marks": [
    {
      "type": "text",
      "encode": {
        "enter": {
          "x": {"value": 50},
          "y": {"value": 40},
          "fontSize": {"value": 16},
          "fontWeight": {"value": "bold"},
          "text": {"value": "Status"}
        }
      }
    },
    {
      "type": "text",
      "encode": {
        "enter": {
          "x": {"value": 250},
          "y": {"value": 40},
          "fontSize": {"value": 16},
          "fontWeight": {"value": "bold"},
          "text": {"value": "Service"}
        }
      }
    },
    {
      "type": "text",
      "encode": {
        "enter": {
          "x": {"value": 450},
          "y": {"value": 40},
          "fontSize": {"value": 16},
          "fontWeight": {"value": "bold"},
          "text": {"value": "Reason"}
        }
      }
    },
    {
      "type": "symbol",
      "from": { "data": "health" },
      "encode": {
        "enter": {
          "x": {"value": 60},
          "y": {"signal": "40 + datum.row * 30"},
          "size": {"value": 250},
          "shape": {"value": "circle"}
        },
        "update": {
          "fill": {
            "signal": "datum._source.status === 'Active' ? '#22c55e' : datum._source.status === 'Idle' ? '#f59e0b' : '#ef4444'"
          }
        }
      }
    },
    {
      "type": "text",
      "from": { "data": "health" },
      "encode": {
        "enter": {
          "x": {"value": 80},
          "y": {"signal": "45 + datum.row * 30"},
          "fontSize": {"value": 13},
          "text": {"signal": "datum._source.status"}
        }
      }
    },
    {
      "type": "text",
      "from": { "data": "health" },
      "encode": {
        "enter": {
          "x": {"value": 250},
          "y": {"signal": "45 + datum.row * 30"},
          "fontSize": {"value": 13},
          "text": {"signal": "datum._source.service"}
        }
      }
    },
    {
      "type": "text",
      "from": { "data": "health" },
      "encode": {
        "enter": {
          "x": {"value": 450},
          "y": {"signal": "45 + datum.row * 30"},
          "fontSize": {"value": 13},
          "text": {"signal": "datum._source.reason"}
        }
      }
    }
  ]
}
```

