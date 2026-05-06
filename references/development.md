# Splunk Development & Customization

## App & Add-on Structure

All Splunk apps and add-ons (TAs) live under `$SPLUNK_HOME/etc/apps/<appname>/`.

```
myapp/
├── app.conf              # App metadata, versioning
├── default/              # Shipped configuration (read-only in operation)
│   ├── inputs.conf       # Input definitions
│   ├── transforms.conf   # Field extractions, lookups
│   ├── props.conf        # Sourcetype parsing
│   ├── savedsearches.conf
│   ├── eventtypes.conf
│   ├── tags.conf
│   ├── macros.conf
│   ├── alert_actions.conf
│   └── data/
│       └── ui/
│           ├── nav/default.xml      # Navigation bar
│           └── views/               # Dashboard XML files
├── local/                # Runtime overrides (never ship this)
├── metadata/
│   ├── default.meta      # Sharing permissions for default objects
│   └── local.meta        # Sharing permissions for local objects
├── lookups/              # CSV lookup files
├── bin/                  # Scripts, custom commands
│   ├── linux_x86_64/     # Platform-specific binaries
│   ├── darwin_x86_64/
│   └── windows_x86_64/
├── appserver/
│   └── static/           # CSS, JS, images served by Splunk Web
│       ├── appIcon.png    # 36x36 app icon
│       └── appIcon_2x.png # 72x72 retina icon
└── README/               # Spec files, documentation
```

### app.conf
```ini
[launcher]
author = My Name
description = Description of the app

[package]
id = myapp
check_for_updates = false

[ui]
is_visible = true
label = My App

[install]
build = 1                # Increment to bust browser cache for static assets
```

### metadata/default.meta
Controls sharing and access for shipped objects:
```ini
# Make all saved searches globally visible
[savedsearches]
export = system

# Restrict a specific lookup
[lookups/sensitive.csv]
access = read : [ admin ], write : [ admin ]
export = none
```

---

## Custom Search Commands

Custom commands extend SPL when native commands are insufficient. Use **Python** with the Splunk SDK and the **Splunk Python SDK's `splunklib.searchcommands`** module.

### Command Types
| Type | Description | Runs on |
|------|-------------|---------|
| **Generating** | Produces events from scratch (no input) | Search head |
| **Streaming** | Transforms each event independently | Indexer or SH |
| **Reporting** | Aggregates events into summary | Search head |
| **Eventing** | Returns ordered events (like sort/dedup) | Search head |

### V2 Protocol (Required for New Commands)
```ini
# commands.conf
[mycommand]
filename = mycommand.py
chunked = true           # V2 protocol
python.version = python3
```

### Example: Streaming Command
```python
# bin/mycommand.py
import sys
import os
sys.path.insert(0, os.path.join(os.path.dirname(__file__), '..', 'lib'))
from splunklib.searchcommands import dispatch, StreamingCommand, Configuration, Option, validators

@Configuration()
class MyCommand(StreamingCommand):
    multiplier = Option(require=False, default=1, validate=validators.Integer())

    def stream(self, events):
        for event in events:
            event['multiplied'] = int(event.get('count', 0)) * self.multiplier
            yield event

dispatch(MyCommand, sys.argv, sys.stdin, sys.stdout, __name__)
```

### Security: Credentials in Custom Commands
- Commands needing API keys or passwords **must run on the search head** — not indexers.
- Store credentials in Splunk's encrypted credential store:
  ```python
  import splunklib.client as client
  service = client.connect(token=self._metadata.searchinfo.session_key)
  creds = service.storage_passwords.list()
  ```
- Never hardcode credentials in scripts.
- Set `required_fields` and `run_in_preview = false` for commands that make external calls.

---

## Modular Inputs

Modular inputs are custom data collection plugins — they generate events and feed them into Splunk's ingestion pipeline.

### Structure
```
myapp/
├── default/inputs.conf   # Input stanza definition for UI
├── bin/myinput.py        # The input script
└── README/inputs.conf.spec  # Parameter spec for validation
```

### inputs.conf.spec
```
[myinput://default]
*This is my custom input.
api_endpoint = <string>
interval = <integer>
```

### Modular Input Script Pattern
```python
#!/usr/bin/env python3
import sys
import os
sys.path.insert(0, os.path.join(os.path.dirname(__file__), '..', 'lib'))
from splunklib.modularinput import Script, Scheme, Argument, Event

class MyInput(Script):
    def get_scheme(self):
        scheme = Scheme("My Input")
        scheme.description = "Collects data from My API"
        scheme.add_argument(Argument("api_endpoint", required_on_create=True))
        return scheme

    def stream_events(self, inputs, ew):
        for input_name, input_item in inputs.inputs.items():
            endpoint = input_item["api_endpoint"]
            # Fetch data...
            event = Event()
            event.stanza = input_name
            event.data = '{"key": "value"}'
            ew.write_event(event)

if __name__ == "__main__":
    sys.exit(MyInput.run(sys.argv))
```

---

## Custom Alert Actions

Alert actions run when a saved search fires an alert. They receive search results as a payload via stdin.

### alert_actions.conf
```ini
[my_alert_action]
label = My Alert Action
description = Sends alert to external system
icon_path = my_icon.png
payload_format = json
is_custom = 1
```

### Alert Action Script
```python
#!/usr/bin/env python3
import sys
import json
import logging

def process_alert(payload):
    config = payload.get('configuration', {})
    results_file = payload.get('results_file', '')
    search_name = payload.get('search_name', '')
    
    # Read results
    with open(results_file, 'r') as f:
        results = json.load(f)
    
    # Process events in results['result']
    for event in results.get('result', []):
        # Send to external system...
        pass

if __name__ == "__main__":
    payload = json.loads(sys.stdin.read())
    process_alert(payload)
```

Alert action scripts run on the **search head** — they have access to the search context and credentials store.

---

## REST API

Splunk exposes 160+ REST endpoints on the **management port (default 8089)**.

### Authentication
```bash
# Session token (basic auth → get token)
curl -k -u admin:password https://splunk:8089/services/auth/login \
  -d "username=admin&password=password" \
  -d "output_mode=json"

# Use token in subsequent requests
curl -k -H "Authorization: Splunk <token>" \
  https://splunk:8089/services/search/jobs?output_mode=json
```

### Key REST Endpoints
| Endpoint | Method | Purpose |
|----------|--------|---------|
| `/services/search/jobs` | POST | Create search job |
| `/services/search/jobs/<sid>` | GET | Check job status |
| `/services/search/jobs/<sid>/results` | GET | Get results |
| `/services/search/jobs/export` | GET | Streaming search export |
| `/services/data/inputs/monitor` | GET/POST | Manage file inputs |
| `/services/data/indexes` | GET/POST | List/create indexes |
| `/servicesNS/<user>/<app>/saved/searches` | GET/POST | Manage saved searches |
| `/services/storage/passwords` | GET/POST | Credential store |
| `/services/apps/local` | GET/POST | App management |

### Search Execution Modes
| Mode | Behavior | Use Case |
|------|----------|---------|
| `exec_mode=normal` | Async; poll for completion | Long-running searches |
| `exec_mode=blocking` | Synchronous; wait for completion | Scripted short queries |
| `exec_mode=oneshot` | Single roundtrip, inline results | Quick queries, small result sets |

```bash
# Oneshot search
curl -k -H "Authorization: Splunk <token>" \
  https://splunk:8089/services/search/jobs \
  -d "search=search index=main earliest=-1h | stats count" \
  -d "exec_mode=oneshot" \
  -d "output_mode=json"
```

### Namespace
All REST calls are namespace-aware: `owner + app + sharing_mode`. Use:
- `/services/` — system namespace (admin-level)
- `/servicesNS/<owner>/<app>/` — app-scoped namespace

---

## Python SDK

### Connection
```python
import splunklib.client as client

service = client.connect(
    host='splunk.example.com',
    port=8089,
    username='admin',
    password='password',
    # OR:
    # splunkToken='<token>'
)
```

### Running Searches
```python
# Oneshot — best for small, fast queries
results = service.jobs.oneshot(
    "search index=web status=500 | stats count by host",
    earliest_time="-1h",
    latest_time="now",
    output_mode="json"
)

# Normal/blocking — for longer searches
import splunklib.results as results_lib

job = service.jobs.create(
    "search index=web | stats count by status",
    earliest_time="-24h"
)
# Wait for completion
while not job.is_done():
    import time; time.sleep(1)
    job.refresh()

# Paginated results
reader = results_lib.JSONResultsReader(
    job.results(output_mode='json', count=100, offset=0)
)
for result in reader:
    if isinstance(result, dict):
        print(result)
```

### SDK Caching Behavior
**Critical**: SDK objects cache state locally. After making changes:
```python
# After updating an object, refresh to see server state
saved_search = service.saved_searches['my_search']
saved_search.update(search="new SPL here")
saved_search.refresh()  # Pull updated state from server
```

---

## CIM — Common Information Model

CIM normalizes field names and values across diverse data sources to enable consistent analytics.

### Why CIM Matters
- ES correlation searches, Splunk app content (e.g., Splunk App for AWS), and OOTB dashboards use CIM field names.
- Without CIM, every data source requires custom SPL — brittle and hard to maintain.
- CIM-compliant data works with all CIM-aware content out of the box.

### Core CIM Data Models (selected)
| Data Model | Key Fields | Use Case |
|------------|-----------|---------|
| `Authentication` | `user`, `src`, `dest`, `action` | Login/auth events |
| `Network_Traffic` | `src_ip`, `dest_ip`, `dest_port`, `bytes_in`, `bytes_out`, `action` | Firewall, netflow |
| `Web` | `src`, `dest`, `url`, `status`, `http_method`, `bytes` | Web/proxy logs |
| `Endpoint` | `user`, `dest`, `process`, `process_name` | EDR, Sysmon |
| `Malware` | `src`, `dest`, `signature`, `category`, `action` | AV events |
| `Alerts` | `src`, `dest`, `signature`, `severity` | IDS/IPS events |
| `Change` | `user`, `object`, `action`, `object_category` | Change/audit logs |

### Implementing CIM Normalization
```ini
# props.conf — alias source field to CIM field name
[my_sourcetype]
FIELDALIAS-cim_user = username AS user
FIELDALIAS-cim_src = src_host AS src
EVAL-action = if(event_type="login_fail", "failure", "success")

# transforms.conf — for complex extractions
[extract_cim_fields]
REGEX = user=(?P<user>[^\s]+)\s+host=(?P<src>[^\s]+)
SOURCE_KEY = _raw
```

**The "Vital 20" CIM fields to map first:** `src`, `dest`, `user`, `action`, `status`, `bytes`, `bytes_in`, `bytes_out`, `duration`, `protocol`, `dest_port`, `src_port`, `app`, `category`, `signature`, `severity`, `object`, `process`, `file_path`, `url`.

### Validating CIM Compliance
Use the **Splunk CIM Validator** (`SA-cim_validator`) from Splunkbase:
```spl
| datamodel Authentication search | head 10
```
If results return with CIM field names populated, your data is normalized.

---

## Data Models & Acceleration

Data models define hierarchical schemas over your data. Acceleration pre-computes `tsidx` summaries for fast `tstats` queries.

### Enabling Acceleration
In Settings → Data models → Edit → Edit Acceleration:
- **Summary Range**: how far back to summarize (e.g., 7 days).
- **Scheduling**: summarization runs every 5 minutes by default.
- **Concurrency**: raise to 75–100% of available search slots for faster backfill.

```ini
# datamodels.conf — programmatic acceleration
[Authentication]
acceleration = true
acceleration.earliest_time = -7d
acceleration.cron_schedule = */5 * * * *
```

**Backfill guidance**: Start with 4–12 hours of backfill, not full history — large backfills overwhelm the scheduler. Extend the range gradually.
