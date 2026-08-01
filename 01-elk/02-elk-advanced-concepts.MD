# Elastic Stack — Advanced Concepts (High Availability, Multi-Cluster, Production Hardening)

This continues from `01-elk-concepts.MD`. That file covers the fundamentals of a single cluster (nodes, shards, replicas, ILM). This file covers what happens when you need that cluster to survive real failures: a lost node, a lost data center, or a lost region — in simple terms.

---

## 1. Making a Single Cluster Highly Available (before you even think about multiple clusters)

### 1.1 Always use an odd number of master-eligible nodes (quorum)

A cluster elects **one master** from its master-eligible nodes. To agree on who the master is without a "split brain" (two nodes both thinking they're master), Elasticsearch uses **quorum voting** — more than half the master-eligible nodes must agree.

- 3 master-eligible nodes → needs 2 votes to elect a master → tolerates **1** master node dying.
- 5 master-eligible nodes → needs 3 votes → tolerates **2** master nodes dying.
- 2 master-eligible nodes is dangerous — if the link between them breaks, neither has a majority, and the cluster can't elect anyone.

**Simple rule: always run 3 (or 5) dedicated master-eligible nodes in production, never 2 or an even number.**

### 1.2 Dedicated master nodes vs. "do everything" nodes

In small/dev clusters (like the labs in this repo), every node does master + data + ingest. In a highly-available production cluster you separate the jobs:

| Node type | Count (typical) | Job |
|---|---|---|
| Dedicated master-eligible | 3 | Only manages cluster state/elections — lightweight but critical, shouldn't be starved by heavy search traffic |
| Data nodes | many | Store shards, handle search/index load |
| Coordinating-only nodes | a few | Absorb client traffic, shield data nodes |
| Ingest nodes | a few (optional) | Pre-process documents |

**Why separate them?** If a master node is also busy serving huge search queries, it might be too slow to respond during an election, causing instability. Isolating masters keeps cluster "brain" functions fast and reliable.

### 1.3 Zone / rack awareness — don't put all copies of a shard in one place

Even with replicas, if a primary shard and its replica happen to live in the **same rack/data-center/availability-zone**, a single power outage can take out both copies at once.

**Shard allocation awareness** tells Elasticsearch "these nodes are in zone A, these are in zone B" (via a node attribute like `node.attr.zone: zone-a`). Elasticsearch then automatically avoids placing a primary and its replica in the same zone.

```yaml
node.attr.zone: zone-a
cluster.routing.allocation.awareness.attributes: zone
```

Analogy: it's like insisting that the master copy and the backup copy of an important document are never stored in the same building.

### 1.4 Handling a lost node automatically (self-healing)

If a data node dies:
1. The master notices (via heartbeats/pings) it stopped responding.
2. Any **replica shards** that were on healthy nodes get **promoted to primary** for the shards that were lost.
3. The master schedules **new replicas** to be created on the remaining nodes to restore full redundancy.
4. Cluster health goes Green → Yellow (while replicas are missing) → Green again once re-replicated.

This is why replicas aren't just "nice to have" — they're what makes a cluster self-healing instead of losing data.

---

## 2. Beyond One Cluster: Primary/Secondary (Active/Passive) Setup

This is a common production pattern: **one "primary" ELK deployment that's always active, and a "secondary" deployment that quietly stays in sync, ready to take over.** In Elastic terminology this is built using **Cross-Cluster Replication (CCR)**, and it's exactly how managed "ELK-as-a-Service" offerings (Elastic Cloud, ECE, ECK) implement disaster recovery between regions.

### 2.1 Cross-Cluster Replication (CCR) — Leader / Follower indices

- **Leader index** — lives on the **primary** cluster. This is the one apps write to normally.
- **Follower index** — lives on the **secondary** cluster. It's a **read-only**, continuously-updated copy of the leader index. Elasticsearch ships new documents from leader → follower in near real-time, automatically.

```
 [Primary Cluster - Region A]              [Secondary Cluster - Region B]
        (ACTIVE - accepts writes)                 (STANDBY - read-only)

   Index: logs-2026.08 (LEADER)   ───ships new docs──▶   logs-2026.08 (FOLLOWER)
        ▲
        │ writes from apps/Beats/Logstash
```

Key simple facts:
- You can only **write** to the leader index. The follower index rejects direct writes — it only accepts data replicated from its leader.
- Replication is **near real-time**, not instant — there's a small lag (usually seconds).
- This is set up per-index (or automatically for all matching indices via an **auto-follow pattern**, useful for daily log indices like `logs-*`).

### 2.2 "Primary always active, secondary has follower indices" — what this actually means

This describes an **active-passive disaster recovery (DR)** architecture:

| | Primary cluster | Secondary cluster |
|---|---|---|
| Role | **Active** — serves all reads and writes | **Passive/Standby** — only receives replicated data |
| Indices | Normal (leader) indices | Follower indices (read-only copies) |
| Used for | Normal day-to-day traffic | Sits idle until needed; can serve read-only reports |
| On disaster | N/A | Gets **promoted**: follower indices are converted to normal writable indices, apps get redirected here |

**Promotion / failover** — if the primary region goes down entirely:
1. You stop replication and "promote" the follower indices on the secondary cluster into regular writable indices (a manual or scripted API call — this is **not automatic** by default, since automatically promoting during a network blip could cause data conflicts).
2. You redirect your applications/Beats/Logstash to point at the secondary cluster.
3. The secondary is now the new "primary" until the original region recovers.

This is different from replica shards (§1.4) in an important way:

| | Replica shards | CCR follower indices |
|---|---|---|
| Scope | Inside **one** cluster | Across **two separate, independent clusters** (can be different data centers/regions/cloud providers) |
| Purpose | Survive a **node** failure | Survive a **cluster/data-center/region** failure |
| Failover | Automatic | Usually manual/controlled ("promote") |

### 2.3 Cross-Cluster Search (CCS) — the "read without copying" alternative

Sometimes you don't want to duplicate data at all — you just want to **query multiple clusters from one place**. **Cross-Cluster Search** lets one Kibana/client send a single search that fans out to several independent clusters and merges the results, without replicating any data.

```
Kibana ──▶ searches across: cluster-eu-* AND cluster-us-*  ──▶ merged results
```

Simple distinction:
- **CCR** = copy the data so a secondary cluster has its own local copy (for DR/failover).
- **CCS** = don't copy anything, just search remotely across clusters (for a "single pane of glass" view over geographically split data).

### 2.4 Active-Active (both clusters serving writes)

Some setups go further: **both** clusters accept writes to *different* indices (e.g., Region A owns `orders-eu-*`, Region B owns `orders-us-*`), and each cluster follows the other's indices for a unified read view via CCR + CCS. This maximizes availability (no single active region) but is more complex — you must avoid ever writing the *same* document in two places at once, since Elasticsearch's CCR isn't a conflict-resolving multi-master system like some databases.

---

## 3. Backups: Snapshot & Restore, and Snapshot Lifecycle Management (SLM)

CCR protects against a cluster going down, but not against **human error** (someone deletes an index) or **corruption**. That's what backups are for.

- A **snapshot** is a point-in-time backup of one or more indices, stored in a **repository** (commonly an S3 bucket, Azure Blob, GCS, or shared filesystem).
- Snapshots are **incremental** — after the first full snapshot, later snapshots only store what changed, so they're fast and space-efficient.
- **SLM (Snapshot Lifecycle Management)** automates this: "take a snapshot every night at 1am, keep the last 30 days, delete older ones" — configured once, runs forever.

```
PUT /_snapshot/my_backup_repo/snapshot_2026_08_02
```

**Restoring** a snapshot recreates the indices exactly as they were — this is your last line of defense, separate from both replicas (node failure) and CCR (region failure).

---

## 4. Rolling Upgrades & Zero-Downtime Maintenance

Because a cluster tolerates losing nodes (§1.4), you can upgrade Elasticsearch **one node at a time** without ever taking the whole cluster offline:

1. Disable shard allocation temporarily (so shards don't try to rebalance while you're mid-restart).
2. Stop one node, upgrade its software, restart it.
3. Wait for it to rejoin and shards to reallocate back (cluster health returns to green).
4. Repeat for the next node.

This is called a **rolling upgrade**, and it's only possible because of the redundancy (replicas + multiple master-eligible nodes) described above — a single-node cluster (like the lab in this repo) always has downtime during an upgrade.

---

## 5. Protecting the Cluster from Overload

### 5.1 Circuit breakers
Elasticsearch tracks memory usage per-request and per-node, and will **reject** a request ("circuit breaker tripped") rather than let it crash the whole node with an out-of-memory error. Think of it like an electrical circuit breaker — it trips to prevent a bigger failure.

### 5.2 Thread pools
Different kinds of work (search, indexing (write), get) each have their own dedicated **thread pool** with a fixed size and queue. If too many search requests pile up, new ones get rejected quickly (`429 Too Many Requests`) instead of queuing forever and making the node unresponsive.

### 5.3 Backpressure on indexing
When writing huge volumes of data, clients should send documents in **bulk requests** (many documents per HTTP call) rather than one at a time — this is dramatically faster and reduces overhead. Well-behaved ingestion (Logstash, Beats, ingest pipelines) all batch writes into bulk requests automatically.

---

## 6. Advanced Security (beyond the basics in file 1)

- **API keys** — short-lived or scoped credentials for applications, instead of using a real user's username/password.
- **Field-level & document-level security** — restrict a role so it can only see *certain fields* of a document, or only documents matching *certain conditions* (e.g., a support agent role can only see tickets for their own team).
- **Audit logging** — records who did what (logins, searches, index changes) for compliance.
- **Role mapping with external identity providers** — connect to LDAP/Active Directory/SAML/OIDC so users log in with existing corporate accounts instead of Elasticsearch-native users.

---

## 7. Managed / Cloud Deployment Models ("ELK-as-a-Service")

This is running Elasticsearch as a managed service rather than self-hosting:

| Option | What it is |
|---|---|
| **Elastic Cloud** | Elastic's own fully-managed SaaS offering — you click a few buttons and get a cluster with monitoring, snapshots, and upgrades handled for you. |
| **ECE (Elastic Cloud Enterprise)** | Run "your own Elastic Cloud" inside your company's infrastructure — same automation, self-hosted. |
| **ECK (Elastic Cloud on Kubernetes)** | An operator that manages Elasticsearch/Kibana clusters as Kubernetes resources (`kind: Elasticsearch`), automating scaling, upgrades, and TLS certs declaratively via YAML. |

All three support **cross-cluster replication between deployments** — which is exactly the "primary always active / secondary with follower indices" pattern from §2, typically set up as: primary deployment in one region, secondary deployment in another region, linked via CCR, with the secondary promoted only if the primary region has an outage.

They also commonly offer **autoscaling** — automatically adding data nodes/storage when disk usage or load crosses a threshold, instead of manually resizing the cluster.

---

## 8. Quick Summary: Layers of "High Availability"

Think of HA as an onion with layers, each protecting against a bigger kind of failure:

| Layer | Protects against | Mechanism |
|---|---|---|
| 1. Replica shards | A single **node** dying | Automatic failover within one cluster (file 1, §2.9) |
| 2. Odd-numbered master-eligible nodes | **Master election** failing / split-brain | Quorum voting (§1.1) |
| 3. Zone/rack awareness | A whole **rack/AZ** failing | Spread primaries & replicas across zones (§1.3) |
| 4. Snapshots (SLM) | **Human error** / data corruption | Point-in-time backups to a repository (§3) |
| 5. Cross-Cluster Replication | A whole **cluster/region/data-center** failing | Primary (active) + Secondary (standby, follower indices), manual promotion on disaster (§2) |
| 6. Cross-Cluster Search | Needing a **unified view** across regions without merging data | Federated search, no data copied (§2.3) |

---

## 9. Glossary (Advanced Terms)

| Term | One-line definition |
|---|---|
| **Quorum** | Majority vote among master-eligible nodes needed to elect a master and avoid split-brain. |
| **Split-brain** | Dangerous state where two nodes both believe they are master, usually due to network partition with insufficient quorum protection. |
| **Shard allocation awareness** | Config that stops Elasticsearch placing a primary and its replica in the same zone/rack. |
| **CCR (Cross-Cluster Replication)** | Continuously replicates a leader index (one cluster) to a follower index (another cluster). |
| **Leader index** | The writable, source-of-truth index in a CCR setup (lives on the primary/active cluster). |
| **Follower index** | The read-only, auto-updated copy in a CCR setup (lives on the secondary/standby cluster). |
| **Auto-follow pattern** | A CCR rule that automatically sets up following for any new index matching a name pattern (e.g., daily log indices). |
| **Promotion (CCR)** | The manual/scripted act of converting a follower index into a normal writable index during failover. |
| **CCS (Cross-Cluster Search)** | Querying multiple independent clusters in one request without copying data. |
| **Snapshot** | An incremental, point-in-time backup of indices stored in a repository (S3, GCS, Azure, filesystem). |
| **SLM (Snapshot Lifecycle Management)** | Automates scheduled snapshot creation and retention/cleanup. |
| **Rolling upgrade** | Upgrading cluster nodes one at a time with zero overall downtime. |
| **Circuit breaker** | Safety mechanism that rejects a request rather than risk an out-of-memory crash. |
| **Thread pool** | Fixed-size worker pool per operation type (search/index/get) that controls concurrency and applies backpressure. |
| **API key** | Scoped, revocable credential for application access, alternative to username/password. |
| **Field/document-level security** | Restricts a role to only certain fields or only documents matching a filter. |
| **ECE / ECK** | Elastic's self-managed "cloud-like" platforms for running Elastic Stack on-prem or on Kubernetes. |
| **Active-passive DR** | One cluster serves live traffic (active); another stays synced via CCR, promoted only on disaster (passive). |
| **Active-active** | Multiple clusters each accept writes to their own indices and replicate/search across each other. |

---

## 10. Suggested Next Steps to See This in Practice

1. In the existing multi-node lab (`02-elastic-lab-multiple-nodes`), inspect `node.roles` and try setting `node.attr.zone` on each container to simulate zone awareness, then check `GET /_cat/shards?v` to see allocation change.
2. Spin up a **second** separate cluster (e.g., copy `02-elastic-lab-multiple-nodes` into a new folder with a different `cluster.name` and ports) and experiment with setting up CCR between them via Kibana's **Stack Management → Cross-Cluster Replication** UI or the `_ccr` API.
3. Try `PUT /_snapshot/my_repo` with a shared filesystem repository, take a manual snapshot, delete an index, then restore it — to feel the difference between "replica-level" and "backup-level" protection.
