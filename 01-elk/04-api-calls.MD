# Elasticsearch / Kibana — Day-to-Day API Cheat Sheet (Troubleshooting & Validation)

A practical, ordered reference of the REST API calls you'll actually reach for day to day — checking cluster health, validating an index/mapping/query, and troubleshooting performance or allocation problems. Concepts behind these calls are explained in `01-elk-concepts.MD` and `02-elk-advanced-concepts.MD`; this file is just the "commands cheat sheet."

**How to run these:** paste them straight into Kibana's **Dev Tools → Console** (Management → Dev Tools) exactly as written below, or as `curl` if you prefer a terminal:

```bash
curl -u elastic:<ELASTIC_PASSWORD> -X GET "http://localhost:9200/_cluster/health?pretty"
```

The sections below are ordered the way you'd actually use them in a real troubleshooting session: **start broad (cluster) → narrow down (nodes → indices → shards → documents/queries) → performance → data-safety (snapshots/ILM) → security → Kibana-side**. A "common recipes" section at the end ties them together into real scenarios.

---

## 1. First Response — Is the Cluster Even Up?

```
GET /                       # basic connectivity + version info
GET /_cluster/health        # 🟢/🟡/🔴 status, node/shard counts — always your FIRST call
GET /_cat/health?v          # same info, one compact line
```

```json
GET /_cluster/health
```
Key fields to check first: `status` (green/yellow/red — see `01-elk-concepts.MD` §2.10), `number_of_nodes`, `unassigned_shards` (should be 0 on a healthy multi-node cluster), `active_shards_percent_as_number` (should be 100).

---

## 2. Cluster-Level Overview & Settings

```
GET /_cluster/health?level=indices          # health broken down per-index
GET /_cluster/health?level=shards           # health broken down per-shard (very verbose)
GET /_cluster/stats                         # aggregate stats: doc counts, storage size, JVM, OS across the whole cluster
GET /_cluster/settings?include_defaults=true&flat_settings=true   # every cluster setting, including defaults
GET /_cluster/pending_tasks                 # cluster-state changes queued but not yet applied (non-empty = master is busy/overloaded)
GET /_cluster/state/master_node             # who is the current elected master
```

Change a cluster setting (e.g. re-enable shard allocation after maintenance):

```json
PUT /_cluster/settings
{
  "persistent": {
    "cluster.routing.allocation.enable": "all"
  }
}
```

---

## 3. Node-Level Diagnostics

```
GET /_cat/nodes?v&h=name,node.role,master,heap.percent,ram.percent,cpu,disk.used_percent
                                              # one-line-per-node table — heap/disk/CPU at a glance
GET /_nodes                                  # full config of every node (roles, plugins, settings)
GET /_nodes/stats                            # live runtime stats per node (JVM, OS, fs, thread pools)
GET /_nodes/stats/jvm                        # just JVM heap/GC stats (narrow down when heap.percent is high)
GET /_nodes/stats/fs                         # just disk usage per node (narrow down on low disk warnings)
GET /_nodes/hot_threads                      # what every node's CPU is busy doing RIGHT NOW — #1 tool for "cluster is slow"
GET /_nodes/<node_id_or_name>/hot_threads    # hot threads for one specific node
```

`hot_threads` is usually the fastest way to answer "why is this node pegged at 100% CPU?" — it prints actual stack traces of the busiest threads.

---

## 4. Index-Level Diagnostics & Validation

```
GET /_cat/indices?v&s=store.size:desc        # every index: health, docs count, size — sorted biggest first
GET /_cat/indices?v&health=red               # ONLY show broken (red) indices — fast triage
GET /_cat/indices?v&health=yellow            # ONLY show under-replicated (yellow) indices
GET /<index>                                 # full definition: settings + mappings + aliases for one index
GET /<index>/_settings                       # just the settings (number_of_shards/replicas, etc.)
GET /<index>/_mapping                        # just the field mappings/types
GET /<index>/_stats                          # per-index stats: doc count, size, indexing/search rates, refresh/merge times
GET /_cat/aliases?v                          # which aliases point at which indices
GET /_cat/shards?v&index=<index>             # every shard of one index: node, primary/replica, size, state
```

Validate an index exists and check its exact shard/replica config before you rely on it:

```
GET /<index>/_settings/index.number_of_shards,index.number_of_replicas
```

Compare actual doc count vs. what you expect (quick sanity check after a bulk load):

```
GET /<index>/_count
```

---

## 5. Shard & Allocation Troubleshooting (Red/Yellow Cluster)

This is the order to run these when `_cluster/health` shows yellow or red:

```
GET /_cat/shards?v&h=index,shard,prirep,state,node,unassigned.reason
                                              # find exactly which shard is unassigned and why (state=UNASSIGNED)
GET /_cluster/allocation/explain             # ask ES to explain WHY the most recent unassigned shard can't be allocated
GET /_cat/allocation?v                       # how many shards + how much disk each node is using — spot a full/overloaded node
GET /_cat/recovery?v&active_only=true        # shards currently being recovered/relocated, and progress %
GET /_cat/thread_pool?v&h=node_name,name,active,queue,rejected
                                              # spot overloaded thread pools (high "rejected" count = requests being dropped, see §9)
```

`_cluster/allocation/explain` with no body explains the *first* unassigned shard it finds; to target one specifically:

```json
GET /_cluster/allocation/explain
{
  "index": "my-index",
  "shard": 0,
  "primary": true
}
```

Common root causes it will report: disk watermark exceeded, allocation filtering/awareness rules, too many shards on one node, or "waiting for a node with the primary's data to rejoin."

---

## 6. Document-Level CRUD & Validation

```
GET  /<index>/_doc/<id>              # fetch one document by ID
HEAD /<index>/_doc/<id>              # check existence only (200/404), no body returned — cheap existence check
POST /<index>/_doc                   # index (create) a document, auto-generated ID
PUT  /<index>/_doc/<id>              # index (create/replace) a document with a specific ID
POST /<index>/_update/<id>           # partial update (merge fields) instead of full replace
DELETE /<index>/_doc/<id>            # delete one document
POST /<index>/_bulk                  # index/update/delete MANY documents in one request (always prefer this over one-at-a-time)
```

Example bulk request (note the required trailing newline):

```json
POST /_bulk
{ "index": { "_index": "products", "_id": "1" } }
{ "name": "Laptop", "price": 999 }
{ "index": { "_index": "products", "_id": "2" } }
{ "name": "Mouse", "price": 25 }
```

---

## 7. Search & Query Debugging / Validation

```
GET /<index>/_search                         # basic search
GET /<index>/_search?q=field:value            # quick "URI search" for one-off ad-hoc checks
POST /<index>/_validate/query?explain=true    # check a query is syntactically/semantically VALID without running it
POST /<index>/_search?explain=true            # for each hit, explain the relevance SCORE calculation
POST /<index>/_search?profile=true            # low-level timing breakdown of every phase of the query (find slow queries)
GET  /<index>/_field_caps?fields=*            # discover a field's type across one or many indices (mapping conflicts)
GET  /_resolve/index/<pattern>                # check which real indices/aliases/data streams a wildcard pattern resolves to
```

Validate a query is well-formed before running it against production data:

```json
POST /my-index/_validate/query?explain=true
{
  "query": {
    "match": { "status": "error" }
  }
}
```

Find out why a document *didn't* match a query you expected it to match — use `_explain` on that specific document:

```
GET /<index>/_explain/<id>
{
  "query": { "match": { "status": "error" } }
}
```

---

## 8. Performance Troubleshooting

```
GET /_nodes/hot_threads                       # (repeated from §3 — the single most useful "why is it slow" command)
GET /_cat/thread_pool?v&h=node_name,name,active,queue,rejected,size
                                              # search/write/get thread pools — rejected > 0 means clients are getting 429s
GET /_nodes/stats/breaker                     # circuit breaker stats — tripped breakers mean requests are being rejected to avoid OOM
GET /_cat/pending_tasks?v                     # cluster-state tasks queued on the master (long list = master overloaded)
GET /_tasks?actions=*search*&detailed=true    # list all currently running search tasks, with description
GET /_tasks?actions=*bulk*&detailed=true      # list all currently running bulk/indexing tasks
POST /_tasks/<task_id>/_cancel                # cancel a specific long-running/stuck task (needs the task to support cancellation)
GET /<index>/_segments                        # segment count/size per shard — many tiny segments = needs a force-merge
POST /<index>/_forcemerge?max_num_segments=1  # merge segments down (only for read-only/older indices — expensive on live indices!)
```

---

## 9. Reindex, Update-By-Query, Delete-By-Query (bulk maintenance)

```json
POST /_reindex
{
  "source": { "index": "old-index" },
  "dest":   { "index": "new-index" }
}
```

```json
POST /<index>/_update_by_query
{
  "query": { "term": { "status": "pending" } },
  "script": { "source": "ctx._source.status = 'processed'" }
}
```

```json
POST /<index>/_delete_by_query
{
  "query": { "range": { "created_at": { "lt": "now-90d/d" } } }
}
```

Check progress of a long-running reindex/update/delete-by-query task (all three return a `task` id when run with `?wait_for_completion=false`):

```
POST /_reindex?wait_for_completion=false
GET  /_tasks/<task_id>
```

---

## 10. Snapshot & Restore Validation (Backups)

```
GET  /_snapshot                               # list registered repositories
GET  /_snapshot/<repo>                        # details of one repository
GET  /_snapshot/<repo>/_all                   # list all snapshots in a repository
GET  /_snapshot/<repo>/<snapshot>/_status     # live progress of a snapshot currently being taken
POST /_snapshot/<repo>/<snapshot>/_verify     # verify a repository is correctly configured/reachable from every node
```

Take a manual snapshot and confirm it succeeded:

```json
PUT /_snapshot/my_repo/manual-snapshot-2026-08-02
{
  "indices": "*",
  "include_global_state": false
}
```
```
GET /_snapshot/my_repo/manual-snapshot-2026-08-02
```
Check the `state` field is `SUCCESS` (not `PARTIAL` or `FAILED`) before trusting a backup.

---

## 11. ILM (Index Lifecycle Management) Validation

```
GET /_ilm/policy                              # list all ILM policies
GET /_ilm/policy/<policy_name>                # one policy's phase definitions
GET /<index>/_ilm/explain                     # WHERE is this index in its lifecycle right now, and why (great for "why hasn't it rolled over yet")
POST /<index>/_ilm/retry                      # retry ILM on an index stuck in an ERROR step
GET /_ilm/status                              # is the ILM service itself running (RUNNING/STOPPED)
```

---

## 12. Security / User & Role Validation

```
GET /_security/user                           # list all users
GET /_security/user/<username>                # one user's roles
GET /_security/role                           # list all roles
GET /_security/role/<role_name>                # one role's exact permissions
GET /_security/_authenticate                  # "who am I?" — validate the credentials you're currently using
POST /_security/user/<username>/_password     # reset a user's password
```

Check what permissions the *current* credentials actually have (useful when debugging a 403):

```
GET /_security/_authenticate
```

Simulate whether a role/user CAN do a specific action, without actually doing it:

```json
POST /_security/user/_has_privileges
{
  "index": [
    { "names": ["my-index"], "privileges": ["read", "write"] }
  ]
}
```

---

## 13. Kibana-Side APIs

```
GET /api/status                               # Kibana's own health (green/yellow/red) — run against :5601, not :9200
GET /api/saved_objects/_find?type=index-pattern   # list all Data Views (formerly "index patterns")
GET /api/saved_objects/_find?type=dashboard       # list all saved dashboards
POST /api/saved_objects/_export               # export saved objects (dashboards/visualizations/data views) as NDJSON
POST /api/saved_objects/_import               # import saved objects from NDJSON
GET /api/spaces/space                         # list all Kibana Spaces
```

Kibana API calls need the same basic-auth credentials as Elasticsearch, plus the `kbn-xsrf` header when writing:

```bash
curl -u elastic:<ELASTIC_PASSWORD> -H "kbn-xsrf: true" -X POST "http://localhost:5601/api/saved_objects/_export" \
  -H "Content-Type: application/json" -d '{"type": "dashboard"}'
```

---

## 14. `_cat` API Quick Reference (human-readable, one line per item)

The `_cat` APIs are the fastest way to eyeball cluster state without parsing JSON — append `?v` for column headers and `?help` on any of them to see all available columns.

| Command | Shows |
|---|---|
| `GET /_cat/health?v` | One-line cluster health |
| `GET /_cat/nodes?v` | Nodes, roles, resource usage |
| `GET /_cat/indices?v` | Indices, health, doc counts, size |
| `GET /_cat/shards?v` | Every shard, its node, and state |
| `GET /_cat/allocation?v` | Disk/shard usage per node |
| `GET /_cat/aliases?v` | Alias → index mappings |
| `GET /_cat/segments?v` | Lucene segment counts/sizes |
| `GET /_cat/recovery?v` | Shard recovery/relocation progress |
| `GET /_cat/thread_pool?v` | Thread pool queue/rejection counts |
| `GET /_cat/pending_tasks?v` | Queued cluster-state changes |
| `GET /_cat/repositories?v` | Registered snapshot repositories |
| `GET /_cat/snapshots/<repo>?v` | Snapshots in a repository |
| `GET /_cat/templates?v` | Index templates |
| `GET /_cat/plugins?v` | Installed plugins per node |
| `GET /_cat/master?v` | Current elected master node |

---

## 15. Common Troubleshooting Recipes (putting it together)

**"Cluster health is red"**
1. `GET /_cat/indices?v&health=red` → find the broken index.
2. `GET /_cat/shards?v&index=<index>` → find the `UNASSIGNED` primary shard.
3. `GET /_cluster/allocation/explain` → read the reason (often: node holding the only copy is down/lost).
4. If the node is truly gone and no replica existed, restore from a snapshot (§10).

**"Cluster health is yellow"**
1. `GET /_cat/shards?v&health=yellow` (or filter indices) → find shards with `state=UNASSIGNED` where `prirep=r` (replica, not primary).
2. `GET /_cluster/allocation/explain` → usually "not enough nodes to place the replica" (normal/expected on a single-node lab) or a disk watermark issue.
3. If it's a real cluster: `GET /_cat/allocation?v` to check for a node stuck above the disk watermark.

**"A node/cluster feels slow"**
1. `GET /_nodes/hot_threads` → see what's actually consuming CPU.
2. `GET /_cat/thread_pool?v` → check for high `queue`/`rejected` counts (overload) or a maxed-out pool.
3. `GET /_nodes/stats/jvm` → check heap usage % and GC time (heap constantly near 100% = undersized JVM heap or a memory leak from oversized aggregations).
4. `POST /<index>/_search?profile=true` on the specific slow query → find which part of the query is expensive.

**"Disk usage warnings / read-only-allow-delete blocks writes"**
1. `GET /_cat/allocation?v` → confirm which node crossed the disk watermark.
2. Free up space (delete old indices via ILM, §11) or add capacity.
3. Once space is freed, remove the auto-applied write block:

```json
PUT /<index>/_settings
{
  "index.blocks.read_only_allow_delete": null
}
```

**"My data isn't showing up in Kibana Discover"**
1. `GET /<index>/_count` → confirm the documents actually exist in Elasticsearch.
2. `GET /_resolve/index/<data-view-pattern>` → confirm the Data View's index pattern actually matches your index name.
3. Check the Data View's configured time field and the Discover time-range picker — most "missing data" issues are just a mismatched time filter.

**"Getting a 403 Forbidden"**
1. `GET /_security/_authenticate` → confirm which user/roles you're actually authenticated as.
2. `POST /_security/user/_has_privileges` → confirm that role truly has the privilege you expect on that index/action.
