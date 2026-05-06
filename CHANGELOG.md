# Changelog

All notable changes to the **superman-splunk** skill will be documented here.

## [1.0.0] — 2026-05-06

### Added
- Initial release of the superman-splunk skill
- `SKILL.md` — main entry point with domain routing table, quick-reference tables (ports, conf files, bucket lifecycle, topology decision tree)
- `references/architecture.md` — 3-tier architecture, forwarder types (UF/HF), indexing pipeline (queue-based), bucket lifecycle, SmartStore, indexer clustering (RF/SF), SHC captain/deployer, Deployment Server, config merge order, performance tuning (THP, parallelIngestionPipelines, limits.conf)
- `references/spl-guide.md` — full SPL command reference, eval functions, rex, stats vs transaction memory model, lookups (CSV/KV Store), macros, event types, tags, workflow actions, tstats vs stats, dashboard types (Classic/Dashboard Studio)
- `references/development.md` — app/TA directory structure, custom commands (V1 and V2 chunked protocol), modular inputs (Python Script class), alert actions, REST API reference (160+ endpoints), Python SDK patterns, CIM normalization (vital 20 fields), data model acceleration
- `references/administration.md` — configuration file precedence, inputs.conf (all stanza types), props.conf (Big 8 settings), transforms.conf (field extractions, routing, masking), indexes.conf (sizing, SmartStore), server.conf (TLS, clustering), Deployment Server serverclass.conf, license management, Monitoring Console dashboards
- `references/security-es.md` — RBAC (authorize.conf, capability model), TLS/SSL configuration, Enterprise Security (correlation searches, notable events, RBA, threat intelligence framework, UEBA), SOAR integration, ITSI (services, KPIs, Episode pages), Splunk Cloud vs Enterprise comparison, hardening checklist
- `evals/evals.json` — 5 benchmark eval prompts across architecture, SPL, development, administration, and ES domains

### Benchmark (v1.0.0)
| Config | Pass Rate |
|--------|-----------|
| With Skill | 96% |
| Without Skill | 88% |
| Delta | +8pp |
