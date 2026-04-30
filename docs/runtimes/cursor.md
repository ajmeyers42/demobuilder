# Running demobuilder in Cursor

## What you need

- Clone or open this repo as the **workspace root** so paths like `skills/demobuilder/SKILL.md` resolve.
- Optional: install **[elastic/agent-skills](https://github.com/elastic/agent-skills)** per [`docs/todo.md`](../todo.md) if you will provision clusters or use cloud/Kibana skills from the Elastic plugin set.

## How Cursor picks up instructions

- **Rules:** [`.cursor/rules/demobuilder.mdc`](../../.cursor/rules/demobuilder.mdc) is set to always apply and points the agent at the orchestrator and `$DEMOBUILDER_ENGAGEMENTS_ROOT` outputs.
- **AGENTS.md:** Cursor reads [`AGENTS.md`](../../AGENTS.md) at the repo root when present — same content as the practical “what the agent should do” manifest.

## Prompting

Examples:

- “Run demobuilder for discovery notes in `~/engagements/acme/discovery/`”
- “Refresh the demo script from `acme-discovery.json` only”
- “Deploy the demo to the cluster in `~/engagements/acme/.env`” (after approval)

Outputs should land under `$DEMOBUILDER_ENGAGEMENTS_ROOT/{slug}/` — default **`~/engagements/{slug}/`** when the env var is unset (see [`docs/engagements-path.md`](../engagements-path.md)).

## Skills location and research order

Use the **in-repo** [`skills/`](../../skills/) tree first. Do not maintain a second copy of skills for Cursor-only; symlink from `~/.claude/skills` to this repo if another tool needs the same files on disk.

When searching for extra skills or external facts: **this repo**, then **github.com/elastic**; for **general web** or non-Elastic sources, **ask the SA before using** — see **Research, skills, and external sources** in [`AGENTS.md`](../../AGENTS.md).

## Model selection by stage

Cursor lets you switch models per chat. Different pipeline stages have different cost/quality
trade-offs. Use this as a guide:

| Stage(s) | Recommended model | Reason |
|----------|------------------|--------|
| **demo-discovery-parser**, **demo-diagnostic-analyzer** | Fast/smaller (e.g. claude-3-5-haiku, gpt-4o-mini) | Structured extraction from large documents — speed matters, creativity does not |
| **demo-opportunity-review** | Fast/smaller | JSON aggregation of already-parsed files; low ambiguity |
| **demo-platform-audit** | Fast/smaller | Matrix comparison against known rules; not generative |
| **demo-script-template**, **demo-ideation** | Full model (Sonnet/Opus/GPT-4o) | Creative, customer-specific narrative; quality dominates |
| **demo-data-modeler** | Full model | Complex multi-index design with interdependencies |
| **demo-ml-designer** | Full model | ML config + anomaly injection planning; correctness-sensitive |
| **demo-kibana-agent-design** | Full model | Agent tool design requires careful schema adherence |
| **demo-vulcan-generate** | Full model | ES|QL + RAG architecture; context-heavy |
| **demo-validator**, **demo-status** | Fast/smaller | Checklist and status — deterministic; no creativity needed |
| **demo-deploy** (bootstrap.py gen) | Full model | Code generation that runs against a live cluster |
| **Resuming a partial build** | Fast/smaller for inventory, full for writing | Use a quick model to check state; switch for active generation |

**Practical flow:** Start a new chat, use a fast model to run inventory and confirm what's
pending, then either continue in that chat with a full model or start a fresh chat with the
full model once you know exactly which stage to run.

## Observability SLOs (integration packages)

For **SLO demos**, **Fleet / Elastic Agent integration packages**, or **Observability package** structure and conventions, prefer the Elastic org repo **[integration-packages-slo](https://github.com/elastic/integration-packages-slo)** alongside in-repo skills and **[elastic/agent-skills](https://github.com/elastic/agent-skills)**. Use it when scripting or explaining SLO-related data ingest—not as a substitute for engagement-specific `bootstrap.py` unless the demo is explicitly package-driven.
