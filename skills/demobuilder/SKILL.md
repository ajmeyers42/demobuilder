---
name: demobuilder
description: >
  Top-level orchestrator for the Elastic demobuilder pipeline. Accepts any combination of
  discovery notes, diagnostic files, architecture diagrams, supplemental notes from the
  discovery team, and existing pipeline outputs; determines which stages need to run,
  executes them in dependency order, and delivers demo-ready artifacts. Demos may emphasize
  search, Observability, Security, or a deliberate mix — scoped to customer needs and
  enterprise-level Elastic capabilities.

  ALWAYS use this skill when the user says "build the demo for X", "run demobuilder",
  "take this from discovery to demo", "full pipeline for [company]", "we have notes and
  a diagnostic — build everything", or provides input files and wants a complete demo build
  rather than a specific pipeline step. Also trigger when the user says "demobuilder" by
  name. Use individual pipeline skills (demo-discovery-parser, demo-script-template, etc.)
  when the user wants only a specific stage.
---

# Demobuilder Orchestrator

You are the project manager for a full demo build. Your job is to figure out where this
engagement is in the pipeline, run every stage that hasn't been completed yet, and hand
off a complete, organized set of artifacts at the end.

You don't do the work yourself — you read each sub-skill's SKILL.md to load its expertise,
execute that stage, then move to the next. Each stage produces files that feed into the
next. Treat each sub-skill as a specialist you're briefing, not a function you're calling.

**External dependencies — `elastic/agent-skills` (Search, Observability, and Security):**
The pipeline is designed to use specialists from `elastic/agent-skills` (separate plugin)
across **all** major Elastic solution areas — not only search and Observability. Install the
**full** plugin so these are **available** in every session, even when a particular
engagement does not use them (e.g. a pure search demo still benefits from a consistent
install; the orchestrator simply does not invoke Security skills for that scenario).

| Area | Examples (non-exhaustive) |
|------|-------------------------|
| **Elasticsearch / search / analytics** | `elasticsearch-esql`, `elasticsearch-file-ingest`, `elasticsearch-authn`, … |
| **Observability** | `observability-manage-slos`, `observability-service-health`, `observability-logs-search`, … |
| **Elastic Security / SIEM** | `security-detection-rule-management`, `security-alert-triage`, `security-case-management`, `security-generate-security-sample-data`, … |
| **Kibana / platform** | `kibana-dashboards`, `kibana-alerting-rules`, `kibana-connectors`, `kibana-vega`, … |
| **Cloud** | `cloud/setup`, `cloud/create-project`, `cloud/manage-project`, … |

Stages **8–9** (provision, deploy) rely on Cloud + Kibana skills; stages **1–7** still call
Security or Observability skills whenever the **discovery, script, or audit** scope requires
them (e.g. SIEM detection demos, hybrid Sec + Obs storylines).

If `elastic/agent-skills` is not installed, surface a clear message rather than failing
silently: install the plugin per `docs/todo.md` — **include Security skills**, not a subset.

**Agent runtimes:** Skills live under `skills/` in the demobuilder repo. Behavior for
Cursor, Claude, and other hosts is unified — see repo root `AGENTS.md` and
`docs/runtimes/`. Do not fork skill content per IDE; only loading paths differ.

**Deploy approval:** Before running **Stage 8 (demo-cloud-provision)** or **Stage 9 (demo-deploy)**
against a **live** cluster, confirm the SA wants to provision or deploy **and** has **reviewed**
`bootstrap.py`, `{slug}-platform-audit`, `{slug}-risks`, `{slug}-demo-checklist.md`, and any
Kibana/ES files the script will apply — unless they state review is complete. **`bootstrap.py --dry-run`**
does not require this gate. See `docs/decisions.md` **D-024** and `AGENTS.md`. Planning stages
(1–7) may proceed when the SA asks to build or refresh artifacts.

**Elastic version scope:**
- **New deployment or Serverless project** — Assume the **latest generally available**
  stack version for that offering **unless the SA specifies otherwise**. Record the
  actual version in `.env` (`ELASTIC_VERSION`) and in any provision log after create.
- **Existing deployment / project / cluster** — **Do not** assume latest. Obtain
  `version` from `GET /` (Elasticsearch) and Kibana `/api/status` (or from diagnostic /
  `demo-diagnostic-analyzer` output) **before** writing demo scripts, data models, or
  execution plans, and thread that version into **demo-platform-audit** and downstream
  artifacts.
- **All scripts, plans, and guidance** — Must match the target stack: ES|QL syntax,
  API shapes, Kibana features, ML APIs, and Agent Builder / Workflows availability all
  depend on version and deployment type. When in doubt, cite the version the guidance
  applies to.

**Deployability on Elastic (`docs/decisions.md` D-025):** Data models, bootstrap payloads,
Agent Builder tools, ML configs, and Kibana imports must be **deployable** on a real cluster
and use **Elastic datatypes and API conventions** — not hand-wavy JSON. When skills or
OpenAPI disagree with a first guess, trust the stack and **`elastic/agent-skills`**
reference behavior.

**Engagement tagging (`docs/decisions.md` D-026):** Generated **`bootstrap.py`** and API payloads
must merge **`demobuilder:<engagement_id>`** into every resource that supports **`tags`**
(SLOs, alerting rules, ML jobs, Agent Builder entities, etc.). Derive `<engagement_id>` per
**`skills/demo-deploy/references/demobuilder-tagging.md`** (`INDEX_PREFIX` normalized, else
`DEMO_SLUG`; optional **`DEMO_ASSET_TAG`** override).

**Demo scope — enterprise capabilities and solution areas:**
- **Assume enterprise-appropriate features** when shaping the demo: prefer capabilities that
  match the **customer outcomes** and pain points in the inputs (discovery, diagnostic,
  supplemental notes), subject to **demo-platform-audit** and license/version reality.
  Do not default to “minimal” or core-search-only unless the customer story is search-only.
- **Inputs** may include: discovery notes, **Elastic diagnostic** exports, **additional
  notes** from the AE/SE/discovery team, and **architecture diagrams** (current-state
  systems, data flows). Treat diagrams as first-class context — extract what they imply
  for integrations, data paths, and operational pain.
- **Use case domains:** Demos apply equally to **Elasticsearch (search / analytics)**,
  **Observability**, and **Elastic Security** — pick the primary domain from the artifacts.
  Past examples often emphasized search; **do not** force search framing when the
  discovery points to logs, APM, SIEM, or detection workflows.
- **Cross-solution demos:** When the customer’s needs span domains, it is **acceptable and
  often desirable** to combine capabilities across **search, Observability, and Security**
  in one storyline (e.g. unified data platform, correlated investigation, shared ES|QL).
  Call this out in the script and platform audit so scope stays honest.

**Narrative — solution first:** Unless the SA says otherwise, demo **scripts and plans**
should lead with **business value and the customer’s key asks** from discovery, then
detail **supporting Elastic capabilities** (how to get there). If primary goals are unclear
in the inputs, the agent should **ask for guidance** before finalizing storyline — see
`demo-script-template`.

**Scenario adaptability (any demo, not one template):** The pipeline is **analytic**, not
prescriptive. Each engagement may emphasize different Elastic surfaces — relevance and
semantic search, log analytics, APM, Synthetics, Security detection and SIEM, Observability
SLOs, ML anomalies, Agent Builder, cross-cluster search, etc. **Nothing** in the stage
list assumes a particular vertical (financial services, retail, public sector, etc.) or a
fixed feature bundle. **demo-platform-audit** and **demo-script-template** narrow what is
feasible and what the story needs; **demo-data-modeler** and **demo-deploy** materialize
only that. Do not retrofit every engagement into a “standard” shape that happened to work
for a prior customer; **read the inputs** and produce artifacts that match **this** demo.

**Additional skills (planning / Kibana / Security):**
- `demo-kibana-agent-design` — when the demo script includes **Elastic Agent Builder** (custom agent, tools, workflows), produce `{slug}-agent-builder-spec.md` per `skills/demo-kibana-agent-design/SKILL.md`.
- **Elastic Security** — when the story includes detection, alerts, cases, or sample security data, read and follow the relevant `security-*` skills from `elastic/agent-skills` (same install as Search/Obs); **platform-audit** must reflect Sec license/tier and feature availability.

**Additional post-deploy skills** available once a cluster is deployed:
- `demo-status` — quick pre-demo readiness pulse check (connectivity, doc counts, ML state, ELSER latency)
- `demo-teardown` — post-demo cleanup; removes all demo resources prefix-aware

## Step 1: Identify the Engagement and Workspace

**Determine the slug** from the customer name in the input files or the user's prompt.
Lowercase, hyphenated: "Citizens Bank" → `citizens-bank`, "Deutsche Telekom SOC-T" → `dt-soct`.
The slug identifies **one engagement** (everything specific to that demo).

**Set the workspace directory** to that engagement’s folder: **`$DEMOBUILDER_ENGAGEMENTS_ROOT/{slug}/`**
where **`DEMOBUILDER_ENGAGEMENTS_ROOT`** defaults to **`$HOME/engagements`** when unset (see
`docs/engagements-path.md`). Only per-demo artifacts belong in `{slug}/`; pipeline code stays in the
demobuilder clone (`skills/`, `docs/`). If the SA sets a different root or absolute workspace, use it;
create the directory if it doesn’t exist.

All output files for this engagement live in the workspace. Use the slug as a prefix for
every file: `{slug}-discovery.json`, `{slug}-demo-script.md`, etc.

## Step 2: Take Inventory

Check the workspace for existing pipeline outputs. Use this to determine where to start —
don't re-run a stage if its output already exists and the inputs haven't changed.

```
Stage                    | Output file                     | Re-run if...
-------------------------|----------------------------------|---------------------------
demo-discovery-parser    | {slug}-discovery.json           | New/changed discovery notes
demo-diagnostic-analyzer | {slug}-current-state.json       | New/changed diagnostic file
demo-platform-audit      | {slug}-platform-audit.json      | discovery or current-state changed
demo-script-template     | {slug}-demo-script.md           | platform-audit changed or user requested
demo-data-modeler        | {slug}-data-model.json          | script changed
demo-ml-designer         | {slug}-ml-config.json           | data-model changed and ML scenes in script
demo-validator           | {slug}-demo-checklist.md        | always run last — regenerate each time
```

Report the inventory to the user before executing:
```
📋 Engagement: [Company] ([slug])
📁 Workspace: [path]

Stage                    Status
─────────────────────────────────────────
Discovery parser         ✅ Complete  ({slug}-discovery.json)
Diagnostic analyzer      ⏭  Skipped  (no diagnostic provided)
Platform audit           ✅ Complete  ({slug}-platform-audit.json)
Script template          🔲 Pending
Data modeler             🔲 Pending
ML designer              🔲 Pending  (will check for ML scenes in script)
Validator                🔲 Pending

Starting from: demo-script-template
```

After this inventory, continue with **planning stages** (1–7) without an extra confirmation
step unless the user asked to pause. **Do not** run provisioning or deploy (stages 8–9)
without explicit approval — see Stage 8–9 notes below.

## Step 3: Detect Available Inputs

Before running any stage, verify what raw inputs are available:

**Discovery notes** — PDF, markdown, plain text, or raw notes provided by the user or
already parsed into `{slug}-discovery.json`. If raw notes are present and no JSON exists,
run `demo-discovery-parser` first.

**Diagnostic file** — ZIP archive or individual JSON API exports from an Elastic cluster.
If present and no `{slug}-current-state.json` exists, run `demo-diagnostic-analyzer`.
If absent, skip the diagnostic stage entirely — it's optional.

**Existing pipeline outputs** — Any `{slug}-*.json` or `{slug}-*.md` files in the workspace.
Use these as inputs to downstream stages. Do not regenerate unless inputs changed.

## Step 4: Execute Each Pending Stage

For each stage that needs to run, in order:

1. **Announce the stage:** `🔄 Running: demo-discovery-parser...`
2. **Read the sub-skill SKILL.md** from the sibling directory:
   `../demo-discovery-parser/SKILL.md` (relative to this file's location)
   This loads the specialist's instructions. Follow them exactly.
3. **Execute the stage** using the loaded instructions and available inputs.
4. **Write outputs** to the workspace directory with the slug prefix.
5. **Announce completion:** `✅ demo-discovery-parser complete → {slug}-discovery.json`
6. **Surface any blockers:** If a stage produces a RED platform audit or critical gaps, pause
   and report before continuing:
   ```
   ⚠️  Platform audit returned RED. Blocking issues:
   - Agent Builder requires 9.x upgrade (current: 8.17.5)
   - ELSER endpoint not deployed

   Recommended action: re-scope demo to exclude Agent Builder, proceed with ES|QL + ML.
   Continuing with adjusted scope...
   ```
   If the blocker is fatal (e.g., no data available at all, zero contacts in discovery),
   stop and ask the user for the missing input rather than producing empty outputs.

### Stage execution order and skip conditions

**Stage 1 — demo-discovery-parser**
- Skip if: `{slug}-discovery.json` exists in workspace AND no new discovery notes provided
- Read: `../demo-discovery-parser/SKILL.md`
- Inputs: discovery notes (PDF/text/markdown)
- Outputs: `{slug}-discovery.json`, `{slug}-confirmation.md`, `{slug}-gaps.md`

**Stage 2 — demo-diagnostic-analyzer** *(optional)*
- Skip if: no diagnostic file provided OR `{slug}-current-state.json` already exists
- Read: `../demo-diagnostic-analyzer/SKILL.md`
- Inputs: diagnostic ZIP or API exports
- Outputs: `{slug}-current-state.json`, `{slug}-architecture.md`, `{slug}-findings.md`

**Stage 3 — demo-platform-audit**
- Skip if: `{slug}-platform-audit.json` exists AND neither discovery nor current-state changed
- Read: `../demo-platform-audit/SKILL.md`
- Inputs: `{slug}-discovery.json`, `{slug}-current-state.json` (if available)
- Outputs: `{slug}-platform-audit.json`, `{slug}-platform-audit.md`
- **Blocker check:** If overall_status is RED, surface the blocking features before
  proceeding. Auto-adjust scope: remove blocked features from the script brief, continue.

**Stage 4 — demo-script-template**
- Skip if: `{slug}-demo-script.md` exists AND platform-audit hasn't changed
- Read: `../demo-script-template/SKILL.md`
- Inputs: `{slug}-discovery.json`, `{slug}-platform-audit.json`
- Outputs: `{slug}-demo-script.md`, `{slug}-demo-brief.md`

**Stage 5 — demo-data-modeler**
- Skip if: `{slug}-data-model.json` exists AND script hasn't changed
- Read: `../demo-data-modeler/SKILL.md` and `../demo-data-modeler/references/mapping-patterns.md`
- Inputs: `{slug}-demo-script.md`, `{slug}-discovery.json`
- Outputs: `{slug}-data-model.json`, `{slug}-data-model.md`, individual mapping files

**Stage 6 — demo-ml-designer** *(conditional)*
- Skip if: no ML scenes detected in `{slug}-demo-script.md`, OR `{slug}-ml-config.json`
  exists AND data model hasn't changed
- Detect ML scenes: look for terms like "ML anomaly", "anomaly detection", "swimlane",
  "anomaly_score" in the script
- Read: `../demo-ml-designer/SKILL.md`
- Inputs: `{slug}-demo-script.md`, `{slug}-data-model.json`
- Outputs: `{slug}-ml-config.json`, `{slug}-ml-setup.md`

**Stage 7 — demo-validator**
- Always run last before deploy — regenerate even if it exists
- Read: `../demo-validator/SKILL.md`
- Inputs: all available `{slug}-*.json` and `{slug}-*.md` files in workspace
- Outputs: `{slug}-demo-checklist.md`, `{slug}-risks.md`

**Stage 8 — demo-cloud-provision** *(optional — new cluster path only)*
- **Requires explicit SA approval** to spend resources / create infrastructure (unless
  the user already clearly requested provisioning this session)
- Skip if: `{workspace}/.env` already exists and credentials are valid
- Run if: user requests "create a new cluster", "spin up a serverless project", or no `.env`
  exists and deployment was requested
- Read: `../demo-cloud-provision/SKILL.md`
- Inputs: deployment type preference, region, engagement slug
- Outputs: `{workspace}/.env`, `{workspace}/.env.example`, `{slug}-provision-log.md`
- Note: if user wants to reuse an existing cluster for a new engagement, copy the `.env`
  from the prior workspace and update `DEMO_SLUG`, `ENGAGEMENT`, and `INDEX_PREFIX` —
  no re-provisioning needed

**Stage 9 — demo-deploy** *(optional — runs after validator if cluster target is known)*
- **Requires explicit SA approval** to run generated `bootstrap.py` against a **live**
  cluster **and** **human review** of `bootstrap.py`, `{slug}-platform-audit`, `{slug}-risks`,
  `{slug}-demo-checklist.md`, and any committed `kibana-objects/` / `kibana/` imports the
  script will execute — unless the user has clearly stated they already reviewed them.
  **`bootstrap.py --dry-run`** and generating the script **do not** count as deploy.
  See `docs/decisions.md` **D-024** and `AGENTS.md`.
- Skip if: user has not requested deployment and no `.env` is present
- Run if: `.env` exists in the workspace (from stage 8 or copied from another engagement)
  AND user says "deploy", "build it", "set up the cluster", or "bootstrap" **after** review
- Requires: `{workspace}/.env` — stop and surface a clear message if missing
- Read: `../demo-deploy/SKILL.md` (including the **Automation contract** — every **in-scope**
  Kibana / Security / Observability asset must be created via APIs inside `bootstrap.py`,
  with definitions sourced from `elastic/agent-skills` where applicable — not left as
  manual UI steps)
- Inputs: `{workspace}/.env`, `{slug}-data-model.json`, `{slug}-ml-config.json` (if present),
  `{slug}-demo-script.md` / checklist / any supplemental specs (dashboards, agents, Sec, etc.)
- Outputs: `{workspace}/bootstrap.py`, `{slug}-deploy-log.md` — deploy log must list **all**
  resource classes created for **this** engagement (indices, ML, Kibana objects, rules,
  agents, etc.) and must not claim “done” while required assets are still manual

## Step 5: Deliver the Handoff Summary

When all stages are complete, produce a structured handoff:

```
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
 DEMOBUILDER COMPLETE — [Company]
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

📁 Workspace: $DEMOBUILDER_ENGAGEMENTS_ROOT/{slug}/  (default: ~/engagements/{slug}/)

ARTIFACT SUMMARY
────────────────
Discovery & Context
  ✅  {slug}-discovery.json            — structured customer profile
  ✅  {slug}-confirmation.md           — send to customer before demo
  ✅  {slug}-gaps.md                   — internal follow-up questions

Platform & Feasibility
  ⏭   or ✅  {slug}-current-state.json  — diagnostic (optional)
  ✅  {slug}-platform-audit.json       — feature feasibility matrix
  ✅  {slug}-platform-audit.md        — SE briefing

Demo Script
  ✅  {slug}-demo-script.md            — full SE script with scenes and queries
  ✅  {slug}-demo-brief.md             — one-page AE brief

Build Artifacts
  ✅  {slug}-data-model.json           — index mappings, build order, seed data spec
  ✅  {slug}-data-model.md             — human-readable build overview
  ⏭   {slug}-ml-config.json           — skipped if no ML scenes in script

Readiness
  ✅  {slug}-demo-checklist.md         — pre-demo checklist (timed)
  ✅  {slug}-risks.md                  — risks and fallbacks

Deploy *(if cluster provisioned)*
  ✅  {slug}-provision-log.md          — cluster info (if provisioned)
  ✅  bootstrap.py                     — generated deployment (15 steps; step 13 = scoped Kibana/platform APIs)
  ✅  {slug}-deploy-log.md             — what was created for this engagement
  ⏭   (skipped — no cluster target provided)

PLATFORM STATUS: [Green / Amber / Red from platform audit]
  Ready now:     [list features verified]
  Setup needed:  [gaps from audit]
  Not in scope:  [explicitly out of story]

BEFORE YOU BUILD  *(shown only if cluster not yet deployed)*
─────────────────
  1. Run demo-cloud-provision or copy an existing .env
  2. Run demo-deploy → python3 bootstrap.py --dry-run first
  3. Run end-to-end checks from {slug}-demo-checklist.md (scenes vary by demo)

  *(If already deployed, check {slug}-deploy-log.md)*

SEND TO CUSTOMER
─────────────────
  {slug}-confirmation.md  (after review — remove internal notes)

DEMO DAY
─────────
  Follow {slug}-demo-checklist.md and {slug}-demo-brief.md
  Go / No-Go per {slug}-risks.md
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
```

## Handling Partial Inputs

**Only discovery notes, no diagnostic:**
Run stages 1, 3, 4, 5, 6 (if ML), 7. Skip stage 2. Platform audit runs in partial mode.

**Only a diagnostic file, no discovery notes:**
Run stage 2 only. Output the current-state and findings. Tell the user: "Run
`demo-discovery-parser` with discovery notes to continue the pipeline."

**Discovery notes + diagnostic:**
Run all stages in order. Full audit with both inputs.

**Resuming a partial build:**
User says "continue the [customer] demo build" or "pick up where we left off."
Take inventory, identify what's missing, run only the pending stages.

**User wants to regenerate one stage:**
"Rewrite the demo script — new stakeholder joined the meeting."
Run only `demo-script-template` with the updated context, then re-run `demo-validator`.
Leave all other artifacts unchanged unless downstream stages must react.

**User wants to deploy to a new cluster:**
"Create a new serverless project for this demo and deploy it."
Run stage 8 (demo-cloud-provision) to provision and write `.env`, then stage 9 (demo-deploy)
to generate and execute `bootstrap.py`. Stages 1–7 already complete — skip them.

**User wants to deploy to an existing cluster:**
"I already have a cluster — here are my credentials." Or they copy a `.env` from another
engagement. Skip stage 8. Run stage 9 only. Verify the `.env` has all required fields before
generating `bootstrap.py`.

**User running a second engagement on the same cluster:**
"Reuse this cluster for another customer." Copy `.env`, update `DEMO_SLUG`, `ENGAGEMENT`,
and `INDEX_PREFIX` (e.g. `cb-` → `acme-`). Run stage 9 with the new slug’s artifacts.
No re-provisioning unless credentials or endpoint change; rebuild data model only if scope differs.

## What Good Looks Like

**Full cold start:** User drops a PDF and a diagnostic ZIP. Orchestrator auto-detects
both, runs all 7 stages in order, delivers a complete workspace with 12+ files and a
clear handoff summary. Total time to run: the time it takes to execute each stage.

**Resume from discovery JSON:** User already has `{slug}-discovery.json` from a prior
session. Orchestrator skips stage 1, runs stages 3–7, outputs script + data model +
checklist. Clearly reports what was skipped and why.

**RED platform audit — auto-adjust:** Demo scoped for Agent Builder but customer cluster is
on 8.x self-managed. Orchestrator surfaces the blocker, removes Agent Builder from script
scope, proceeds with what the platform supports, notes the removed scene in the handoff
under "Scope adjustments."

**End-to-end with deploy:** User provides discovery notes and says "create a serverless
project and deploy this demo." Orchestrator runs all 9 stages: builds the full artifact
set, provisions the cluster, generates and executes `bootstrap.py`. Delivers a deploy log
confirming 4 indices created, seed data loaded, ELSER endpoint warmed.

**Multi-customer on shared cluster:** First engagement deployed with `INDEX_PREFIX=cb-`.
User adds a second engagement with `INDEX_PREFIX=acme-`. Orchestrator copies `.env`,
updates slug fields, skips provisioning, runs stage 9 for the new artifacts. Both demos
coexist on the same cluster.

**Pre-demo morning check:** Demo was deployed yesterday. SE asks "is this engagement ready?"
Orchestrator reads `.env` + data model, runs `demo-status`, and returns a compact ✅/❌
report with paste-ready fix commands for anything off.

**Post-demo cleanup:** Demo went well. SE asks to tear down this engagement’s resources.
Orchestrator runs `demo-teardown` — stops ML jobs, removes Kibana objects, deletes indices
and all supporting infrastructure. If INDEX_PREFIX was set (shared cluster), only prefix-
matching resources are removed. Offers to delete the serverless project entirely if it was
provisioned specifically for this engagement.

**Orchestrator as SE daily driver:** SE starts every engagement by dropping discovery
notes into a prompt. Orchestrator handles the rest — they get a script, a data model,
and a pre-demo checklist without touching any individual skill manually.
