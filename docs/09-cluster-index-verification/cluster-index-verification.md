# Cluster and Index Verification & Monitoring Permissions

<!-- **Author:** Abhinav   -->
**Version:** 1.0  
<!-- **Created Date:** 2026-07-28  
**Updated Date:** 2026-07-28 -->

This document references the predefined action groups and specific granular permissions required to monitor and verify the health, status, and task states of the Wazuh Indexer / OpenSearch cluster without allowing any structural changes.

## 1. Monitoring Action Groups (Shorthand Names)

Use these shorthand names in the respective configuration fields to apply multiple monitoring permissions at once.

| Action Group Name | Type | What It Allows |
| :--- | :--- | :--- |
| `cluster_monitor` | Cluster | Read-only access to all cluster monitoring APIs (health, stats, info, pending tasks, etc.). |
| `indices_monitor` | Cluster | Cluster-level alias that grants monitoring permission across all indices (stats, segments, recovery). |
| `monitor` | Index | Allows read-only monitoring of a specific index (stats, segments, recovery, shard stores). |

## 2. Granular Cluster Monitoring Permissions (`cluster:monitor/*`)

These permissions grant raw visibility into the cluster state, health, and task activity. They should be applied in the **Cluster Permissions** field of a role.

| Permission | Description |
| :--- | :--- |
| `cluster:monitor/main` | Returns basic cluster info (name, version, UUID) via the root `/` endpoint. |
| `cluster:monitor/health` | Retrieves the cluster health status (green/yellow/red) and shard counts. |
| `cluster:monitor/state` | Returns the full cluster state (routing table, metadata, blocks, nodes). |
| `cluster:monitor/stats` | Returns cluster-wide statistics (node counts, index counts, JVM, OS, etc.). |
| `cluster:monitor/nodes/info` | Returns node-level information (hostnames, roles, attributes, plugins, and version information). Consider restricting this permission for tenant-facing roles. |
| `cluster:monitor/nodes/stats` | Returns per-node metrics (JVM heap, CPU, disk, thread pools, indices). |
| `cluster:monitor/nodes/hot_threads` | Returns hot thread info for diagnosing high CPU on individual nodes. |
| `cluster:monitor/nodes/liveness` | Internal liveness ping used by nodes to check peer reachability. |
| `cluster:monitor/pending_tasks` | Lists cluster-level tasks waiting in the master task queue. |
| `cluster:monitor/task` | Retrieves details of a specific running or completed task by task ID. |
| `cluster:monitor/tasks/list` | Lists all currently running tasks across every node in the cluster. |

Note: Evaluate whether cluster:monitor/state is required for tenant-facing roles because it exposes detailed cluster metadata.

## Optional Operational Permissions

The following permissions are not required for monitoring and allow operational intervention:

- cluster:monitor/tasks/cancel

## 3. Granular Index Monitoring Permissions (`indices:monitor/*`)

These permissions provide visibility into individual index operational health metrics and must be applied in the **Index Permissions** configuration block.

| Permission | Description |
| :--- | :--- |
| `indices:monitor/stats` | Returns index-level statistics (docs, store size, query cache, etc.). |
| `indices:monitor/settings/get` | Retrieves index settings via the monitoring endpoint. |
| `indices:monitor/segments` | Returns Lucene segment information for the index (count, size, doc count per segment). |
| `indices:monitor/shard_stores` | Returns shard store metadata and allocation status. |
| `indices:monitor/recovery` | Returns recovery status (primary/replica promotion, peer recovery progress). |
| `indices:monitor/upgrade` | Returns upgrade status showing whether shards need upgrading. |