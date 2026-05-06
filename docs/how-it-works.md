# How the Superman Splunk Skill Works

## Skill Architecture

AI skills (in the [Superpowers](https://github.com/superpowers-ai/superpowers) format) are structured Markdown files that give an AI assistant domain-specific knowledge and behavior guidelines. They work across Claude Code, GitHub Copilot CLI, and Gemini CLI.

The superman-splunk skill uses a **hub-and-spoke** design:

```
SKILL.md  (hub — always loaded)
    │
    ├── references/architecture.md    (loaded for arch/infra questions)
    ├── references/spl-guide.md       (loaded for SPL/search questions)
    ├── references/development.md     (loaded for app/TA/SDK questions)
    ├── references/administration.md  (loaded for conf/admin questions)
    └── references/security-es.md     (loaded for ES/SOAR/RBAC questions)
```

### Entry Point: SKILL.md

`SKILL.md` contains:

1. **YAML frontmatter** — the `description` field that the skill manager uses to decide *when* to activate the skill. The description uses natural language that triggers on Splunk-related user prompts.

2. **Domain routing table** — a Markdown table listing which reference file to read for each topic area.

3. **Expert behavior rules** — specific behavioral guidelines for the AI when answering Splunk questions (e.g., "always specify the conf file, stanza, and attribute", "warn about cluster-wide vs local scope").

4. **Quick reference tables** — port numbers, key conf files, bucket lifecycle — facts that are fast to reference without loading a full file.

### Reference Files: Lazy Loading

The AI only reads the reference files it needs for the current question. This keeps token usage efficient:

- A question about SPL → reads `spl-guide.md` only (~334 lines)
- A question about indexer clustering → reads `architecture.md` only (~241 lines)
- A broad "how do I set up ES from scratch?" → reads multiple files

Each reference file is self-contained with real configuration examples, not just conceptual descriptions.

## How the Knowledge Was Built

The skill's reference content was built through a three-stage automated research pipeline:

### Stage 1: Multi-Notebook Research (Google NotebookLM)

Three parallel research notebooks were created in Google NotebookLM, each focused on a distinct Splunk domain:

| Notebook | Domain | Sources |
|----------|--------|---------|
| Splunk Architecture & Core | Architecture, indexing, clustering, performance | ~128 web sources |
| Splunk Development | App dev, SDK, REST API, custom commands, CIM | ~80 web sources |
| Splunk Admin & Security | Administration, conf files, ES, SOAR, RBAC | ~85 web sources |

Each notebook was queried with 8–12 targeted questions per domain to extract structured findings.

### Stage 2: Synthesis

A synthesis agent merged the three sets of findings, deduplicated overlapping content, identified gaps, and organized the output into the five reference domains.

### Stage 3: Skill Authoring

The synthesized research was structured into the hub-and-spoke format:
- Technical facts were converted to working configuration examples
- Best practices were condensed into actionable rules in `SKILL.md`
- Content was validated against known Splunk behavior

## Evaluation Methodology

The skill was evaluated using 5 real-world Splunk scenarios:

1. **Indexer clustering** — "our cluster shows incomplete search results after a peer went down"
2. **SPL for security** — "detect credential stuffing from firewall logs"
3. **TA development** — "build a modular input TA that polls a REST API"
4. **Performance troubleshooting** — "indexing pipeline is slow, typingQueue saturated"
5. **Risk-Based Alerting** — "implement RBA in ES from scratch"

For each scenario, the AI was run twice — once with the skill and once without. Responses were graded against 5 assertions per scenario (25 total), checking for presence of specific Splunk-accurate content.

**Result:** Skill improved pass rate from 88% → 96% (+8 percentage points) across all scenarios.
