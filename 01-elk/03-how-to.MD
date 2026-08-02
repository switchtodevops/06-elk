# Elastic Stack — How To (Questions & Answers)

This file is a running Q&A log of practical "how do I do X?" questions about Elasticsearch/Kibana, answered in detail but in simple terms. Related background concepts live in `01-elk-concepts.MD` and `02-elk-advanced-concepts.MD`.

---

## 01 — How to enable the Stack Monitoring dashboard in Kibana?

### In simple terms

**Stack Monitoring** is Kibana's built-in "health dashboard" for the Elastic Stack itself — it shows you CPU, memory, disk, JVM heap, indexing/search rates, node status, and shard health for your Elasticsearch nodes and Kibana instances, over time. Think of it like a **car dashboard** (speedometer, fuel gauge, engine temperature) but for your cluster instead of your data. It answers questions like "is a node running out of disk?", "is heap usage dangerously high?", "did indexing throughput suddenly drop?" — *before* they turn into an outage.

By default, Stack Monitoring is **not collecting anything** — you have to turn on "collection" (the process of gathering these metrics) before the dashboard has data to show.

### The two ways to collect monitoring data

| Method | How it works | When to use |
|---|---|---|
| **Legacy/Internal collection** | Elasticsearch and Kibana push their own metrics directly into special `.monitoring-*` indices on the *same* cluster (or a dedicated monitoring cluster), via a setting you flip on. No extra software needed. | Quick, zero-install way to see monitoring in a **dev/lab/small cluster** (like the labs in this repo). Deprecated for production. |
| **Metricbeat collection (recommended)** | A separate lightweight **Metricbeat** agent (with the `elasticsearch-xpack` and `kibana-xpack` modules) polls each node's monitoring API and ships the metrics to a (ideally separate) **monitoring cluster**. | Production — decouples monitoring data from the cluster being monitored, so if the monitored cluster has problems, you can still see *why* on the separate monitoring cluster. |

Elastic's own guidance: use internal collection to get monitoring turned on quickly, but migrate to Metricbeat for anything beyond a personal lab, since self-monitoring a struggling cluster with data stored *on that same struggling cluster* is a bit like a patient trying to take their own vital signs while unconscious.

---

### Method A — Quick way for this repo's labs (Internal/legacy collection via the Kibana UI)

This is the fastest path and works well for `02-elastic-lab-multiple-nodes` or `03-elastic-lab-single-node` in this repo (Elastic Stack `8.19.19`, security enabled).

1. **Start the stack** and open Kibana at `http://localhost:5601`, then log in as `elastic` (password from the lab's `.env` file, e.g. `ELASTIC_PASSWORD`).
2. Open the main menu (☰, top-left) → scroll down to **Management** → **Stack Monitoring**.
   *(Direct URL: `http://localhost:5601/app/monitoring`)*
3. Kibana detects that monitoring isn't set up yet and shows a **"Turn on monitoring"** button — click it.
4. This flips the setting `xpack.monitoring.collection.enabled` to `true` on the Elasticsearch cluster (equivalent to running the API call below yourself), which tells every node to start periodically publishing its own stats into `.monitoring-es-*` indices.

   You can do the same thing manually from **Dev Tools** instead of clicking the button:

   ```
   PUT /_cluster/settings
   {
     "persistent": {
       "xpack.monitoring.collection.enabled": true
     }
   }
   ```
5. Wait ~10–20 seconds (the default collection interval) and refresh the **Stack Monitoring** page. You should now see:
   - An **Elasticsearch** panel — cluster health, number of nodes, indices, shards, JVM heap, CPU.
   - A **Kibana** panel — instance status, requests/sec, response times.
6. Click into a node/index to drill down into detailed metric charts (search rate, indexing rate, disk usage, GC time, etc.) over the selected time range (top-right time picker, just like Discover).

**Security note:** since these labs run with `xpack.security.enabled: true`, the `elastic` superuser (or a user with the built-in `monitoring_user` role) is required to view the Stack Monitoring app, and a user with `manage` cluster privileges is required to turn collection on. The lab's default `elastic` user already has both.

---

### Method B — Production-style setup (dedicated monitoring cluster + Metricbeat)

For a real production cluster, the recommended layout is: **production cluster → Metricbeat → monitoring cluster → Kibana (pointed at the monitoring cluster) → Stack Monitoring UI**.

1. **Disable internal collection** on the production cluster (if it was ever turned on), so nodes stop self-reporting and Metricbeat becomes the only source of truth:

   ```
   PUT /_cluster/settings
   {
     "persistent": {
       "xpack.monitoring.collection.enabled": false
     }
   }
   ```

2. **Stand up (or designate) a separate monitoring cluster** — even a small single-node cluster is fine. This is where the metrics *about* your production cluster will live, so it survives even if production has problems.

3. **Install Metricbeat** on a machine that can reach both clusters, then enable the two relevant modules:

   ```bash
   metricbeat modules enable elasticsearch-xpack kibana-xpack
   ```

4. **Point each module at the cluster it should monitor** — edit `modules.d/elasticsearch-xpack.yml` (and the Kibana equivalent) so `hosts` points at your **production** Elasticsearch/Kibana, with credentials that have the `remote_monitoring_collector` role:

   ```yaml
   - module: elasticsearch
     xpack.enabled: true
     period: 10s
     hosts: ["http://production-es:9200"]
     username: "remote_monitoring_user"
     password: "changeme"
   ```

5. **Point Metricbeat's output at the monitoring cluster** (not production!) in `metricbeat.yml`:

   ```yaml
   output.elasticsearch:
     hosts: ["http://monitoring-es:9200"]
     username: "elastic"
     password: "changeme"
   ```

6. Start Metricbeat (`metricbeat setup` once, then `metricbeat -e`). It will create `.monitoring-es-*` / `.monitoring-kibana-*` indices on the **monitoring cluster**.

7. In **Kibana** (connected to the monitoring cluster), open **Stack Monitoring** — it will automatically detect the incoming Metricbeat data and display the same dashboards as Method A, but now sourced from a safely-separated cluster.

---

### Quick comparison recap

| | Internal collection (Method A) | Metricbeat collection (Method B) |
|---|---|---|
| Setup effort | One click / one API call | Install & configure Metricbeat, ideally a 2nd cluster |
| Extra moving parts | None | Metricbeat process, optional separate cluster |
| Survives production cluster failure? | No — data lives on the same cluster being monitored | Yes — data lives on a separate monitoring cluster |
| Status | Good for dev/lab | Recommended, production-grade |
| Good fit for this repo's labs | ✅ Yes | Optional (overkill for a single local lab) |

### Glossary (for this question)

| Term | One-line definition |
|---|---|
| **Stack Monitoring** | Kibana's built-in app showing health/performance metrics (CPU, heap, disk, indexing/search rates) for Elasticsearch and Kibana itself. |
| **Internal/legacy collection** | Nodes push their own monitoring metrics directly into `.monitoring-*` indices, controlled by `xpack.monitoring.collection.enabled`. |
| **Metricbeat collection** | A separate Metricbeat agent polls monitoring APIs and ships metrics to a (usually separate) monitoring cluster — the recommended production method. |
| **Monitoring cluster** | A dedicated Elasticsearch cluster that only stores monitoring data about one or more *other* ("production") clusters. |
| **`remote_monitoring_collector` / `remote_monitoring_user`** | Built-in roles/user used by Metricbeat to read monitoring data from a production cluster and write it to a monitoring cluster. |
