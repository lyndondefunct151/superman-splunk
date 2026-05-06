# Splunk Architecture & Core Concepts

## The Three-Tier Architecture

Splunk separates concerns into three logical tiers:

```
Data Sources → [Forwarders] → [Indexers] → [Search Heads] → Users
```

### Forwarder Tier
Forwarders collect data from sources and ship it to indexers. Two primary types:

**Universal Forwarder (UF)**
- Lightweight agent (~5 MB RAM); preferred for production deployment.
- Handles: file monitoring (`monitor://`), scripted inputs, network inputs (TCP/UDP syslog), Windows event logs.
- Does NOT parse events — raw data passes through to indexers.
- Only parses `outputs.conf`, `inputs.conf`, and `deploymentclient.conf`.
- Config managed via Deployment Server or manually.

**Heavy Forwarder (HF)**
- Full Splunk instance running in forwarding mode.
- Can parse, filter (nullQueue), mask, route, and transform data before forwarding.
- Higher resource usage — use only when parsing near-source is necessary.
- Acts as an intermediate tier for complex routing or protocol bridging.

### Indexer Tier
Indexers ingest, parse, and store events. They also run search jobs when dispatched by a search head.

**Data ingestion pipeline** (queue-based, each queue is a FIFO buffer):
```
Input data → [inputQueue] → parsing processor → [parsingQueue]
           → [aggQueue] (line merging/event breaking)
           → [typingQueue] (timestamp extraction, transforms)
           → [indexQueue] → disk (rawdata + .tsidx)
```

Data is processed in **64 KB blocks**. The pipeline adds metadata (`host`, `source`, `sourcetype`, `index`) at the input stage.

**What gets written to disk:**
- `rawdata/` — compressed raw event text (`.gz` slices, `journal.gz`)
- `tsidx/` — term/time index files (`.tsidx`) enabling keyword + time search
- `metadata/` — per-index metadata about field values and sources

### Search Head Tier
Search heads receive user queries, dispatch them to indexers, merge partial results, apply post-processing (sort, dedup, etc.), and return results.

- Search heads do NOT store indexed data (unless also acting as indexers in small deployments).
- Search heads hold **knowledge objects** (saved searches, field extractions, dashboards, alerts).
- In SHC (Search Head Clustering), knowledge objects are synchronized across members.

---

## Bucket Architecture & Storage

### Bucket Types
| Bucket State | Description | Typical Storage |
|-------------|-------------|-----------------|
| **hot** | Active write bucket; indexed events written here | Fast SSD |
| **warm** | Closed hot bucket; searchable, no new writes | Fast SSD or SAS |
| **cold** | Older warm buckets rolled here by age/size | Slow HDD or NAS |
| **frozen** | Deleted (default) or archived externally | Archive/S3 |
| **thawed** | Frozen data restored for temporary search | Any |

### Bucket Roll-Over Rules (indexes.conf)
```ini
[myindex]
maxHotBuckets = 3                    # max concurrent hot buckets
maxDataSize = auto                   # 750 MB default, auto_high_volume = 10 GB
frozenTimePeriodInSecs = 7776000     # default ~90 days (not 6 years — tune carefully)
maxTotalDataSizeMB = 500000          # total index size cap
minFreeSpace = 20000                 # min free disk MB (enterprise baseline ~20000)
coldPath = $SPLUNK_DB/myindex/colddb
homePath = $SPLUNK_DB/myindex/db     # hot + warm
thawedPath = $SPLUNK_DB/myindex/thaweddb
```

Use `auto_high_volume` for indexes ingesting **>10 GB/day** — sets hot bucket max to 10 GB.

### SmartStore
SmartStore decouples compute from storage:
- Hot and actively warm buckets remain **local** on the indexer.
- Older warm bucket primaries are pushed to **remote object storage** (S3, GCS, Azure Blob).
- Indexers cache frequently accessed remote buckets locally in a **cache manager**.
- Eliminates the classic cold tier for large deployments — cold lives in object storage.
- Requires indexer clustering; configured in `indexes.conf` with `remotePath` and `[cachemanager]`.

```ini
[volume:remote_store]
storageType = remote
path = s3://my-bucket/splunk

[myindex]
remotePath = volume:remote_store/myindex
```

---

## Distributed Deployment & Clustering

### Indexer Clustering
Provides **high availability** and **data replication** across indexers.

**Key concepts:**
- **Cluster Manager** (formerly "cluster master"): coordinates the cluster, tracks bucket locations, manages fixup tasks. Does NOT index data.
- **Replication Factor (RF)**: total number of bucket copies across all peers. RF=3 means 3 copies.
- **Search Factor (SF)**: number of **searchable** copies. SF ≤ RF.
- **Bucket fixup**: automatic process to re-replicate under-replicated buckets after a peer failure.

**Minimal cluster:** RF=2, SF=2 (requires ≥2 peers + 1 cluster manager).
**Production recommendation:** RF=3, SF=2 (requires ≥3 peers).

```ini
# server.conf on cluster manager
[clustering]
mode = manager
replication_factor = 3
search_factor = 2
pass4SymmKey = <shared_secret>
cluster_label = prod_cluster

# server.conf on indexer peer
[clustering]
mode = peer
manager_uri = https://cluster-manager:8089
pass4SymmKey = <same_shared_secret>
```

Apps/configs deployed to indexer peers via the **Cluster Manager** using the manager-apps directory:
```
$SPLUNK_HOME/etc/manager-apps/<app>/
```
Push with: `splunk apply cluster-bundle`

### Search Head Clustering (SHC)
Provides **search tier HA** and **concurrent search capacity**.

**Key concepts:**
- **Captain**: elected member that coordinates scheduled search jobs, assigns them to members.
- **Deployer**: separate Splunk instance that pushes apps/config to SHC members.
- Members replicate knowledge objects and job artifacts to each other.
- Captain election uses Raft-like consensus.

```ini
# server.conf on each SHC member
[shclustering]
mode = member
mgmt_uri = https://this-member:8089
replication_port = 34777
shcluster_label = shc_prod
pass4SymmKey = <shared_secret>
conf_deploy_fetch_url = https://deployer:8089
```

Push apps to SHC via deployer:
```
$SPLUNK_HOME/etc/shcluster/apps/<app>/
splunk apply shcluster-bundle -target https://captain:8089
```

**SHC gotcha**: Oversizing an SHC (too many members) adds replication overhead without proportional search capacity gain. 3–5 members is typical; beyond 7 becomes counterproductive.

### Deployment Server
Centralized configuration distribution for **forwarders** (not indexer peers).

```ini
# serverclass.conf on Deployment Server
[serverClass:linux_servers]
whitelist.0 = *.prod.example.com

[serverClass:linux_servers:app:my_inputs_app]
stateOnClient = enabled
restartSplunkd = true
```

Forwarders connect via `deploymentclient.conf`:
```ini
[deployment-client]
[target-broker:deploymentServer]
targetUri = https://deploy-server:8089
```

---

## Knowledge Objects & Bundles

Knowledge objects are user-created content objects: saved searches, field extractions, lookups, event types, tags, macros, dashboards, and alerts.

**Knowledge bundles**: search heads package knowledge objects and replicate them to indexer peers so that distributed searches can apply transformations and field extractions at the indexer tier. **Bundle bloat causes performance degradation** — large bundles increase dispatch time for every search.

**Prevent bundle bloat:**
- Delete stale saved searches, dashboards, and alerts.
- Move large lookup CSV files to **KV Store** (avoids bundling).
- Set `export = system` only for knowledge objects that genuinely need peer replication.

### Configuration Merge Order (highest to lowest precedence)
```
1. $SPLUNK_HOME/etc/system/local/
2. $SPLUNK_HOME/etc/apps/<app>/local/   ← Production changes go HERE
3. $SPLUNK_HOME/etc/apps/<app>/default/
4. $SPLUNK_HOME/etc/system/default/
```

**Never edit `/default` configs.** Changes in `/default` are overwritten on upgrade.

---

## Performance Tuning

### Linux OS Settings
- **Disable Transparent Huge Pages (THP)**: enabled THP degrades Splunk indexing/search by ~30%.
  ```bash
  echo never > /sys/kernel/mm/transparent_hugepage/enabled
  echo never > /sys/kernel/mm/transparent_hugepage/defrag
  ```
- **Ulimits**: increase `nofile` (open files) and `nproc` (processes) for the splunk user.
- **I/O scheduler**: use `deadline` or `noop` for SSDs.

### Indexer Performance
- Keep hot/warm buckets on **SSD** — disk throughput is the #1 indexing bottleneck.
- Tune `parallelIngestionPipelines` in `server.conf` for multi-core indexers (default 1; set 2 for high-volume indexers).
- Monitor queue fill rates in Monitoring Console (Settings → Monitoring Console → Indexing Performance). Persistent `typingQueue` saturation indicates parsing bottleneck.

### Search Performance
- Use `index=` and explicit `sourcetype=` to reduce data scanned.
- Use the **earliest/latest** time pickers — avoid "all time" searches.
- Use `tstats` for accelerated data model queries instead of raw `stats`.
- Limit `| transaction` usage — it buffers all matching events in memory.
- Tune `limits.conf`:
  ```ini
  [search]
  max_searches_per_cpu = 1      # default 1; increase cautiously
  base_max_searches = 6
  ```

### Monitoring Console
Key dashboards to watch:
- **Search Usage Statistics**: active searches, search latency, artifact sizes.
- **Scheduler Activity**: scheduled search queue depth, skip rate.
- **Indexing Performance**: per-indexer queue fill rates, event throughput (EPS).
- **Resource Usage**: CPU/memory per process, hot bucket count.
