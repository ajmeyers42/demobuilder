# demobuilder

Elastic demobuilder — a reusable agent pipeline for turning customer discovery into working demos.

## What This Is

A collection of skills that take a sales engagement from raw discovery notes through to a built, deployed, and validated Elastic demo. Each skill is a discrete step in the pipeline; they're designed to be used independently or chained together by the `demobuilder` orchestrator.

## Quick Start

Drop a discovery note (PDF, markdown, raw text) into a prompt and say **"build the demo for [company]"**. The `demobuilder` orchestrator handles the rest — detecting available inputs, running each pipeline stage in order, and delivering a complete workspace with all demo artifacts.

```
demobuilder-workspace/
└── citizens-bank/
    ├── citizens-bank-discovery.json      ← structured customer profile
    ├── citizens-bank-confirmation.md     ← send to customer
    ├── citizens-bank-gaps.md             ← internal follow-up questions
    ├── citizens-bank-platform-audit.json ← feature feasibility matrix
    ├── citizens-bank-demo-script.md      ← full SE script
    ├── citizens-bank-demo-brief.md       ← one-page AE brief
    ├── citizens-bank-data-model.json     ← index mappings + build order
    ├── citizens-bank-ml-config.json      ← ML job configs (if applicable)
    └── citizens-bank-demo-checklist.md   ← pre-demo checklist (timed)
```

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

## Key Design Principles

**Each skill is a specialist.** Drop into any stage of the pipeline independently — you don't need to run the full pipeline every time. The orchestrator handles sequencing when you want it.

**Outputs feed inputs.** Every skill's JSON output is designed as a machine-readable input to the next skill. The discovery JSON drives the platform audit; the platform audit constrains the script; the script shapes the data model.

**Nothing is hallucinated.** Confirmation docs use only the customer's own language. Platform audits only clear features that are actually supported on the customer's platform. Scripts are grounded in specific pain points from discovery — not generic feature showcases.

**Resumes intelligently.** The orchestrator inventories existing outputs before running. If you change one thing (e.g., the audience composition changes), it re-runs only the affected downstream stages and leaves everything else intact.

## Validation Coverage

| Skill | Validated Against |
|---|---|
| `demo-discovery-parser` | Citizens Bank, IHG Club, Thermo Fisher, Lowe's — 97.5% benchmark |
| `demo-diagnostic-analyzer` | Deutsche Telekom SOC-T (152-node, 1.18PB cluster) |
| `demo-platform-audit` | Deutsche Telekom SOC-T (8.17.5 self-managed) |
| `demo-script-template` | Citizens Bank (champion → DM-present re-scope) |
| `demo-data-modeler` | Citizens Bank fraud demo (8-step build order, 4 indices) |
| `demo-validator` | Citizens Bank (timed checklist, 6 go/no-go criteria) |

## Repo Structure

```
demobuilder/
├── README.md
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
    └── demo-validator/
        ├── SKILL.md
        └── evals/evals.json
```
