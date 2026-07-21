# Wazuh Multi-Tenant MSSP Implementation

## 1. Objective

This document describes the implementation of a multi-tenant Wazuh MSSP proof of concept.

The objective is to provide tenant-level visibility and access control using:

* Wazuh Manager
* Wazuh Indexer
* Wazuh Dashboard
* OpenSearch Security Plugin
* Tenant-specific indices
* Role-Based Access Control (RBAC)

The implementation focuses on:

* Alert isolation
* File Integrity Monitoring (FIM) visibility
* Inventory isolation
* Vulnerability data isolation
* Tenant-specific OpenSearch roles and users

> **POC Scope:** Alerts are routed directly into tenant-specific indices. Inventory and vulnerability data are initially stored in shared Wazuh state indices and are copied into tenant-specific indices using the OpenSearch `_reindex` API. This is a manual POC process and must be repeated when new or updated state data is required in the tenant index.

---

## 2. High-Level Architecture

```text
Wazuh Agent
        |
        v
Wazuh Manager
        |
        +------------------------------+
        |                              |
        v                              v
alerts.json                     Indexer Connector
        |                              |
        v                              v
Filebeat                    Shared wazuh-states-* indices
        |                              |
        v                              v
Tenant-specific alerts       Manual _reindex by agent.name
indices                      into tenant-specific state indices
        |                              |
        +--------------+---------------+
                       |
                       v
OpenSearch Security Plugin
                       |
                       v
Wazuh Dashboard Tenant View
```

---

## 3. Component Responsibilities

| Component                  | Responsibility                                                                                     |
| -------------------------- | -------------------------------------------------------------------------------------------------- |
| Wazuh Agent                | Collects endpoint logs, FIM events, Syscollector inventory, and security telemetry                 |
| Wazuh Manager              | Analyzes events, generates alerts, maintains agent inventory, and performs vulnerability detection |
| Filebeat                   | Reads Wazuh alerts from `alerts.json` and sends alerts to Wazuh Indexer                            |
| Indexer Connector          | Sends inventory and vulnerability state data from Wazuh Manager to Wazuh Indexer                   |
| Wazuh Indexer              | Stores alerts, inventory, vulnerabilities, and tenant-specific copied indices                      |
| OpenSearch Security Plugin | Enforces index-level RBAC and Dashboard tenant permissions                                         |
| Wazuh Dashboard            | Provides tenant-specific dashboards, Discover access, and visualizations                           |

---

## 4. Data Flow

### 4.1 Alerts and FIM Events

Alerts are event-based telemetry.

```text
Wazuh Agent
        |
        v
Wazuh Manager
        |
        v
/var/ossec/logs/alerts/alerts.json
        |
        v
Filebeat
        |
        v
wazuh-tenanta-alerts-*
```

Examples include:

* Syslog events
* SSH failed-login alerts
* Malware alerts
* Authentication failures
* FIM alerts

FIM events are alerts, not state documents. Therefore, FIM isolation is achieved through tenant-specific alert indexing.

### 4.2 Inventory and Vulnerability State Data

Inventory is state-based telemetry.

```text
Wazuh Agent Syscollector
        |
        v
Wazuh Manager internal database
        |
        v
Indexer Connector
        |
        v
wazuh-states-*
        |
        v
Manual OpenSearch _reindex API
        |
        v
wazuh-tenanta-states-*
```

Examples include:

* Installed packages
* Running processes
* Local users
* Open ports
* Network interfaces
* Vulnerability records

---

## 5. Why Filebeat Is Not Used for Inventory

Filebeat is designed for append-only logs and event streams.

Inventory and vulnerability information requires:

* State synchronization
* Updates to existing records
* Deduplication
* Latest-state maintenance

For this reason, Wazuh uses the Indexer Connector instead of Filebeat for inventory and vulnerability data.

---

# Part A — Prerequisites and Naming Convention

## Step 1: Confirm Docker Container Names

Run the following command on the Wazuh Docker host:

```bash
docker ps --format "table {{.Names}}\t{{.Status}}"
```

Expected container names in this guide:

```text
single-node-wazuh.manager-1
single-node-wazuh.indexer-1
single-node-wazuh.dashboard-1
```

If your container names are different, replace them in every command in this document.

---

## Step 2: Define Tenant Names and Agent Mapping

This guide uses the following example mapping:

| Tenant   | OpenSearch tenant | Agent name              | Tenant alert index prefix |
| -------- | ----------------- | ----------------------- | ------------------------- |
| Tenant A | `TenantA`         | `Kibana`                | `wazuh-tenanta-alerts-*`  |
| Tenant B | `TenantB`         | `<TENANT_B_AGENT_NAME>` | `wazuh-tenantb-alerts-*`  |

Replace the example agent name `Kibana` with the actual Wazuh agent name assigned to Tenant A.

Verify agent names:

```bash
docker exec -it single-node-wazuh.manager-1 /var/ossec/bin/agent_control -l
```

---

## Step 3: Define Index Names

This POC uses the following tenant-specific index names.

| Data Type           | Tenant A Index                             |
| ------------------- | ------------------------------------------ |
| Alerts and FIM      | `wazuh-tenanta-alerts-*`                   |
| Inventory packages  | `wazuh-tenanta-states-inventory-packages`  |
| Inventory processes | `wazuh-tenanta-states-inventory-processes` |
| Inventory ports     | `wazuh-tenanta-states-inventory-ports`     |
| Inventory users     | `wazuh-tenanta-states-inventory-users`     |
| Inventory networks  | `wazuh-tenanta-states-inventory-networks`  |
| Vulnerabilities     | `wazuh-tenanta-states-vulnerabilities`     |

---

# Part B — Configure OpenSearch Security Plugin

## Step 4: Enter the Wazuh Indexer Container

```bash
docker exec -it single-node-wazuh.indexer-1 bash
```

---

## Step 5: Access the Security Configuration Directory

```bash
cd /usr/share/wazuh-indexer/config/opensearch-security/
```

Back up the security configuration before making changes:

```bash
cp -a /usr/share/wazuh-indexer/config/opensearch-security \
/usr/share/wazuh-indexer/config/opensearch-security.backup
```

---

## Step 6: Configure OpenSearch Dashboard Tenants

Open the tenant configuration file:

```bash
vi tenants.yml
```

Add the following entries below the `_meta` section:

```yaml
TenantA:
  reserved: false
  hidden: false
  description: "Tenant A Dashboard"

TenantB:
  reserved: false
  hidden: false
  description: "Tenant B Dashboard"
```

The complete structure should be similar to:

```yaml
_meta:
  type: "tenants"
  config_version: 2

TenantA:
  reserved: false
  hidden: false
  description: "Tenant A Dashboard"

TenantB:
  reserved: false
  hidden: false
  description: "Tenant B Dashboard"
```

Save the file.

---

## Step 7: Configure Tenant A Role

Open the roles configuration file:

```bash
vi roles.yml
```

Add the following role:

```yaml
tenantA_role:
  reserved: false
  hidden: false
  cluster_permissions:
    - "cluster_composite_ops_ro"
  index_permissions:
    - index_patterns:
        - "wazuh-tenanta-alerts-*"
        - "wazuh-tenanta-states-vulnerabilities-*"
        - "wazuh-tenanta-states-inventory-packages-*"
        - "wazuh-tenanta-states-inventory-users-*"
        - "wazuh-tenanta-states-inventory-networks-*"
        - "wazuh-tenanta-states-inventory-ports-*"
        - "wazuh-tenanta-states-inventory-processes-*"
      allowed_actions:
        - "read"
        - "search"
        - "indices:data/read/*"
  tenant_permissions:
    - tenant_patterns:
        - "TenantA"
      allowed_actions:
        - "kibana_all_read"
```

Save the file.

> This role grants read-only access only to Tenant A index patterns and to saved objects stored in the `TenantA` OpenSearch Dashboard tenant.

---

## Step 8: Configure Tenant B Role

In `roles.yml`, add the following role:

```yaml
tenantB_role:
  reserved: false
  hidden: false
  cluster_permissions:
    - "cluster_composite_ops_ro"
  index_permissions:
    - index_patterns:
        - "wazuh-tenantb-alerts-*"
        - "wazuh-tenantb-states-vulnerabilities-*"
        - "wazuh-tenantb-states-inventory-packages-*"
        - "wazuh-tenantb-states-inventory-users-*"
        - "wazuh-tenantb-states-inventory-networks-*"
        - "wazuh-tenantb-states-inventory-ports-*"
        - "wazuh-tenantb-states-inventory-processes-*"
      allowed_actions:
        - "read"
        - "search"
        - "indices:data/read/*"
  tenant_permissions:
    - tenant_patterns:
        - "TenantB"
      allowed_actions:
        - "kibana_all_read"
```

Save the file.

---

## Step 9: Create Tenant Users

Open the internal users configuration file:

```bash
vi internal_users.yml
```

Generate a password hash for each user. Run the following command inside the Indexer container:

```bash
/usr/share/wazuh-indexer/plugins/opensearch-security/tools/hash.sh
```

Enter a strong password when prompted and copy the generated hash.

Add Tenant A user configuration:

```yaml
tenantA_user:
  hash: "<TENANT_A_BCRYPT_PASSWORD_HASH>"
  reserved: false
  backend_roles:
    - "tenantA_role"
  description: "Tenant A User"
```

Add Tenant B user configuration:

```yaml
tenantB_user:
  hash: "<TENANT_B_BCRYPT_PASSWORD_HASH>"
  reserved: false
  backend_roles:
    - "tenantB_role"
  description: "Tenant B User"
```

Replace the placeholder values with the generated bcrypt password hashes.

Save the file.

---

## Step 10: Configure Role Mapping

Open the role mapping configuration file:

```bash
vi roles_mapping.yml
```

Add the following mappings:

```yaml
tenantA_role:
  reserved: false
  hidden: false
  backend_roles:
    - "tenantA_role"
  users:
    - "tenantA_user"
  description: "Maps Tenant A user to Tenant A role"

tenantB_role:
  reserved: false
  hidden: false
  backend_roles:
    - "tenantB_role"
  users:
    - "tenantB_user"
  description: "Maps Tenant B user to Tenant B role"
```

Save the file.

---

## Step 11: Apply OpenSearch Security Configuration

Run the following command inside the Wazuh Indexer container:

```bash
bash /usr/share/wazuh-indexer/plugins/opensearch-security/tools/securityadmin.sh \
-cd /usr/share/wazuh-indexer/config/opensearch-security/ \
-icl \
-key /usr/share/wazuh-indexer/certs/admin-key.pem \
-cert /usr/share/wazuh-indexer/certs/admin.pem \
-cacert /usr/share/wazuh-indexer/certs/root-ca.pem \
-nhnv \
-h localhost \
-p 9200
```

The security administration tool applies tenant, role, user, and role mapping changes to the OpenSearch Security Plugin.

Exit the Indexer container:

```bash
exit
```

---

# Part C — Configure Tenant-Specific Alert Isolation

## Step 12: Identify the Filebeat Configuration File

Enter the Wazuh Manager container:

```bash
docker exec -it single-node-wazuh.manager-1 bash
```

Locate the Filebeat configuration file:

```bash
find / -name filebeat.yml 2>/dev/null
```

In a standard Wazuh Docker deployment, the Filebeat configuration is commonly located at:

```text
/etc/filebeat/filebeat.yml
```

---

## Step 13: Back Up Filebeat Configuration

```bash
cp /etc/filebeat/filebeat.yml /etc/filebeat/filebeat.yml.backup
```

---

## Step 14: Configure Tenant A Alert Routing

Open the Filebeat configuration:

```bash
vi /etc/filebeat/filebeat.yml
```

Locate the existing `output.elasticsearch` or `output.opensearch` section.

Add the following `indices` configuration under the output section. Keep the existing `hosts`, credentials, SSL settings, and default index configuration unchanged.

```yaml
indices:
  - index: "wazuh-tenanta-alerts-%{+yyyy.MM.dd}"
    when.equals:
      agent.name: "Kibana"
```

Replace `Kibana` with the actual Tenant A agent name.

For Tenant B, add:

```yaml
  - index: "wazuh-tenantb-alerts-%{+yyyy.MM.dd}"
    when.equals:
      agent.name: "<TENANT_B_AGENT_NAME>"
```

Keep the existing default Wazuh alert index as the final fallback:

```yaml
  - index: "wazuh-alerts-4.x-%{+yyyy.MM.dd}"
```

Example:

```yaml
output.elasticsearch:
  hosts: ["https://single-node-wazuh.indexer-1:9200"]
  username: "admin"
  password: "<INDEXER_PASSWORD>"
  ssl.verification_mode: none
  indices:
    - index: "wazuh-tenanta-alerts-%{+yyyy.MM.dd}"
      when.equals:
        agent.name: "Kibana"
    - index: "wazuh-tenantb-alerts-%{+yyyy.MM.dd}"
      when.equals:
        agent.name: "<TENANT_B_AGENT_NAME>"
    - index: "wazuh-alerts-4.x-%{+yyyy.MM.dd}"
```

Save the file.

---

## Step 15: Validate Filebeat Configuration

Run:

```bash
filebeat test config -c /etc/filebeat/filebeat.yml
```

Expected output:

```text
Config OK
```

Restart Filebeat:

```bash
service filebeat restart
```

Check Filebeat status:

```bash
service filebeat status
```

Exit the Manager container:

```bash
exit
```

---

## Step 16: Verify Tenant-Specific Alert Index

Run the following command from the Docker host:

```bash
docker exec -it single-node-wazuh.indexer-1 \
curl -k -u admin:<INDEXER_PASSWORD> \
"https://localhost:9200/_cat/indices/wazuh-tenanta-alerts-*?v"
```

Generate an alert from the Tenant A endpoint, then verify that a tenant-specific alert index is created.

Expected index pattern:

```text
wazuh-tenanta-alerts-YYYY.MM.dd
```

---

# Part D — Enable and Verify Indexer Connector

## Step 17: Locate Persistent Wazuh Manager Configuration

On the Docker host, move to the Wazuh Docker single-node directory:

```bash
cd ~/wazuh-docker/single-node
```

The persistent Wazuh Manager configuration file is commonly:

```text
config/wazuh_cluster/wazuh_manager.conf
```

Back up the configuration:

```bash
cp config/wazuh_cluster/wazuh_manager.conf \
config/wazuh_cluster/wazuh_manager.conf.backup
```

---

## Step 18: Enable the Indexer Connector

Open the persistent manager configuration file:

```bash
nano config/wazuh_cluster/wazuh_manager.conf
```

Locate the `<indexer>` block.

If the block contains:

```xml
<indexer>
  <enabled>no</enabled>
</indexer>
```

Change it to:

```xml
<indexer>
  <enabled>yes</enabled>
</indexer>
```

If the `<indexer>` block does not exist, add the following inside the `<ossec_config>` section:

```xml
<indexer>
  <enabled>yes</enabled>
  <hosts>
    <host>https://single-node-wazuh.indexer-1:9200</host>
  </hosts>
  <ssl>
    <certificate_authorities>
      <ca>/etc/ssl/root-ca.pem</ca>
    </certificate_authorities>
    <certificate>/etc/ssl/filebeat.pem</certificate>
    <key>/etc/ssl/filebeat-key.pem</key>
  </ssl>
</indexer>
```

> Use the certificate paths already present in the Wazuh Manager container. Do not replace working certificate paths in an existing configuration.

Save the file.

---

## Step 19: Restart the Wazuh Manager Container

```bash
docker restart single-node-wazuh.manager-1
```

Wait for the container to become healthy:

```bash
docker ps --filter "name=single-node-wazuh.manager-1"
```

---

## Step 20: Configure Indexer Credentials in Wazuh Keystore

Enter the Manager container:

```bash
docker exec -it single-node-wazuh.manager-1 bash
```

Store the Indexer username:

```bash
echo -n "admin" | /var/ossec/bin/wazuh-keystore -f indexer -k username
```

Store the Indexer password:

```bash
echo -n "<INDEXER_PASSWORD>" | /var/ossec/bin/wazuh-keystore -f indexer -k password
```

Exit the container:

```bash
exit
```

---

## Step 21: Verify Indexer Connector Initialization

Run:

```bash
docker exec -it single-node-wazuh.manager-1 \
sh -c 'grep -i "IndexerConnector" /var/ossec/logs/ossec.log | tail -n 20'
```

Expected output includes a successful Indexer Connector initialization message.

The Indexer Connector forwards Wazuh inventory and vulnerability state information to the Wazuh Indexer.

---

## Step 22: Verify Shared State Indices

```bash
docker exec -it single-node-wazuh.indexer-1 \
curl -k -u admin:<INDEXER_PASSWORD> \
"https://localhost:9200/_cat/indices?v" | grep wazuh-states
```

Expected shared index patterns can include:

```text
wazuh-states-inventory-packages-*
wazuh-states-inventory-processes-*
wazuh-states-inventory-ports-*
wazuh-states-inventory-users-*
wazuh-states-inventory-networks-*
wazuh-states-vulnerabilities-*
```

---

# Part E — Manual Tenant Inventory Isolation Using _reindex

## Step 23: Confirm Tenant A Agent Name in Shared Inventory Index

Run:

```bash
docker exec -it single-node-wazuh.indexer-1 \
curl -k -u admin:<INDEXER_PASSWORD> \
"https://localhost:9200/wazuh-states-inventory-packages-*/_search?q=agent.name:Kibana&pretty&size=5"
```

Replace `Kibana` with the Tenant A agent name.

Confirm that the response contains Tenant A package inventory documents.

---

## Step 24: Reindex Tenant A Package Inventory

Run:

```bash
docker exec -i single-node-wazuh.indexer-1 \
curl -k -u admin:<INDEXER_PASSWORD> \
-X POST "https://localhost:9200/_reindex" \
-H "Content-Type: application/json" \
-d '{
  "source": {
    "index": "wazuh-states-inventory-packages-*",
    "query": {
      "term": {
        "agent.name.keyword": "Kibana"
      }
    }
  },
  "dest": {
    "index": "wazuh-tenanta-states-inventory-packages"
  }
}'
```

Replace `Kibana` with the Tenant A agent name.

---

## Step 25: Reindex Tenant A Process Inventory

```bash
docker exec -i single-node-wazuh.indexer-1 \
curl -k -u admin:<INDEXER_PASSWORD> \
-X POST "https://localhost:9200/_reindex" \
-H "Content-Type: application/json" \
-d '{
  "source": {
    "index": "wazuh-states-inventory-processes-*",
    "query": {
      "term": {
        "agent.name.keyword": "Kibana"
      }
    }
  },
  "dest": {
    "index": "wazuh-tenanta-states-inventory-processes"
  }
}'
```

---

## Step 26: Reindex Tenant A Port Inventory

```bash
docker exec -i single-node-wazuh.indexer-1 \
curl -k -u admin:<INDEXER_PASSWORD> \
-X POST "https://localhost:9200/_reindex" \
-H "Content-Type: application/json" \
-d '{
  "source": {
    "index": "wazuh-states-inventory-ports-*",
    "query": {
      "term": {
        "agent.name.keyword": "Kibana"
      }
    }
  },
  "dest": {
    "index": "wazuh-tenanta-states-inventory-ports"
  }
}'
```

---

## Step 27: Reindex Tenant A User Inventory

```bash
docker exec -i single-node-wazuh.indexer-1 \
curl -k -u admin:<INDEXER_PASSWORD> \
-X POST "https://localhost:9200/_reindex" \
-H "Content-Type: application/json" \
-d '{
  "source": {
    "index": "wazuh-states-inventory-users-*",
    "query": {
      "term": {
        "agent.name.keyword": "Kibana"
      }
    }
  },
  "dest": {
    "index": "wazuh-tenanta-states-inventory-users"
  }
}'
```

---

## Step 28: Reindex Tenant A Network Inventory

```bash
docker exec -i single-node-wazuh.indexer-1 \
curl -k -u admin:<INDEXER_PASSWORD> \
-X POST "https://localhost:9200/_reindex" \
-H "Content-Type: application/json" \
-d '{
  "source": {
    "index": "wazuh-states-inventory-networks-*",
    "query": {
      "term": {
        "agent.name.keyword": "Kibana"
      }
    }
  },
  "dest": {
    "index": "wazuh-tenanta-states-inventory-networks"
  }
}'
```

---

## Step 29: Verify Tenant A Inventory Indices

```bash
docker exec -it single-node-wazuh.indexer-1 \
curl -k -u admin:<INDEXER_PASSWORD> \
"https://localhost:9200/_cat/indices/wazuh-tenanta-states-inventory-*?v"
```

---

# Part F — Manual Tenant Vulnerability Isolation Using _reindex

## Step 30: Verify Shared Vulnerability Index

```bash
docker exec -it single-node-wazuh.indexer-1 \
curl -k -u admin:<INDEXER_PASSWORD> \
"https://localhost:9200/_cat/indices?v" | grep wazuh-states-vulnerabilities
```

Expected shared index pattern:

```text
wazuh-states-vulnerabilities-*
```

---

## Step 31: Verify Tenant A Vulnerability Documents

```bash
docker exec -it single-node-wazuh.indexer-1 \
curl -k -u admin:<INDEXER_PASSWORD> \
"https://localhost:9200/wazuh-states-vulnerabilities-*/_search?q=agent.name:Kibana&pretty&size=5"
```

---

## Step 32: Reindex Tenant A Vulnerability Documents

```bash
docker exec -i single-node-wazuh.indexer-1 \
curl -k -u admin:<INDEXER_PASSWORD> \
-X POST "https://localhost:9200/_reindex" \
-H "Content-Type: application/json" \
-d '{
  "source": {
    "index": "wazuh-states-vulnerabilities-*",
    "query": {
      "term": {
        "agent.name.keyword": "Kibana"
      }
    }
  },
  "dest": {
    "index": "wazuh-tenanta-states-vulnerabilities"
  }
}'
```

Replace `Kibana` with the Tenant A agent name.

---

## Step 33: Verify Tenant A Vulnerability Index

```bash
docker exec -it single-node-wazuh.indexer-1 \
curl -k -u admin:<INDEXER_PASSWORD> \
"https://localhost:9200/wazuh-tenanta-states-vulnerabilities/_search?pretty&size=10"
```

---

# Part G — Create Tenant-Specific Discover Data Views

## Step 34: Log In as Tenant A User

Open Wazuh Dashboard and log in with:

```text
Username: tenantA_user
Password: <TENANT_A_PASSWORD>
```

Select the `TenantA` Dashboard tenant when prompted.

---

## Step 35: Create Tenant A Alert Data View

Navigate to:

```text
OpenSearch Dashboards Management → Data views
```

Create a data view:

```text
Name: Tenant A Alerts
Index pattern: wazuh-tenanta-alerts-*
Time field: @timestamp
```

---

## Step 36: Create Tenant A Inventory and Vulnerability Data Views

Create the following data views:

```text
Tenant A Packages
wazuh-tenanta-states-inventory-packages*

Tenant A Processes
wazuh-tenanta-states-inventory-processes*

Tenant A Ports
wazuh-tenanta-states-inventory-ports*

Tenant A Users
wazuh-tenanta-states-inventory-users*

Tenant A Networks
wazuh-tenanta-states-inventory-networks*

Tenant A Vulnerabilities
wazuh-tenanta-states-vulnerabilities*
```

Use the appropriate timestamp field if one is available; otherwise, create the data view without a time filter.

---

# Part H — Validation

## Step 37: Validate Tenant A Alert Access

Log in as `tenantA_user`.

Open Discover and select `Tenant A Alerts`.

Verify that alerts from Tenant A agents are visible.

Attempt to create or access a data view for:

```text
wazuh-tenantb-alerts-*
```

The request should be denied because `tenantA_role` does not have permissions for Tenant B indices.

---

## Step 38: Validate Tenant A Inventory Access

Open Discover and select `Tenant A Packages`.

Verify that package inventory documents contain only the Tenant A agent name.

Example Discover query:

```text
agent.name: "Kibana"
```

---

## Step 39: Validate Tenant A Vulnerability Access

Open Discover and select `Tenant A Vulnerabilities`.

Verify that vulnerability documents contain only Tenant A agent data.

---

# Part I — Important POC Limitations

## 1. _reindex Is Not Continuous Synchronization

The `_reindex` API copies documents that exist at the time the command is executed.

When Wazuh receives new inventory scans, package changes, process updates, port changes, or vulnerability updates, those documents continue to be written to the shared `wazuh-states-*` indices.

To refresh tenant-specific POC indices, rerun the relevant `_reindex` command.

---

## 2. Existing Destination Documents Are Not Automatically Removed

The `_reindex` API copies matching documents into the destination index. It does not automatically remove documents that no longer match the source state.

For a clean POC refresh, delete and recreate the tenant-specific destination index before rerunning `_reindex`.

Example for Tenant A package inventory:

```bash
docker exec -it single-node-wazuh.indexer-1 \
curl -k -u admin:<INDEXER_PASSWORD> \
-X DELETE "https://localhost:9200/wazuh-tenanta-states-inventory-packages"
```

Then rerun the package inventory `_reindex` command.

---

## 3. Dashboard Tenants Do Not Replace Index Permissions

OpenSearch Dashboard tenants isolate saved objects such as dashboards, visualizations, and data views.

Index-level permissions in `roles.yml` are what prevent a tenant user from reading another tenant’s indices.

---

## 4. Wazuh API Access Requires Separate Restriction

This POC enforces tenant isolation for index data through OpenSearch Security Plugin permissions.

Wazuh Dashboard features that query the Wazuh Manager API may require additional Wazuh RBAC configuration to restrict each tenant user to only its own agents.

---

# Part J — Final Architecture Achieved

| Feature                 | POC Isolation Method                                                                 |
| ----------------------- | ------------------------------------------------------------------------------------ |
| Alerts                  | Filebeat routes alerts into tenant-specific alert indices                            |
| FIM                     | FIM is stored as alerts and follows tenant-specific alert routing                    |
| Inventory               | Manual `_reindex` from shared state indices using `agent.name` filtering             |
| Vulnerabilities         | Manual `_reindex` from shared vulnerability state index using `agent.name` filtering |
| RBAC                    | OpenSearch Security Plugin role and index permissions                                |
| Dashboard saved objects | OpenSearch Dashboard tenants                                                         |

```text
Wazuh Agent
        |
        v
Wazuh Manager
        |
        +----------------------------+
        |                            |
        v                            v
Filebeat                     Indexer Connector
        |                            |
        v                            v
Tenant alert indices          Shared Wazuh state indices
        |                            |
        +-------------+--------------+
                      |
                      v
OpenSearch Security Plugin RBAC
                      |
                      v
Tenant-specific Dashboard view
```

---

#
