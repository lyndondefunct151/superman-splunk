# Splunk Administration & Configuration

## Configuration File System

### Precedence (highest to lowest)
```
1. $SPLUNK_HOME/etc/system/local/         ← Highest: System-level overrides
2. $SPLUNK_HOME/etc/apps/<app>/local/     ← App-specific overrides (production changes here)
3. $SPLUNK_HOME/etc/apps/<app>/default/   ← Shipped app defaults
4. $SPLUNK_HOME/etc/system/default/       ← Splunk's built-in defaults (lowest)
```

**Golden Rule:** Never edit files in `/default/`. Changes there are overwritten by upgrades.
Always create or edit files in `/local/`.

### Merging Behavior
- Most settings: **last-write-wins** within a precedence level.
- List settings (e.g., `REPORT-*`, `LOOKUP-*`): **merged** across all layers.
- Stanzas are merged field by field; an app `/local` can override only specific attributes.

### Checking Effective Configuration
```bash
# Show merged effective configuration for a specific conf file
splunk btool inputs list --debug
splunk btool props list --debug | grep -A5 "mysourcetype"

# Check which file a setting comes from
splunk btool transforms list --debug | grep -B2 "my_transform"
```

---

## inputs.conf — Data Collection

Defines where Splunk collects data from.

### File Monitor
```ini
[monitor:///var/log/apache2/*.log]
index = web
sourcetype = access_combined
disabled = false
followTail = 0       # 0 = read from beginning of new files; 1 = tail only
recursive = true
whitelist = \.log$
blacklist = \.gz$
```

### Network (TCP/UDP/Syslog)
```ini
[tcp://9514]
connection_host = dns
sourcetype = syslog
index = network

[udp://514]
sourcetype = syslog
index = network
no_appending_timestamp = true
```

### Scripted Input
```ini
[script://./bin/my_collector.py]
interval = 300        # run every 5 minutes
sourcetype = my_data
index = custom
disabled = false
```

### HEC — HTTP Event Collector
```ini
[http]
disabled = false
port = 8088
enableSSL = true

[http://my_token_name]
token = <uuid>
index = web
sourcetype = json_events
```

Send events via HEC:
```bash
curl -k -H "Authorization: Splunk <HEC_TOKEN>" \
  https://splunk:8088/services/collector/event \
  -d '{"event": {"key": "value"}, "sourcetype": "json_events", "index": "web"}'
```

---

## props.conf — Event Parsing

Controls how Splunk breaks raw data into events, extracts timestamps, and applies transforms.

### The "Big 8" Parsing Settings

```ini
[my_sourcetype]
# 1. Line breaking — regex to identify event boundaries
LINE_BREAKER = ([\r\n]+)         # default: split on newlines
# For multi-line events:
LINE_BREAKER = (END_OF_EVENT)    # regex matching event delimiter

# 2. Line merging — combine multiple lines into one event
SHOULD_LINEMERGE = false         # default: true (dangerous for high-volume)

# 3. Max event size
TRUNCATE = 10000                 # max chars per event (default 10000)

# 4. Timestamp lookahead — how far into event to look for timestamp
MAX_TIMESTAMP_LOOKAHEAD = 30     # characters (default 128; smaller = faster)

# 5. Timestamp prefix — text before the timestamp
TIME_PREFIX = timestamp=

# 6. Timestamp format
TIME_FORMAT = %Y-%m-%dT%H:%M:%S%z

# 7. Event breaker (for streaming, HEC aggregator)
EVENT_BREAKER_ENABLE = true
EVENT_BREAKER = \n---END---\n

# 8. Charset
CHARSET = UTF-8
```

### Field Extractions in props.conf
```ini
[access_combined]
# Regex extraction
EXTRACT-status = \s(?P<http_status>\d{3})\s

# Transform-based extraction
REPORT-parsed_fields = my_kv_transform

# Calculated field
EVAL-is_bot = if(match(useragent, "(?i)bot|spider|crawler"), "true", "false")

# Field alias (rename without removing original)
FIELDALIAS-cim_action = method AS http_method

# Lookup enrichment (automatic)
LOOKUP-geoip = geoip_lookup ip AS clientip OUTPUT country, city

# KV mode — automatic key=value extraction
KV_MODE = auto           # options: none, auto, auto_escaped, multi, json, xml

# JSON event parsing
KV_MODE = json           # parses entire event as JSON
```

---

## transforms.conf — Field Transforms & Routing

### Regex Field Extraction
```ini
[extract_session]
REGEX = session_id=(?P<session>[a-f0-9]+)
SOURCE_KEY = _raw
```

### Field Lookup
```ini
[user_lookup]
filename = users.csv
case_sensitive_match = false
batch_index_query = true     # performance boost for large lookups
match_type = WILDCARD(src_ip) CIDR(src_cidr)  # special match types
```

### Data Masking (PII Redaction)
```ini
[mask_ssn]
REGEX = \d{3}-\d{2}-(?P<last4>\d{4})
FORMAT = SSN-REDACTED-$1
SOURCE_KEY = _raw
DEST_KEY = _raw              # overwrite the raw event
```
Apply in props.conf: `SEDCMD-mask_ssn = s/\d{3}-\d{2}-\d{4}/XXX-XX-XXXX/g`

### Event Routing (nullQueue)
Route unwanted events to null (discard):
```ini
[route_to_null]
REGEX = ^\s*$               # discard blank lines
DEST_KEY = queue
FORMAT = nullQueue
```

Apply in props.conf:
```ini
[my_sourcetype]
TRANSFORMS-drop_blank = route_to_null
```

### Index-time Field Extraction
Force fields to be indexed (available without field extraction at search time):
```ini
[extract_indexed_field]
REGEX = severity=(?P<severity>\w+)
SOURCE_KEY = _raw
WRITE_META = true           # writes to metadata, indexed
```

---

## indexes.conf — Storage Configuration

```ini
[myapp_index]
# Storage paths
homePath   = $SPLUNK_DB/myapp/db          # hot + warm buckets
coldPath   = $SPLUNK_DB/myapp/colddb      # cold buckets
thawedPath = $SPLUNK_DB/myapp/thaweddb

# Bucket sizing
maxHotBuckets = 3
maxDataSize = auto                  # 750 MB; use auto_high_volume for >10 GB/day
# maxDataSize = auto_high_volume    # 10 GB hot buckets

# Retention
frozenTimePeriodInSecs = 7776000    # 90 days
maxTotalDataSizeMB = 200000         # 200 GB total size cap

# Disk safety
minFreeSpace = 20000                # MB — enterprise baseline

# Freeze/archive
coldToFrozenScript = /opt/scripts/archive_to_s3.py  # custom freeze script
# coldToFrozenDir = /archive/splunk/myapp           # OR copy-to-dir

# Replication (for indexer clustering)
repFactor = auto                    # inherit from cluster config
```

---

## outputs.conf — Forwarding

```ini
# Default forwarding group
[tcpout]
defaultGroup = primary_indexers
autoLBFrequency = 30         # rebalance load every 30 seconds
autoLBVolume = 10485760      # OR rebalance after 10 MB

# Primary indexer group
[tcpout:primary_indexers]
server = idx1:9997, idx2:9997, idx3:9997
useACK = true                # indexer acknowledgment for data durability

# Compressed forwarding
compressed = true

# SSL for forwarder-to-indexer
sslCertPath = $SPLUNK_HOME/etc/certs/forwarder.pem
sslRootCAPath = $SPLUNK_HOME/etc/auth/ca.pem
sslVerifyServerCert = true
sslVerifyServerName = true
```

---

## server.conf — Core System Settings

```ini
[general]
serverName = splunk-indexer-01
pass4SymmKey = <SECRET>      # shared key for cluster authentication — PROTECT THIS

[sslConfig]
enableSplunkdSSL = true
sslVersions = tls1.2, tls1.3
cipherSuite = ECDHE-RSA-AES256-GCM-SHA384:...
requireClientCert = false

# Forwarder SSL verification
sslVerifyServerCert = true
sslVerifyServerName = true
sslRootCAPath = $SPLUNK_HOME/etc/auth/ca.pem

[diskUsage]
minFreeSpace = 20000         # global minimum free disk (MB)
```

---

## Deployment Server

Deployment Server (DS) distributes apps and configurations to forwarders at scale.

### serverclass.conf
```ini
# Assign forwarders to classes by hostname/IP/FQDN
[serverClass:linux_production]
whitelist.0 = *.linux.prod.example.com
blacklist.0 = *.dev.*

# Assign an app to a server class
[serverClass:linux_production:app:linux_inputs]
stateOnClient = enabled
restartSplunkd = true

# Deploy to specific hosts
[serverClass:web_tier]
whitelist.0 = webserver01, webserver02, webserver03

[serverClass:web_tier:app:web_inputs]
stateOnClient = enabled
```

Apps to deploy: `$SPLUNK_HOME/etc/deployment-apps/<appname>/`

### Reload after changes
```bash
splunk reload deploy-server
```

---

## License Management

Splunk licensing is based on **daily ingested volume** (not storage size).

### Key Principles
- Daily license quota = total GB/day indexed.
- Violations: 5 within 30 days triggers a warning stack that disables search.
- License usage is tracked per `host`, `source`, `sourcetype`, and `index`.

### Monitoring License Usage
```spl
index=_internal source=*license_usage.log type=Usage
| stats sum(b) AS bytes BY s, st, h
| eval GB = round(bytes/1073741824, 3)
| sort -GB
```

Find top license consumers:
```spl
index=_internal source=*license_usage.log type=Usage earliest=-1d
| stats sum(b) AS total_bytes BY sourcetype
| eval GB_per_day = round(total_bytes/1073741824, 2)
| sort -GB_per_day | head 20
```

### Reducing License Usage
- **Filter at collection time**: use props.conf `TRANSFORMS-null` to drop high-volume low-value events before indexing.
- **Aggregation at HF**: use Heavy Forwarder to pre-aggregate metrics before indexing.
- **Summary indexes**: write pre-aggregated results to a summary index; don't keep raw high-volume data.
- **Routing**: send debug-level logs to a different, smaller-retention index.

---

## Monitoring Console

Access: Settings → Monitoring Console (or `_internal` index queries).

### Key Health Metrics
```spl
# Indexing throughput (EPS)
index=_internal source=*metrics.log group=per_index_thruput series!=_*
| timechart span=5m sum(eps) AS events_per_sec

# Search latency
index=_internal source=*scheduler.log status=success
| stats avg(run_time) AS avg_run_sec, p95(run_time) AS p95_run_sec BY savedsearch_name
| sort -avg_run_sec | head 20

# Queue fill rates (pipeline health)
index=_internal source=*metrics.log group=queue
| timechart span=1m avg(fill_perc) BY name

# License usage today
index=_internal source=*license_usage.log type=Usage earliest=@d
| stats sum(b) AS bytes | eval GB = round(bytes/1073741824, 2)
```

---

## Upgrade & Maintenance Best Practices

1. **Version control all configs**: keep `$SPLUNK_HOME/etc/` (minus `system/default/`) in Git.
2. **Validate before applying**: use `btool` to check for syntax errors.
3. **Cluster upgrades**: upgrade cluster manager first, then rolling upgrade peers (never upgrade all peers simultaneously).
4. **SHC upgrades**: use the `splunk upgrade-shcluster-members` procedure.
5. **Restart requirements**: most conf changes require `splunk restart` or at minimum `splunk reload <conf>`.
6. **Don't run as root**: run Splunk as a dedicated service account with least-privilege.
7. **Change default passwords**: never leave `admin`/`changeme` in production.
8. **Protect `pass4SymmKey`**: this key authenticates cluster members — treat it like a TLS private key.
