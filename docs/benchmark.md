# Benchmark Methodology & Results

## Overview

The superman-splunk skill was benchmarked by comparing AI responses **with** vs **without** the skill across 5 real-world Splunk scenarios. Each scenario was designed to test a distinct knowledge domain and surface information that the base model would likely get wrong or miss entirely.

---

## Methodology

### Evaluation Design

For each eval scenario:
1. The same prompt was sent to the AI agent **twice** — once with the skill loaded, once without.
2. Each response was graded against **5 assertions** (25 total across all evals).
3. Assertions checked for presence of specific, accurate Splunk content — not just general correctness.

### Grading Criteria

Assertions tested for:
- **Presence of specific technical facts** (e.g., "mentions Transparent Huge Pages")
- **Completeness** (e.g., "provides practical from-scratch implementation steps, not just theory")
- **Splunk-specific accuracy** (e.g., correct conf file + stanza syntax)
- **Production-relevant guidance** (e.g., warns about cluster-wide vs instance-local scope)

---

## Eval Scenarios

### Eval 1 — Indexer Cluster Incomplete Results

> *"Our Splunk indexer cluster shows incomplete search results after one peer went down and came back. Some events are missing. Walk me through why this happens and how to fix it."*

**Focus**: Indexer clustering mechanics, bucket replication, fixup tasks, RF/SF concepts.

**Key assertions**:
- Explains RF vs SF distinction
- Mentions `splunk apply cluster-bundle` or fixup process
- Describes what "searchable" vs "replicated-only" copies mean
- Covers the cluster manager's role in fixup coordination
- Provides actionable recovery steps

---

### Eval 2 — Credential Stuffing Detection SPL

> *"Write me a Splunk SPL query to detect credential stuffing attacks. I have firewall logs in index=network and authentication logs in index=auth. Show me a complete, production-ready search."*

**Focus**: SPL expertise, security use-case, cross-index search, stats/time-window logic.

**Key assertions**:
- Uses both `index=network` and `index=auth`
- Implements time-window correlation (e.g., many failures followed by success)
- Uses `stats` (not `transaction`) for efficiency
- Includes threshold logic (count > N)
- Provides a complete, runnable query

---

### Eval 3 — Technology Add-On (TA) Modular Input Development

> *"Build me a Technology Add-On (TA) for Splunk that polls a REST API every 5 minutes and indexes the results. Include the directory structure, Python code for the modular input, the inputs.conf stanza, and show me how to handle credentials securely without hardcoding them."*

**Focus**: App/TA development, modular input framework, credential handling, V2 protocol.

**Key assertions**:
- Correct TA directory structure (`default/`, `bin/`, `metadata/`)
- Complete Python `Script` class with `get_scheme()` and `stream_events()`
- Correct `inputs.conf` stanza with `interval=300`
- Uses `storage_passwords` API (not hardcoded credentials)
- Mentions V2 chunked protocol for modern development

*Note: Both with-skill and without-skill agents missed the V2 protocol assertion — identified as a skill improvement opportunity.*

---

### Eval 4 — Indexing Pipeline Performance Troubleshooting

> *"Our Splunk indexing performance has degraded significantly. The Monitoring Console shows the typingQueue is consistently near 100% full on all indexers. Walk me through diagnosing the root cause and the specific settings I should tune."*

**Focus**: Indexing pipeline architecture, queue bottlenecks, Linux OS tuning, parallelIngestionPipelines.

**Key assertions**:
- Correctly identifies `typingQueue` saturation as a parsing-stage bottleneck
- Mentions **Transparent Huge Pages (THP)** disabling as a critical Linux OS fix
- Recommends `props.conf` optimization (SHOULD_LINEMERGE, MAX_TIMESTAMP_LOOKAHEAD)
- Provides diagnostic approach (Monitoring Console dashboards, btool)
- Mentions `parallelIngestionPipelines` in `server.conf`

*Differentiator: Only the with-skill agent mentioned THP. This is a well-known Splunk gotcha that the base model misses.*

---

### Eval 5 — Risk-Based Alerting (RBA) Implementation

> *"I want to implement Risk-Based Alerting (RBA) in Splunk Enterprise Security from scratch. My team is drowning in alert fatigue. Explain what RBA is, how it works in ES, and give me a step-by-step implementation plan."*

**Focus**: Enterprise Security, RBA workflow, risk index, correlation searches, alert fatigue reduction.

**Key assertions**:
- Explains RBA core concept (accumulate risk scores vs direct alerting)
- Explains risk-contributing correlation searches
- Mentions the `risk` index and how risk events are stored
- Addresses alert fatigue reduction as the primary business goal
- Provides practical from-scratch implementation steps (not just theory)

*Differentiator: Only the with-skill agent provided concrete practical steps (e.g., creating risk factors, scoring model, conversion workflow). Baseline response was theory-heavy.*

---

## Results

| Eval | With Skill | Without Skill | Differentiator |
|------|:----------:|:-------------:|----------------|
| Eval 1 (Clustering) | 5/5 ✅ | 5/5 ✅ | Both strong — base model knows clustering |
| Eval 2 (SPL) | 5/5 ✅ | 5/5 ✅ | Both strong — SPL within base model knowledge |
| Eval 3 (TA Dev) | 4/5 ⚠️ | 4/5 ⚠️ | Both miss V2 protocol — skill improvement needed |
| Eval 4 (Performance) | 5/5 ✅ | 4/5 ❌ | Skill surfaces THP; baseline misses it |
| Eval 5 (RBA) | 5/5 ✅ | 4/5 ❌ | Skill gives practical steps; baseline gives theory |

**Summary:**

| Metric | With Skill | Without Skill | Delta |
|--------|:----------:|:-------------:|:-----:|
| Pass Rate | **96%** | **88%** | **+8pp** |
| Avg Response Time | 273s | 497s | 224s faster |

---

## Skill Gap Identified

**V2 Custom Command Protocol** — neither with-skill nor without-skill agents surfaced the V2 chunked protocol (`chunked=true` in `commands.conf`) in the TA development eval. The information exists in `references/development.md` but is not prominent enough to be reliably surfaced. This is tracked in the [roadmap](../README.md#-roadmap).
