# Non-Wazuh SOC Client Onboarding & Pipeline Implementation Guide

*GitHub → Nginx → Logstash → OpenSearch (ityug-raw-logs)*

Consolidated reference covering client onboarding, secure webhook ingestion, and index lifecycle management.

Prepared for internal SOC / MSSP operations use
Version 1.0

---

## Table of Contents

1. [Executive Summary](#1-executive-summary)
2. [Scope and Audience](#2-scope-and-audience)
3. [Part A — SOC Client Onboarding Plan](#part-a--soc-client-onboarding-plan)
4. [Part B — Secure Webhook Ingestion: Nginx Reverse Proxy (Replacing ngrok)](#part-b--secure-webhook-ingestion-nginx-reverse-proxy-replacing-ngrok)
5. [Part C — OpenSearch Index Lifecycle: ISM Rollover](#part-c--opensearch-index-lifecycle-ism-rollover)
6. [Part D — Consolidated End-to-End Architecture](#part-d--consolidated-end-to-end-architecture)
7. [Appendix](#appendix)

---

## 1. Executive Summary

This document consolidates three working artifacts produced during the setup of a non-Wazuh SOC client pipeline into a single, structured reference: (1) the client onboarding plan, (2) the Nginx reverse-proxy implementation that replaced a temporary ngrok tunnel for GitHub webhook delivery, and (3) the OpenSearch Index State Management (ISM) rollover configuration that keeps the resulting index (`ityug-raw-logs`) healthy over time.

Together these three pieces describe one continuous data path: a client's source events (GitHub, GitLab, network telemetry, Kubernetes, etc.) are collected, forwarded securely over the public internet, parsed by Logstash, written into a rolling OpenSearch index, and evaluated by alert monitors — all wrapped in a repeatable onboarding process so the same pattern can be applied to future non-Wazuh clients.

The goal of this guide is to give any engineer picking up the project a single source of truth: what information must be collected from a client before work starts, how the ingress layer is built and secured, and how the storage layer is configured to roll over and retain data without manual intervention.

## 2. Scope and Audience

This guide is intended for SOC engineers, MSSP platform administrators, and DevOps/backend engineers involved in onboarding a new client that does not use a native Wazuh agent. It assumes familiarity with Linux system administration, basic networking (DNS, TLS, reverse proxies), and the Elastic/OpenSearch ecosystem (Logstash, indices, aliases, ISM).

- **Part A** defines the business and technical information that must be gathered from the client and the steps to provision their tenant.
- **Part B** is a step-by-step implementation of the secure webhook ingress (Nginx reverse proxy + Let's Encrypt TLS), replacing the ngrok tunnel used during initial testing.
- **Part C** is a step-by-step implementation of automatic index rollover using OpenSearch ISM, so a single alias always points at the current write index.
- **Part D** ties the three parts together into one end-to-end architecture diagram.
- The **Appendix** provides a glossary, a security checklist, and a consolidated command reference.

---

## Part A — SOC Client Onboarding Plan

### A.1 Purpose

Define a standardized onboarding process for clients that will be monitored without a native Wazuh agent, including how the appropriate ingestion pipeline is chosen for each client based on their environment and monitoring requirements.

> **Note:** The pilot rollout referenced in Parts B and C uses a GitHub test repository as the source. The original planning notes referenced GitLab as the "current selection" — confirm with the client which source system(s) are in scope before repeating this process for a new tenant, since the pipeline choice (Part B/C) depends on it.

### A.2 Pre-Requisites Checklist

Collect the following information from the client before provisioning begins. Items marked **TBD** are still being standardized internally and should be confirmed with the CISO / platform owner at the time of onboarding.

#### A.2.1 Environment and Data Profile

- Number of endpoints to be monitored
- Type of endpoints — e.g. GitLab, GitHub, network telemetry (NetFlow, Suricata, Zeek, etc.), Kubernetes, and other log sources
- Expected data volume (e.g. 30 GB/day or more)
- Networking requirements for log forwarding from the endpoint to the Logstash server — confirm required ports and protocols are reachable
- Subscription / licensing limits (**TBD**)

#### A.2.2 Platform and Index Configuration

- Wazuh Indexer indices to be created
- Index rollover strategy (see Part C)
- ISM policy — thresholds to be agreed with the customer
- RBAC roles — default or custom
- Dashboards required
- Wazuh / Logstash routing preference
- Monitor logic, triggers, and actions — defined per client requirement
- Retention policy — standard (CISO-defined) or custom per client
- Tenant name
- Agent ID — default sequence unless the client requires otherwise
- Region
- Compliance requirements (e.g. data residency, industry regulations)
- Backup strategy

#### A.2.3 Platform Access and Security

- Dedicated SOC platform vs. shared/multi-tenant platform
- If shared: communicate platform guidelines and limitations, including the isolation scope between tenants
- Confirm whether SSO is required for platform access
- If SSO is requested, provide RBAC role mapping details
- Access privileges for each RBAC role
- Collect user/member details required to provision platform access
- Per-user role assignment

#### A.2.4 Integrations (TBD)

- SIEM integration
- SOAR integration
- Ticketing system integration

#### A.2.5 Commercial and Operational

- Point of contact / contact details
- Agreement start date and total agreement period / tenure
- Alert reporting style (webhook)
- Report selection — on-demand or frequency-based
- Onboarding confirmation and access handover
- Maintenance and support model (**TBD**)

### A.3 Onboarding Workflow

Once the pre-requisites above have been collected, onboarding proceeds in two stages: tenant registration, followed by platform (MSSP) setup.

#### A.3.1 Stage 1 — Tenant Registration / Organization

1. Application: register the tenant with all details collected during the pre-requisites phase.
2. Enroll the new tenant using the information gathered from the client.
3. Prepare a backend engineering checklist, or raise a ticket, capturing the tenant's configuration.
4. Send the setup request / ticket to the backend engineering team.

#### A.3.2 Stage 2 — Begin Platform (MSSP) Setup

1. Template creation.
2. Tenant-specific index creation.
3. Update Filebeat/Logstash configuration to accommodate alert routing for the new tenant.
4. Deploy the endpoint/agent, or configure agentless collection where no agent is used.
5. Verify connectivity from the source to the ingestion layer.
6. Confirm agent/source health.
7. Configure RBAC.
8. Create index patterns.
9. Run platform validation checks before handover.

#### A.3.3 Reference Pipeline — GitHub Test Repository

The current pilot pipeline collects regular/default Git repository events (pushes, webhook deliveries) and converts them into alerts. It is used as the reference implementation for Parts B and C of this guide, and as the template for onboarding future clients with a similar source profile.

### A.4 Onboarding Sign-off Checklist

Before handing the tenant over to steady-state operations, confirm the following:

- [ ] All pre-requisite information in A.2 has been collected and recorded against the tenant record
- [ ] Ingestion pipeline (Part B) is deployed, TLS-secured, and verified end to end
- [ ] Index and rollover policy (Part C) are created and verified
- [ ] RBAC roles and (if applicable) SSO are configured and tested with a sample user
- [ ] Dashboards and index patterns are visible to the client's assigned users
- [ ] Retention and backup policy is documented and agreed with the client
- [ ] Point of contact, escalation path, and support/maintenance model are documented
- [ ] Client has received access credentials and onboarding confirmation

---

## Part B — Secure Webhook Ingestion: Nginx Reverse Proxy (Replacing ngrok)

### B.1 Objective

During initial testing, GitHub webhook events reached Logstash through an ngrok tunnel. ngrok is convenient for prototyping but is not suitable for production: the public URL is not permanent, traffic passes through a third-party relay, and there is no control over TLS certificate management. This section replaces that tunnel with a permanent, self-hosted Nginx reverse proxy fronted by a real domain name and a Let's Encrypt TLS certificate.

**Before: ngrok tunnel**

```
GitHub
  |  HTTPS
  v
ngrok
  |  Tunnel
  v
Logstash (:8080)
```

**After: Nginx reverse proxy**

```
GitHub
  |  HTTPS
  v
nonwazuh.vidyayug.com   (DNS)
  v
Nginx (:443)   -- Reverse Proxy -->  Logstash HTTPS (:8080)
  v
Logstash Filters
  v
OpenSearch (ityug-raw-logs)
```

### B.2 Prerequisites

| Component | Value |
|---|---|
| VM IP | 192.168.35.37 |
| Domain | nonwazuh.vidyayug.com |
| Logstash | Installed |
| Nginx | To be installed |
| Certbot | To be installed |
| Public port | 443 |
| Logstash port | 8080 |

### B.3 Implementation Steps

**Step 1 — Verify Logstash is working**

Confirm the Logstash service is active:

```bash
sudo systemctl status logstash
```

Verify Logstash is listening on the expected port:

```bash
sudo ss -tlnp | grep 8080

# Expected:
LISTEN  *:8080
```

Because the Logstash HTTP input is configured with `ssl => true`, test it over HTTPS directly:

```bash
curl -vk https://127.0.0.1:8080

# Expected:
HTTP/1.1 200 OK
ok
```

> **Note:** If this direct test does not return 200 OK, stop here and fix Logstash before continuing — the reverse proxy cannot succeed if the origin server is not responding correctly.

**Step 2 — Install Nginx**

```bash
sudo apt update
sudo apt install nginx -y
sudo systemctl enable nginx
sudo systemctl start nginx
sudo systemctl status nginx
```

**Step 3 — Allow Firewall Ports**

```bash
sudo ufw allow 80/tcp
sudo ufw allow 443/tcp
sudo ufw status
```

**Step 4 — Create the Nginx Reverse Proxy Configuration**

```bash
sudo nano /etc/nginx/sites-available/nonwazuh
```

Paste the following server block:

```nginx
server {
    listen 80;
    server_name nonwazuh.vidyayug.com;

    location / {
        proxy_pass https://127.0.0.1:8080;
        proxy_ssl_verify off;
        proxy_http_version 1.1;
        proxy_set_header Host $host;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
```

**Step 5 — Enable the Site**

```bash
sudo ln -s /etc/nginx/sites-available/nonwazuh /etc/nginx/sites-enabled/
ls -l /etc/nginx/sites-enabled/
```

**Step 6 — Validate and Reload Nginx**

```bash
sudo nginx -t
# Expected: syntax is ok / test is successful

sudo systemctl reload nginx
```

**Step 7 — Verify the Reverse Proxy Locally**

```bash
curl -H "Host: nonwazuh.vidyayug.com" http://127.0.0.1
# Expected: ok
```

> **Note:** If you receive `502 Bad Gateway`, check the Nginx error log with `sudo tail -f /var/log/nginx/error.log`. In this deployment the cause was a protocol mismatch: Logstash expects HTTPS, so `proxy_pass` had to be changed from `http://127.0.0.1:8080` to `https://127.0.0.1:8080` with `proxy_ssl_verify off;` added, which resolved the issue.

**Step 8 — Install Let's Encrypt (Certbot)**

```bash
sudo apt install certbot python3-certbot-nginx -y
sudo certbot --nginx
# When prompted, choose: nonwazuh.vidyayug.com
```

Certbot automatically issues the certificate, updates the Nginx configuration to serve it, enables HTTPS, and configures automatic renewal.

**Step 9 — Verify HTTPS**

```bash
sudo ss -tlnp | grep 443
# Expected: LISTEN 443

curl -vk https://127.0.0.1
# Expected: HTTP/1.1 200 OK / ok
# Certificate CN=nonwazuh.vidyayug.com, Issuer=Let's Encrypt
```

**Step 10 — Configure DNS**

Create an A record pointing the subdomain at the VM's public IP:

```
Host:   nonwazuh
Value:  <public IP of VM>
# Example: nonwazuh.vidyayug.com  ->  38.xxx.xxx.xxx
```

Verify propagation:

```bash
nslookup nonwazuh.vidyayug.com
```

**Step 11 — Verify Public Access**

From a separate machine (not the VM itself):

```bash
curl -vk https://nonwazuh.vidyayug.com
# Expected: HTTP/1.1 200 OK / ok
```

**Step 12 — Update the GitHub Webhook**

In the GitHub repository: **Settings → Webhooks → Edit**, replace the old ngrok URL with the new domain:

```
Old:  https://xxxx.ngrok-free.app
New:  https://nonwazuh.vidyayug.com
```

**Step 13 — Test the Webhook**

From the GitHub webhook's **Recent Deliveries** tab, click **Redeliver**, or push a test commit to the repository.

**Step 14 — End-to-End Verification**

Monitor Nginx access logs:

```bash
sudo tail -f /var/log/nginx/access.log
```

Monitor Logstash logs:

```bash
sudo journalctl -u logstash -f
```

Confirm the event was indexed in OpenSearch (`ityug-raw-logs`), and, if the monitor condition was satisfied, confirm the corresponding alert landed in `ityug-alerts`.

### B.4 Final Architecture

```
Internet
  |  HTTPS POST
  v
nonwazuh.vidyayug.com  --DNS-->  Public IP of VM
  |  TCP 443
  v
+-----------------------+
|  Nginx Reverse Proxy  |
+-----------------------+
  |  proxy_pass https://127.0.0.1:8080
  v
+-----------------------------+
|  Logstash HTTP Input :8080  |
|  (SSL enabled)              |
+-----------------------------+
  |  JSON decoding + Logstash filters
  v
OpenSearch index: ityug-raw-logs
  v
Alert Monitor (if matched)
  v
ityug-alerts
```

This is a clean, reusable implementation. The same steps can be applied on any new VM by changing only the domain name and the VM's public IP address, making it repeatable for future non-Wazuh clients onboarded per Part A.

---

## Part C — OpenSearch Index Lifecycle: ISM Rollover

### C.1 Purpose

A single, ever-growing index is hard to manage: it gets slow to search, hard to delete piecemeal by age, and risky to reindex. This section configures OpenSearch's Index State Management (ISM) plugin to automatically "roll over" the `ityug-raw-logs` index into a new physical index once it reaches a size threshold, while Logstash and downstream monitors keep writing to and reading from a single stable alias — with zero configuration changes required at rollover time.

### C.2 Architecture

```
GitHub / NetFlow
  v
Logstash   (index => ityug-raw-logs)
  v
Write Alias: ityug-raw-logs
  v
+-------------------------+
| ityug-raw-logs-000001   |   <- current write index
+-------------------------+
  |  reaches 5 MB primary shard size
  v
Automatic Rollover (ISM)
  v
+-------------------------+
| ityug-raw-logs-000002   |   <- new write index
+-------------------------+
  v
OpenSearch Monitor  (reads via the alias, unaffected by rollover)
```

### C.3 Implementation Steps

**Step 1 — Create the ISM Policy**

```bash
curl -k -u admin:SecretPassword \
  -X PUT "https://192.168.35.41:9200/_plugins/_ism/policies/ityug-raw-rollover-policy" \
  -H "Content-Type: application/json" \
  -d '{
  "policy": {
    "description": "Rollover policy for ityug raw logs",
    "default_state": "hot",
    "states": [
      {
        "name": "hot",
        "actions": [
          {
            "rollover": { "min_size": "5mb", "copy_alias": false },
            "retry": { "count": 3, "backoff": "exponential", "delay": "1m" }
          }
        ]
      }
    ],
    "ism_template": [
      { "index_patterns": ["ityug-raw-logs-*"], "priority": 100 }
    ]
  }
}'
```

> **Note:** 5 MB is a deliberately small threshold used for testing rollover behavior quickly. For production, size this to the client's actual data volume (see A.2.1) — typical production thresholds are in the tens of gigabytes, often combined with a `max_age` condition (e.g. 1 day) so indices roll over on a predictable schedule even during low-traffic periods.

Verify the policy was created:

```bash
curl -k -u admin:SecretPassword \
  https://192.168.35.41:9200/_plugins/_ism/policies/ityug-raw-rollover-policy?pretty
```

**Step 2 — Create the Initial Index**

The first index in a rollover series must end in `-000001`:

```bash
curl -k -u admin:SecretPassword \
  -X PUT https://192.168.35.41:9200/ityug-raw-logs-000001 \
  -H "Content-Type: application/json" \
  -d '{
  "settings": {
    "plugins.index_state_management.rollover_alias": "ityug-raw-logs"
  },
  "aliases": {
    "ityug-raw-logs": { "is_write_index": true }
  }
}'
```

Verify the alias:

```bash
curl -k -u admin:SecretPassword https://192.168.35.41:9200/_cat/aliases?v
```

Expected: the `ityug-raw-logs` alias points to `ityug-raw-logs-000001` as the write index.

**Step 3 — Verify Index Settings**

```bash
curl -k -u admin:SecretPassword https://192.168.35.41:9200/ityug-raw-logs-000001/_settings?pretty
```

Confirm `rollover_alias` is set to `ityug-raw-logs`.

**Step 4 — Configure Logstash to Write to the Alias, Not a Dated Index**

Do not write to a date-suffixed index pattern such as:

```
index => "ityug-raw-logs-%{+YYYY.MM.dd}"
```

Instead, write to the rollover alias itself:

```ruby
opensearch {
  hosts => ["https://192.168.35.41:9200"]
  user => "admin"
  password => "SecretPassword"
  ssl => true
  ssl_certificate_verification => false
  index => "ityug-raw-logs"
}
```

Restart Logstash to apply the change:

```bash
sudo systemctl restart logstash
```

**Step 5 — Generate Test Logs**

Push a GitHub event (or generate NetFlow traffic), then verify it landed in the current write index:

```bash
curl -k -u admin:SecretPassword https://192.168.35.41:9200/_cat/indices/ityug-raw-logs*?v
# Initially: ityug-raw-logs-000001
```

**Step 6 — Verify ISM Is Tracking the Index**

```bash
curl -k -u admin:SecretPassword https://192.168.35.41:9200/_plugins/_ism/explain/ityug-raw-logs-000001?pretty
# Initially: state=hot, action=rollover, condition=pending
```

**Step 7 — Check Current Index Size**

```bash
curl -k -u admin:SecretPassword https://192.168.35.41:9200/ityug-raw-logs-000001/_stats/store?pretty
# Look for: size_in_bytes
```

**Step 8 — Wait for the Threshold**

Once the primary shard reaches the configured threshold (5 MB in this test policy), ISM evaluates the rollover condition on its next scheduled check.

**Step 9 — Automatic Rollover**

`ityug-raw-logs-000001` automatically becomes read-only, OpenSearch creates `ityug-raw-logs-000002`, and the write alias moves from `000001` to `000002` — no manual action required.

**Step 10 — Verify the Rollover**

```bash
curl -k -u admin:SecretPassword https://192.168.35.41:9200/_cat/indices/ityug-raw-logs*?v
# Now shows both: ityug-raw-logs-000001 and ityug-raw-logs-000002

curl -k -u admin:SecretPassword https://192.168.35.41:9200/_cat/aliases?v
# Alias ityug-raw-logs now points to 000002 (write index)
```

**Step 11 — Verify Logstash Is Writing to the New Index**

Generate another GitHub webhook event, then re-check the indices:

```bash
curl -k -u admin:SecretPassword https://192.168.35.41:9200/_cat/indices/ityug-raw-logs*?v
```

Expected: the document count for `000001` stays unchanged while the count for `000002` increases — confirming GitHub → Logstash → alias → 000002 is working end to end.

**Step 12 — Confirm the Monitor Requires No Changes**

Because the alert monitor is configured against the alias (`ityug-raw-logs`) rather than a specific physical index (`ityug-raw-logs-000001`), it automatically reads the current write index through the alias and does not need to be modified after a rollover.

**Step 13 — Manual Rollover (for Testing)**

To force a rollover on demand, e.g. to validate the policy without waiting for the size threshold:

```bash
curl -k -u admin:SecretPassword -X POST https://192.168.35.41:9200/ityug-raw-logs/_rollover?pretty
```

### C.4 Quick Reference Commands

| Purpose | Command |
|---|---|
| List indices | `curl -k -u admin:*** https://<host>:9200/_cat/indices/ityug-raw-logs*?v` |
| List aliases | `curl -k -u admin:*** https://<host>:9200/_cat/aliases?v` |
| Explain ISM state | `curl -k -u admin:*** https://<host>:9200/_plugins/_ism/explain/ityug-raw-logs-000001?pretty` |
| Check store size | `curl -k -u admin:*** https://<host>:9200/ityug-raw-logs-000001/_stats/store?pretty` |
| Force rollover | `curl -k -u admin:*** -X POST https://<host>:9200/ityug-raw-logs/_rollover?pretty` |

---

## Part D — Consolidated End-to-End Architecture

Parts B and C describe two halves of the same pipeline. Combined, a single event travels through the following path from the client's source system to a searchable, self-managing index:

```
Client source: GitHub / GitLab / NetFlow / Suricata / Zeek / Kubernetes
  |  HTTPS webhook / telemetry
  v
DNS:  <client-subdomain>.vidyayug.com   (e.g. nonwazuh.vidyayug.com)
  v
Nginx reverse proxy  (:443, TLS via Let's Encrypt)   [Part B]
  v
Logstash HTTP input  (:8080, SSL)  ->  filters / parsing
  v
Write alias: <tenant>-raw-logs   ->  current -NNNNNN index          [Part C]
  v
ISM policy: rollover on size / age threshold, no downtime, no client-side change
  v
OpenSearch Monitor  (reads via alias)
  v
<tenant>-alerts  ->  Reporting / Ticketing / SOAR integration       [Part A]
```

Because both the ingress layer (Part B) and the storage layer (Part C) are built around a stable alias/domain rather than a hardcoded physical index or a temporary tunnel URL, this entire pattern is reusable: onboarding a new client (Part A) is largely a matter of repeating Part B with a new subdomain/VM and Part C with a new tenant-prefixed index name.

---

## Appendix

### Appendix 1 — Glossary

| Term | Definition |
|---|---|
| **ISM** | Index State Management — an OpenSearch plugin that automates actions on indices (e.g. rollover, delete) based on age, size, or document count. |
| **Rollover alias** | An alias that always points to the current "write" index in a series (e.g. `-000001`, `-000002` …), letting producers and consumers use a stable name. |
| **Write index** | The single physical index in a rollover series currently accepting new documents. |
| **MSSP** | Managed Security Service Provider — the multi-tenant SOC platform onboarding each client as a tenant. |
| **RBAC** | Role-Based Access Control — restricting platform/dashboard access based on a user's assigned role. |
| **SSO** | Single Sign-On — allowing a client's users to authenticate using their own organization's identity provider. |
| **NetFlow** | A network protocol for collecting IP traffic metadata, commonly used for network telemetry ingestion. |
| **Suricata / Zeek** | Open-source network security monitoring / intrusion detection engines, used as telemetry sources. |
| **SIEM / SOAR** | Security Information and Event Management / Security Orchestration, Automation and Response — downstream systems that may consume alerts from this pipeline. |

### Appendix 2 — Security Checklist

The commands in Parts B and C use inline credentials and `-k` (disable TLS verification) for convenience during testing. Before treating a client pipeline as production-ready, confirm:

- [ ] OpenSearch admin credentials are moved out of shell history / scripts and into a secrets manager or environment-scoped credentials file
- [ ] `curl -k` / `proxy_ssl_verify off` are replaced with proper certificate validation once internal CA or valid certificates are in place end to end
- [ ] The Nginx server block is upgraded to also terminate TLS on 443 with the Let's Encrypt certificate (Certbot does this automatically in Step 8, but confirm the resulting config only serves HTTPS, with HTTP redirecting to HTTPS)
- [ ] GitHub webhook secret validation is enabled so Logstash/Nginx only accept signed payloads
- [ ] Firewall rules restrict inbound access to only the ports required (80/443), and Logstash's port 8080 is not directly reachable from the public internet
- [ ] Let's Encrypt auto-renewal is confirmed working (`sudo certbot renew --dry-run`)
- [ ] ISM size/age thresholds are set for production data volumes, not the 5 MB testing value
- [ ] Retention policy and backup strategy from Part A.2.2 are actually implemented (e.g. an ISM delete/snapshot state after the hot phase), not just documented

### Appendix 3 — Source Documents

This guide consolidates and edits the following working documents:

- SOC Client Onboarding Plan (non-Wazuh)
- Nginx Implementation — Replacing ngrok
- OpenSearch ISM Rollover Implementation Guide

Content has been corrected for grammar and terminology (e.g. "Telementary" → "telemetry", "RBACK" → "RBAC"), reorganized into a single narrative, and supplemented with explanatory notes, a security checklist, and a glossary that were not present in the originals.
