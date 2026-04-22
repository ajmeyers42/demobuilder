# demobuilder

Elastic demobuilder — a reusable agent pipeline for turning customer discovery into working demos.

## What This Is

A collection of skills that take a sales engagement from raw discovery notes through to a built, deployed, and validated Elastic demo. Each skill is a discrete step in the pipeline; they're designed to be used independently or chained together by the `demobuilder` orchestrator.

## Pipeline Overview

```
                    ╔══════════════════════════════════════╗
                    ║        demobuilder orchestrator       ║
                    ║  drop any inputs · stages auto-run   ║
                    ║  skips completed · resumes on re-run ║
                    ╚══════════════╤═══════════════════════╝
                                   │
           ┌───────────────────────┴─────────────────────┐
           │                                             │
    ┌──────▼───────────────┐               ┌─────────────▼──────────────┐
    │   Discovery Notes    │               │     Diagnostic File        │
    │   PDF · md · text    │               │   ZIP · API exports        │
    └──────┬───────────────┘               └─────────────┬──────────────┘
           │                                             │
    ┌──────▼───────────────┐               ┌─────────────▼──────────────┐
    │ demo-discovery-      │               │  demo-diagnostic-          │
    │ parser               │               │  analyzer        optional  │
    │                      │               │                            │
    │ → discovery.json     │               │ → current-state.json       │
    │ → confirmation.md    │               │ → findings.md              │
    │ → gaps.md            │               └─────────────┬──────────────┘
    └──────┬───────────────┘                             │
           └───────────────────┬─────────────────────────┘
                               │
                    ┌──────────▼──────────┐
                    │  demo-platform-     │
                    │  audit              │
                    │                     │
                    │ → platform-audit    │
                    │   .json / .md       │
                    └──────────┬──────────┘
                               │
                    ┌──────────▼──────────┐
                    │  demo-script-       │
                    │  template           │
                    │                     │
                    │ → demo-script.md    │
                    │ → demo-brief.md     │
                    └──────────┬──────────┘
                               │
                    ┌──────────▼──────────┐
                    │  demo-data-         │
                    │  modeler            │
                    │                     │
                    │ → data-model.json   │
                    │ → mapping files     │
                    └──────────┬──────────┘
                               │
                    ┌──────────▼──────────┐
                    │  demo-ml-           │
                    │  designer           │  ← conditional
                    │                     │    ML scenes only
                    │ → ml-config.json    │
                    └──────────┬──────────┘
                               │
                    ┌──────────▼──────────┐
                    │  demo-validator     │  ← always runs last
                    │                     │
                    │ → demo-checklist.md │
                    │ → risks.md          │
                    └──────────┬──────────┘
                               │
           ┌───────────────────┴──────────────────────────┐
           │              deploy phase  (optional)         │
           │                                               │
    ┌──────▼───────────────┐               ┌──────────────▼──────────┐
    │  demo-cloud-         │               │  demo-deploy            │
    │  provision           ├──────────────►│                         │
    │                      │               │ → bootstrap.py          │
    │  new cluster or      │               │ → deploy-log.md         │
    │  copy existing .env  │               └──────────────┬──────────┘
    │  → .env              │                              │
    └──────────────────────┘                              │
                                          ┌───────────────┴────────────┐
                                          │                            │
                               ┌──────────▼──────────┐  ┌─────────────▼──────┐
                               │   demo-status       │  │   demo-teardown    │
                               │   readiness check   │  │   post-demo        │
                               │   any time pre-demo │  │   cleanup          │
                               └─────────────────────┘  └────────────────────┘
```

## Agent runtimes (Cursor, Claude, others)

Demobuilder is **agent-first**: one shared [`skills/`](skills/) tree; engagement outputs live under **`~/engagements/{slug}/`** by default (or **`$DEMOBUILDER_ENGAGEMENTS_ROOT/{slug}/`** if you override the root — see [`docs/engagements-path.md`](docs/engagements-path.md)). No parallel `cursor/` vs `claude/` skill copies — only thin glue:

| File | Purpose |
|------|---------|
| [`AGENTS.md`](AGENTS.md) | What the assistant should do (orchestrator, engagement root env var, approvals) |
| [`.cursor/rules/demobuilder.mdc`](.cursor/rules/demobuilder.mdc) | Cursor rule pointing at the orchestrator |
| [`docs/runtimes/cursor.md`](docs/runtimes/cursor.md) | Using this repo in Cursor (includes **SLO / integration packages** → **[elastic/integration-packages-slo](https://github.com/elastic/integration-packages-slo)**) |
| [`docs/runtimes/claude.md`](docs/runtimes/claude.md) | Claude Code / Claude projects |

## Quick Start

Drop a discovery note (PDF, markdown, raw text) into a prompt and say **"build the demo for [company]"**. The `demobuilder` orchestrator handles the rest — detecting available inputs, running each pipeline stage in order, and delivering a complete workspace with all demo artifacts.

**Default:** engagement data lives under **`~/engagements/`** (i.e. **`$HOME/engagements`**). Set **`DEMOBUILDER_ENGAGEMENTS_ROOT`** only if you want a different parent directory. Each engagement is a subfolder — not inside this git repo:

```
$DEMOBUILDER_ENGAGEMENTS_ROOT/   # defaults to $HOME/engagements when unset
└── {customer-slug}/                 ← one folder per engagement (demo-specific)
    ├── .env                                    ← cluster credentials (never commit)
    ├── .env.example                            ← template, safe to copy
    ├── {customer-slug}-discovery.json          ← structured customer profile
    ├── {customer-slug}-confirmation.md         ← send to customer
    ├── {customer-slug}-gaps.md                   ← internal follow-up questions
    ├── {customer-slug}-platform-audit.json      ← feature feasibility matrix
    ├── {customer-slug}-demo-script.md            ← full SE script
    ├── {customer-slug}-demo-brief.md             ← one-page AE brief
    ├── {customer-slug}-data-model.json           ← index mappings + build order
    ├── {customer-slug}-ml-config.json            ← ML job configs (if applicable)
    ├── {customer-slug}-demo-checklist.md         ← pre-demo checklist (timed)
    ├── {customer-slug}-risks.md                  ← risks and fallbacks
    ├── bootstrap.py                              ← single deploy driver (ES + data + ML + Kibana API import)
    ├── {customer-slug}-deploy-log.md             ← what was created, doc counts
    ├── kibana-objects/                           ← optional: committed .ndjson (dashboards, etc.)
    ├── kibana/                                   ← optional: workflows JSON/YAML, agent JSON (imported by bootstrap)
    └── elasticsearch/                            ← optional: declarative ES JSON (if not inlined in bootstrap)
```

This repository contains [`engagements/README.md`](engagements/README.md) as a pointer only.

## Pipeline

| Skill | Input | Output | Status |
|---|---|---|---|
| `demobuilder` | Any combination of inputs | Runs full pipeline, delivers handoff summary | ✅ v1 |
| `demo-discovery-parser` | Discovery notes (PDF, markdown, raw text) | `{slug}-discovery.json`, `{slug}-confirmation.md`, `{slug}-gaps.md` | ✅ v1 |
| `demo-diagnostic-analyzer` | Elastic diagnostic ZIP or API exports | `{slug}-current-state.json`, `{slug}-architecture.md`, `{slug}-findings.md` | ✅ v1 |
| `demo-platform-audit` | Discovery JSON + optional diagnostic | `{slug}-platform-audit.json`, `{slug}-platform-audit.md` | ✅ v1 |
| `demo-script-template` | Discovery JSON + platform audit | `{slug}-demo-script.md`, `{slug}-demo-brief.md` | ✅ v1 |
| `demo-data-modeler` | Demo script + discovery JSON | `{slug}-data-model.json`, `{slug}-data-model.md`, mapping files | ✅ v1 |
| `demo-ml-designer` | Demo script + data model | `{slug}-ml-config.json`, `{slug}-ml-setup.md` | ✅ v1 |
| `demo-validator` | All pipeline outputs | `{slug}-demo-checklist.md`, `{slug}-risks.md` | ✅ v1 |
| `demo-kibana-agent-design` | Demo script + discovery (Agent Builder in scope) | `{slug}-agent-builder-spec.md` (optional) | ✅ v1 |
| `demo-cloud-provision` | Deployment type, region, slug | `{slug}/.env`, `{slug}/.env.example`, `{slug}-provision-log.md` | ✅ v1 |
| `demo-deploy` | `.env` + data model + ML config | `bootstrap.py`, `{slug}-deploy-log.md` | ✅ v1 |
| `demo-status` | `.env` + data model + ML config | Terminal status report (✅/❌ per resource, paste-ready fix commands) | ✅ v1 |
| `demo-teardown` | `.env` + data model | `teardown.py`, `{slug}-teardown-log.md` | ✅ v1 |

## Key Design Principles

**Each skill is a specialist.** Drop into any stage of the pipeline independently — you don't need to run the full pipeline every time. The orchestrator handles sequencing when you want it.

**Outputs feed inputs.** Every skill's JSON output is designed as a machine-readable input to the next skill. The discovery JSON drives the platform audit; the platform audit constrains the script; the script shapes the data model.

**Nothing is hallucinated.** Confirmation docs use only the customer's own language. Platform audits only clear features that are actually supported on the customer's platform. Scripts are grounded in specific pain points from discovery — not generic feature showcases.

**Stack version is explicit.** New cloud resources default to **latest GA** unless specified. Existing clusters require a **validated** version (diagnostic or `GET /`) before scripting and build artifacts; ES|QL and APIs are scoped accordingly. See `docs/decisions.md` D-020.

**Solution scope matches the customer.** Inputs include discovery notes, diagnostics, supplemental team notes, and sometimes **architecture diagrams**. Demos showcase **enterprise-appropriate** Elastic capabilities for the outcomes described — **search**, **Observability**, **Security**, or a **combined** storyline when that fits. See `docs/decisions.md` D-021.

**Solution first.** Scripts and plans lead with **business value** and the customer’s **key asks**, then the **capabilities** that deliver them — unless the SA directs otherwise. See `docs/decisions.md` D-022.

**Resumes intelligently.** The orchestrator inventories existing outputs before running. If you change one thing (e.g., the audience composition changes), it re-runs only the affected downstream stages and leaves everything else intact.

**Credentials stay local.** Each engagement workspace under `$DEMOBUILDER_ENGAGEMENTS_ROOT/{slug}/` has its own `.env` — never committed with the repo, never shared between customers unless you explicitly copy and update it. `INDEX_PREFIX` namespaces all resources when sharing a cluster across multiple demos.

**Kibana assets as files; review before deploy.** Save exported dashboards and related saved objects under `kibana-objects/` and/or `kibana/`; `bootstrap.py` imports them via Kibana APIs (see `docs/decisions.md` **D-024**, **D-017**). Do **not** run provision/deploy against a live cluster until the SA has reviewed `bootstrap.py`, platform audit, risks, and checklist — `bootstrap.py --dry-run` is fine without deploy. See `AGENTS.md`.

**Elastic truthfulness.** Every defined asset must be **deployable** on Elastic using supported APIs and **conform to Elasticsearch datatypes and product conventions** (mappings, Agent Builder tool params, rules, saved objects) — see `docs/decisions.md` **D-025**. Do not specify abstract types where the stack expects `keyword`, `text`, `date`, etc.

**Engagement tags (D-026).** Every deployed asset with a `tags` field carries `demobuilder:<engagement_id>` for discovery and cleanup — see `skills/demo-deploy/references/demobuilder-tagging.md`.

## Validation Coverage

| Skill | Validated Against |
|---|---|
| `demo-discovery-parser` | Sample discovery notes from multiple customer interactions across 4 verticals — 97.5% benchmark |
| `demo-diagnostic-analyzer` | Sample diagnostic exports from multiple customer environments (large-scale self-managed) |
| `demo-platform-audit` | Sample diagnostics from multiple customer self-managed deployments |
| `demo-script-template` | Sample discovery notes from multiple customer interactions (single-contact and executive-present scenarios) |
| `demo-data-modeler` | Sample discovery notes from multiple customer interactions (fraud detection use case) |
| `demo-validator` | Sample pipeline outputs from multiple customer interactions |
| `demo-cloud-provision` | Evals written — serverless project + shared cluster namespace copy |
| `demo-deploy` | Evals written — isolated cluster full deploy + shared cluster prefix deploy |
| `demo-status` | Evals written — readiness check + ML-focused readiness check |
| `demo-teardown` | Evals written — isolated cluster teardown + shared cluster prefix teardown |

## Repo Structure

```
demobuilder/
├── AGENTS.md                 ← agent behavior (orchestrator, DEMOBUILDER_ENGAGEMENTS_ROOT, approvals)
├── README.md
├── .cursor/
│   └── rules/
│       └── demobuilder.mdc   ← Cursor: load orchestrator + engagement path convention
├── docs/
│   ├── engagements-path.md   ← where per-customer folders live (outside git)
│   └── runtimes/             ← Cursor vs Claude setup (no duplicated skills)
├── engagements/
│   └── README.md             ← pointer only; real workspaces use $DEMOBUILDER_ENGAGEMENTS_ROOT
└── skills/
    ├── demobuilder/                        ← orchestrator
    │   ├── SKILL.md
    │   └── evals/evals.json
    ├── demo-discovery-parser/
    │   ├── SKILL.md
    │   └── evals/evals.json
    ├── demo-diagnostic-analyzer/
    │   ├── SKILL.md
    │   └── evals/evals.json
    ├── demo-platform-audit/
    │   ├── SKILL.md
    │   └── evals/evals.json
    ├── demo-script-template/
    │   ├── SKILL.md
    │   └── evals/evals.json
    ├── demo-data-modeler/
    │   ├── SKILL.md
    │   ├── evals/evals.json
    │   └── references/mapping-patterns.md  ← ES mapping syntax reference
    ├── demo-ml-designer/
    │   ├── SKILL.md
    │   └── evals/evals.json
    ├── demo-validator/
    │   ├── SKILL.md
    │   └── evals/evals.json
    ├── demo-kibana-agent-design/
    │   └── SKILL.md
    ├── demo-cloud-provision/
    │   ├── SKILL.md
    │   └── evals/evals.json
    ├── demo-deploy/
    │   ├── SKILL.md
    │   ├── evals/evals.json
    │   └── references/
    │       ├── env-reference.md            ← .env field docs + multi-customer workflow
    │       ├── demobuilder-tagging.md      ← D-026 engagement tag spec
    │       ├── serverless-differences.md   ← feature flags, ML, ELSER, Workflows, saved objects
    │       └── workflow-patterns.md        ← Kibana Workflows YAML + Liquid syntax
    ├── demo-status/
    │   ├── SKILL.md
    │   ├── demo_status.py
    │   ├── references/status-expected.example.json
    │   └── evals/evals.json
    └── demo-teardown/
        ├── SKILL.md
        └── evals/evals.json
```

## Dependencies

**elastic/agent-skills** (https://github.com/elastic/agent-skills) must be installed
**in full** alongside demobuilder — **Search, Observability, and Elastic Security** skills
are all part of the same plugin. Install everything so Security-heavy or hybrid demos work
without a second setup; search-only engagements simply do not invoke Security skills.

Representative skills used (non-exhaustive):

| Skill area | Examples | Purpose in demobuilder |
|---|---|---|
| Cloud | `cloud/setup`, `cloud/create-project`, `cloud/manage-project` | EC_API_KEY, provision, teardown |
| Kibana | `kibana/agent-builder`, `kibana/kibana-dashboards`, `kibana/kibana-connectors` | Deploy, dashboards, Workflows connectors |
| Observability | `observability-manage-slos`, `observability-logs-search`, … | SLOs, logs analysis when in scope |
| **Security** | `security-detection-rule-management`, `security-alert-triage`, `security-case-management`, `security-generate-security-sample-data`, … | Detection rules, triage, cases, sample data when demo is Sec or hybrid |
| Elasticsearch | `elasticsearch/elasticsearch-esql`, … | Queries, demo-status |

Run `cloud-setup` once to configure your Elastic Cloud API key before using
`demo-cloud-provision`. See `docs/todo.md` for the full setup checklist.

## Documentation

| File | Contents |
|---|---|
| `AGENTS.md` | Agent-first behavior: skills path, `$DEMOBUILDER_ENGAGEMENTS_ROOT` outputs, deploy approval |
| `docs/engagements-path.md` | Engagement root (`~/engagements` default) |
| `docs/runtimes/cursor.md` | Running demobuilder in Cursor |
| `docs/runtimes/claude.md` | Running demobuilder in Claude Code / Claude projects |
| `docs/postmortem.md` | Session post-mortem: lessons learned, friction points, design validation |
| `docs/decisions.md` | Architecture decision log with rationale |
| `docs/todo.md` | Open items requiring user action (installs, credentials, validations) |
| `docs/references-observability-slo.md` | Elastic Guide + API links for programmatic SLOs and burn-rate rules |
| `docs/references-kibana-apis.md` | Kibana Saved Objects + Alerting (rules & connectors); complements SLO reference |
| `skills/demo-deploy/references/env-reference.md` | `.env` field reference, multi-customer workflow, API key permissions |
| `skills/demo-deploy/references/demobuilder-tagging.md` | `demobuilder:<id>` on all tagged deploy assets (D-026) |
| `skills/demo-deploy/references/serverless-differences.md` | Serverless/ECH behavioral differences: feature flags, ML, ELSER, Workflows YAML, saved objects |
| `skills/demo-deploy/references/workflow-patterns.md` | Kibana Workflows YAML patterns and Liquid syntax reference |
| `skills/demo-status/demo_status.py` | NDJSON inventory + version check + tagged SLOs + agent + space + workflows (pre-demo checks) |
