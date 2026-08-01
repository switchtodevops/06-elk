# Elastic Stack (ELK) Concepts — Beginner to Advanced

This guide explains **every core concept** of the Elastic Stack (Elasticsearch, Logstash, Kibana, Beats) in plain, simple language. It's written so that a complete beginner can read top to bottom and come away understanding how the whole system works — nodes, indices, shards, mappings, and every major Kibana component.

Wherever helpful, the examples refer to the two labs in this repo:
- `02-elastic-lab-multiple-nodes` — a **3-node cluster** (`es01`, `es02`, `es03`)
- `03-elastic-lab-single-node` — a **single-node cluster**

---

## 1. The Big Picture: What is "ELK" / "Elastic Stack"?

**ELK** originally stood for **Elasticsearch + Logstash + Kibana**. Today it's usually called the **Elastic Stack**, and often includes a fourth piece, **Beats**:

| Component | Simple explanation | Analogy |
|---|---|---|
| **Elasticsearch** | The database/search engine. Stores your data and lets you search/analyze it instantly, even across billions of records. | The "warehouse + super-fast librarian" that stores books and can find any sentence in any book instantly. |
| **Logstash** | A data pipeline tool. Collects data from many sources, transforms/cleans it, and sends it to Elasticsearch. | The "conveyor belt + sorting machine" that processes raw materials before they go into the warehouse. |
| **Kibana** | The web UI. Lets humans search, visualize, and build dashboards on top of the data stored in Elasticsearch. | The "showroom / control panel" where you look at what's in the warehouse. |
| **Beats** | Lightweight agents installed on servers to ship logs/metrics (e.g., Filebeat for logs, Metricbeat for metrics) into Logstash or Elasticsearch. | The "delivery trucks" that continuously bring new goods to the warehouse. |

**Typical flow of data:**

```
[Servers/Apps/Logs] → Beats → Logstash (optional) → Elasticsearch → Kibana (you view it here)
```

You don't always need all four. A minimal setup (like the labs in this repo) is just **Elasticsearch + Kibana**: Elasticsearch stores and searches the data, Kibana visualizes it.

---

## 2. Elasticsearch Fundamentals

### 2.1 What is Elasticsearch?

Elasticsearch is a **distributed, document-oriented search and analytics engine**. Let's unpack each word:

- **Distributed** — it can run across many machines (nodes) instead of just one, for speed and reliability.
- **Document-oriented** — it stores data as **JSON documents** (like a record), not as rows/columns in a traditional relational table.
- **Search and analytics engine** — it's built from the ground up to search text and numbers extremely fast, and to aggregate/summarize large amounts of data (counts, averages, histograms, etc.).

### 2.2 Document — the basic unit of data

A **document** is a single JSON object — the smallest piece of data you store. It's similar to a "row" in a SQL database, but more flexible (fields can be nested, arrays, missing, etc.).

```json
{
  "name": "John Doe",
  "age": 30,
  "city": "Berlin",
  "tags": ["customer", "vip"]
}
```

Each document has a unique **`_id`** and belongs to exactly one **index**.

### 2.3 Index — where documents live

An **index** is a collection of related documents — similar to a "table" in a relational database, or a "folder" that groups similar files.

Examples: `products`, `logs-2026.08`, `customers`.

```
Index "products"
 ├── Document 1: { "name": "Laptop", "price": 999 }
 ├── Document 2: { "name": "Mouse",  "price": 25  }
 └── Document 3: { "name": "Keyboard", "price": 45 }
```

> In older Elasticsearch versions, an index could contain multiple "types" (like sub-tables). Since Elasticsearch 7.x, **each index has only one implicit type (`_doc`)** — this "types" concept is deprecated/removed. You just think in terms of indices now.

### 2.4 Mapping — the schema of an index

A **mapping** defines what fields a document has and their **data types** (text, keyword, integer, date, boolean, geo_point, etc.). It's similar to a table schema in SQL, but:

- Elasticsearch can **auto-detect** mappings if you don't define them (called "dynamic mapping").
- You *can* define an **explicit mapping** up front for control and correctness (recommended for production).

```json
PUT /products
{
  "mappings": {
    "properties": {
      "name":  { "type": "text" },
      "price": { "type": "float" },
      "in_stock": { "type": "boolean" },
      "created_at": { "type": "date" }
    }
  }
}
```

Two important field types beginners confuse:

| Type | Use for | Behavior |
|---|---|---|
| `text` | Full-text search (e.g., product descriptions) | Gets broken into individual words ("analyzed") so you can search partial matches. |
| `keyword` | Exact values (e.g., status codes, IDs, tags, emails) | Stored as-is, used for exact matching, filtering, sorting, and aggregations. |

### 2.5 The Inverted Index — how Elasticsearch searches so fast

This is the "magic" behind Elasticsearch's speed. Instead of scanning every document to find a word, Elasticsearch builds an **inverted index**: a map from each word → list of documents that contain it (like the index at the back of a textbook).

```
Word        → Documents containing it
"laptop"    → [doc1, doc5, doc9]
"wireless"  → [doc2, doc5]
"keyboard"  → [doc3, doc5]
```

So searching for "laptop" is an instant lookup instead of reading every document. This is why Elasticsearch is dramatically faster than a plain database for text search.

### 2.6 Cluster — a group of nodes working together

A **cluster** is a collection of one or more Elasticsearch servers (nodes) that together hold your entire data set and provide search/indexing across all of it. All nodes in a cluster share the same **cluster name**.

In this repo:
- `02-elastic-lab-multiple-nodes`: `CLUSTER_NAME=elastic-wsl-cluster`, made of 3 nodes: `es01`, `es02`, `es03`.
- `03-elastic-lab-single-node`: `cluster.name: elastic-wsl-cluster`, made of a single node.

A cluster gives you two big benefits over a single server:
1. **Scalability** — spread data and load across many machines.
2. **High availability** — if one node dies, the cluster keeps working using the surviving nodes.

### 2.7 Node — a single Elasticsearch server

A **node** is one running instance of Elasticsearch (usually one process/container/VM). Every node:
- Has a unique `node.name` (e.g., `es01`, `es02`, `es03`).
- Joins a cluster using the same `cluster.name`.
- Stores data and/or helps coordinate/manage the cluster, depending on its **roles**.

### 2.8 Node Roles — what job each node performs

A node can perform one or more **roles**. In small setups (like our labs), a single node often does *everything*. In large production clusters, roles are usually **split across dedicated nodes** for performance and stability.

| Role | Config value | What it does | Simple analogy |
|---|---|---|---|
| **Master-eligible** | `master` | Can be elected as the **master node**. Manages cluster-wide "housekeeping": creating/deleting indices, tracking which nodes are in the cluster, deciding where shards live. Does **not** handle search/index requests for your data itself. | The **manager** of a warehouse — plans where things go, but doesn't carry boxes. |
| **Data** | `data` | Stores the actual documents/shards and executes search & indexing operations on them. This is the workhorse role. | The **warehouse worker** who actually stores and retrieves boxes. |
| **Ingest** | `ingest` | Pre-processes documents before indexing (e.g., parse a log line, extract fields, rename fields) using "ingest pipelines" — similar to a lightweight Logstash inside Elasticsearch. | The **quality-control station** on the conveyor belt before boxes reach the shelf. |
| **Coordinating (only)** | *(a node with no other roles)* | Routes client requests, collects results from data nodes, merges them, and returns the final response to the user. Every node does some coordinating work, but a "coordinating-only" node has *no other role* and exists purely to load-balance requests. | The **front-desk receptionist** who takes your request, asks the right departments, and hands you the combined answer. |
| **Machine Learning (`ml`)** | `ml` | Runs anomaly detection and other ML jobs. | The **data analyst** spotting unusual patterns. |
| **Remote Cluster Client** | `remote_cluster_client` | Allows this node to connect to other (remote) clusters, needed for cross-cluster search/replication. | The **liaison officer** who talks to other warehouses. |
| **Transform** | `transform` | Runs continuous/batch data transform jobs (turns raw data into summarized indices). | The **report generator**. |
| **Voting-only** | `voting_only` | Master-eligible but can never actually become master — only helps with elections (used to keep an odd number of voters without adding full master resource cost). | A **tie-breaker voter** who never becomes chairperson. |

**In this repo's multi-node lab**, every node is configured with:
```yaml
node.roles: master,data,ingest,remote_cluster_client
```
This means `es01`, `es02`, and `es03` are all "do everything" nodes — common for small clusters/dev/labs. In real production clusters with many nodes, you would typically dedicate:
- 3 nodes as **master-eligible only** (for stability — master duties are lightweight but critical),
- Many nodes as **data** nodes (for storage/search capacity),
- A few **coordinating-only** nodes (to shield data nodes from client traffic spikes),
- Optionally dedicated **ingest** or **ml** nodes.

#### Master node vs. Master-eligible node (important distinction)

- **Master-eligible node**: *can* become the master (has `master` role).
- **The master node**: the *one* master-eligible node that is currently **elected** to do the job, at any given time.

Only **one node is the active master** at a time in a healthy cluster. If it fails, the remaining master-eligible nodes hold an **election** and pick a new master automatically — this is what keeps the cluster self-healing.

What the master node actually does:
- Tracks cluster **state**: which indices exist, their mappings/settings, which nodes are alive.
- Decides **shard allocation** — which node should hold which shard.
- Does **not** handle your search/index requests directly — that's the data nodes' job. This is why the master role is lightweight and shouldn't be overloaded with data duties in large clusters.

#### Data node — deeper look

A **data node** holds actual shards (data) and does the heavy lifting:
- Executing CRUD (create/read/update/delete) operations.
- Executing search queries and aggregations on the shards it holds.
- This is the role that needs the most CPU, RAM, and disk.

You can even sub-specialize data nodes by **data tier** (introduced for time-series/log use cases and Index Lifecycle Management):

| Data tier | Role name | Purpose |
|---|---|---|
| Hot | `data_hot` | Newest, most frequently queried/written data. Fastest (often SSD) hardware. |
| Warm | `data_warm` | Older data, queried less often, no longer written to. |
| Cold | `data_cold` | Rarely accessed, kept for compliance/history, can be more compressed. |
| Frozen | `data_frozen` | Extremely rarely accessed, cheapest storage (can be backed by object storage like S3). |

This "hot-warm-cold-frozen" architecture is common for log/metrics use cases (typical ELK use case) to control cost as data ages.

### 2.9 Shards — how an index is physically split up

Here's the key idea that makes Elasticsearch scale: **an index is not stored as one single blob** — it's split into pieces called **shards**, and those shards are distributed across the nodes in the cluster.

Think of an index as a **deck of cards** too big for one person to hold. You split it into a few piles (shards) and hand them to different people (nodes) to hold and search through in parallel.

There are two kinds of shards:

#### Primary shards
- The original pieces of your index's data.
- The number of primary shards is set **when you create the index** and (in most cases) **cannot be changed later** without reindexing.
- If you have 3 primary shards, your index's data is split into 3 parts, e.g. distributed across `es01`, `es02`, `es03`.

#### Replica shards
- A **copy** of a primary shard, kept on a **different node** than its primary.
- Purpose: **high availability** (if the node holding the primary dies, a replica is promoted to primary — no data loss) and **read performance** (searches can be served by either the primary or any replica, spreading out query load).
- The number of replicas can be changed anytime, even after the index is created.

**Example:** Index `products` with `3 primary shards` and `1 replica each` = 6 shards total, spread across the 3-node cluster like this:

```
Node es01: [P0] [R1]
Node es02: [P1] [R2]
Node es03: [P2] [R0]

P = Primary shard, R = Replica shard (of the shard with matching number)
```

Notice replica `R0` (a copy of primary `P0`) is placed on a **different** node than `P0` — Elasticsearch never puts a primary and its own replica on the same node, since that would defeat the purpose of redundancy.

**Why not just use 1 giant shard?**
- A single shard has a practical size/performance ceiling.
- Splitting into shards lets you parallelize search across nodes and CPUs.
- But *too many* tiny shards is also bad (overhead per shard) — this is called **oversharding**. Sizing shards correctly is a real production skill.

### 2.10 Cluster Health — Green / Yellow / Red

Elasticsearch reports overall cluster health as a simple traffic-light status:

| Status | Meaning |
|---|---|
| 🟢 **Green** | All primary **and** replica shards are allocated/active. Everything is fully healthy. |
| 🟡 **Yellow** | All primary shards are active, but at least one **replica** is missing/unallocated (e.g., you have replicas configured but not enough nodes to place them — common on a **single-node cluster**, since a replica can never be on the same node as its primary!). Your data is safe and searchable, just not fully redundant yet. |
| 🔴 **Red** | At least one **primary** shard is missing/unallocated. Some data is unavailable — this is a real problem. |

> This is exactly why the `03-elastic-lab-single-node` setup will typically show **yellow** if you create an index with the default 1 replica — there's only one node, so the replica has nowhere to go. That's expected and harmless in a single-node dev/lab environment.

Check it yourself:
```
GET /_cluster/health
```

### 2.11 How a write (indexing) request flows

1. Client sends a document to **any** node (that node becomes the "coordinating node" for this request).
2. That node calculates which **primary shard** should own the document (based on a hash of the document's `_id` by default) and forwards the request there.
3. The primary shard node writes the document, then **forwards the write to all replica shards** of that primary.
4. Once enough copies acknowledge (based on the configured consistency), the write is confirmed back to the client.

### 2.12 How a read (search) request flows

1. Client sends a search query to **any** node (the coordinating node for this request).
2. That node forwards the query to **one copy** (primary or replica — whichever is available/least busy) of **every shard** in the index, so it only reads each piece of data once.
3. Each shard searches its own local data and returns its top matches.
4. The coordinating node **merges and re-ranks** all shard results into one final result list and returns it to the client.

This "scatter-gather" pattern is why more shards (up to a point) = more parallelism = faster searches on large data sets.

### 2.13 Segments and Lucene (under the hood)

Each shard is actually powered by **Apache Lucene**, a lower-level search library. Internally, a shard is composed of many small immutable files called **segments**. When you index a document, it first goes into an in-memory buffer, then gets periodically written ("refreshed") into a new segment. Over time, small segments are merged into bigger ones in the background ("segment merging") for efficiency. You don't need to manage this directly — just know it's why:
- Newly indexed documents aren't *instantly* searchable (default refresh interval ~1 second) — this delay is called **near real-time (NRT) search**.
- Deletes/updates in Elasticsearch are implemented as "mark old segment doc as deleted + write a new document," reclaimed later during merges.

### 2.14 REST API — how you talk to Elasticsearch

Elasticsearch exposes everything (search, indexing, cluster management) via a simple **HTTP REST API**, using JSON. Examples (these are exactly what you'd type into Kibana's Dev Tools console):

```
GET  /_cluster/health              # cluster status
GET  /_cat/nodes?v                 # list nodes in human-readable table
GET  /_cat/indices?v               # list all indices
PUT  /products                     # create an index
POST /products/_doc                # index (add) a document
GET  /products/_search?q=laptop    # search
DELETE /products                   # delete an index
```

### 2.15 Security basics (X-Pack Security)

Modern Elasticsearch (used in both labs here, `xpack.security.enabled: true`) has security built in:

- **Users** — accounts that log in (e.g., the built-in superuser `elastic`).
- **Roles** — permission sets (e.g., "can read index `logs-*`, cannot delete anything").
- **`kibana_system`** — a special built-in service account Kibana uses internally to talk to Elasticsearch (you'll notice both labs set a `KIBANA_PASSWORD`/`KIBANA_SYSTEM_PASSWORD` for exactly this account — it's not a human user).
- **TLS/SSL** — encrypts traffic. In `02-elastic-lab-multiple-nodes`, transport traffic *between nodes* is encrypted with certificates (`xpack.security.transport.ssl.enabled: true`) generated by the `setup` container using `elasticsearch-certutil`. In both labs, the HTTP layer (`xpack.security.http.ssl.enabled: false`) is left unencrypted since these are local dev labs, not production.

### 2.16 Index Lifecycle Management (ILM) — managing data over time

For time-series data (logs, metrics), you don't want one giant ever-growing index. **ILM** automates moving an index through **phases** as it ages:

| Phase | Typical action |
|---|---|
| **Hot** | Actively being written to and queried. Lives on fast hardware. |
| **Warm** | No longer written to, queried less. Can reduce replicas, merge segments (force merge) to save resources. |
| **Cold** | Rarely queried; can be moved to cheaper storage. |
| **Frozen** | Almost never queried; minimal resources, often backed by cheap object storage. |
| **Delete** | Automatically deleted after a retention period (e.g., 90 days). |

This pairs with the **hot/warm/cold/frozen data tier node roles** mentioned earlier (§2.8) and with **Data Streams** (below).

### 2.17 Index Templates & Data Streams

- **Index Template**: a predefined mapping/settings blueprint automatically applied whenever a new index matching a name pattern (e.g., `logs-*`) is created. Saves you from manually defining mappings every time.
- **Data Stream**: a convenient abstraction for append-only, time-series data (like logs). You write to one logical name (e.g., `logs-app-default`), and Elasticsearch automatically manages the underlying rotating indices (called **backing indices**) behind the scenes, working hand-in-hand with ILM.

### 2.18 Aggregations — analytics, not just search

Beyond search, Elasticsearch can **summarize** data on the fly, similar to `GROUP BY` in SQL:

```
POST /products/_search
{
  "size": 0,
  "aggs": {
    "avg_price": { "avg": { "field": "price" } },
    "by_category": { "terms": { "field": "category.keyword" } }
  }
}
```

This aggregation engine is the foundation for every Kibana chart and dashboard — when you build a bar chart in Kibana, under the hood it's just sending an aggregation query like this one.

---

## 3. Kibana — Concepts and Components

Kibana is the web application you open in your browser (port `5601` in both labs) to interact with the data stored in Elasticsearch. Below are all its major building blocks.

### 3.1 Data Views (formerly "Index Patterns")

Before you can explore data in Kibana, you must tell it **which Elasticsearch index (or indices) to look at**. A **Data View** is essentially a saved pointer like `logs-*` or `products`, plus metadata about field types, that Kibana uses across almost every feature (Discover, Visualize, Dashboards).

> Historically called "Index Patterns" — you'll still see this term in older docs/versions.

### 3.2 Discover

The **Discover** page lets you browse and search raw documents interactively, like a search bar over your data — filter by time range, search with the Kibana Query Language (KQL), inspect individual documents, and see field value distributions. This is usually the *first* place you go to explore new data.

### 3.3 KQL (Kibana Query Language) & DQL

A simplified, user-friendly query syntax used in Kibana's search bars, e.g.:
```
status:"error" and service:"payments"
response_time > 500
```
Kibana translates this into Elasticsearch queries under the hood, so you don't need to hand-write raw JSON queries for everyday exploration.

### 3.4 Visualize / Lens

Tools to turn aggregation queries into charts without writing code:
- **Lens** — the modern drag-and-drop visualization builder (recommended starting point). Drag a field onto an axis, pick a chart type (bar, line, pie, table, etc.), and Kibana builds the aggregation for you.
- **Visualize Library** — the older/classic visualization editors, still used for some specialized chart types (e.g., TSVB, Timelion, Vega).
- **Maps** — geospatial visualizations (e.g., plotting `geo_point` fields on a world map).

### 3.5 Dashboards

A **Dashboard** is a single page combining multiple saved visualizations, tables, and filters into one view — e.g., an "Ops Overview" dashboard showing error rate, request volume, and latency charts side by side. Dashboards can be filtered globally (by time range or search query) and shared/exported.

### 3.6 Canvas

A more "freeform, presentation-style" way to build pixel-perfect, live-data infographics/reports — think of it as PowerPoint slides that auto-update from live Elasticsearch data.

### 3.7 Dev Tools (Console)

A built-in code editor inside Kibana for sending raw REST API requests directly to Elasticsearch, with autocomplete — this is the fastest way to run the example API calls from §2.14 without needing `curl`. Found under **Management → Dev Tools**.

### 3.8 Stack Management

The admin section of Kibana, where you configure almost everything behind the scenes:
- **Index Management** — view/create/delete indices, see shard counts, health.
- **Index Lifecycle Policies** — configure ILM phases (§2.16).
- **Data Views** — manage the data views described in §3.1.
- **Saved Objects** — every dashboard, visualization, and data view is stored as a "saved object" in a special internal Kibana index; this screen lets you export/import/inspect them.
- **Roles & Users** — manage Elasticsearch security (§2.15) directly from the UI.
- **Spaces** — see below.

### 3.9 Spaces

**Spaces** let you partition Kibana's dashboards/visualizations/saved objects into separate, isolated workspaces within a single Kibana instance — e.g., a "Marketing" space and an "Engineering" space, each seeing only their own dashboards, without needing separate Kibana deployments.

### 3.10 Alerting & Rules

Kibana lets you define **rules** that run on a schedule, check conditions against your Elasticsearch data (e.g., "error count > 100 in the last 5 minutes"), and trigger **actions** (send an email, Slack message, webhook, create a case) when the condition is met. Useful for turning dashboards from "something you have to watch" into "something that watches you."

### 3.11 Machine Learning jobs

If your cluster has nodes with the `ml` role (§2.8), Kibana provides a UI to create:
- **Anomaly detection jobs** — automatically learn "normal" patterns in a metric over time and flag unusual spikes/dips.
- **Data frame analytics** — regression/classification/outlier-detection jobs on your indexed data.

### 3.12 Observability & Security solutions

On top of the core stack, Kibana ships pre-built solutions for common use cases (these are really just curated dashboards + data models + apps on top of the same Elasticsearch data):
- **Observability** — APM (application performance monitoring), logs, and infrastructure metrics.
- **Security (SIEM)** — threat hunting, detection rules, endpoint security.

### 3.13 Fleet & Elastic Agent

**Elastic Agent** is a single unified agent (successor to individual Beats) that you install on a server to collect logs/metrics/security data and ship it to Elasticsearch. **Fleet** is the Kibana UI to centrally manage, configure, and monitor all your deployed Elastic Agents from one place.

---

## 4. Logstash & Beats (the "L" and the collectors)

Not present in this repo's labs (which are Elasticsearch + Kibana only), but good to know conceptually since they complete the "ELK" picture:

### 4.1 Logstash

A server-side data processing pipeline with 3 stages:
```
Input → Filter → Output
```
- **Input**: where data comes from (files, Kafka, Beats, databases, etc.)
- **Filter**: transform/parse it (e.g., `grok` to parse unstructured log lines into structured fields, `mutate` to rename/convert fields, `date` to parse timestamps).
- **Output**: where it goes (usually Elasticsearch, but can be anywhere).

### 4.2 Beats

Lightweight, single-purpose data shippers written in Go, installed close to the data source:

| Beat | Ships |
|---|---|
| **Filebeat** | Log files |
| **Metricbeat** | System/service metrics (CPU, memory, etc.) |
| **Packetbeat** | Network traffic data |
| **Winlogbeat** | Windows event logs |
| **Heartbeat** | Uptime/availability monitoring |

Beats are "lightweight" on purpose — they just collect and forward, leaving the heavy transformation work to Logstash or Elasticsearch ingest pipelines (§2.8).

### 4.3 Elasticsearch Ingest Pipelines (Logstash-lite, built into Elasticsearch)

If your transformations are simple, you can skip Logstash entirely and define an **ingest pipeline** directly in Elasticsearch (run by nodes with the `ingest` role), made of **processors** (e.g., `grok`, `set`, `rename`, `date`) applied to documents right before they're indexed.

---

## 5. How Everything Fits Together (Architecture Diagram)

```
                        ┌─────────────────────────────────────────┐
                        │              Elasticsearch Cluster        │
                        │         (cluster.name: elastic-wsl-cluster)│
                        │                                            │
   Data sources         │   ┌────────┐   ┌────────┐   ┌────────┐    │
   (apps, logs,   ───▶  │   │  es01  │   │  es02  │   │  es03  │    │
    Beats, etc.)        │   │ master │   │ master │   │ master │    │
                        │   │  data  │   │  data  │   │  data  │    │
                        │   │ ingest │   │ ingest │   │ ingest │    │
                        │   └────────┘   └────────┘   └────────┘    │
                        │      (one of them is the ACTIVE master)   │
                        └─────────────────────────────────────────┘
                                          ▲
                                          │ HTTP REST API (port 9200)
                                          │
                                    ┌───────────┐
                                    │  Kibana   │  (port 5601)
                                    │ (web UI)  │
                                    └───────────┘
                                          ▲
                                          │ browser
                                          │
                                       [ You ]
```

Mapping this to the repo's labs:
- `03-elastic-lab-single-node`: the box above collapses to **one** node doing master + data + ingest, and `discovery.type: single-node` tells Elasticsearch not to wait for other nodes to form a cluster.
- `02-elastic-lab-multiple-nodes`: exactly the 3-node picture above — `es01`/`es02`/`es03` discover each other via `discovery.seed_hosts` and elect a master among themselves via `cluster.initial_master_nodes`.

---

## 6. Glossary (Quick Reference)

| Term | One-line definition |
|---|---|
| **Cluster** | Group of nodes working together, sharing a `cluster.name`. |
| **Node** | One running Elasticsearch instance/server. |
| **Master node** | The currently elected node responsible for cluster-wide management (not data). |
| **Master-eligible node** | A node with the `master` role that *could* be elected master. |
| **Data node** | A node that stores shards and executes search/index operations. |
| **Coordinating node** | Whichever node receives a client request and routes/merges it — every node can do this. |
| **Index** | A named collection of related documents (like a table). |
| **Document** | A single JSON record (like a row). |
| **Mapping** | The schema — field names and types — for an index. |
| **Shard** | A horizontal slice of an index's data; the unit of distribution across nodes. |
| **Primary shard** | An original shard slice; count fixed at index creation. |
| **Replica shard** | A copy of a primary shard on a different node, for redundancy and read throughput. |
| **Inverted index** | The word → document lookup structure that makes text search fast. |
| **Segment** | Small immutable Lucene file that makes up a shard internally. |
| **Cluster health (green/yellow/red)** | Overall status based on shard allocation. |
| **ILM (Index Lifecycle Management)** | Automates moving/deleting indices through hot/warm/cold/frozen/delete phases. |
| **Data stream** | Abstraction for append-only, time-series data across rotating backing indices. |
| **Data View** | Kibana's pointer to which index/indices to explore (formerly "index pattern"). |
| **Discover** | Kibana page for raw document search/browsing. |
| **Lens / Visualize** | Kibana tools for building charts from aggregations. |
| **Dashboard** | A page combining multiple saved visualizations. |
| **Dev Tools / Console** | Kibana's built-in raw REST API client. |
| **Space** | An isolated workspace inside one Kibana instance. |
| **Beats** | Lightweight per-purpose data shippers (Filebeat, Metricbeat, etc.). |
| **Logstash** | Server-side ETL pipeline (Input → Filter → Output) for data before it hits Elasticsearch. |
| **Ingest pipeline** | Lightweight, built-in Elasticsearch equivalent of Logstash filtering. |
| **Fleet / Elastic Agent** | Centralized management of unified data-shipping agents. |

---

## 7. Suggested Learning Path (Practical Next Steps)

1. Start the single-node lab (`03-elastic-lab-single-node`) and open Kibana at `http://localhost:5601`.
2. Log in with the `elastic` user and the password from its `.env` file.
3. Go to **Dev Tools** and run: `GET /_cluster/health`, `PUT /test-index`, `POST /test-index/_doc` (add a sample document), `GET /test-index/_search`.
4. Go to **Stack Management → Index Management** and look at the shard count/health for `test-index` — notice it's likely **yellow** (§2.10) since there's only one node.
5. Create a **Data View** for `test-index*`, then explore it in **Discover**.
6. Build a simple chart in **Lens**, then add it to a **Dashboard**.
7. Once comfortable, spin up `02-elastic-lab-multiple-nodes` and repeat step 3–4: notice the cluster health for a new index with 1 replica turns **green** now that there are 3 nodes to place shards on, and run `GET /_cat/shards?v` to *see* how Elasticsearch distributed primaries and replicas across `es01`, `es02`, and `es03`.
