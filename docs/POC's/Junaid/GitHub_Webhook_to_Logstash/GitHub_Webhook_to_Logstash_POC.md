# POC: GitHub Webhook to Logstash -Wazuh Indexer 

## 1. Objective

Demonstrate how GitHub repository activity can be collected through a webhook, ingested by Logstash, stored in Wazuh Indexer, monitored by OpenSearch Alerting, and written to a separate alert-history index.

This POC covers both raw GitHub event ingestion and alert generation.

    GitHub Repository
            ↓
    GitHub Webhook
            ↓
    ngrok Public HTTPS URL
            ↓
    Logstash Pipeline 1 — Port 8080
            ↓
    Wazuh Indexer — github-logs-*
            ↓
    OpenSearch Alerting Monitor
            ↓
    Logstash Pipeline 2 — Port 8081
            ↓
    Wazuh Indexer — github-alerts-history

## 2. Expected Outcome

After completing this POC:

1.  A GitHub push event is delivered to Logstash through an ngrok public URL.
2.  Logstash enriches the event and stores it in `github-logs-*`.
3.  An OpenSearch Alerting monitor detects GitHub activity.
4.  The monitor sends an alert payload to Logstash Pipeline 2.
5.  Logstash Pipeline 2 stores the alert in `github-alerts-history`.
6.  Raw GitHub events and generated alert history are visible in Wazuh Dashboard Dev Tools.

## 3. Environment Details

| Component           | Details                                         |
|---------------------|-------------------------------------------------|
| GitHub              | Repository and webhook source                   |
| Logstash Server     | `<IP-address>`                                 |
| Wazuh Indexer       | `<IP-address>`                            |
| Pipeline 1          | GitHub webhook receiver on port `8080`          |
| Pipeline 2          | Alerting webhook receiver on port `8081`        |
| Raw Event Index     | `github-logs-*`                                 |
| Alert History Index | `github-alerts-history`                         |
| Indexer User        | `admin`                                         |
| ngrok               | Public HTTPS tunnel for GitHub webhook delivery |

# Part A: Prerequisite Validation

## Step 1: Verify Logstash service

On the Logstash server, run:

    sudo systemctl status logstash

Expected result:

    Active: active (running)

## Step 2: Verify connectivity from Logstash to Wazuh Indexer

Run:

    nc -zv <IP-address>

Expected result:

    Connection to <IP-address> 9200 port [tcp/*] succeeded!

If this fails, confirm routing, firewall rules, and Wazuh Indexer availability before continuing.

# Part B: GitHub Repository Setup

## Step 3: Create a test repository in GitHub

1.  Log in to GitHub.
2.  Click the **+** icon in the top-right corner.
3.  Select **New repository**.
4.  Enter the repository name:


    github-wazuh-poc

5.  Select **Public** or **Private** according to the requirement.
6.  Select **Add a README file**.
7.  Click **Create repository**.

## Step 4: Clone the repository and create an initial commit

On a local workstation:

    git clone https://github.com/<YOUR-GITHUB-USERNAME>/github-wazuh-poc.git
    cd github-wazuh-poc

Create a test file:

    echo "GitHub Wazuh POC test" > poc-test.txt

Commit and push it:

    git add .
    git commit -m "Initial GitHub Wazuh POC commit"
    git push origin main

This confirms that the repository is working before webhook configuration.

# Part C: Install and Configure ngrok

## Step 5: Install ngrok on the Logstash server

Connect to the Logstash server:

    ssh log1@<IP-address>

Install ngrok:

    sudo snap install ngrok

Verify the installation:

    ngrok version

Expected example:

    ngrok version 3.x.x

## Step 6: Configure ngrok authentication

1.  Create or log in to an ngrok account.
2.  Open the ngrok dashboard.
3.  Copy the account authtoken.
4.  Configure it on the Logstash server:


    ngrok config add-authtoken <YOUR_NGROK_AUTHTOKEN>

Validate the configuration:

    ngrok config check

Expected result:

    Valid configuration file

> Never expose the ngrok authtoken. If it is accidentally exposed, revoke it immediately and generate a new token.

# Part D: Configure Logstash Pipeline 1

Step 7A: Confirm the Logstash HTTP input plugin

The GitHub webhook pipeline and the OpenSearch Alerting webhook pipeline both use the Logstash HTTP input plugin.

Check whether it is installed:

    sudo /usr/share/logstash/bin/logstash-plugin list --verbose | grep logstash-input-http

Expected output:

    logstash-input-http

If it is not installed, install it:

    sudo /usr/share/logstash/bin/logstash-plugin install logstash-input-http

After installation, restart Logstash:

    sudo systemctl restart logstash

Verify it again:

    sudo /usr/share/logstash/bin/logstash-plugin list --verbose | grep logstash-input-http

## Step 7: Confirm the OpenSearch Logstash output plugin

Run:

    sudo /usr/share/logstash/bin/logstash-plugin list --verbose | grep logstash-output-opensearch

If the plugin is not listed, install it:

    sudo /usr/share/logstash/bin/logstash-plugin install logstash-output-opensearch
    sudo systemctl restart logstash

## Step 8: Create TLS certificate files for Logstash

Create the certificate directory:

    sudo mkdir -p /etc/logstash/certs

Create a private key:

    sudo openssl genrsa -out /etc/logstash/certs/logstash.key 2048

Create a certificate signing request:

    sudo openssl req -new \
    -key /etc/logstash/certs/logstash.key \
    -out /tmp/logstash.csr \
    -subj "/C=IN/ST=TG/L=HYD/O=VIDYAYUG/OU=SOC/CN=logstash"

If your Wazuh Root CA certificate and key are securely available on the Logstash server, sign the certificate:

    sudo openssl x509 -req \
    -in /tmp/logstash.csr \
    -CA /tmp/root-ca.pem \
    -CAkey /tmp/root-ca.key \
    -CAcreateserial \
    -out /etc/logstash/certs/logstash.crt \
    -days 3650 \
    -sha256

Set permissions:

    sudo chown -R logstash:logstash /etc/logstash/certs
    sudo chmod 600 /etc/logstash/certs/logstash.key
    sudo chmod 644 /etc/logstash/certs/logstash.crt

Verify the certificate:

    sudo openssl x509 -in /etc/logstash/certs/logstash.crt -noout -subject -issuer -dates

> For production, use a certificate with a valid hostname and trusted certificate chain. For this POC, ngrok provides the public HTTPS certificate presented to GitHub.

## Step 9: Create the GitHub ingestion pipeline configuration

Create the directory:

    sudo mkdir -p /etc/logstash/conf.d/github-junaid

Create the configuration file:

    sudo nano /etc/logstash/conf.d/github-junaid/github.conf

Paste the following configuration:


    input {
      http {
        host => "0.0.0.0"
        port => 8080

        ssl => true
        ssl_certificate => "/etc/logstash/certs/logstash.crt"
        ssl_key => "/etc/logstash/certs/logstash.key"

        codec => json
      }
    }

    filter {

      mutate {
        add_field => {
          "github_event_type" => "%{[headers][x-github-event]}"
        }
      }

      mutate {
        add_field => {
          "source_platform" => "github"
          "event_category"  => "repository_activity"
          "environment"     => "poc"
          "tenant"          => "NON-AGENT-TENANT"
        }
      }

      mutate {
        replace => {
          "message" => "GitHub event=%{[headers][x-github-event]} action=%{[action]} repo=%{[repository][full_name]} user=%{[sender][login]} branch=%{[ref]} tenant=NON-AGENT-TENANT"
        }
      }

    }

    output {

      stdout {
        codec => rubydebug
      }

      file {
        path => "/tmp/github-events.log"
        codec => rubydebug
      }

      opensearch {
        hosts => ["https://<IP-address>:9200"]
        user => "admin"
        password => "SecretPassword"
        index => "github-logs-%{+YYYY.MM.dd}"

        ssl => true
        ssl_certificate_verification => false
      }

    }

Save the file:

    Ctrl + O → Enter → Ctrl + X

## Step 10: Register Pipeline 1

Edit the Logstash pipelines file:

    sudo nano /etc/logstash/pipelines.yml

Add:

    - pipeline.id: github-ingestion
      path.config: "/etc/logstash/conf.d/github-junaid/github.conf"

Save the file.

## Step 11: Validate and start Pipeline 1

Validate the Logstash configuration:

    sudo /usr/share/logstash/bin/logstash --path.settings /etc/logstash -t

Expected result:

    Configuration OK

Restart Logstash:

    sudo systemctl restart logstash

Confirm that port `8080` is listening:

    sudo ss -tlnp | grep 8080

Expected result:

    LISTEN ... *:8080 ... java

Check Logstash logs:

    sudo tail -f /var/log/logstash/logstash-plain.log

Expected messages:

    Pipeline started {"pipeline.id"=>"github-ingestion"}
    Starting http input listener {:address=>"0.0.0.0:8080", :ssl_enabled=>true}

## Step 12: Test Pipeline 1 locally

Run from the Logstash server:

    curl -k -X POST https://localhost:8080 \
    -H "Content-Type: application/json" \
    -d '{"test":"local-logstash-test"}' -v

Expected result:

    HTTP/1.1 200 OK
    ok

Verify that Logstash received the event:

    sudo tail -f /tmp/github-events.log

# Part E: Start the ngrok Tunnel

## Step 13: Start ngrok for the HTTPS Logstash listener

Because Logstash is listening with HTTPS on port `8080`, start ngrok with:

    ngrok http https://localhost:8080

Do not use:

    ngrok http 8080

That forwards plain HTTP to an HTTPS Logstash listener and can cause:

    ERR_NGROK_3004
    The server returned an invalid or incomplete HTTP response

When ngrok starts, copy the public forwarding URL.

Expected format:

    Forwarding https://<generated-name>.ngrok-free.dev -> https://localhost:8080

Keep this terminal running throughout the POC.

## Step 14: Test the ngrok public URL

From another terminal, run:

    curl -k -X POST https://<ACTIVE_NGROK_URL> \
    -H "Content-Type: application/json" \
    -d '{"test":"ngrok-test"}' -v

Expected result:

    HTTP/2 200
    ok

If it fails, verify:

    pgrep -af ngrok
    sudo ss -tlnp | grep 8080
    sudo tail -f /var/log/logstash/logstash-plain.log

# Part F: Configure the GitHub Webhook

## Step 15: Add the webhook in GitHub

In the GitHub repository, navigate to:

    Repository → Settings → Webhooks → Add webhook

Configure the webhook:

| GitHub Field     | Value                                           |
|------------------|-------------------------------------------------|
| Payload URL      | `https://<ACTIVE_NGROK_URL>`                    |
| Content type     | `application/json`                              |
| Secret           | Leave blank for this POC, or configure a secret |
| SSL verification | Enable                                          |
| Events           | Select **Just the push event**                  |
| Active           | Enable                                          |

Click **Add webhook**.

GitHub sends a `ping` event immediately after creating the webhook.

## Step 16: Verify GitHub webhook delivery

Navigate to:

    Repository → Settings → Webhooks → Select webhook → Recent Deliveries

A successful delivery must show:

    Response: 200

If GitHub returns an error, verify:

    sudo ss -tlnp | grep 8080
    pgrep -af ngrok
    sudo tail -f /var/log/logstash/logstash-plain.log

## Step 17: Generate a GitHub push event

On the local machine where the repository was cloned:

    echo "GitHub webhook event $(date)" >> poc-test.txt
    git add poc-test.txt
    git commit -m "Generate GitHub webhook event"
    git push origin main

## Step 18: Verify GitHub event ingestion in Logstash

On the Logstash server:

    sudo tail -f /tmp/github-events.log

Expected fields include:

    github_event_type
    repository
    sender
    tenant
    source_platform
    message

# Part G: Verify GitHub Logs in Wazuh Indexer

## Step 19: Check GitHub index creation

In Wazuh Dashboard, navigate to:

    Dev Tools

Run:

    GET _cat/indices/github-*?v

Expected result:

    green open github-logs-YYYY.MM.DD

Check document count:

    GET github-logs-*/_count

View recent GitHub events:

    GET github-logs-*/_search
    {
      "size": 10,
      "sort": [
        {
          "@timestamp": {
            "order": "desc"
          }
        }
      ]
    }

At this stage, the raw ingestion flow is complete:

    GitHub → ngrok → Logstash Pipeline 1 → github-logs-*

# Part H: Create the Alert History Index

## Step 20: Create the alert-history index

In Wazuh Dashboard Dev Tools, run:

    PUT github-alerts-history
    {
      "settings": {
        "number_of_shards": 1,
        "number_of_replicas": 0
      }
    }

Verify index creation:

    GET _cat/indices/github-alerts-history?v

# Part I: Configure Logstash Pipeline 2

## Step 21: Create the alert webhook receiver configuration

On the Logstash server, create the directory:

    sudo mkdir -p /etc/logstash/conf.d/github-alerts

Create the configuration file:

    sudo nano /etc/logstash/conf.d/github-alerts/github-alerts.conf

Paste:

    input {
      http {
        host => "0.0.0.0"
        port => 8081
        codec => json
      }
    }

    filter {

      mutate {
        add_field => {
          "alert_source" => "opensearch-alerting"
        }
      }

    }

    output {

      stdout {
        codec => rubydebug
      }

      file {
        path => "/tmp/github-alerts.log"
        codec => rubydebug
      }

      opensearch {
        hosts => ["https://<IP-address>:9200"]
        user => "admin"
        password => "SecretPassword"

        index => "github-alerts-history"

        ssl => true
        ssl_certificate_verification => false
      }

    }

Save the file.

## Step 22: Register Pipeline 2

Edit the Logstash pipelines file:

    sudo nano /etc/logstash/pipelines.yml

Ensure both pipelines are present:

    - pipeline.id: github-ingestion
      path.config: "/etc/logstash/conf.d/github-junaid/github.conf"

    - pipeline.id: github-alerts
      path.config: "/etc/logstash/conf.d/github-alerts/github-alerts.conf"

Validate the configuration:

    sudo /usr/share/logstash/bin/logstash --path.settings /etc/logstash -t

Restart Logstash:

    sudo systemctl restart logstash

Verify both listening ports:

    sudo ss -tlnp | egrep '8080|8081'

Expected result:

    *:8080 LISTEN
    *:8081 LISTEN

## Step 23: Test Pipeline 2 manually

Run from the Logstash server:

    curl -X POST http://localhost:8081 \
    -H "Content-Type: application/json" \
    -d '{
      "monitor":"github-test",
      "trigger":"force-push",
      "severity":"1",
      "hits":"1"
    }'

Expected result:

    ok

In Wazuh Dashboard Dev Tools, verify:

    GET github-alerts-history/_search
    {
      "sort": [
        {
          "@timestamp": {
            "order": "desc"
          }
        }
      ]
    }

# Part J: Configure OpenSearch Alerting

## Step 24: Create the webhook notification channel

In Wazuh Dashboard Dev Tools,Under the Indexer Management run:

    POST _plugins/_notifications/configs
    {
      "config": {
        "name": "github-webhook",
        "description": "Send GitHub alerts to Logstash Pipeline 2",
        "config_type": "webhook",
        "is_enabled": true,
        "webhook": {
          "url": "http://<IP-address>:8081",
          "header_params": {
            "Content-Type": "application/json"
          },
          "method": "POST"
        }
      }
    }

Verify the channel:

    GET _plugins/_notifications/configs

> Ensure the Wazuh Indexer host can reach `<IP-address>:8081`. For production, protect this endpoint with TLS, authentication, and network restrictions.

## Step 25: Create the GitHub Activity Monitor in the Dashboard

Navigate to:

    Wazuh Dashboard → Alerting → Monitors → Create monitor

Configure the monitor:

| Field           | Value                     |
|-----------------|---------------------------|
| Monitor name    | `GitHub Activity Monitor` |
| Monitor type    | Per query monitor         |
| Defining method | Visual editor             |
| Schedule        | Every 1 minute            |
| Cluster         | `wazuh-cluster (Local)`   |
| Index           | `github-logs-*`           |
| Time field      | `@timestamp`              |
| Query           | Match all                 |

## Step 26: Create the monitor trigger

Click **Add trigger** and configure:

| Field             | Value                     |
|-------------------|---------------------------|
| Trigger name      | `GitHub Activity Trigger` |
| Severity          | `1 (Highest)`             |
| Trigger condition | `IS ABOVE`                |
| Value             | `0`                       |
| Metric            | `COUNT of documents`      |

This trigger creates an alert when at least one GitHub event exists during the monitor evaluation period.

## Step 27: Add the alert action

Under **Actions**, add a notification action.

| Field       | Value                           |
|-------------|---------------------------------|
| Action name | `Send GitHub Alert to Logstash` |
| Channel     | `github-webhook`                |

Use this message body:

    {
      "monitor": "{{ctx.monitor.name}}",
      "trigger": "{{ctx.trigger.name}}",
      "severity": "{{ctx.trigger.severity}}",
      "hits": "{{ctx.results.0.hits.total.value}}",
      "time": "{{ctx.periodEnd}}"
    }

Click **Create**.

# Part K: Final End-to-End Demonstration

## Step 28: Push a final GitHub commit

On the local repository machine:

    echo "Final  demo $(date)" >> poc-test.txt
    git add .
    git commit -m "Final GitHub alert demonstration"
    git push origin main

Wait approximately one minute for the alert monitor to evaluate the new event.

## Step 29: Verify each stage

### Verify raw GitHub event ingestion

On the Logstash server:

    sudo tail -f /tmp/github-events.log

### Verify alert webhook reception

On the Logstash server:

    sudo tail -f /tmp/github-alerts.log

### Verify raw GitHub events in Wazuh Indexer

In Wazuh Dashboard Dev Tools:

    GET github-logs-*/_search
    {
      "size": 10,
      "sort": [
        {
          "@timestamp": {
            "order": "desc"
          }
        }
      ]
    }

### Verify alert history in Wazuh Indexer

    GET github-alerts-history/_search
    {
      "size": 10,
      "sort": [
        {
          "@timestamp": {
            "order": "desc"
          }
        }
      ]
    }

Expected alert-history document fields:

    {
      "monitor": "GitHub Activity Monitor",
      "trigger": "GitHub Activity Trigger",
      "severity": "1",
      "hits": "1",
      "alert_source": "opensearch-alerting"
    }

# POC Success Criteria

| Validation     | Expected Result                                                      |
|----------------|----------------------------------------------------------------------|
| GitHub webhook | GitHub Recent Deliveries returns HTTP `200`                          |
| Pipeline 1     | GitHub event appears in `/tmp/github-events.log`                     |
| Raw index      | Event is stored in `github-logs-*`                                   |
| Monitor        | GitHub Activity Monitor triggers on new activity                     |
| Pipeline 2     | Alert payload appears in `/tmp/github-alerts.log`                    |
| Alert index    | Alert is stored in `github-alerts-history`                           |
| Final proof    | Raw event and alert history are visible in Wazuh Dashboard Dev Tools |


