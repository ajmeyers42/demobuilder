# Pipeline

The demobuilder pipeline runs end-to-end when you invoke the `demobuilder` orchestrator, or you can drop into any single skill independently. Each skill's JSON output is a machine-readable input to the next stage — nothing is re-inferred downstream.

For a quick lookup of when to invoke a specific skill without running the full pipeline, see [skills-index.md](skills-index.md).

## Stage flow

```
                    ╔══════════════════════════════════════╗
                    ║        demobuilder orchestrator       ║
                    ║  drop any inputs · stages auto-run   ║
                    ║  skips completed · resumes on re-run ║
                    ╚══════════════╤═══════════════════════╝
                                   │
                    ┌──────────────▼──────────────┐
                    │  Step 0: Currency check      │
                    │  demobuilder + hive-mind     │
                    │  repos up to date?           │
                    └──────────────┬──────────────┘
                                   │
                    ┌──────────────▼──────────────┐
                    │  demo-ideation   optional   │
                    │  SA coaching + archetypes    │
                    │  when direction is unclear   │
                    │  → {slug}-ideation.md        │
                    └──────────────┬──────────────┘
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
                    │  demo-opportunity-  │
                    │  review             │
                    │                     │
                    │ → opportunity-      │
                    │   summary.md        │
                    │ → opportunity-      │
                    │   profile.json      │
                    └──────────┬──────────┘
                               │
                     Team alignment review
                     SDR · AE · SA confirm
                     MEDDPIC gate  ──────►  🔴 not qualified: stop
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
      conditional if scripted:
      Agent Builder → demo-kibana-agent-design
      AI component  → token-visibility
               │
      conditional (ES|QL-heavy / RAG / integrations):
      ┌─────────────────────────────────┐
      │  demo-vulcan-generate optional  │
      │  validated ES|QL + synth data   │
      │  EPR grounding + RAG pipelines  │
      │ → vulcan-queries.json           │
      │ → vulcan-data/*.csv             │
      └────────────────┬────────────────┘
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
           │              deploy phase  (SA approval req.) │
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

## Skills reference

| Skill | Input | Output | Version |
|---|---|---|---|
| `demobuilder` | Any combination of inputs | Runs full pipeline, delivers handoff summary | v2 |
| `demo-ideation` | SA goals, customer vertical (optional) | `demo/{slug}-ideation.md` — frozen archetype + value contract | v2 |
| `demo-discovery-parser` | Discovery notes (PDF, markdown, raw text) | `demo/{slug}-discovery.json`, `opportunity/{slug}-confirmation.md`, `opportunity/{slug}-gaps.md` | v1 |
| `demo-diagnostic-analyzer` | Elastic diagnostic ZIP or API exports | `demo/{slug}-current-state.json`, `demo/{slug}-architecture.md`, `demo/{slug}-findings.md` | v1 |
| `demo-opportunity-review` | All parsed discovery + diagnostic outputs | `opportunity/{slug}-opportunity-summary.md` (team review), `opportunity/{slug}-opportunity-profile.json` | v1 |
| `demo-platform-audit` | Discovery JSON + opportunity profile (pre-scopes audit) + optional diagnostic | `demo/{slug}-platform-audit.json`, `demo/{slug}-platform-audit.md` | v1 |
| `demo-script-template` | Discovery JSON + platform audit | `demo/{slug}-demo-script.md`, `opportunity/{slug}-demo-brief.md` | v2 |
| `demo-vulcan-generate` | Demo script (ES\|QL / RAG / integration-grounded demos) | `data/{slug}-vulcan-queries.json`, `data/{slug}-vulcan-data-profile.json`, `data/seed/*.csv` | v1 |
| `demo-data-modeler` | Demo script + discovery JSON + optional Vulcan outputs | `data/{slug}-data-model.json`, `data/{slug}-data-model.md`, `data/mappings/`, `data/pipelines/` | v2 |
| `demo-fleet-integrations` | Data model + discovery + demo script | `deploy/{slug}-integrations-manifest.json`, `demo/{slug}-integration-assets.md` | v1 |
| `demo-ml-designer` | Demo script + data model | `data/{slug}-ml-config.json`, `data/{slug}-ml-setup.md` | v1 |
| `demo-validator` | All pipeline outputs | `deploy/{slug}-demo-checklist.md`, `deploy/{slug}-risks.md` | v1 |
| `demo-kibana-agent-design` | Demo script + discovery (Agent Builder in scope) | `demo/{slug}-agent-builder-spec.md` | v2 |
| `token-visibility` | Engagement slug + `.env` | Token tracking index + AI Cost dashboard (auto-included in Agent Builder demos) | v2 |
| `demo-cloud-provision` | Deployment type, region, slug | `{slug}/.env`, `{slug}/.env.example`, `deploy/{slug}-provision-log.md` | v1 |
| `demo-deploy` | `.env` + pipeline outputs | `deploy/bootstrap.py`, `deploy/{slug}-deploy-log.md` | v2 |
| `demo-status` | `.env` | Terminal readiness report (✅/❌ per resource, fix commands) | v1 |
| `demo-teardown` | `.env` | `deploy/teardown.py`, `deploy/{slug}-teardown-log.md` | v1 |

## Engagement workspace layout

Every engagement produces a folder under `$DEMOBUILDER_ENGAGEMENTS_ROOT/{slug}/` (default `~/engagements/{slug}/`). Artifacts are organized into four audience-scoped subfolders (D-037):

```
{slug}/
├── .env                              ← cluster credentials (never commit — always at root)
├── .env.example                      ← template, safe to copy
│
├── opportunity/                      ← AE / SDR / SA: team alignment gate
│   ├── {slug}-opportunity-summary.md   ← living team review doc (SDR/AE/SA)
│   ├── {slug}-opportunity-profile.json ← structured MEDDPIC + qualification data
│   ├── {slug}-confirmation.md          ← customer-facing confirmation (send after discovery)
│   ├── {slug}-gaps.md                  ← internal follow-up questions
│   └── {slug}-demo-brief.md            ← one-page AE brief (hand to AE before demo)
│
├── demo/                             ← SA: design & planning intelligence
│   ├── {slug}-ideation.md              ← archetype + value contract (if ideation ran)
│   ├── {slug}-discovery.json           ← structured customer profile
│   ├── {slug}-current-state.json       ← diagnostic profile (if diagnostic ran)
│   ├── {slug}-architecture.md          ← current architecture summary (if diagnostic ran)
│   ├── {slug}-findings.md              ← diagnostic findings + demo readiness (if diagnostic ran)
│   ├── {slug}-platform-audit.json      ← feature feasibility matrix
│   ├── {slug}-platform-audit.md        ← SE briefing
│   ├── {slug}-demo-script.md           ← full SE script with scenes + timing
│   └── {slug}-agent-builder-spec.md    ← Agent Builder spec (if applicable)
│
├── data/                             ← SA / engineer: data model + generation
│   ├── {slug}-data-model.json          ← index mappings + build order
│   ├── {slug}-data-model.md            ← human-readable build overview
│   ├── {slug}-ml-config.json           ← ML job configs (if applicable)
│   ├── {slug}-ml-setup.md              ← ML setup guide (if applicable)
│   ├── {slug}-vulcan-queries.json      ← validated ES|QL queries from Vulcan (if ran)
│   ├── {slug}-vulcan-data-profile.json ← data profile summary from Vulcan (if ran)
│   ├── {slug}-vulcan-query-results.json← query pass/fail results from Vulcan (if ran)
│   ├── seed/                           ← synthetic CSV datasets from Vulcan (if ran)
│   ├── mappings/                       ← per-index mapping JSON files
│   └── pipelines/                      ← per-pipeline JSON files
│
└── deploy/                           ← SA: cluster assets + readiness
    ├── bootstrap.py                    ← single deploy driver
    ├── teardown.py                     ← generated on first teardown run
    ├── {slug}-provision-log.md         ← cluster provisioning record
    ├── {slug}-demo-checklist.md        ← timed pre-demo checklist
    ├── {slug}-risks.md                 ← risks and fallbacks
    ├── {slug}-deploy-log.md            ← what was created, doc counts
    ├── {slug}-teardown-log.md          ← what was deleted + when
    ├── kibana-objects/                 ← optional: committed .ndjson exports
    ├── kibana/                         ← optional: Workflows YAML, agent JSON
    └── elasticsearch/                  ← optional: declarative ES JSON
```

See [docs/engagements-path.md](engagements-path.md) for the env-var override and multi-customer isolation patterns.

## Validation coverage

| Skill | Validated against |
|---|---|
| `demo-discovery-parser` | Sample notes from multiple customer interactions across 4 verticals — 97.5% benchmark |
| `demo-diagnostic-analyzer` | Sample diagnostic exports from multiple customer environments (large-scale self-managed) |
| `demo-opportunity-review` | Evals written — initial MEDDPIC review + living-document update |
| `demo-platform-audit` | Sample diagnostics from multiple customer self-managed deployments |
| `demo-script-template` | Sample notes from multiple customer interactions (single-contact and executive-present scenarios) |
| `demo-vulcan-generate` | Evals written — ES\|QL parameterized demo + integration-grounded data + RAG pipeline |
| `demo-data-modeler` | Sample notes from multiple customer interactions (fraud detection use case) |
| `demo-ideation` | Evals written — cold-start demo direction + vague operational AI prompt |
| `demo-kibana-agent-design` | Evals written — Agent Builder spec + blocked Agent Builder fallback |
| `token-visibility` | Evals written — Agent Builder default inclusion + opt-out behavior |
| `demo-validator` | Sample pipeline outputs from multiple customer interactions |
| `demo-cloud-provision` | Evals written — serverless project + shared cluster namespace copy |
| `demo-deploy` | Evals written — isolated cluster full deploy + shared cluster prefix deploy |
| `demo-status` | Evals written — readiness check + ML-focused readiness check |
| `demo-teardown` | Evals written — isolated cluster teardown + shared cluster prefix teardown |

See [../schemas/](../schemas/) for initial JSON contracts used by machine-readable outputs.
