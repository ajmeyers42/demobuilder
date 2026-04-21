# Running demobuilder in Cursor

## What you need

- Clone or open this repo as the **workspace root** so paths like `skills/demobuilder/SKILL.md` resolve.
- Optional: install **[elastic/agent-skills](https://github.com/elastic/agent-skills)** per [`docs/todo.md`](../todo.md) if you will provision clusters or use cloud/Kibana skills from the Elastic plugin set.

## How Cursor picks up instructions

- **Rules:** [`.cursor/rules/demobuilder.mdc`](../../.cursor/rules/demobuilder.mdc) is set to always apply and points the agent at the orchestrator and `$DEMOBUILDER_ENGAGEMENTS_ROOT` outputs.
- **AGENTS.md:** Cursor reads [`AGENTS.md`](../../AGENTS.md) at the repo root when present — same content as the practical “what the agent should do” manifest.

## Prompting

Examples:

- “Run demobuilder for discovery notes in `$DEMOBUILDER_ENGAGEMENTS_ROOT/acme/discovery/`”
- “Refresh the demo script from `acme-discovery.json` only”
- “Deploy the demo to the cluster in `$DEMOBUILDER_ENGAGEMENTS_ROOT/acme/.env`” (after approval)

Outputs should land under `$DEMOBUILDER_ENGAGEMENTS_ROOT/{slug}/` (see [`docs/engagements-path.md`](../engagements-path.md)).

## Skills location

Use the **in-repo** [`skills/`](../../skills/) tree. Do not maintain a second copy of skills for Cursor-only; symlink from `~/.claude/skills` to this repo if another tool needs the same files on disk.
