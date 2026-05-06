# SPL (Search Processing Language) & Knowledge Objects

## SPL Fundamentals

SPL is a **pipeline language**: each command feeds its output to the next.

```
<search terms> | command1 args | command2 args | command3 args
```

The leading search terms filter raw events from the index. Commands after `|` transform the result set.

### Search Best Practices
1. **Always specify index**: `index=web_logs` — avoids scanning every index.
2. **Narrow time range first**: use the time picker or `earliest=-1h latest=now`.
3. **Filter early**: put `WHERE`, `search`, or field filters before expensive commands.
4. **Most restrictive filter first**: reduces event volume for subsequent commands.

```spl
index=security sourcetype=firewall action=blocked
| stats count by src_ip, dest_port
| where count > 100
| sort -count
```

---

## Core SPL Commands

### Retrieval & Filtering
| Command | Purpose | Example |
|---------|---------|---------|
| `search` | Filter events | `search status=500` |
| `where` | Filter with eval expressions | `where len(url) > 100` |
| `head` / `tail` | First/last N results | `head 100` |
| `dedup` | Remove duplicate field values | `dedup session_id` |
| `fields` | Include/exclude fields | `fields + src_ip, dest_ip` |
| `rename` | Rename fields | `rename src_ip AS source` |
| `table` | Display specific columns | `table _time, src_ip, action` |

### Transforming Commands
| Command | Purpose | Notes |
|---------|---------|-------|
| `eval` | Create/derive fields | Most versatile command |
| `rex` | Regex field extraction | Expensive; use `props.conf` for persistence |
| `erex` | Auto-regex extraction | Useful for discovery |
| `extract` | Apply persistent field extractions | Runs existing transforms |
| `convert` | Type conversion | `convert timeformat="%Y-%m-%d" ctime(_time)` |
| `lookup` | Enrich events from CSV or KV Store | `lookup geoip.csv ip AS src_ip` |
| `inputlookup` / `outputlookup` | Read/write lookup tables | Full table operations |
| `mvexpand` | Expand multivalue field to rows | Essential for array-valued fields |
| `makemv` | Split field into multivalue | `makemv delim="," tags` |

### Aggregation Commands
```spl
# stats — aggregate over entire result set
| stats count, avg(bytes) AS avg_bytes BY host, sourcetype

# timechart — stats over time (x-axis is _time)
| timechart span=1h count BY status

# chart — stats with two grouping dimensions
| chart avg(response_time) OVER host BY status

# top / rare — most/least frequent values
| top limit=20 url
```

**`stats` vs `transaction`:**
- `stats`: memory-efficient, fast, operates on single events or aggregated values. **Prefer this.**
- `transaction`: groups related events into a single event by shared field + time window. Memory-intensive; only use when you genuinely need the grouped event.

```spl
# DON'T — slow and memory-heavy
| transaction session_id maxspan=30m

# DO — if you just need duration
| stats min(_time) AS start, max(_time) AS end BY session_id
| eval duration = end - start
```

### Eval Functions
```spl
| eval field = expression

# String functions
| eval lower_url = lower(url)
| eval upper_host = upper(host)
| eval trimmed = trim(message)
| eval substr_val = substr(field, 1, 5)
| eval len_val = len(field)
| eval joined = field1 . " " . field2       # string concatenation

# Math
| eval ratio = bytes / requests
| eval rounded = round(value, 2)

# Conditional
| eval severity = if(status >= 500, "error", if(status >= 400, "warn", "ok"))
| eval category = case(
    status >= 500, "server_error",
    status >= 400, "client_error",
    status >= 300, "redirect",
    1=1, "success")

# Null checks
| eval has_value = if(isnull(field), "no", "yes")
| eval coalesced = coalesce(field1, field2, "default")

# Time
| eval human_time = strftime(_time, "%Y-%m-%d %H:%M:%S")
| eval epoch = strptime("2024-01-01", "%Y-%m-%d")
| eval age_days = (now() - _time) / 86400
```

### Rex (Regex Extraction)
```spl
# Named group extraction — adds field to event
| rex field=_raw "user=(?<username>[^\s]+)\s+action=(?<action>\w+)"

# Mode sed — string replacement (for masking)
| rex field=email mode=sed "s/(@.+)/REDACTED/g"
```

**Performance note**: `rex` runs at search time and re-parses every event. For recurring extractions, define them in `props.conf` instead:
```ini
[source::...log]
EXTRACT-username = user=(?P<username>[^\s]+)
```

---

## Lookups

Lookups enrich events with additional fields from an external table.

### CSV Lookup
```ini
# transforms.conf
[geoip_lookup]
filename = geoip.csv
case_sensitive_match = false

# props.conf — automatic lookup
[source::firewall*]
LOOKUP-geoip = geoip_lookup ip AS src_ip OUTPUT country, city
```

```spl
# Manual lookup in SPL
| lookup geoip.csv ip AS src_ip OUTPUT country, city
```

### KV Store Lookup
Use KV Store for:
- Large lookup tables (>10 MB) — avoids bundle replication.
- Frequently updated data — CSV lookups require file replacement.
- Lookups updated programmatically via REST API.

```ini
# transforms.conf
[asset_lookup]
collection = assets_collection
external_type = kvstore
fields_list = ip, hostname, owner, department
```

---

## Knowledge Objects

### Macros
Reusable SPL fragments. Essential for DRY (Don't Repeat Yourself) search development.

```ini
# macros.conf
[base_web_search(1)]
definition = index=web sourcetype=access_combined host=$host$
args = host
iseval = false
```

```spl
# Usage
`base_web_search("webserver01")` | stats count by status
```

### Event Types
Save a search condition as a named, reusable classification.

```ini
# eventtypes.conf
[failed_login]
search = sourcetype=auth action=failure
priority = 1
```

```spl
# Events now have field eventtype="failed_login"
sourcetype=auth | where eventtype="failed_login"
```

### Tags
Apply human-readable labels to field=value pairs.

```ini
# tags.conf
[eventtype=failed_login]
authentication = enabled
failure = enabled
```

```spl
# Search by tag
tag::authentication=failure
```

### Workflow Actions
UI-driven actions that launch follow-up searches or HTTP requests from event fields.

Configured in Settings → Fields → Workflow actions:
- **Search action**: run a new search using current event fields.
- **Link action**: open a URL with field values interpolated (e.g., VirusTotal lookup on IP).

### Field Extractions
Three methods:
1. **Regex (EXTRACT-)**: `EXTRACT-myfield = (?<fieldname>pattern)` in `props.conf`
2. **Delimiter (REPORT-)**: CSV, JSON, XML auto-parsing via transforms
3. **Calculated (EVAL-)**: `EVAL-severity = if(status>=500,"high","low")` in `props.conf`

```ini
# props.conf
[access_combined]
EXTRACT-status_code = \s(?P<http_status>\d{3})\s
EVAL-is_error = if(http_status >= 400, "true", "false")
```

---

## Dashboards

### SimpleXML (Classic Dashboards)
XML-based, stored in `$SPLUNK_HOME/etc/apps/<app>/local/data/ui/views/`.

```xml
<dashboard>
  <label>My Dashboard</label>
  <row>
    <panel>
      <chart>
        <title>Errors Over Time</title>
        <search>
          <query>index=web status>=500 | timechart count</query>
          <earliest>-24h</earliest>
          <latest>now</latest>
        </search>
        <option name="charting.chart">line</option>
      </chart>
    </panel>
  </row>
</dashboard>
```

### Dashboard Studio
Newer JSON-based experience with pixel-perfect layout, richer visualization control, and absolute/grid positioning.
- Created and edited in the Splunk Web UI editor.
- Stored as JSON in `data/ui/views/` with `version="1.1"` attribute.
- Supports dynamic inputs, drilldowns, and conditional formatting.

### Splunk Web Framework
For fully custom UI applications using JavaScript/React patterns:
- SplunkJS Stack with Backbone.js (legacy).
- React-based apps using `@splunk/react-ui` component library.
- Best for complex interactive apps that go beyond dashboard panels.

---

## `tstats` — High-Performance Aggregation

`tstats` queries **accelerated data models** and **tsidx index files** directly — much faster than searching raw events.

```spl
# Count web events using Network Traffic data model
| tstats count FROM datamodel=Web WHERE Web.status>=400
  BY Web.src_ip, Web.status, _time span=1h

# Using summariesonly=true — only uses accelerated summaries (fastest)
| tstats summariesonly=true count FROM datamodel=Authentication
  WHERE Authentication.action=failure
  BY Authentication.src, _time span=5m
```

**Rules for `tstats`:**
- Field names must use the **data model dotted notation** (`Web.url`, `Authentication.user`).
- `summariesonly=false` (default) falls back to raw if no summary — slower but complete.
- Data model must be accelerated for full benefit.

---

## Common SPL Patterns

### Top Talkers
```spl
index=firewall | stats sum(bytes) AS total_bytes BY src_ip | sort -total_bytes | head 20
```

### Error Rate Over Time
```spl
index=web | timechart span=5m count(eval(status>=500)) AS errors, count AS total
| eval error_rate = round(errors/total*100, 2)
```

### Joining Two Data Sources
```spl
# subsearch join (for small datasets)
index=web [search index=threat_intel | fields ip | rename ip AS client_ip]

# stats-based join (more scalable)
index=web | stats count BY client_ip
| join client_ip [search index=threat_intel | table ip, threat_category | rename ip AS client_ip]
```

### Session Analysis Without transaction
```spl
index=web
| stats min(_time) AS session_start, max(_time) AS session_end, count AS page_views BY session_id, user
| eval session_duration_min = round((session_end - session_start) / 60, 1)
| where session_duration_min > 0
```

### Field Presence Audit
```spl
index=myindex | fieldsummary | table field, count, distinct_count, is_exact, numeric_count
```
