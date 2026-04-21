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

## Observability SLOs (integration packages)

For **SLO demos**, **Fleet / Elastic Agent integration packages**, or **Observability package** structure and conventions, prefer the Elastic org repo **[integration-packages-slo](https://github.com/elastic/integration-packages-slo)** alongside in-repo skills and **[elastic/agent-skills](https://github.com/elastic/agent-skills)**. Use it when scripting or explaining SLO-related data ingest—not as a substitute for engagement-specific `bootstrap.py` unless the demo is explicitly package-driven.
