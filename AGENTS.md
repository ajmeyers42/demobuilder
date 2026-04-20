# Demobuilder — agent instructions

This repository is meant to be driven by an **assistant** (Cursor, Claude Code, Elastic Agent, or similar) helping a **Solutions Architect**, not by asking the SA to manually run repo scripts as the primary path.

## Canonical behavior

1. **Orchestrator:** Read and follow [`skills/demobuilder/SKILL.md`](skills/demobuilder/SKILL.md). Sub-skills live alongside it under [`skills/`](skills/).
2. **Outputs:** Write all engagement artifacts under **`engagements/{slug}/`** (or the workspace path the SA chose). Same tree regardless of which tool hosts the agent.
3. **Execution:** Run terminal commands, API calls, and skill workflows **on behalf of the SA** when they agree — scripts in this repo are **backends**, not the SA’s homework.
4. **Approvals:** Do **not** run `demo-cloud-provision` or `demo-deploy` (create/spend cloud resources, mutate clusters, run `bootstrap.py`) until the SA has **explicitly** asked to provision or deploy in this session, or confirmed after you ask. Artifact stages (discovery → validator) can run when the SA asks to build or refresh the demo plan.

## Runtime-specific setup

| Runtime | Setup |
|--------|--------|
| **Cursor** | [`docs/runtimes/cursor.md`](docs/runtimes/cursor.md) |
| **Claude Code / Claude projects** | [`docs/runtimes/claude.md`](docs/runtimes/claude.md) |

Do **not** duplicate skill bodies per IDE. One [`skills/`](skills/) tree; thin glue only (this file, Cursor rules, Claude plugin paths).

## External dependencies

Cloud provisioning and some Kibana operations require **[elastic/agent-skills](https://github.com/elastic/agent-skills)** (see [`docs/todo.md`](docs/todo.md)). If missing, say so clearly instead of failing silently.
