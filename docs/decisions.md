# demobuilder — Architecture Decisions

*Rationale for the choices baked into the pipeline. Updated as significant decisions are made.*
*See `docs/postmortem.md` for the full session post-mortem that prompted many of these.*

---

## D-001: Per-engagement `.env` file for credential isolation

**Decision:** Each engagement workspace (`~/demobuilder-workspace/{slug}/`) holds its own `.env` file with cluster credentials. No global config.

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
