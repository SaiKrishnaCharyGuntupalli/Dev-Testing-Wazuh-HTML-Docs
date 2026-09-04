# IMPLEMENTATION PROCESS TO MONITOR AWS CLOUDTRAIL HEALTH

## Prerequisites 

**AWS side**

- An active AWS CloudTrail trail already created and logging (your existing pipeline's trail) 

- IAM permissions to create IAM users/policies (or ask an admin to do it) 

- A dedicated IAM user (e.g. cloudtrail-health-monitor) with a CloudTrailHealthReadOnly policy granting cloudtrail:GetTrailStatus 

- Access Key ID + Secret Access Key generated for that IAM user 

**Wazuh / OpenSearch side** 

- Working Wazuh Indexer (OpenSearch) node reachable from the Logstash VM on port 9200 

- Wazuh Indexer username and password with permission to create an index and write documents 

- Wazuh Dashboard access to create a Vega visualization 

**Logstash VM side** 

- Linux VM with Python 3 already installed (confirm with python3 --version) 

- pip3 available, with boto3 and requests libraries installed 

- Outbound network access from the VM to AWS API endpoints and to the Wazuh Indexer 

- cron installed and running on the VM (standard on most Linux distros) 

- Sudo/root access to create the script directory, log directory, and edit crontab 

Knowledge/values to have on hand before starting 

- AWS region of the trail 

- Exact trail name 

- Wazuh Indexer host/IP and port 

- Target index name to use (e.g. aws-cloudtrail-health-status) 

CloudTrail is actually simpler — AWS gives you a purpose-built API for exactly this: **GetTrailStatus**. It tells you directly:

- **IsLogging** → is the trail turned on at all
- **LatestDeliveryTime** → when the last log file was written to S3
- **LatestDeliveryError** → if delivery is failing

So the status logic becomes:

| Condition | Status |
|---|---|
| IsLogging = False | **Inactive** |
| Delivery error present | **Inactive** |
| Last delivery within 20 min | **Active** |
| Last delivery older than 20 min (or none yet) | **Idle** |

Same shape as your Azure script (get creds → call API → evaluate → push to OpenSearch → cron), just a cleaner check.

---

## WHAT YOU NEED, AND WHERE TO FIND IT

| Value | Where to Get It |
|---|---|
| AWS_ACCESS_KEY_ID / AWS_SECRET_ACCESS_KEY | IAM → Users → create a dedicated user (e.g. `cloudtrail-health-monitor`) → Security credentials → Create access key |
| AWS_REGION | The region your trail is in — CloudTrail console, top-right region selector, or on the trail's detail page |
| TRAIL_NAME | CloudTrail → Trails → the "Trail name" column (same trail from your ingestion pipeline) |
| OPENSEARCH_HOST / INDEX_NAME / username / password | Same Wazuh Indexer node you already use; just a new index name |

---

## FOR AWS ACCESS KEY ID / AWS SECRET ACCESS KEY

### 1. Create the IAM Policy First

AWS Console → search "IAM" → left sidebar Policies → Create policy → click JSON tab → paste:

```json
{"Version":"2012-10-17","Statement":[{"Effect":"Allow","Action":"cloudtrail:GetTrailStatus","Resource":"*"}]}
```

→ Next → Name it `CloudTrailHealthReadOnly` → Create policy.

### 2. Create the IAM User

IAM → left sidebar Users → Create user → Name it `cloudtrail-health-monitor` → do NOT tick "Provide user access to the AWS Management Console" (this user only needs API access, not a login) → Next.

### 3. Attach the Policy to the User

On the permissions step, choose "Attach policies directly" → search `CloudTrailHealthReadOnly` → tick it → Next → Create user.

### 4. Open the New User and Go to Security Credentials

IAM → Users → click `cloudtrail-health-monitor` → click the "Security credentials" tab (next to Permissions, Groups, Tags).

### 5. Create the Access Key

Scroll to "Access keys" → Create access key → when asked for use case, choose "Application running outside AWS" (also labelled Third-party service in some accounts) → tick the confirmation checkbox → Next → Description tag is optional → Create access key.

### 6. Copy Both Values Now — Shown Only Once

You'll see Access key ID and Secret access key on screen. Copy both immediately (or click Download .csv file). Once you leave this page, AWS will never show the secret again — if you lose it you must delete the key and create a new one.

### 7. Paste Into the Script

In `aws_cloudtrail_health.py`, set `AWS_ACCESS_KEY_ID` to the Access key ID and `AWS_SECRET_ACCESS_KEY` to the Secret access key you just copied.

**Note:** Take any VM which can access Wazuh Indexer (e.g. Logstash VM).

---

## SETUP ON THE LOGSTASH VM

Since Python's already installed:

**Note:** If Python is not installed, then install it first, then start following the remaining steps.

```bash
pip3 install boto3 requests
sudo mkdir -p /opt/aws-cloudtrail-health-monitor
cd /opt/aws-cloudtrail-health-monitor
```

Save the below `aws_cloudtrail_health.py` here, fill in the CONFIG block at the top with the 8 values from the table above:

**AWS CloudTrail health file:**

```python
import boto3
import requests
import json

from datetime import datetime, timezone
import urllib3

urllib3.disable_warnings()

# =========================
# CONFIG (EDIT THIS)
# =========================

AWS_ACCESS_KEY_ID = "your-access-key-id"
AWS_SECRET_ACCESS_KEY = "your-secret-access-key"
AWS_REGION = "your-region"       # e.g. ap-south-1
TRAIL_NAME = "your-trail-name"   # e.g. org-cloudtrail

# Wazuh Indexer (OpenSearch)
OPENSEARCH_HOST = "https://xx.xx.xx.xx:9200"
OPENSEARCH_USER = "your-wazuh-indexer-username"
OPENSEARCH_PASS = "your-wazuh-indexer-password"
INDEX_NAME = "aws-cloudtrail-health-status"

# If the last delivered log file is older than this, we call it Idle
# instead of Active. CloudTrail typically delivers within ~15 min.
IDLE_THRESHOLD_MINUTES = 20

# =========================
# GET CLOUDTRAIL STATUS
# =========================

def get_trail_status():
    try:
        client = boto3.client(
            "cloudtrail",
            region_name=AWS_REGION,
            aws_access_key_id=AWS_ACCESS_KEY_ID,
            aws_secret_access_key=AWS_SECRET_ACCESS_KEY,
        )
        return client.get_trail_status(Name=TRAIL_NAME)
    except Exception as e:
        print("Error calling AWS API:", e)
        return None

# =========================
# CHECK STATUS LOGIC
# =========================

def evaluate_status(trail_status):
    if trail_status is None:
        return "Inactive", "No response from AWS API"

    try:
        is_logging = trail_status.get("IsLogging", False)

        if not is_logging:
            return "Inactive", "Trail logging is turned off"

        delivery_error = trail_status.get("LatestDeliveryError")
        if delivery_error:
            return "Inactive", f"Delivery error: {delivery_error}"

        latest_delivery = trail_status.get("LatestDeliveryTime")

        if latest_delivery is None:
            return "Idle", "Logging is on, but no log file delivered yet"

        age_minutes = (datetime.now(timezone.utc) - latest_delivery).total_seconds() / 60

        if age_minutes <= IDLE_THRESHOLD_MINUTES:
            return "Active", f"Last log delivered {int(age_minutes)} min ago"

        return "Idle", f"No new logs in {int(age_minutes)} min"

    except Exception:
        return "Inactive", "Error while parsing trail status"

# =========================
# SEND TO OPENSEARCH
# =========================

def send_to_opensearch(doc):
    url = f"{OPENSEARCH_HOST}/{INDEX_NAME}/_doc"
    headers = {"Content-Type": "application/json"}

    r = requests.post(
        url,
        auth=(OPENSEARCH_USER, OPENSEARCH_PASS),
        headers=headers,
        json=doc,
        verify=False
    )

    print("Response:", r.text)
    return r.status_code

# =========================
# MAIN
# =========================

def main():
    trail_status = get_trail_status()
    status, reason = evaluate_status(trail_status)

    doc = {
        "@timestamp": datetime.now(timezone.utc).isoformat(),
        "source": "aws_cloudtrail_health",
        "service": TRAIL_NAME,
        "status": status,
        "reason": reason
    }

    code = send_to_opensearch(doc)

    print("Status:", status)
    print("Reason:", reason)
    print("OpenSearch response:", code)


if __name__ == "__main__":
    main()
```

**then:**

```bash
chmod +x aws_cloudtrail_health.py
python3 aws_cloudtrail_health.py
```

Expected output: `Status: Active` (or Idle/Inactive) and `OpenSearch response: 201`.

Test the index manually first, same as you did for Azure:

```bash
curl -k -u username:password -X PUT \
  "https://<indexer-ip>:9200/aws-cloudtrail-health-status"
```

---

## AUTOMATE WITH CRON (EVERY MINUTE)

Your script only runs once when you type `python3 aws_cloudtrail_health.py`. Cron is what makes it run automatically, forever, every minute, without you sitting there. The curl command is just you checking — from the terminal — "did the last cron run actually save data into Wazuh?"

Everything below runs **on the Logstash VM**, in a terminal (SSH into it).

### 1. Create the Log Folder and File

Run: `sudo mkdir -p /var/log/aws-cloudtrail-health-monitor` then `sudo touch /var/log/aws-cloudtrail-health-monitor/aws_health.log`. Cron writes output to this file, not your screen — if the folder/file don't exist first, cron silently fails to log and you'll have no way to debug it.

### 2. Find Your Exact python3 Path

Run: `which python3`. It usually prints `/usr/bin/python3` — but confirm it, because cron does not use your normal shell's PATH. If the path in your crontab doesn't match this exact output, the cron job will fail silently.

### 3. Open Your Crontab for Editing

Run: `crontab -e`. First time, it may ask you to pick an editor (nano is easiest — press 1 or 2 if prompted). This opens a blank/empty file scoped to your current user.

### 4. Add the Schedule Line

On a new line, paste:

```
* * * * * /usr/bin/python3 /opt/aws-cloudtrail-health-monitor/aws_cloudtrail_health.py >> /var/log/aws-cloudtrail-health-monitor/aws_health.log 2>&1
```

Meaning: the five stars = "every minute, every hour, every day" (cron's schedule format). The `>>` sends normal output into your log file. The `2>&1` sends errors into that same log file too — so if the script crashes, you'll see why.

### 5. Save and Exit

In nano: `Ctrl+O`, `Enter` (save), then `Ctrl+X` (exit). You'll see a message like "crontab: installing new crontab" — that confirms it's active. No need to restart any service; cron picks it up immediately.

### 6. Watch the Log to Confirm It's Actually Running

Run: `tail -f /var/log/aws-cloudtrail-health-monitor/aws_health.log` and wait up to 60 seconds. You should see a new "Status: ..." and "OpenSearch response: 201" line appear on its own — that's cron firing the script for you, unattended. `Ctrl+C` to stop watching.

---

## NOW THE curl COMMAND — WHY YOU NEED IT, IN PLAIN TERMS

Your script pushes a document into OpenSearch every minute. The curl command just asks OpenSearch, "show me the most recent document you have," so you can confirm it's actually arriving — not just that cron *ran*, but that the data *landed*.

```bash
curl -k -u username:password \
  "https://<indexer-ip>:9200/aws-cloudtrail-health-status/_search?size=1&sort=@timestamp:desc&pretty"
```

**Breaking down each part:**

- `-k` → ignore the SSL certificate warning (your indexer likely uses a self-signed cert)
- `-u username:password` → your Wazuh Indexer login (the same OPENSEARCH_USER/OPENSEARCH_PASS you put in the script)
- `<indexer-ip>:9200` → replace with your actual Wazuh Indexer IP (same one from OPENSEARCH_HOST in the script, without `https://`)
- `aws-cloudtrail-health-status` → the index name we're querying
- `_search?size=1&sort=@timestamp:desc` → "give me only 1 result, the newest one first"
- `&pretty` → format the JSON output so it's readable instead of one long line

Run this on the Logstash VM (or any machine that can reach the indexer on port 9200). If it's working, you'll see a JSON block ending in something like:

```json
"_source": {
  "status": "Active",
  "service": "your-trail-name",
  "reason": "Last log delivered 3 min ago"
}
```

If you instead see `"hits": []` (empty), it means the index exists but nothing's been written yet — check the cron log from the previous step for errors.

---

## TESTING

Right now you're only checking the *latest* doc — that doesn't prove new ones are being added, just that at least one exists. To prove growth, you need the **document count**, checked twice with a pause in between.

### 1. Get the Current Count

```bash
curl -k -u username:password \
  "https://<indexer-ip>:9200/aws-cloudtrail-health-status/_count?pretty"
```

This returns just a number — total documents in that index right now, e.g. `"count": 42`.

### 2. Wait a Minute or Two, Then Run the Exact Same Command Again

If the number went up (e.g. 42 → 44), cron is genuinely inserting new documents each run — confirmed. If it's stuck at the same number, cron isn't firing or the script is failing before it reaches OpenSearch — go back to the log file to see why.

### 3. Easier — Watch It Live, Auto-Refreshing Every 10 Seconds

(so you don't have to type the command repeatedly):

```bash
watch -n 10 'curl -sk -u username:password \
  "https://<indexer-ip>:9200/aws-cloudtrail-health-status/_count?pretty"'
```

Leave this running in a terminal for 2–3 minutes and you should see "count" tick up roughly once per minute. `Ctrl+C` to stop watching.

### 4. If You Want to See the Actual Timestamps Stacking Up, Not Just a Number

```bash
curl -k -u username:password \
  "https://<indexer-ip>:9200/aws-cloudtrail-health-status/_search?size=5&sort=@timestamp:desc&pretty" \
  | grep "@timestamp"
```

This prints the 5 most recent `@timestamp` values — you should see them roughly 60 seconds apart, matching your cron schedule.

---

## VEGA DASHBOARD

Wazuh → ☰ → Visualize → Create visualization → Vega → delete the sample, paste `aws_cloudtrail_health_vega.json`'s contents → **Update** → **Save**.

**AWS CloudTrail health Vega file:**

```json
{
  "$schema": "https://vega.github.io/schema/vega/v5.json",
  "autosize": "none",
  "width": 1000,
  "height": 300,
  "title": {
    "text": "AWS CloudTrail Health",
    "fontSize": 20,
    "anchor": "start"
  },
  "data": [
    {
      "name": "health",
      "url": {
        "%context%": true,
        "index": "aws-cloudtrail-health-status",
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


