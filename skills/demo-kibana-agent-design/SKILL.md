---
name: demo-kibana-agent-design
description: >
  Produces a concrete Elastic Agent Builder definition for a demobuilder engagement: agent
  name, system instructions, behaviors, tool list (ES|QL, index search, workflow tools), and
  workflow linkage. Aligns with Kibana 9.3+ Agent Builder (GA on Elastic Stack since 9.3).
  Use when the demo script includes a custom agent (e.g. Fraud Assistant), when the SA asks
  to "define the Agent Builder agent", "spec the Kibana agent", or when refreshing agent
  prompts/tools after script changes. Does not replace bootstrap.py or elastic/agent-skills
  for API execution — it documents what to build in Kibana.

  ALWAYS use this skill when the user is defining or revising an Agent Builder agent for a
  customer demo scripted in demobuilder.
---

# Demo Kibana Agent Design (Agent Builder)

You are specifying how the SE implements **Elastic Agent Builder** in Kibana for a demo. Ground everything in the **demo script**, **discovery JSON**, and **platform audit**. Official product entry point and UI paths: **[Get started with Elastic Agent Builder](https://www.elastic.co/docs/explore-analyze/ai-features/agent-builder/get-started)** (Elastic Docs).

## Canonical references (read before writing)

- **This repo:** `skills/demo-deploy/references/workflow-patterns.md` — workflow `id` vs name, Agent Builder tool wiring, `GET /api/workflows`.
- **Elastic org:** **[elastic/workflows](https://github.com/elastic/workflows)** — Elastic Workflow Library (YAML examples, docs). Prefer this for workflow *authoring*; use `workflow-patterns.md` for Kibana API + agent handoff.
- **Elastic Docs (same topic):** [Agent Builder get started](https://www.elastic.co/docs/explore-analyze/ai-features/agent-builder/get-started), [custom agents](https://www.elastic.co/docs/explore-analyze/ai-features/agent-builder/custom-agents), [custom tools](https://www.elastic.co/docs/explore-analyze/ai-features/agent-builder/tools/custom-tools), [Agent Builder API tutorial](https://www.elastic.co/docs/explore-analyze/ai-features/agent-builder/agent-builder-api-tutorial) if programmatic.

## Step 1: Confirm scope

From `{slug}-platform-audit.json` and cluster version: Agent Builder must be **available** (e.g. Elastic Stack **9.3+** GA per docs; Serverless as documented). If not green, say so and point to audit remediation — do not write a full agent spec that cannot run.

## Step 2: Name and role

- **Agent display name** — customer-facing (e.g. `Fraud Assistant`).
- **One-line role** — what the agent does for the champion vs leadership (POC boundaries).

## Step 3: System instructions (prompt)

Write **copy-paste-ready** agent instructions that:

1. **Ground** answers in named indices/data streams and KB indices from the data model (no invented `claim_id` / IDs).
2. **Route behavior:** when to use **ES|QL tools** vs **index search** vs **workflow tool**.
3. **Link** to the customer SLA language from discovery (e.g. acknowledge vs resolve windows).
4. **Safety:** POC only; no production or PII claims.

## Step 4: Tools (9.3 Agent Builder)

List each tool with **type** and **purpose**:

| Type | Use |
|------|-----|
| **ES|QL** | Parameterized queries the SA tests in the tool editor (SLA breakdowns, lookups). |
| **Index search** | Scoped patterns to claims + KB; semantic/policy questions. |
| **Workflow** | **Workflow `id`** (from API) that performs **create Case** or other scripted action — not the workflow *name*. |

Do **not** specify **MCP tools** unless the SA explicitly asks — default demobuilder demos use Elastic-native tools only.

## Step 5: Workflow linkage

- Name the **workflow** as scripted in `{slug}-demo-script.md`.
- Remind: create workflow first → copy **`id`** → attach to **Workflow tool** in Agent Builder per `workflow-patterns.md`.

## Step 6: Output

Write **`{slug}-agent-builder-spec.md`** in the engagement workspace (`$DEMOBUILDER_ENGAGEMENTS_ROOT/{slug}/`) containing:

- Prerequisites (Kibana URL, privileges — link [permissions doc](https://www.elastic.co/docs/explore-analyze/ai-features/agent-builder/permissions) if needed).
- Navigation (Agents, **AI Agent** button — per get-started doc for the deployment type).
- Full **system instructions** block.
- **Tools** table + test prompts for each tool.
- **Demo prompts** (3–5) copied from or aligned with the demo script scenes.

If the SA only wants an inline section inside `{slug}-demo-script.md`, merge the same content there instead of a separate file — but prefer a **spec file** when tools/workflows are non-trivial so `demo-deploy` / SA handoff stays traceable.

## Handoff

Point the SA to **elastic/agent-skills** for Kibana API automation if they generate configs via API; otherwise UI build following the spec is sufficient for POC.
