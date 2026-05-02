# demobuilder — Open Items Requiring User Action

*Generated from the post-mortem and current skill review. Updated as items are resolved.*
*Last updated: 2026-05-01*

For **agent behavior** (orchestrator path, `$DEMOBUILDER_ENGAGEMENTS_ROOT` outputs, deploy approvals), see
[`AGENTS.md`](../AGENTS.md), [`docs/engagements-path.md`](../docs/engagements-path.md), and [`docs/runtimes/`](../docs/runtimes/).

---

## 🔴 Blocking — Pipeline Won't Work Without These

### 1. Install elastic/agent-skills
The demobuilder pipeline depends on skills from `https://github.com/elastic/agent-skills` for actual API integration with Elastic Cloud and Kibana. Without them, `demo-cloud-provision` has no mechanism to call the Cloud API, and `demo-deploy` can't create Agent Builder configs or Kibana dashboards.

**Install the full plugin** so **Search, Observability, and Elastic Security** skills are all available. A given engagement may only use search — but SIEM/detection demos require Security skills without a separate install.

**Skills needed (install all — representative list):**
- `cloud/setup` — configure EC_API_KEY (must run first)
- `cloud/create-project` — create serverless projects
- `cloud/manage-project` — connect to existing projects, delete after demo
- `kibana/agent-builder` — create/update agents in Agent Builder
- `kibana/kibana-dashboards` — deploy dashboards and Lens visualizations
- `kibana/kibana-connectors` — set up email/webhook connectors for Workflows
- **Security (SIEM / detection / cases):** e.g. `security-detection-rule-management`, `security-alert-triage`, `security-case-management`, `security-generate-security-sample-data` — required for Security-scoped or hybrid demos

**How:** Clone `https://github.com/elastic/agent-skills` and install per its README, or install via the Claude Code plugin mechanism. These should be available alongside the demobuilder skills in the same session.

### 2. Run `cloud-setup` once to configure EC_API_KEY
Before `demo-cloud-provision` can create any serverless project, the `cloud-setup` skill needs to run to configure and validate your Elastic Cloud API key. This is a one-time setup.

**How:** In a Claude session with the elastic/agent-skills installed, say: "set up my Elastic Cloud credentials" and follow the prompts. **Never paste the API key directly into chat** — the skill handles secure entry.

---

## 🟡 Required Before First Live Demo Deploy

### 3. Validate bootstrap.py against a real cluster
`demo-deploy` generates `deploy/bootstrap.py` from the data model, but it has not yet been executed against a real cluster. The generated script follows the correct patterns, but edge cases (network timeouts, ELSER model load time, Kibana ndjson import formatting) may surface only on a live run.

**Action:** Pick one demo (Citizens Bank is the best-validated). Engagement root defaults to **`~/engagements`** (see [`docs/engagements-path.md`](../docs/engagements-path.md)):
```bash
cd /path/to/demobuilder
ROOT="${DEMOBUILDER_ENGAGEMENTS_ROOT:-$HOME/engagements}"
set -a && source "$ROOT/citizens-bank/.env" && set +a
python3 "$ROOT/citizens-bank/deploy/bootstrap.py" --dry-run
# Review output, then:
python3 "$ROOT/citizens-bank/deploy/bootstrap.py"
```
Note any failures and report back — the script template in `demo-deploy/SKILL.md` may need adjustments.

### 4. Kibana saved objects — files in the engagement workspace
`deploy/bootstrap.py` imports from paths such as `deploy/kibana-objects/{slug}-*.ndjson` and optional
`deploy/kibana/workflows/*`, `deploy/kibana/agent/*.json` (see `docs/decisions.md` **D-024**). Those files
are **authored** using **`elastic/agent-skills`** (e.g. `kibana-dashboards`) and **export-first**
from Kibana when needed (**D-017**), then **saved under `{engagement_dir}/deploy/`** — not generated
on the fly during deploy unless `demo-kibana-builder` exists.

**Action:** Keep dashboards/agent exports under `deploy/`; ensure `deploy/bootstrap.py`
references them and runs **`saved_objects/_import`** (and related APIs) — not manual UI import.

**Review gate:** Do **not** run `deploy/bootstrap.py` against a cluster until the SA has reviewed
`deploy/bootstrap.py`, `demo/{slug}-platform-audit`, `deploy/{slug}-risks`, and `deploy/{slug}-demo-checklist.md` (**D-024**, `AGENTS.md`).

### 5. Configure email connector for Workflows demos
The `online-order-notification` Workflow (Lowe's demo) and any demo using Kibana Workflows that send email requires a pre-configured Kibana Email connector. The `kibana-connectors` skill handles this, but the SMTP settings (or SES/Mailgun config) need to be provided.

**Action:** Using the `kibana-connectors` skill, configure an email connector in the demo cluster before running the Lowe's bootstrap.

---

## 🔵 Backlog — Skills to Build Next

### 6. `demo-kibana-builder` *(optional accelerator)*
Generates baseline Kibana saved objects (dashboards, Lens, ES|QL panels) from `data/{slug}-data-model.json`
+ `demo/{slug}-demo-script.md`. **Not required** if the SA commits exports under `deploy/kibana-objects/`
as in **D-024**; this skill would automate first-time **generation** only.

**Priority:** Medium — improves repeatability; file-based imports remain the contract.
**Inputs:** `data/{slug}-data-model.json`, `demo/{slug}-demo-script.md`
**Outputs:** `deploy/kibana-objects/{slug}-dashboards.ndjson`, etc.
**Depends on:** `kibana-dashboards` skill from elastic/agent-skills

### 7. `demo-data-generator`
Generates seed data as reviewable flat JSON files from the data model's sample data spec, before `deploy/bootstrap.py` runs. Makes seed data version-controllable and re-loadable independently.

**Priority:** Medium — current approach (generating data inside bootstrap.py) works but isn't reviewable.
**Inputs:** `data/{slug}-data-model.json`
**Outputs:** `data/{slug}-{index-name}-seed.json` (one file per index)

### 8. `demo-refresh`
Re-injects anomaly data T-2h before a demo. Separate from `deploy/bootstrap.py --step 14` because timing, verification, and failure modes are different from initial deployment. SEs will run this the morning of every demo.

**Priority:** Medium — until built, SEs use `python3 deploy/bootstrap.py --step 14` directly.
**Inputs:** `{engagement_dir}/.env`, `data/{slug}-ml-config.json`
**Outputs:** confirmation that injection docs are indexed and ML score ≥ 75 on target

### 9. `demo-fleet-integrations`
Installs and configures Elastic Fleet integration packages (EPM) for demo environments. Provides managed assets — ingest pipelines, index templates, ILM policies, dashboards — from the Integrations catalog rather than hand-rolled equivalents. Supports both agent-based collection and synthetic bulk ingest into integration-managed indices.

**Priority:** Medium — originated from 2026lenovoGAIaaS engagement.
**Status:** Stub SKILL.md exists at `skills/demo-fleet-integrations/SKILL.md` — implementation pending.
**Inputs:** `demo/{slug}-demo-script.md`, `data/{slug}-data-model.json`, `.env`
**Outputs:** Fleet policy JSON, integration package install steps, integration-managed index templates

### 10. `demo-ingest-pipeline-design`
Designs and generates Elasticsearch ingest pipeline definitions from a field mapping specification. Produces deployment-ready pipeline JSON artifacts consumed by `kibana-streams-manage` (for Streams wiring) or directly deployed by `deploy/bootstrap.py` step 4.

**Priority:** Medium — originated from 2026lenovoGAIaaS engagement.
**Status:** Stub SKILL.md exists at `skills/demo-ingest-pipeline-design/SKILL.md` — implementation pending.
**Inputs:** `data/{slug}-data-model.json`, field samples or log examples
**Outputs:** `data/pipelines/{slug}-{name}-pipeline.json`

### 11. `kibana-streams-manage`
Creates, configures, and wires Kibana Streams with ingest pipelines for demo environments. Extends the read-only `kibana-streams` skill with write-side operations (create stream, set routing, attach pipeline, configure retention).

**Priority:** Medium — originated from 2026lenovoGAIaaS engagement. Required when the demo story is Streams-native ingest.
**Status:** Stub SKILL.md exists at `skills/kibana-streams-manage/SKILL.md` — implementation pending.
**Depends on:** `demo-ingest-pipeline-design` (pipelines must exist before wiring)

---

## ⚪ Process / Infrastructure

### 12. Add `read:org` scope to GitHub token
The current keychain token has `repo` scope but not `read:org`, which blocks `gh auth login` and the gh CLI. Not blocking for demobuilder usage, but limits automation.

**Action:** Regenerate the GitHub PAT at `github.com/settings/tokens` and include `read:org`.

### 13. Run evals for demo-cloud-provision and demo-deploy
Both skills have `evals/evals.json` written but the eval loop (run → grade → benchmark) hasn't been executed. These evals require a real cluster to be meaningful.

**Action:** After completing items 1–3 above, run the evals for these two skills using the skill-creator eval loop.

### 14. Provide reference repos before any Workflow or Agent Builder build

Before any demo that includes Kibana Workflows or Agent Builder, load these repos into context:
- `https://github.com/elastic/workflows` — authoritative Workflow YAML examples
- `https://github.com/elastic/kibana-agent-builder-sdk` — Agent Builder tool/agent API schema

**How:** Either clone locally and reference by path, or provide the URL and ask Claude to fetch the README and examples before starting. Without these, Workflow builds will require multiple debugging cycles.

### 15. Validate `.ml-anomalies-*` field names on first Serverless ML build

On first use of ML anomaly detection on Serverless, run this before writing any query or dashboard:
```
GET .ml-anomalies-*/_mapping
```
The Serverless field names differ from documentation: use `record_score`, `timestamp`, `partition_field_value`, `by_field_value`. See `skills/demo-deploy/references/serverless-differences.md`.

### 16. Update `skills/demo-status/demo_status.py` for D-037 subfolder layout

The `demo_status.py` helper script still resolves Kibana saved objects from `kibana-objects/*.ndjson`
(old flat layout). After D-037, those files live under `deploy/kibana-objects/`. The script will
silently find zero saved objects on any engagement created after the reorganization.

**Action:** Update the glob patterns in `demo_status.py` to search `deploy/kibana-objects/` and
`deploy/kibana/**/*.ndjson` instead of the root-level paths. Verify against a live engagement.

---

## ✅ Resolved

- **Workspace root convention** — Engagement workspaces live under **`~/engagements/{slug}/`** by default (or **`$DEMOBUILDER_ENGAGEMENTS_ROOT/{slug}/`** if set) outside the repo. See `docs/decisions.md` D-019, D-023 and [`docs/engagements-path.md`](../docs/engagements-path.md).
- **Engagement folder reorganization (D-037)** — All pipeline outputs are now organized into audience-scoped subfolders: `opportunity/` (AE/SDR), `demo/` (SA design), `data/` (data model / ML / seed), `deploy/` (bootstrap, teardown, Kibana objects). `.env` remains at the engagement root. All skill SKILL.md files, `scripts/inventory.py`, `docs/pipeline.md`, and `docs/decisions.md` updated accordingly. Merged to main via PR #5.
- **Cloud-synced engagement root** — Google Drive (and OneDrive) documented as a valid `DEMOBUILDER_ENGAGEMENTS_ROOT` with pathing, sync latency, and credential security guidance. See `docs/engagements-path.md`.
- **Pipeline efficiency layer** — Pipeline state file, context pruning strategy, model routing guidance, subagent parallelization map, and `scripts/inventory.py` zero-AI CLI added. See `docs/efficiency.md`.
- **Vulcan integration** — `demo-vulcan-generate` skill added as optional Stage 4.5 in the pipeline. Vulcan outputs land in `data/` and `data/seed/`. See `docs/decisions.md` D-034.
