# demobuilder — Architecture Decisions

*Rationale for the choices baked into the pipeline. Updated as significant decisions are made.*
*See `docs/postmortem.md` for the full session post-mortem that prompted many of these.*

---

## D-020: Default to latest GA for new stacks; validate version for existing

**Decision:** When **creating** a new Elastic Cloud deployment or Serverless project for a
demo, use the **latest generally available** stack version for that product **unless** the
SA requests a specific version. When **using an existing** deployment, project, or cluster,
**resolve and record** Elasticsearch and Kibana versions (e.g. `GET /`, `/api/status`, or
diagnostic output) **before** producing scripts, data models, or plans. All guidance and
automation must be **scoped to that version** and deployment type.

**Rationale:** ES|QL, APIs, ML, Kibana embeddables, Agent Builder, and Workflows all vary
by version; assuming “latest” on a customer’s 8.x cluster causes failed demos.

**Applied to:** `skills/demobuilder/SKILL.md`, `skills/demo-cloud-provision/SKILL.md`,
`skills/demo-platform-audit/SKILL.md`, `skills/demo-script-template/SKILL.md`, `AGENTS.md`,
`.cursor/rules/demobuilder.mdc`, `README.md`.

**Date:** 2026-04-20 | **Session:** version policy

---

## D-021: Enterprise showcase; multi-input; search / Observability / Security / cross-solution

**Decision:** Demos should **assume enterprise-level Elastic features** are in play when they
address customer outcomes in the inputs — subject to platform audit, license, and version.
**Inputs** may include discovery notes, diagnostic files, supplemental notes from the
discovery team, and **architecture diagrams** illustrating current-state environments.
The primary use case may be **search / analytics**, **Observability**, **Elastic Security**,
or a **deliberate combination**; the pipeline must not default to “core search only” when
artifacts point elsewhere. **Cross-solution** demos (e.g. unified data, correlated
investigation) are acceptable when they match stated needs.

**Rationale:** Pre-sales stories follow the customer’s domain; many engagements are
Observability- or Security-led. Diagrams and team addenda are common and should be
first-class context.

**Applied to:** `skills/demobuilder/SKILL.md`, `skills/demo-discovery-parser/SKILL.md`,
`skills/demo-script-template/SKILL.md`, `AGENTS.md`, `.cursor/rules/demobuilder.mdc`, `README.md`.

**Date:** 2026-04-20 | **Session:** solution scope and inputs

---

## D-022: Solution-first narrative in scripts and plans

**Decision:** Unless the SA specifies otherwise, demo **scripts** and **plans** should
structure the storyline **solution first**: lead with **outcomes and business value**
linked to the customer’s **key asks** from discovery inputs, then describe **supporting
Elastic capabilities** (data, queries, ML, Security/Observability apps, agents, etc.) that
realize those outcomes. If primary goals or asks are **not clear** from artifacts, the agent
should **ask the SA for guidance** rather than guessing the headline narrative.

**Rationale:** Executives and business sponsors need the “why” before the “how”; technical
depth still follows, but order matters for retention and credibility.

**Applied to:** `skills/demo-script-template/SKILL.md`, `skills/demobuilder/SKILL.md`,
`skills/demo-validator/SKILL.md`, `AGENTS.md`, `.cursor/rules/demobuilder.mdc`, `README.md`.

**Date:** 2026-04-20 | **Session:** narrative arc

---

## D-001: Per-engagement `.env` file for credential isolation

**Decision:** Each engagement workspace (`engagements/{slug}/` under the demobuilder repo) holds its own `.env` file with cluster credentials. No global config.

**Rationale:** An SE running demos for Citizens Bank and IHG Club simultaneously — possibly on the same cluster — needs clean separation of credentials and namespace. A global config would require constant switching and risks cross-contamination.

**Implications:** The `p(name)` helper in `bootstrap.py` applies `INDEX_PREFIX` to every resource name, so indices, pipelines, templates, ML jobs, and Kibana index patterns all carry the engagement's namespace prefix. Copy workflow: `cp citizens-bank/.env ihg-club/.env` then update 3 fields (DEMO_SLUG, ENGAGEMENT, INDEX_PREFIX).

**Date:** 2026-04-15 | **Session:** initial build

---

## D-002: Idempotent bootstrap.py with check-before-create

**Decision:** Every resource creation in `bootstrap.py` checks for existence first. `--step N` flag resumes from a specific step. Data load checks doc count before loading (90% threshold).

**Rationale:** A failed deploy at step 9 shouldn't require tearing down steps 1–8. Demo environments are time-pressured — if something fails, you need to pick up where you left off, not start over.

**Implications:** Scripts are longer (each step has a check + create path), but they're safe to re-run. The check-before-create pattern also means running bootstrap on a half-deployed cluster is safe.

**Date:** 2026-04-15 | **Session:** initial build

---

## D-003: Separate provision and deploy skills

**Decision:** `demo-cloud-provision` and `demo-deploy` are distinct skills with a clean handoff via `.env`.

**Rationale:** An SE might provision once and deploy many times (adding a second customer on the same cluster with a different INDEX_PREFIX). Or they might bring their own existing cluster and skip provisioning entirely. Combining the two would force unnecessary reprovisioning or complex conditional logic.

**Implications:** `demo-cloud-provision` is optional. `demo-deploy` only needs a valid `.env` — it doesn't care how that `.env` was created.

**Date:** 2026-04-15 | **Session:** initial build

---

## D-004: `demo_critical_docs` as first-class concept in the data model

**Decision:** The data model spec includes a `demo_critical_docs` array for each index: specific documents that must exist, be individually verified by `_id` or unique field, and produce specific demo behavior.

**Rationale:** A demo where the script says "here's merchant VND-0412 with 7 suspicious claims" and the cluster has none of those documents fails visibly in front of the customer. Bulk doc count statistics don't catch this. Named, individually verified documents are the safety net.

**Implications:** `bootstrap.py` indexes demo_critical_docs individually (not in bulk), verifies each one, and reports specifically on their presence. `demo-status` spot-checks them as part of its readiness check. `demo-validator` lists them explicitly in the pre-demo checklist.

**Date:** 2026-04-15 | **Session:** initial build

---

## D-005: Two-dimensional shard density metric

**Decision:** `demo-diagnostic-analyzer` reports shard density on two axes: shards/GB of data AND (total shards / node count) / heap_GB_per_node.

**Rationale:** Discovered during real-data validation against Deutsche Telekom SOC-T (152-node, 1.18PB cluster). The first metric alone rated DT as healthy (0.027 shards/GB) — which it was from a data-sizing perspective. But 211 shards/node with 30GB heap is a different signal worth surfacing. One metric without the other is misleading in opposite directions depending on cluster topology.

**Thresholds:**
- Shards/GB: <0.1 (>10GB avg) → healthy, 0.1–1 → monitor, 1–20 → elevated, >20 → many tiny shards
- Shards/node/heap: <20 → healthy, 20–40 → elevated, >40 → high

**Date:** 2026-04-15 | **Session:** initial build (corrected via DT validation)

---

## D-006: Negative assertions in evals must specify scope

**Decision:** Eval assertions that check for the *absence* of content (e.g., "no competitor mentions") must specify which section of the output is being checked, not the full document.

**Rationale:** A skill may correctly mention a competitor in an internal "do not mention" instruction block for SE awareness — this is correct behavior. A blunt full-document assertion will falsely fail. Assertions like "not mentioned in talking points or scene narration" are precise; "not mentioned anywhere" is brittle.

**Applied to:** `demo-script-template` evals, and as a general rule for all future negative assertions.

**Date:** 2026-04-15 | **Session:** post-mortem

---

## D-007: elastic/agent-skills as the API integration layer

**Decision:** `demo-cloud-provision` and `demo-deploy` delegate cloud and Kibana API calls to skills from `elastic/agent-skills` (https://github.com/elastic/agent-skills) rather than implementing API clients from scratch.

**Rationale:** The Elastic-maintained skills handle auth, error handling, and API version differences for Cloud, Kibana, and Elasticsearch APIs. Duplicating this in demobuilder skills would create maintenance debt and diverge from the maintained implementations.

**Skills used:**
| elastic/agent-skill | Used by |
|---|---|
| `cloud-setup` | demo-cloud-provision (prerequisite) |
| `cloud-create-project` | demo-cloud-provision (serverless path) |
| `cloud-manage-project` | demo-cloud-provision (reuse path), demo-teardown (delete project) |
| `kibana-dashboards` | demo-deploy (if .ndjson doesn't exist), demo-kibana-builder (planned) |
| `kibana-agent-builder` | demo-deploy (agent config creation) |
| `kibana-connectors` | demo-deploy (Workflow email connectors) |
| `elasticsearch-esql` | demo-status (spot-check queries) |

**Gap:** No elastic/agent-skills exist yet for ML anomaly detection, Kibana Workflows (9.3), ingest pipelines, or index templates. These remain handled by `bootstrap.py` directly.

**Date:** 2026-04-15 | **Session:** post-mortem

---

## D-008: demo-status and demo-teardown as lifecycle skills

**Decision:** Add `demo-status` (pre-demo readiness pulse check) and `demo-teardown` (post-demo cleanup) to the skill set as first-class pipeline members.

**Rationale:** The pipeline was missing bookends: a way to quickly verify a deployed demo is healthy before going live, and a way to cleanly remove everything afterward. Without `demo-status`, SEs have to manually check 6 different things. Without `demo-teardown`, demo clusters accumulate resources and billing continues after demos end.

**Design principles:**
- `demo-status` runs in <60 seconds, produces ✅/❌ per resource, gives paste-ready fix commands
- `demo-teardown` is prefix-aware (only removes `{INDEX_PREFIX}*` resources on shared clusters), has `--dry-run`, generates a teardown log, offers to delete the cluster project if it was provisioned for this engagement only

**Date:** 2026-04-15 | **Session:** post-mortem

---

## D-009: Orchestrator references elastic/agent-skills explicitly and surfaces missing-plugin errors

**Decision:** The `demobuilder` orchestrator notes the elastic/agent-skills dependency upfront and tells the SE clearly if those skills aren't installed rather than failing silently at stage 8.

**Rationale:** An SE who completes stages 1–7 successfully and then hits a cryptic error at stage 8 because a dependency is missing will lose trust in the tool. A clear, actionable error message ("install elastic/agent-skills, see docs/todo.md") is better than a runtime failure.

**Date:** 2026-04-15 | **Session:** post-mortem

---

## D-010: docs/ directory for pipeline-level documentation

**Decision:** Add a `docs/` directory to the demobuilder repo for pipeline-wide documentation: postmortem, decisions log, and user-action todo list.

**Files:**
- `docs/postmortem.md` — full session post-mortem with lessons learned
- `docs/decisions.md` — this file; rationale for architectural choices
- `docs/todo.md` — items requiring user action (installs, credentials, validations)

**Rationale:** Pipeline-level knowledge was previously only in conversation history, which gets summarized and lost across context windows. These docs make it durable and accessible to any Claude session (or human) picking up the work cold.

**Date:** 2026-04-15 | **Session:** post-mortem

---

## D-011: Feature flag verification applies to Serverless AND ECH

**Decision:** `demo-cloud-provision` Step 4.5 verifies that Agent Builder and Kibana Workflows feature flags are enabled on **both Serverless and ECH deployments** before any build work begins.

**Rationale:** From the first-gen postmortem: Agent Builder and Workflows are not enabled by default on new projects. Initially documented as Serverless-only, but confirmed to apply equally to ECH until these features reach GA. Workflows is expected to GA with Elastic 9.4 — at that point the Workflows check can be relaxed for ECH, but Agent Builder may still require a flag. Always verify both.

**Applied to:** `demo-cloud-provision/SKILL.md` Step 4.5. `references/serverless-differences.md` Feature Flags section.

**Date:** 2026-04-15 | **Session:** first-gen review; corrected 2026-04-15

---

## D-012: Serverless ML field names documented as hard requirement

**Decision:** `demo-ml-designer` documents the Serverless `.ml-anomalies-*` field name differences and requires a `GET .ml-anomalies-*/_mapping` check before writing any query or dashboard panel.

**Rationale:** From the first-gen postmortem: all four ML dashboard panels had to be corrected after deployment because `anomaly_score`/`@timestamp`/`store_id`/`sku` do not exist on Serverless — the actual fields are `record_score`/`timestamp`/`partition_field_value`/`by_field_value`. A 2-minute mapping check prevents hours of rework.

**Applied to:** `demo-ml-designer/SKILL.md`. `references/serverless-differences.md`.

**Date:** 2026-04-15 | **Session:** first-gen review

---

## D-013: Workflow YAML reference required before writing any Workflow code

**Decision:** `demo-deploy` requires the `elastic/workflows` and `elastic/kibana-agent-builder-sdk` repos to be in context before any Workflow YAML or Agent Builder API call is written. 30-minute escalation rule: if progress stalls on an undocumented API, surface it as a blocker immediately.

**Rationale:** From the first-gen postmortem: Workflow debugging took ~3h and Agent Builder schema took ~2h because reference material was only found after problems were encountered. The `| first` Liquid filter, `_geo_distance` sort limitation, and `pattern` vs. `index` Agent Builder field were all documentable upfront.

**Applied to:** `demo-deploy/SKILL.md`. `references/workflow-patterns.md`. `references/serverless-differences.md`.

**Date:** 2026-04-15 | **Session:** first-gen review

---

## D-014: ELSER service body differs between Serverless and ECH

**Decision:** `bootstrap.py` uses `"service": "elser"` on Serverless (no `model_id`) and `"service": "elasticsearch"` with explicit `model_id` on ECH/self-managed.

**Rationale:** The actual working Serverless ELSER body uses `"service": "elser"` — the prior demobuilder implementation was using the wrong service name for Serverless, which would have caused step 8 to fail on every Serverless deploy.

**Applied to:** `demo-deploy/SKILL.md`. `demo-ml-designer/SKILL.md`.

**Date:** 2026-04-15 | **Session:** first-gen review

---

## D-015: Test session cleanup at T-10min is a required checklist item

**Decision:** The demo validator checklist includes a mandatory T-10min step to delete test agent sessions before going live.

**Rationale:** From the first-gen postmortem: pre-demo testing populates the session history index with test conversation turns that appear during the live demo. A single `_delete_by_query` on `@timestamp < now-10m` fixes it in seconds; not doing it risks surfacing test data during the demo.

**Applied to:** `demo-validator/SKILL.md`. `demo-deploy/SKILL.md` completion summary.

**Date:** 2026-04-15 | **Session:** first-gen review

---

## D-016: KIBANA_API_KEY is a required .env field for all Kibana asset operations

**Decision:** `KIBANA_API_KEY` is a required `.env` field used for **all** Kibana asset operations (Agent Builder, Workflows, Dashboards, Connectors, Saved Objects import) across all deployment types. It is not a fallback from `ES_API_KEY`. `bootstrap.py` uses `KB_KEY` (read from `KIBANA_API_KEY`) for all `kb()` calls.

**Rationale:** API key privilege requirements for Kibana vs. Elasticsearch are under active product change. Keeping separate keys is the safe default until product confirms a unified approach. The first-gen added this field after hitting 401 responses mid-build — it is now a provisioning-time requirement rather than a discovered fix.

**Applied to:** `references/env-reference.md`. `demo-cloud-provision/SKILL.md` `.env` and `.env.example` templates. `demo-deploy/SKILL.md` bootstrap.py credential block.

**Date:** 2026-04-15 | **Session:** first-gen review; elevated to required 2026-04-15

---

## D-017: Export-first dashboard pattern; never hand-write Lens JSON

**Decision:** Kibana dashboard saved objects must be exported from a working live panel. `migrationVersion` and `coreMigrationVersion` must be stripped before import or commit.

**Rationale:** From the first-gen postmortem: hand-written Lens panels took ~3h due to format errors. The Serverless inline `embeddableConfig.attributes` format differs from all public examples and changes between versions. Export-first produces a valid template in minutes.

**Applied to:** `demo-deploy/SKILL.md` Kibana step. `references/serverless-differences.md`.

**Date:** 2026-04-15 | **Session:** first-gen review

---

## D-018: ML datafeed geo_point workaround via runtime_mappings

**Decision:** When the datafeed source index contains a `geo_point` field, `demo-ml-designer` adds a `runtime_mappings` shadow to prevent datafeed failure.

**Rationale:** ML datafeeds cannot natively consume geo_point fields. The first-gen hit this with `store_location` and fixed it with a runtime mapping that shadows the field as a keyword emitting an empty string.

**Applied to:** `demo-ml-designer/SKILL.md` datafeed config section.

**Date:** 2026-04-15 | **Session:** first-gen review

---

## D-019: Engagement collateral grouped under `engagements/` subfolder

**Decision:** All per-engagement workspaces live under `engagements/{slug}/` relative to the **demobuilder repository root**, not under a separate home-directory tree such as `~/demobuilder-workspace/`.

**Rationale:** `{slug}` is always one engagement (demo-specific). Everything else belongs to the demobuilder codebase (`skills/`, `docs/`, scripts) or shared references — not mixed into engagement folders. Repo-local `engagements/` keeps the SA working in one clone; `.gitignore` can still omit credentials from VCS.

**Applied to:** `demobuilder/SKILL.md`, `demo-cloud-provision/SKILL.md`, `demo-deploy/SKILL.md`, `demo-status/SKILL.md`, `demo-teardown/SKILL.md`, `demo-deploy/references/env-reference.md`, `README.md`. `docs/todo.md` item 11 closed.

**Date:** 2026-04-20 | **Session:** workspace organization | **Updated:** 2026-04-20 — default path repo-local
