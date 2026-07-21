# OpenSearch Roles and Internal Users (Should be executed in VM)

## Overview

This section covers creating tenants, roles, users, role mappings, and updating permissions via the OpenSearch Security REST API. All commands are run on the Wazuh Manager/Indexer host.


## Step 1 — Create Tenants

```bash
curl -k -u admin:SecretPassword -X PUT "https://localhost:9200/_plugins/_security/api/tenants/TenantBWorkspace" \
  -H 'Content-Type: application/json' \
  -d '{"description": "Workspace for TenantB"}'
```

---

## Step 2 — Create or update tenant role with index, cluster, and tenant permissions

This step creates a role that combines OpenSearch cluster permissions, index permissions, and tenant permissions for the tenant workspace.

```bash
curl -k -u admin:SecretPassword -X PUT "https://localhost:9200/_plugins/_security/api/roles/<role_name>" \
  -H 'Content-Type: application/json' \
  -d '{
    "cluster_permissions": ["cluster_composite_ops_ro", "cluster_monitor"],
    "index_permissions": [
      {"index_patterns": ["<index-pattern>-*"], "allowed_actions": ["read","indices_monitor"]}
    ],
    "tenant_permissions": [
      {"tenant_patterns": ["<WorkspaceName>"], "allowed_actions": ["kibana_all_read"]}
    ]
  }'
```

---

## Step 3 — Seed Default Index Patterns

Run these after creating the tenant to seed the standard Wazuh index patterns into the tenant workspace:

```bash
curl -k -u admin:SecretPassword -X POST "https://192.168.45.13/api/saved_objects/index-pattern/wazuh-alerts" \
  -H "Content-Type: application/json" \
  -H "osd-xsrf: true" \
  -H "securitytenant: TenantBWorkspace" \
  -d '{"attributes": {"title": "wazuh-alerts-*", "timeFieldName": "timestamp"}}'

curl -k -u admin:SecretPassword -X POST "https://192.168.45.13/api/saved_objects/index-pattern/wazuh-monitoring" \
  -H "Content-Type: application/json" \
  -H "osd-xsrf: true" \
  -H "securitytenant: TenantBWorkspace" \
  -d '{"attributes": {"title": "wazuh-monitoring-*", "timeFieldName": "timestamp"}}'

curl -k -u admin:SecretPassword -X POST "https://192.168.45.13/api/saved_objects/index-pattern/wazuh-statistics" \
  -H "Content-Type: application/json" \
  -H "osd-xsrf: true" \
  -H "securitytenant: TenantBWorkspace" \
  -d '{"attributes": {"title": "wazuh-statistics-*", "timeFieldName": "timestamp"}}'
```

---

## Step 4 — Seed Tenant-Specific Alert Pattern

```bash
curl -k -u admin:SecretPassword -X POST "https://192.168.45.13/api/saved_objects/index-pattern/tenantb-alerts" \
  -H "Content-Type: application/json" \
  -H "osd-xsrf: true" \
  -H "securitytenant: TenantBWorkspace" \
  -d '{"attributes": {"title": "wazuh-tenant-tenantb-alerts-*", "timeFieldName": "timestamp"}}'
```

---

## Step 4 — Full RBAC Setup Script

This script creates both tenant users, both roles, both role mappings, and both OpenSearch tenants in one run.

Save as `rbac_setup.sh` and run it on the Wazuh Manager host:

```bash
#!/bin/bash

HOST="https://localhost:9200"
ADMIN_USER="admin"
ADMIN_PASS="SecretPassword"

echo "====== Creating TenantA User ======"
curl -X PUT "$HOST/_plugins/_security/api/internalusers/tenanta-user" \
  -H "Content-Type: application/json" \
  -u $ADMIN_USER:$ADMIN_PASS --insecure \
  -d '{
    "password": "TenantA@Pass123!",
    "opendistro_security_roles": [],
    "backend_roles": [],
    "attributes": {
      "tenant": "tenanta"
    }
  }'
echo " TenantA User Created"

echo "====== Creating TenantB User ======"
curl -X PUT "$HOST/_plugins/_security/api/internalusers/tenantb-user" \
  -H "Content-Type: application/json" \
  -u $ADMIN_USER:$ADMIN_PASS --insecure \
  -d '{
    "password": "TenantB@Pass123!",
    "opendistro_security_roles": [],
    "backend_roles": [],
    "attributes": {
      "tenant": "tenantb"
    }
  }'
echo " TenantB User Created"

echo "====== Creating TenantA Role ======"
curl -X PUT "$HOST/_plugins/_security/api/roles/tenanta-role" \
  -H "Content-Type: application/json" \
  -u $ADMIN_USER:$ADMIN_PASS --insecure \
  -d '{
    "cluster_permissions": [
      "cluster_composite_ops_ro"
    ],
    "index_permissions": [
      {
        "index_patterns": [
          "wazuh-tenanta-alerts-*",
          "wazuh-tenanta-states-vulnerabilities-wazuh.manager",
          "wazuh-tenanta-states-inventory-*-wazuh.manager"
        ],
        "dls": "",
        "fls": [],
        "masked_fields": [],
        "allowed_actions": [
          "read",
          "indices:data/read/search",
          "indices:data/read/msearch",
          "indices:admin/mappings/get"
        ]
      }
    ],
    "tenant_permissions": [
      {
        "tenant_patterns": ["TenantAWorkspace"],
        "allowed_actions": ["kibana_all_read"]
      }
    ]
  }'
echo ""

echo "====== Creating TenantB Role ======"
curl -X PUT "$HOST/_plugins/_security/api/roles/tenantb-role" \
  -H "Content-Type: application/json" \
  -u $ADMIN_USER:$ADMIN_PASS --insecure \
  -d '{
    "cluster_permissions": [
      "cluster_composite_ops_ro"
    ],
    "index_permissions": [
      {
        "index_patterns": [
          "wazuh-tenantb-alerts-*",
          "wazuh-tenantb-states-vulnerabilities-wazuh.manager",
          "wazuh-tenantb-states-inventory-*-wazuh.manager"
        ],
        "dls": "",
        "fls": [],
        "masked_fields": [],
        "allowed_actions": [
          "read",
          "indices:data/read/search",
          "indices:data/read/msearch",
          "indices:admin/mappings/get"
        ]
      }
    ],
    "tenant_permissions": [
      {
        "tenant_patterns": ["TenantBWorkspace"],
        "allowed_actions": ["kibana_all_read"]
      }
    ]
  }'
echo ""

echo "====== Creating TenantA Role Mapping ======"
curl -X PUT "$HOST/_plugins/_security/api/rolesmapping/tenanta-role" \
  -H "Content-Type: application/json" \
  -u $ADMIN_USER:$ADMIN_PASS --insecure \
  -d '{
    "backend_roles": [],
    "hosts": [],
    "users": ["tenanta-user"]
  }'
echo ""

echo "====== Creating TenantB Role Mapping ======"
curl -X PUT "$HOST/_plugins/_security/api/rolesmapping/tenantb-role" \
  -H "Content-Type: application/json" \
  -u $ADMIN_USER:$ADMIN_PASS --insecure \
  -d '{
    "backend_roles": [],
    "hosts": [],
    "users": ["tenantb-user"]
  }'
echo ""

echo "RBAC Setup Done!"
```

---

## Step 5 — Verification Script

Save as `rbac_verify.sh` and run to confirm everything was created correctly:

```bash
#!/bin/bash

HOST="https://localhost:9200"
ADMIN_USER="admin"
ADMIN_PASS="SecretPassword"

echo "====== Check Users ======"
curl -X GET "$HOST/_plugins/_security/api/internalusers/tenanta-user?pretty" \
  -u $ADMIN_USER:$ADMIN_PASS --insecure

curl -X GET "$HOST/_plugins/_security/api/internalusers/tenantb-user?pretty" \
  -u $ADMIN_USER:$ADMIN_PASS --insecure

echo "====== Check Roles ======"
curl -X GET "$HOST/_plugins/_security/api/roles/tenanta-role?pretty" \
  -u $ADMIN_USER:$ADMIN_PASS --insecure

curl -X GET "$HOST/_plugins/_security/api/roles/tenantb-role?pretty" \
  -u $ADMIN_USER:$ADMIN_PASS --insecure

echo "====== Check Role Mappings ======"
curl -X GET "$HOST/_plugins/_security/api/rolesmapping/tenanta-role?pretty" \
  -u $ADMIN_USER:$ADMIN_PASS --insecure

curl -X GET "$HOST/_plugins/_security/api/rolesmapping/tenantb-role?pretty" \
  -u $ADMIN_USER:$ADMIN_PASS --insecure

echo "====== Check Tenants ======"
curl -X GET "$HOST/_plugins/_security/api/tenants?pretty" \
  -u $ADMIN_USER:$ADMIN_PASS --insecure

echo "====== Test TenantA User Access ======"
# Should SUCCEED — tenanta index
curl -X GET "$HOST/wazuh-tenanta-alerts-*/_count?pretty" \
  -u tenanta-user:TenantA@Pass123! --insecure

# Should FAIL — tenantb index blocked
curl -X GET "$HOST/wazuh-tenantb-alerts-*/_count?pretty" \
  -u tenanta-user:TenantA@Pass123! --insecure

echo "====== Test TenantB User Access ======"
# Should SUCCEED — tenantb index
curl -X GET "$HOST/wazuh-tenantb-alerts-*/_count?pretty" \
  -u tenantb-user:TenantB@Pass123! --insecure

# Should FAIL — tenanta index blocked
curl -X GET "$HOST/wazuh-tenanta-alerts-*/_count?pretty" \
  -u tenantb-user:TenantB@Pass123! --insecure

echo "Verification Done!"
```

---

## Step 6 — Update Role Permissions Script

Run this after initial setup to add full cluster monitor permissions, `.kibana*` index access, and upgrade tenant permissions to `kibana_all_write`:

```bash
#!/bin/bash

HOST="https://localhost:9200"
USER="admin"
PASS="SecretPassword"

echo "====== Update TenantA Role — Add Permissions ======"
curl -X PATCH "$HOST/_plugins/_security/api/roles/tenanta-role" \
  -H "Content-Type: application/json" \
  -u $USER:$PASS --insecure \
  -d '[
    {
      "op": "add",
      "path": "/cluster_permissions",
      "value": [
        "cluster_composite_ops",
        "cluster_composite_ops_ro",
        "cluster:monitor/main",
        "cluster:monitor/health",
        "cluster:monitor/state",
        "cluster:monitor/nodes/info",
        "cluster:monitor/nodes/stats",
        "indices:data/read/scroll",
        "indices:data/read/scroll/clear"
      ]
    },
    {
      "op": "add",
      "path": "/index_permissions/-",
      "value": {
        "index_patterns": [".kibana*"],
        "allowed_actions": [
          "read",
          "write",
          "delete",
          "create_index",
          "indices:data/read*",
          "indices:data/write*",
          "indices:admin*"
        ]
      }
    },
    {
      "op": "add",
      "path": "/tenant_permissions",
      "value": [
        {
          "tenant_patterns": ["tenanta"],
          "allowed_actions": ["kibana_all_write"]
        }
      ]
    }
  ]'
echo ""

echo "====== Update TenantB Role — Add Permissions ======"
curl -X PATCH "$HOST/_plugins/_security/api/roles/tenantb-role" \
  -H "Content-Type: application/json" \
  -u $USER:$PASS --insecure \
  -d '[
    {
      "op": "add",
      "path": "/cluster_permissions",
      "value": [
        "cluster_composite_ops",
        "cluster_composite_ops_ro",
        "cluster:monitor/main",
        "cluster:monitor/health",
        "cluster:monitor/state",
        "cluster:monitor/nodes/info",
        "cluster:monitor/nodes/stats",
        "indices:data/read/scroll",
        "indices:data/read/scroll/clear"
      ]
    },
    {
      "op": "add",
      "path": "/index_permissions/-",
      "value": {
        "index_patterns": [".kibana*"],
        "allowed_actions": [
          "read",
          "write",
          "delete",
          "create_index",
          "indices:data/read*",
          "indices:data/write*",
          "indices:admin*"
        ]
      }
    },
    {
      "op": "add",
      "path": "/tenant_permissions",
      "value": [
        {
          "tenant_patterns": ["tenantb"],
          "allowed_actions": ["kibana_all_write"]
        }
      ]
    }
  ]'
echo ""

echo "====== Add kibana_user Mapping — CRITICAL ======"
curl -X PUT "$HOST/_plugins/_security/api/rolesmapping/kibana_user" \
  -H "Content-Type: application/json" \
  -u $USER:$PASS --insecure \
  -d '{
    "users": ["tenanta-user", "tenantb-user"]
  }'
echo ""

echo "====== Verify ======"
curl -X GET "$HOST/_plugins/_security/api/roles/tenanta-role?pretty" \
  -u $USER:$PASS --insecure

curl -X GET "$HOST/_plugins/_security/api/roles/tenantb-role?pretty" \
  -u $USER:$PASS --insecure

curl -X GET "$HOST/_plugins/_security/api/rolesmapping/kibana_user?pretty" \
  -u $USER:$PASS --insecure

echo "Done!"
```

!!! warning "kibana_user mapping is critical"
    Both tenant users must be mapped to the `kibana_user` role. Without this mapping, the Wazuh Dashboard will fail to initialize for those users even if their tenant role is correct.