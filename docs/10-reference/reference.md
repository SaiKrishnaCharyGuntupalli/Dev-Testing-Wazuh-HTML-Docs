# Wazuh Dashboard — Security Role Permissions Master Reference

**Source:** OpenSearch Security Plugin (used by Wazuh Indexer / Wazuh Dashboard)  
**Configuration Location:** Indexer Management → Security → Edit Role → Cluster Permissions & Index Permissions

---

## SECTION 1 — CLUSTER PERMISSIONS

Applied in the **Cluster Permissions** field of a role. These control cluster-wide operations independent of any specific index.

### 1.1 Cluster-Level Action Groups
Shorthand names you can type directly to apply multiple permissions at once.

| Action Group Name | What It Allows |
| :--- | :--- |
| `cluster_all` | Full access to all cluster-level operations; alias for `cluster:*` wildcard. |
| `cluster_monitor` | Read-only access to all cluster monitoring APIs (health, stats, info, pending tasks, etc.). |
| `cluster_composite_ops` | Permits bulk, multi-get (mget), multi-search (msearch), multi-term vector, and reindex operations. |
| `cluster_composite_ops_ro` | Read-only composite ops: msearch, mget, and multi-term vector (no write operations). |
| `manage_snapshots` | Allows creating, deleting, restoring, and querying snapshots and snapshot repositories. |
| `manage_point_in_time` | Allows creating and deleting Point-in-Time (PIT) contexts for consistent paginated searches. |
| `indices_monitor` | Cluster-level alias that grants monitoring permission across all indices (stats, segments, recovery). |

### 1.2 Administrative Permissions (`cluster:admin/*`)
Control cluster configuration, snapshots, ingest pipelines, scripts, templates, and plugin-specific operations.

| Permission | Description |
| :--- | :--- |
| `cluster:admin/settings/update` | Dynamically updates transient or persistent cluster settings via `_cluster/settings`. |
| `cluster:admin/reroute` | Manually triggers shard rerouting (allocate, cancel, move) via `_cluster/reroute`. |
| `cluster:admin/snapshot/create` | Initiates a snapshot of one or more indices to a registered repository. |
| `cluster:admin/snapshot/delete` | Deletes one or more snapshots from a repository. |
| `cluster:admin/snapshot/restore` | Restores indices from an existing snapshot in a repository. |
| `cluster:admin/snapshot/status` | Returns the status of an in-progress or recently completed snapshot. |
| `cluster:admin/snapshot/get` | Retrieves metadata and status for one or more snapshots. |
| `cluster:admin/repository/put` | Registers or updates a snapshot repository (S3, FS, GCS, Azure, etc.). |
| `cluster:admin/repository/get` | Retrieves information about one or more registered snapshot repositories. |
| `cluster:admin/repository/delete` | Removes a registered snapshot repository (does not delete existing snapshots). |
| `cluster:admin/repository/verify` | Verifies that a snapshot repository is accessible from all nodes. |
| `cluster:admin/ingest/pipeline/put` | Creates or updates an ingest pipeline used to pre-process documents before indexing. |
| `cluster:admin/ingest/pipeline/get` | Retrieves one or more ingest pipeline definitions. |
| `cluster:admin/ingest/pipeline/delete` | Deletes an ingest pipeline by name. |
| `cluster:admin/ingest/pipeline/simulate` | Tests an ingest pipeline against sample documents without indexing them. |
| `cluster:admin/script/put` | Stores a stored Painless (or other language) script cluster-wide. |
| `cluster:admin/script/get` | Retrieves a stored script by its ID. |
| `cluster:admin/script/delete` | Deletes a stored script by its ID. |
| `cluster:admin/script/context/get_action_context` | Returns the list of allowed fields/variables for a script context. |
| `cluster:admin/reindex/rethrottle` | Changes the requests-per-second throttle rate of a running reindex task. |
| `cluster:admin/nodes/reload_secure_settings` | Reloads encrypted keystore settings on nodes without a restart. |
| `cluster:admin/index_template/put` | Creates or updates an index template (legacy `_template` API). |
| `cluster:admin/index_template/get` | Retrieves one or more index templates. |
| `cluster:admin/index_template/delete` | Deletes an index template by name. |
| `cluster:admin/index_template/simulate` | Simulates index template application to determine which template would match a given index name. |
| `cluster:admin/component_template/put` | Creates or updates a component template (composable template building block). |
| `cluster:admin/component_template/get` | Retrieves one or more component templates. |
| `cluster:admin/component_template/delete` | Deletes a component template. |
| `cluster:admin/indices/dangling/list` | Lists dangling indices (indices known to a node but not part of cluster state). |
| `cluster:admin/indices/dangling/import` | Imports a dangling index back into the cluster state. |
| `cluster:admin/indices/dangling/delete` | Permanently deletes a dangling index. |
| `cluster:admin/opensearch/ql/datasources/read` | Reads query language data source configurations (OpenSearch SQL/PPL plugin). |
| `cluster:admin/opensearch/ql/datasources/write` | Creates or updates query language data source configurations. |
| `cluster:admin/opensearch/ql/datasources/delete` | Deletes a query language data source configuration. |
| `cluster:admin/opendistro/alerting/alerts/acknowledge` | Acknowledges one or more triggered alerts in the Alerting plugin. |
| `cluster:admin/opendistro/alerting/destination/write` | Creates or updates an alerting notification destination (Slack, webhook, etc.). |
| `cluster:admin/opendistro/alerting/destination/delete` | Deletes an alerting notification destination. |
| `cluster:admin/opendistro/alerting/monitor/write` | Creates or updates an alerting monitor definition. |
| `cluster:admin/opendistro/alerting/monitor/delete` | Deletes an alerting monitor by ID. |
| `cluster:admin/opendistro/alerting/monitor/execute` | Manually executes an alerting monitor for testing. |
| `cluster:admin/opendistro/alerting/monitor/read` | Reads alerting monitor definitions and history. |
| `cluster:admin/opendistro/reports/instance/list` | Lists report instances (generated report history) in the Reports plugin. |
| `cluster:admin/opendistro/reports/instance/get` | Retrieves a specific report instance by ID. |
| `cluster:admin/opendistro/reports/menu/download` | Downloads a generated report from the Reports plugin. |
| `cluster:admin/opendistro/reports/definition/create` | Creates a new report definition (schedule, template, format). |
| `cluster:admin/opendistro/reports/definition/update` | Updates an existing report definition. |
| `cluster:admin/opendistro/reports/definition/on_demand` | Triggers an on-demand report generation from a report definition. |
| `cluster:admin/opendistro/reports/definition/delete` | Deletes a report definition by ID. |
| `cluster:admin/opendistro/reports/definition/list` | Lists all report definitions in the cluster. |
| `cluster:admin/opendistro/reports/definition/get` | Retrieves a specific report definition by ID. |
| `cluster:admin/opensearch/observability/create` | Creates an Observability object (panel, visualization, notebook). |
| `cluster:admin/opensearch/observability/update` | Updates an existing Observability object. |
| `cluster:admin/opensearch/observability/delete` | Deletes an Observability object by ID. |
| `cluster:admin/opensearch/observability/get` | Retrieves one or more Observability objects. |

### 1.3 Cross-Index / Scroll Permissions
These `indices:*` permissions operate across multiple indices and **must** be placed in **Cluster Permissions**, not Index Permissions.

| Permission | Description |
| :--- | :--- |
| `indices:data/read/scroll` | Retrieves the next batch of results from an open scroll context. |
| `indices:data/read/scroll/clear` | Closes and releases resources held by an open scroll context. |
| `indices:data/read/mget` | Retrieves multiple documents by ID in a single request (cluster-level multi-get). |
| `indices:data/read/msearch` | Executes multiple search requests in a single HTTP call (multi-search). |
| `indices:data/read/msearch/template` | Executes multiple search template requests in a single HTTP call. |
| `indices:data/write/bulk` | Allows bulk indexing, update, and delete operations across one or more indices. |
| `indices:data/write/reindex` | Allows the `_reindex` API to copy documents from a source index to a destination. |
| `indices:admin/template/put` | Creates or updates a legacy index template via the `_template` API. |
| `indices:admin/template/get` | Retrieves one or more legacy index templates. |
| `indices:admin/template/delete` | Deletes a legacy index template by name. |
| `indices:admin/index_template/put` | Creates or updates a composable index template via the `_index_template` API. |
| `indices:admin/index_template/get` | Retrieves one or more composable index templates. |
| `indices:admin/index_template/simulate` | Simulates a composable index template to preview its settings/mappings. |

---

## SECTION 2 — INDEX PERMISSIONS

Applied in the **Index Permissions** field of a role (associated with a specific index pattern). These control operations on targeted indices.

### 2.1 Index-Level Action Groups
Shorthand names used in Index Permissions `allowed_actions` to apply groups of individual permissions at once.

| Action Group Name | What It Allows |
| :--- | :--- |
| `indices_all` | Full access to every index operation (alias for `indices:*` wildcard). |
| `read` | Allows searches, gets, mget, explain, field caps, msearch, scroll, and mapping reads on an index. |
| `write` | Allows indexing (create/update) documents and related bulk write operations on an index. |
| `delete` | Allows deleting documents by ID or via delete-by-query on an index. |
| `crud` | Combines `read` + `write` + `delete` — full document CRUD without admin-level operations. |
| `search` | Allows search, msearch, and scroll operations (subset of `read`). |
| `get` | Allows fetching individual documents by ID (`GET /{index}/_doc/{id}`). |
| `index` | Allows creating and updating (indexing) documents in an index. |
| `suggest` | Allows using the suggest/autocomplete endpoint on an index. |
| `manage` | Allows all index admin operations (open, close, refresh, flush, settings, aliases, mappings). |
| `monitor` | Allows read-only monitoring of an index (stats, segments, recovery, shard stores). |
| `create_index` | Allows creating a new index but not writing documents or changing settings. |
| `manage_aliases` | Allows adding, removing, and listing index aliases. |
| `view_index_metadata` | Allows reading index metadata (mappings, settings, aliases) without querying data. |
| `data_access` | Combined alias for `read` + `write` — data-plane access without administrative operations. |

### 2.2 Data Read Permissions (`indices:data/read/*`)
Allow users to retrieve documents, run searches, and read index data without modifying anything.

| Permission | Description |
| :--- | :--- |
| `indices:data/read/search` | Executes a search query against an index. |
| `indices:data/read/search/template` | Executes a stored search template against an index. |
| `indices:data/read/get` | Fetches a single document by its ID from an index. |
| `indices:data/read/mget` | Fetches multiple documents by ID from one or more indices in one request. |
| `indices:data/read/explain` | Returns an explanation of why a document did or did not match a query. |
| `indices:data/read/scroll` | Retrieves subsequent pages of scroll search results on this index. |
| `indices:data/read/scroll/clear` | Closes the scroll context associated with this index's scroll search. |
| `indices:data/read/field_caps` | Returns the capabilities (types, searchability) of fields across matching indices. |
| `indices:data/read/terms_enum` | Returns terms matching a prefix for autocomplete/suggestion use cases. |
| `indices:data/read/tv` | Retrieves term vectors (term frequency, positions) for a document. |
| `indices:data/read/mtv` | Retrieves term vectors for multiple documents in a single request. |
| `indices:data/read/percolate` | Checks which stored percolator queries match a given document. |
| `indices:data/read/pit_segments` | Retrieves low-level segment information within a Point-in-Time context. |

### 2.3 Data Write Permissions (`indices:data/write/*`)
Allow users to create, update, and delete documents within an index.

| Permission | Description |
| :--- | :--- |
| `indices:data/write/index` | Indexes (creates or replaces) a document in the index. |
| `indices:data/write/update` | Updates an existing document partially (using scripts or partial doc). |
| `indices:data/write/update/byquery` | Updates documents matching a query using the update-by-query API. |
| `indices:data/write/delete` | Deletes a document from the index by its ID. |
| `indices:data/write/delete/byquery` | Deletes all documents matching a query in the index. |
| `indices:data/write/bulk` | Executes bulk index, update, and delete operations on this index. |
| `indices:data/write/bulk[s]` | Shard-level component of a bulk request; typically internal/background use. |

### 2.4 Index Administration Permissions (`indices:admin/*`)
Allow managing the index itself — creation, deletion, settings, mappings, aliases, and operational actions.

| Permission | Description |
| :--- | :--- |
| `indices:admin/create` | Creates the index with optional settings and mappings. |
| `indices:admin/delete` | Permanently deletes the index and all its data. |
| `indices:admin/open` | Opens a closed index to make it available for reads and writes. |
| `indices:admin/close` | Closes the index to block access and reduce resource usage. |
| `indices:admin/get` | Retrieves index-level metadata (settings, mappings, aliases) for the index. |
| `indices:admin/exists` | Checks whether the index exists (`HEAD /{index}` endpoint). |
| `indices:admin/refresh` | Forces a refresh so recently indexed documents become searchable immediately. |
| `indices:admin/refresh[s]` | Shard-level refresh operation; typically internal/background use. |
| `indices:admin/flush` | Flushes the index (fsync transaction log and clear it) for durability. |
| `indices:admin/flush[s]` | Shard-level flush; typically internal. |
| `indices:admin/forcemerge` | Triggers a force merge (segment merge) to optimize storage and performance. |
| `indices:admin/cache/clear` | Clears caches (query, field data, request) for the index. |
| `indices:admin/shrink` | Shrinks the index into a new index with fewer primary shards. |
| `indices:admin/split` | Splits the index into a new index with more primary shards. |
| `indices:admin/clone` | Clones the index into a new index with the same number of shards. |
| `indices:admin/rollover` | Rolls over an alias to a new index when size/age/doc-count conditions are met. |
| `indices:admin/aliases` | Adds, removes, or updates index aliases. |
| `indices:admin/aliases/get` | Retrieves the aliases associated with the index. |
| `indices:admin/aliases/exists` | Checks whether a specific alias exists on the index. |
| `indices:admin/mappings/put` | Adds or updates field mappings on the index. |
| `indices:admin/mappings/get` | Retrieves the current field mappings for the index. |
| `indices:admin/mappings/fields/get` | Retrieves mappings for specific fields only. |
| `indices:admin/settings/update` | Updates index-level settings (e.g. replicas, refresh interval). |
| `indices:admin/settings/get` | Retrieves the current index settings. |
| `indices:admin/template/put` | Creates or updates an index template (legacy; per-index context). |
| `indices:admin/template/get` | Retrieves index template definitions. |
| `indices:admin/template/delete` | Deletes a legacy index template. |
| `indices:admin/validate/query` | Validates a query against the index mappings without executing it. |
| `indices:admin/upgrade` | Upgrades the index to the current Lucene format. |
| `indices:admin/analyze` | Runs the analysis chain on text to inspect tokenization results. |
| `indices:admin/seq_no/global_checkpoint_sync` | Syncs the global checkpoint across shards for sequence-number tracking. |
| `indices:admin/resolve/index` | Resolves a list of index names, aliases, and data streams to concrete indices. |
| `indices:admin/shard_stores` | Retrieves shard store information (allocation, store exceptions) for the index. |
| `indices:admin/synced_flush` | Performs a synced flush (deprecated in recent versions, used for fast restarts). |
| `indices:admin/block/add` | Adds a block to the index (e.g., read-only, no-new-delete). |
| `indices:admin/warmers/put` | Adds a warmer query to pre-warm the index cache (deprecated in newer versions). |

### 2.5 System Index Permissions (`system:admin/*`)
Special permissions required to access protected system indices used by OpenSearch plugins.

| Permission | Description |
| :--- | :--- |
| `system:admin/system_index` | Grants access to protected system indices (e.g., `.opendistro-alerting-config`); requires explicit grant. |
| `system:admin/close_recovery_context` | Internal action that closes recovery context after a primary promotion. |
| `system:admin/replication/*` | Wildcard covering all cross-cluster replication admin actions on the index. |