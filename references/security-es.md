# Splunk Security, ES & Enterprise Features

## Role-Based Access Control (RBAC)

### Core Concepts
- **Roles** define capabilities (what a user can do) and indexes (what data they can see).
- **Users** are assigned one or more roles; they inherit the union of all role capabilities.
- Capabilities are atomic permissions: `search`, `edit_monitor`, `admin_all_objects`, etc.
- Roles are defined in `authorize.conf`.

### authorize.conf
```ini
[role_analyst]
importRoles = user                  # inherit base user capabilities
srchIndexesAllowed = web;security   # indexes this role can search
srchIndexesDefault = web            # default search scope
srchDiskQuota = 50000               # MB of disk per user for search artifacts
srchJobsQuota = 10                  # concurrent searches per user
rtSrchJobsQuota = 5                 # real-time searches per user

# Capabilities
grantableRoles = role_analyst       # can grant roles up to this level
capability::schedule_search = enabled
capability::edit_own_saved_searches = enabled
capability::list_inputs = enabled
# Do NOT grant: admin_all_objects, edit_monitor on shared roles

[role_security_analyst]
importRoles = role_analyst
srchIndexesAllowed = web;security;network;endpoint
capability::edit_correlationsearches = enabled   # ES capability
capability::ess_analyst = enabled                # ES analyst role
```

### RBAC Best Practices
- Use **modular roles**: build small, focused roles and compose them via `importRoles`.
- Map roles to **SAML/LDAP/AD groups** — manage group membership in the directory, not Splunk.
- Apply **least privilege**: grant only the indexes and capabilities a role genuinely needs.
- Audit with:
  ```spl
  index=_audit action=search user!=splunk-system-user
  | stats count by user, search | sort -count
  ```
- Use **index-based segmentation** to prevent cross-team data visibility.

### LDAP/AD Integration
```ini
# authentication.conf
[authentication]
authType = LDAP
authSettings = myLDAP

[myLDAP]
host = dc.example.com
port = 636
SSLEnabled = 1
bindDN = cn=splunk-bind,ou=service,dc=example,dc=com
bindDNpassword = <stored in credential vault>
userBaseDN = ou=users,dc=example,dc=com
userNameAttribute = sAMAccountName
realNameAttribute = displayName
groupBaseDN = ou=groups,dc=example,dc=com
groupMemberAttribute = member
```

Map LDAP groups to Splunk roles in Settings → Authentication → Map Groups.

---

## TLS/SSL Configuration

### server.conf — Management Port (8089)
```ini
[sslConfig]
enableSplunkdSSL = true
sslVersions = tls1.2, tls1.3
cipherSuite = ECDHE-RSA-AES256-GCM-SHA384:ECDHE-RSA-AES128-GCM-SHA256

# Server certificate (PEM, RSA-encrypted private key)
serverCert = $SPLUNK_HOME/etc/certs/splunkd.pem
sslRootCAPath = $SPLUNK_HOME/etc/auth/ca.pem

# Client certificate verification (optional but recommended)
requireClientCert = false
sslVerifyServerCert = true
sslVerifyServerName = true
```

### Forwarder → Indexer SSL (outputs.conf + inputs.conf)
```ini
# outputs.conf on forwarder
[tcpout:indexers]
server = idx1:9997, idx2:9997
sslCertPath = $SPLUNK_HOME/etc/certs/forwarder.pem
sslRootCAPath = $SPLUNK_HOME/etc/auth/ca.pem
sslVerifyServerCert = true
sslVerifyServerName = true

# inputs.conf on indexer — receiving with SSL
[splunktcp-ssl:9997]
disabled = false

[SSL]
serverCert = $SPLUNK_HOME/etc/certs/indexer.pem
sslRootCAPath = $SPLUNK_HOME/etc/auth/ca.pem
requireClientCert = true   # require forwarder to present cert
```

### Certificate Best Practices
- Use **PEM format** with **RSA-encrypted private keys**.
- Never use self-signed certs in production (except for inter-Splunk traffic with internal CA).
- Rotate certificates before expiry — expired certs break all Splunk communication.
- Store private keys with `chmod 400` owned by the splunk service account.

---

## Splunk Enterprise Security (ES)

ES is a premium app providing SIEM capabilities on top of the Splunk platform.

### Core ES Architecture
```
Data Sources → [CIM-Normalized Indexes] → [Data Models] → [ES Content]
                                                               ↓
                              Correlation Searches → Notable Events → Incident Review
                                                               ↓
                              Risk-Based Alerting → Risk Index → Risk Notable Events
```

### Correlation Searches
Scheduled SPL searches that detect suspicious patterns. When conditions match, they create **Notable Events** in the Incident Review dashboard.

```spl
-- Example: Detect brute force login
index=security sourcetype=auth action=failure
| stats count BY user, src
| where count > 10
```

Correlation search configuration:
- Scheduling: typically every 5–15 minutes.
- **Next Steps**: actions triggered on notable (Jira ticket, SOAR playbook, email).
- **Drilldown searches**: contextual SPL for analysts to investigate.

### Notable Events
Created when a correlation search fires. Key fields:
| Field | Description |
|-------|-------------|
| `rule_name` | Correlation search that fired |
| `src` | Source IP/host involved |
| `dest` | Destination IP/host |
| `user` | User involved |
| `severity` | Critical/High/Medium/Low/Informational |
| `status` | New/In Progress/Resolved/Closed |
| `owner` | Assigned analyst |
| `event_id` | Unique notable ID |

### Risk-Based Alerting (RBA)
Instead of direct alerts, correlations contribute **risk scores** to **risk objects** (users/systems). An alert fires only when accumulated risk exceeds a threshold.

```spl
-- Risk modifier: add +40 risk to a user for suspicious activity
| eval risk_object = user
| eval risk_object_type = "user"
| eval risk_score = 40
| eval risk_message = "Multiple failed logins from unusual location"
| collect index=risk
```

Risk factors allow conditional scoring:
- +5 for contractor/external users
- +10 if access occurs outside business hours
- +20 if source IP matches threat intel

**Benefits of RBA**: Reduces alert fatigue by aggregating low-confidence signals before alerting. Aligns naturally with MITRE ATT&CK tactics/techniques.

### Threat Intelligence
ES integrates threat intel feeds into 9 KV Store collections:
| Collection | Data Type |
|-----------|-----------|
| `ip_intel` | Malicious IP addresses |
| `domain_intel` | Malicious domains |
| `file_intel` | Malicious file hashes |
| `url_intel` | Malicious URLs |
| `user_intel` | Compromised users |
| `certificate_intel` | Suspicious certificates |
| `email_intel` | Malicious email addresses |
| `http_intel` | Malicious HTTP patterns |
| `process_intel` | Malicious process names/hashes |

Intel matches are written to `threat_activity` and feed notable events automatically.

Adding custom intel:
```spl
| inputlookup ip_intel_lookup
| append [| makeresults | eval ip="192.168.1.100", description="Internal threat actor", threat_key="custom"]
| outputlookup ip_intel_lookup
```

### MITRE ATT&CK Integration
ES provides MITRE ATT&CK framework alignment:
- Map correlation searches to specific techniques (T1110 for brute force, etc.).
- Use the ES MITRE ATT&CK framework view to visualize coverage.
- Tag `mitre_technique_id` in notable events.

---

## Splunk SOAR Integration

SOAR (Security Orchestration, Automation, and Response) automates analyst workflows via playbooks.

### ES → SOAR Flow
1. ES correlation search fires → creates notable event.
2. ES **Adaptive Response Action** sends notable to SOAR via webhook/API.
3. SOAR triggers a **playbook** based on event type/severity.
4. Playbook automatically: enriches (IP reputation lookup, user HR data), contains (block IP, disable account), notifies (Slack/Jira/email).
5. SOAR writes results back to ES notable event.

### Key SOAR Concepts
- **Playbooks**: Python-based or visual automation workflows.
- **Apps**: SOAR connectors for third-party tools (firewall, AD, ticketing, threat intel).
- **Containers**: SOAR's internal representation of security events.
- **Actions**: Individual automation steps (run query, block IP, create ticket).

---

## Splunk ITSI — IT Service Intelligence

ITSI provides service-health monitoring and ML-based anomaly detection.

### Core Concepts
| Concept | Description |
|---------|-------------|
| **Service** | Logical IT service (e.g., "Web Application", "Database Tier") |
| **KPI** | Key Performance Indicator measuring service health (e.g., response time, error rate) |
| **Health Score** | 0–100 calculated from KPI thresholds |
| **Episode** | Correlated alert grouping related service degradation events |
| **Glass Table** | Custom visual service topology dashboard |

### KPI Configuration
Each KPI is a saved SPL search returning a single metric value:
```spl
index=web sourcetype=access_combined
| stats avg(response_time_ms) AS avg_response_time
```

Thresholds define health score mapping:
- response_time < 200ms → 100 (normal)
- 200–500ms → 75 (warning)
- 500–1000ms → 50 (medium)
- > 1000ms → 25 (high severity)

### Predictive Analytics (Adaptive Thresholding)
ITSI can auto-learn normal baseline and flag anomalies using ML-based adaptive thresholds — removes need to manually set static thresholds.

---

## SmartStore Deep Dive

SmartStore enables massive-scale retention by moving data to object storage.

```ini
# indexes.conf — enable SmartStore for an index
[volume:remote_store]
storageType = remote
path = s3://my-splunk-bucket/data
remote.s3.endpoint = https://s3.amazonaws.com
remote.s3.access_key = <key>          # prefer IAM roles over static keys
remote.s3.secret_key = <secret>

[large_retention_index]
remotePath = volume:remote_store/$_index_name
# Hot buckets stay local; older warm buckets move to S3
maxGlobalRawDataSizeMB = 50000        # local cache size limit

[cachemanager]
maxCacheSize = 50000                  # MB of local cache for remote data
```

**SmartStore requirements:**
- Indexer clustering must be enabled.
- All indexer peers must have access to the same remote storage.
- Once an index is converted to SmartStore, it cannot be easily reverted.

---

## Splunk Cloud vs Enterprise

| Feature | Enterprise | Cloud |
|---------|-----------|-------|
| Infrastructure | Customer-managed | Splunk-managed |
| Custom apps | Full control | Vetted apps only (Splunk vetting required) |
| Config file access | Direct filesystem | Limited (via REST/UI) |
| Upgrade control | Customer-scheduled | Splunk-managed, less control |
| Data residency | Full control | Region selection available |
| HEC | Configurable | Available (port 443) |
| SmartStore | Self-configured | Included/managed |
| Compliance | Customer-responsible | SOC2, FedRAMP available |

**Cloud limitations to know:**
- Can't run arbitrary scripts or binaries — custom commands must be Python-only.
- Forwarder management is the same; data collection is identical.
- `inputs.conf` is managed via the Splunk UI or Deployment Server.
- Some advanced `limits.conf` settings are not user-configurable.

---

## Security Hardening Checklist

```
✅ Change default admin password immediately after install
✅ Create named user accounts; disable shared accounts
✅ Do NOT run splunkd as root (use dedicated service account)
✅ Enable SSL/TLS on all ports (Web:8000, mgmt:8089, forwarding:9997)
✅ Use sslVerifyServerCert=true and sslVerifyServerName=true
✅ Rotate pass4SymmKey (shared cluster secret) — requires cluster restart
✅ Implement modular RBAC; avoid granting admin_all_objects broadly
✅ Restrict Splunk Web to internal networks (firewall/LB rules)
✅ Enable audit logging: index=_audit
✅ Monitor for privilege escalation: searches in _audit for role changes
✅ Keep configs in version control (Git) — exclude sensitive files
✅ Use Splunk Secret Storage for credentials (never hardcode in conf files)
✅ Disable THP (Transparent Huge Pages) on Linux hosts
✅ Regular certificate rotation before expiry
✅ Validate cluster health before and after upgrades
```

### Audit Logging Queries
```spl
-- Failed login attempts
index=_audit action=login status=failure
| stats count BY user, src | sort -count

-- Admin privilege changes
index=_audit action=edit_roles OR action=edit_user
| table _time, user, action, info

-- Search activity monitoring
index=_audit action=search NOT user=splunk-system-user
| stats count, values(search) AS searches BY user
| sort -count
```
