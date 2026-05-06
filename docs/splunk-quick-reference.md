# Splunk Quick Reference Cheatsheet

A fast-access reference for the most commonly needed Splunk facts.

## Ports

| Service | Default Port | Notes |
|---------|-------------|-------|
| Splunk Web (HTTP/HTTPS) | 8000 | User-facing UI |
| Management API / splunkd | 8089 | REST API, CLI, SDK |
| Indexer receiving port | 9997 | Forwarder → Indexer TCP |
| KV Store (MongoDB) | 8191 | Internal, not externally exposed |
| Bucket replication | 8080 | Indexer cluster peer replication |
| SHC replication | 34777 | Search head cluster member replication |
| HEC (HTTP Event Collector) | 8088 | Direct event ingestion over HTTP |
| Syslog TCP/UDP | 514 (configurable) | In `inputs.conf` |

---

## Key Configuration Files

| File | Location | Purpose |
|------|----------|---------|
| `inputs.conf` | Forwarder & Indexer | Defines data sources (file monitors, network inputs, scripted inputs, HEC) |
| `props.conf` | Indexer & Search Head | Event parsing: sourcetype assignment, timestamps, line-breaking, EXTRACT transforms |
| `transforms.conf` | Indexer & Search Head | Field extractions (REPORT), routing (DEST_KEY=queue, nullQueue), field aliases, masking |
| `outputs.conf` | Forwarder | Defines indexer targets, load balancing, SSL |
| `indexes.conf` | Indexer | Index definitions, storage paths, retention (frozenTimePeriodInSecs), bucket sizing |
| `server.conf` | All tiers | TLS, clustering, SHC, Deployment Server client, shared secrets (pass4SymmKey) |
| `authorize.conf` | Search Head | RBAC: role definitions, capabilities, index access |
| `serverclass.conf` | Deployment Server | Forwarder groupings and app assignments |
| `limits.conf` | Search Head & Indexer | Search concurrency, memory caps, command-specific limits |
| `alert_actions.conf` | Search Head | Custom alert action definitions |
| `commands.conf` | Search Head | Custom search command registration |
| `app.conf` | All apps | App metadata, version, build, UI visibility |
| `deploymentclient.conf` | Forwarder | Deployment Server connection target |

---

## Configuration Precedence (Highest → Lowest)

```
1. $SPLUNK_HOME/etc/system/local/           ← System overrides
2. $SPLUNK_HOME/etc/apps/<app>/local/       ← App overrides  ← PUT CHANGES HERE
3. $SPLUNK_HOME/etc/apps/<app>/default/     ← App defaults (shipped)
4. $SPLUNK_HOME/etc/system/default/         ← Splunk built-ins (never edit)
```

**Rule:** Never edit `/default/` — it is overwritten on upgrade.

---

## Bucket Lifecycle

```
hot  →  warm  →  cold  →  frozen
                          (deleted or archived)
                     ↗
                 thawed
               (restored for search)
```

| State | Writable | Searchable | Typical Storage |
|-------|----------|------------|-----------------|
| hot | ✅ | ✅ | Fast SSD |
| warm | ❌ | ✅ | SSD / SAS |
| cold | ❌ | ✅ | HDD / NAS |
| frozen | ❌ | ❌ (unless thawed) | Archive / S3 |
| thawed | ❌ | ✅ | Any |

---

## Clustering Quick Reference

### Indexer Cluster
| Concept | Description |
|---------|-------------|
| Replication Factor (RF) | Total number of bucket copies across all peers |
| Search Factor (SF) | Number of **searchable** copies (SF ≤ RF) |
| Cluster Manager | Coordinates cluster; does NOT index data |
| Minimum viable | RF=2, SF=2, ≥2 peers + 1 manager |
| Production recommendation | RF=3, SF=2, ≥3 peers |

### Search Head Cluster (SHC)
| Concept | Description |
|---------|-------------|
| Captain | Elected coordinator for scheduled searches |
| Deployer | Separate instance that pushes apps to SHC members |
| Replication port | 34777 (default) |
| Sweet spot | 3–5 members; beyond 7 degrades performance |

---

## SPL Command Reference (Most Common)

### Aggregation & Statistics
```spl
| stats count, avg(field), max(field), values(field) by group_field
| eventstats avg(duration) as avg_dur by host    # adds column without reducing rows
| streamstats count by session_id                # running count per group
| timechart span=1h count by sourcetype
```

### Transformations
```spl
| eval status=if(code>=400, "error", "ok")
| eval label=case(code<200,"info", code<300,"ok", code<400,"redirect", true(),"error")
| rex field=_raw "(?P<user>[a-z]+)@(?P<domain>[^\"]+)"
| rename old_field AS new_field
| fields - unwanted_field1, unwanted_field2
```

### Lookups
```spl
| lookup asset_inventory.csv ip OUTPUT hostname, owner
| inputlookup threat_intel.csv | where category="malware"
| outputlookup new_lookup.csv   # write current results to a lookup file
```

### Subsearches & Joins
```spl
index=web [ search index=security action=blocked | fields src_ip ]
| join type=left src_ip [ search index=asset_db | fields src_ip, hostname ]
```

### Filtering & Deduplication
```spl
| where duration > 5000
| dedup src_ip, dest_port sortby -_time
| top limit=10 url by status_code
```

### Performance Tips
- Use `tstats` instead of `stats` on accelerated data models — 10–100× faster
- Push extractions to `props.conf` EXTRACT- transforms instead of inline `rex`
- Avoid `| transaction` for large datasets — use `stats` instead
- Always set `index=`, `sourcetype=`, and a time range before piping

---

## Key SPL Functions (eval)

| Function | Example | Result |
|----------|---------|--------|
| `if(X, Y, Z)` | `if(status=200, "ok", "fail")` | conditional |
| `case(...)` | `case(code<400,"ok", true(),"err")` | multi-branch |
| `coalesce(A, B)` | `coalesce(user, src_ip)` | first non-null |
| `mvindex(mv, 0)` | `mvindex(split(path,"/"), -1)` | multivalue index |
| `strftime(X, fmt)` | `strftime(_time, "%Y-%m-%d")` | time → string |
| `strptime(X, fmt)` | `strptime(date_str, "%m/%d/%Y")` | string → epoch |
| `len(X)` | `len(user)` | string length |
| `match(X, regex)` | `match(url, "admin")` | boolean regex |
| `urldecode(X)` | `urldecode(encoded_url)` | URL decode |
| `cidrmatch(cidr, ip)` | `cidrmatch("10.0.0.0/8", src)` | CIDR match |

---

## CLI Quick Commands

```bash
# Service management
splunk start | stop | restart | status

# Search from CLI
splunk search 'index=_internal | head 5' -auth admin:password

# Apply cluster bundle (indexer cluster)
splunk apply cluster-bundle -answer-yes

# Apply SHC bundle
splunk apply shcluster-bundle -target https://captain:8089 -auth admin:password

# Check Deployment Server clients
splunk list deploy-clients

# Validate a config file
splunk btool inputs list --debug
splunk btool props list sourcetype=syslog --debug

# Check index disk usage
splunk list index

# Start with a specific conf dir (for testing)
SPLUNK_HOME=/opt/splunk splunk start
```

---

## Enterprise Security (ES) Quick Reference

| Component | Description |
|-----------|-------------|
| Correlation Search | Scheduled SPL that generates Notable Events when conditions match |
| Notable Event | Alert instance in the Incident Review queue |
| Risk Score | Cumulative score attached to a system/user identity |
| Risk-Based Alerting (RBA) | Aggregates risk contributions before alerting; reduces alert fatigue |
| Threat Intelligence | IOC framework (IP, domain, hash, email); ingested via threat_intel modular input |
| UEBA | User/Entity Behavior Analytics; anomaly scoring over time |
| `risk` index | Stores risk events from contributing correlation searches |
| Asset & Identity | `asset_lookup`, `identity_lookup` — critical for context enrichment |
