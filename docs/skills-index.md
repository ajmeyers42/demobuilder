# Skills Index

Use `demobuilder` when you want the guided end-to-end pipeline. Use an individual skill when you know the exact stage you want to run or refresh.

## Guided Pipeline

| Skill | Use when | Primary outputs |
|---|---|---|
| `demobuilder` | Build or resume a complete engagement from any combination of discovery notes, diagnostics, and existing outputs | Runs the pipeline, skips completed stages, delivers the final handoff |

## Planning and Qualification

| Skill | Use when | Primary outputs |
|---|---|---|
| `demo-ideation` | No discovery notes exist yet and the SA needs help choosing a demo direction, archetype, and wow moments | `{slug}-ideation.md` |
| `demo-discovery-parser` | Discovery notes, meeting notes, or sales notes need to become structured customer context | `{slug}-discovery.json`, `{slug}-confirmation.md`, `{slug}-gaps.md` |
| `demo-diagnostic-analyzer` | Elastic diagnostic ZIPs or API exports need to become a current-state profile | `{slug}-current-state.json`, `{slug}-architecture.md`, `{slug}-findings.md` |
| `demo-opportunity-review` | Parsed discovery and diagnostics need MEDDPIC qualification and SDR/AE/SA team alignment before demo planning | `{slug}-opportunity-summary.md`, `{slug}-opportunity-profile.json` |
| `demo-platform-audit` | Planned scope needs to be checked against version, license, deployment type, and known platform constraints | `{slug}-platform-audit.json`, `{slug}-platform-audit.md` |

## Demo Design

| Skill | Use when | Primary outputs |
|---|---|---|
| `demo-script-template` | Discovery and platform audit are ready and the SA needs a solution-first demo script and AE brief | `{slug}-demo-script.md`, `{slug}-demo-brief.md` |
| `demo-vulcan-generate` | Script has 5+ ES\|QL queries, semantic/RAG search, or integration-grounded data (Fleet/Beats) — run before demo-data-modeler | `{slug}-vulcan-queries.json`, `{slug}-vulcan-data-profile.json`, `vulcan-data/*.csv` |
| `demo-data-modeler` | The script is ready and the demo needs indices, mappings, data streams, pipelines, seed data, and build order | `{slug}-data-model.json`, `{slug}-data-model.md`, mapping files |
| `demo-ml-designer` | The script includes ML anomaly detection, data frame analytics, anomaly injection, or model deployment scenes | `{slug}-ml-config.json`, `{slug}-ml-setup.md` |
| `demo-kibana-agent-design` | The script includes Elastic Agent Builder custom agents, tools, workflows, or multi-agent orchestration | `{slug}-agent-builder-spec.md` |
| `token-visibility` | Agent Builder or another AI component is in scope and the demo should include AI cost and usage transparency | `{prefix}agent-sessions` spec, AI Cost + Usage dashboard guidance |

## Readiness, Deploy, and Operations

| Skill | Use when | Primary outputs |
|---|---|---|
| `demo-validator` | The planning/build artifacts are ready and the SE needs a pre-demo checklist and go/no-go risk register | `{slug}-demo-checklist.md`, `{slug}-risks.md` |
| `demo-cloud-provision` | A new Elastic Cloud deployment or Serverless project is needed, or a reusable `.env` must be prepared | `.env`, `.env.example`, `{slug}-provision-log.md` |
| `demo-deploy` | The SA approved live cluster changes and wants `bootstrap.py` generated or executed | `bootstrap.py`, `{slug}-deploy-log.md` |
| `demo-status` | A deployed demo needs a quick health/readiness pulse check before the meeting | Terminal readiness report |
| `demo-teardown` | A deployed demo should be cleaned up after the meeting | `teardown.py`, `{slug}-teardown-log.md` |

## Invocation Tips

- If you have raw notes and want a full build, invoke `demobuilder`.
- If you already have `{slug}-discovery.json` and only want qualification, invoke `demo-opportunity-review`.
- If you only changed the storyline, rerun `demo-script-template`, then rerun downstream affected stages (`demo-data-modeler`, `demo-ml-designer` if relevant, and `demo-validator`).
- If Agent Builder appears in the script, run `demo-kibana-agent-design` and `token-visibility` before deploy planning.
- If the script has 5+ ES|QL queries, semantic search, or Fleet/Beats integrations, run `demo-vulcan-generate` before `demo-data-modeler` to get cluster-validated queries and synthetic CSV data.
- If you only need to check the environment on demo morning, invoke `demo-status`; do not rerun the whole pipeline.
