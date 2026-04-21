# Demobuilder — agent instructions

This repository is meant to be driven by an **assistant** (Cursor, Claude Code, Elastic Agent, or similar) helping a **Solutions Architect**, not by asking the SA to manually run repo scripts as the primary path.

## Canonical behavior

1. **Orchestrator:** Read and follow [`skills/demobuilder/SKILL.md`](skills/demobuilder/SKILL.md). Sub-skills live alongside it under [`skills/`](skills/).
2. **Outputs:** Write all engagement artifacts under **`$DEMOBUILDER_ENGAGEMENTS_ROOT/{slug}/`** (`{slug}` = one engagement). If **`DEMOBUILDER_ENGAGEMENTS_ROOT`** is unset, default to **`$HOME/engagements`** (see [`docs/engagements-path.md`](docs/engagements-path.md)). The git repo holds pipeline code only (`skills/`, `docs/`), not customer workspaces. Only ask the SA for a different root if they need a non-default path.
3. **Execution:** Run terminal commands, API calls, and skill workflows **on behalf of the SA** when they agree — scripts in this repo are **backends**, not the SA’s homework.
4. **Approvals:** Do **not** run `demo-cloud-provision` or `demo-deploy` (create/spend cloud resources, mutate clusters, run `bootstrap.py`) until the SA has **explicitly** asked to provision or deploy in this session, or confirmed after you ask. Artifact stages (discovery → validator) can run when the SA asks to build or refresh the demo plan.

5. **Elastic version:** For **new** cloud deployments or Serverless projects, default to the **latest GA** stack for that product unless the SA specifies a version. For **existing** deployments, **resolve and validate** `version` (e.g. `GET /`, Kibana `/api/status`, or diagnostic output) **before** producing scripts, data models, or step-by-step plans. Scope all guidance, ES|QL, and API usage to that version (and deployment type).

6. **Enterprise scope and solution areas:** Prefer **enterprise-appropriate** Elastic capabilities that address the customer outcomes in the inputs (discovery, diagnostics, team notes, **architecture diagrams** when supplied). Demos may center **search**, **Observability**, **Security**, or a **deliberate mix** — use cross-solution storylines when that best matches the customer need; do not default to core-search-only unless the artifacts are search-only.

7. **Solution first:** Scripts and plans should **lead with outcomes** tied to the customer’s **key asks** from discovery, then show **supporting capabilities**. If those asks are not clear, **ask the SA** before locking the storyline.

## Runtime-specific setup

| Runtime | Setup |
|--------|--------|
| **Cursor** | [`docs/runtimes/cursor.md`](docs/runtimes/cursor.md) |
| **Claude Code / Claude projects** | [`docs/runtimes/claude.md`](docs/runtimes/claude.md) |

Do **not** duplicate skill bodies per IDE. One [`skills/`](skills/) tree; thin glue only (this file, Cursor rules, Claude plugin paths).

## External dependencies

Cloud provisioning and some Kibana operations require **[elastic/agent-skills](https://github.com/elastic/agent-skills)** (see [`docs/todo.md`](docs/todo.md)). If missing, say so clearly instead of failing silently.
