# demobuilder — Open Items Requiring User Action

*Generated from the post-mortem and current skill review. Updated as items are resolved.*
*Last updated: 2026-04-15*

For **agent behavior** (orchestrator path, `engagements/` outputs, deploy approvals), see
[`AGENTS.md`](../AGENTS.md) and [`docs/runtimes/`](../docs/runtimes/).

---

## 🔴 Blocking — Pipeline Won't Work Without These

### 1. Install elastic/agent-skills
The demobuilder pipeline depends on skills from `https://github.com/elastic/agent-skills` for actual API integration with Elastic Cloud and Kibana. Without them, `demo-cloud-provision` has no mechanism to call the Cloud API, and `demo-deploy` can't create Agent Builder configs or Kibana dashboards.

**Skills needed (install all):**
- `cloud/setup` — configure EC_API_KEY (must run first)
- `cloud/create-project` — create serverless projects
- `cloud/manage-project` — connect to existing projects, delete after demo
- `kibana/agent-builder` — create/update agents in Agent Builder
- `kibana/kibana-dashboards` — deploy dashboards and Lens visualizations
- `kibana/kibana-connectors` — set up email/webhook connectors for Workflows

**How:** Clone `https://github.com/elastic/agent-skills` and install per its README, or install via the Claude Code plugin mechanism. These should be available alongside the demobuilder skills in the same session.

### 2. Run `cloud-setup` once to configure EC_API_KEY
Before `demo-cloud-provision` can create any serverless project, the `cloud-setup` skill needs to run to configure and validate your Elastic Cloud API key. This is a one-time setup.

**How:** In a Claude session with the elastic/agent-skills installed, say: "set up my Elastic Cloud credentials" and follow the prompts. **Never paste the API key directly into chat** — the skill handles secure entry.

---

## 🟡 Required Before First Live Demo Deploy

### 3. Validate bootstrap.py against a real cluster
`demo-deploy` generates `bootstrap.py` from the data model, but it has not yet been executed against a real cluster. The generated script follows the correct patterns, but edge cases (network timeouts, ELSER model load time, Kibana ndjson import formatting) may surface only on a live run.

**Action:** Pick one demo (Citizens Bank is the best-validated) and run from the **demobuilder repo root**:
```bash
cd /path/to/demobuilder
set -a && source engagements/citizens-bank/.env && set +a
python3 engagements/citizens-bank/bootstrap.py --dry-run
# Review output, then:
python3 engagements/citizens-bank/bootstrap.py
```
Note any failures and report back — the script template in `demo-deploy/SKILL.md` may need adjustments.

### 4. Generate and validate Kibana saved objects
`bootstrap.py` imports Kibana objects from `.ndjson` files, but these files don't exist yet — they need to be created. Until `demo-kibana-builder` is built (see below), dashboards and agent configs need to be created manually in Kibana and exported.

**Action:** For the Citizens Bank demo, manually build in Kibana:
- "Fraud Operations" dashboard (key visualizations from the script)
- Associate agent config (4 tools as defined in the data model)
- Export both as `.ndjson` and save to the workspace

This is the current gap in the pipeline — see the `demo-kibana-builder` skill in the backlog.

### 5. Configure email connector for Workflows demos
The `online-order-notification` Workflow (Lowe's demo) and any demo using Kibana Workflows that send email requires a pre-configured Kibana Email connector. The `kibana-connectors` skill handles this, but the SMTP settings (or SES/Mailgun config) need to be provided.

**Action:** Using the `kibana-connectors` skill, configure an email connector in the demo cluster before running the Lowe's bootstrap.

---

## 🔵 Backlog — Skills to Build Next

### 6. `demo-kibana-builder`
Generates Kibana saved objects (dashboards, Lens visualizations, ES|QL panels, index patterns) from the data model JSON. Currently the missing step between "cluster bootstrapped" and "demo visually ready."

**Priority:** High — without this, Kibana object creation is a manual step for every new demo.
**Inputs:** `{slug}-data-model.json`, `{slug}-demo-script.md`
**Outputs:** `kibana-objects/{slug}-dashboards.ndjson`, `kibana-objects/{slug}-agent-config.ndjson`
**Depends on:** `kibana-dashboards` skill from elastic/agent-skills

### 7. `demo-data-generator`
Generates seed data as reviewable flat JSON files from the data model's sample data spec, before `bootstrap.py` runs. Makes seed data version-controllable and re-loadable independently.

**Priority:** Medium — current approach (generating data inside bootstrap.py) works but isn't reviewable.
**Inputs:** `{slug}-data-model.json`
**Outputs:** `data/{slug}-{index-name}-seed.json` (one file per index)

### 8. `demo-refresh`
Re-injects anomaly data T-2h before a demo. Separate from `bootstrap.py --step 14` because timing, verification, and failure modes are different from initial deployment. SEs will run this the morning of every demo.

**Priority:** Medium — until built, SEs use `python3 bootstrap.py --step 14` directly.
**Inputs:** `{slug}/.env`, `{slug}-ml-config.json`
**Outputs:** confirmation that injection docs are indexed and ML score ≥ 75 on target

---

## ⚪ Process / Infrastructure

### 9. Add `read:org` scope to GitHub token
The current keychain token has `repo` scope but not `read:org`, which blocks `gh auth login` and the gh CLI. Not blocking for demobuilder usage, but limits automation.

**Action:** Regenerate the GitHub PAT at `github.com/settings/tokens` and include `read:org`.

### 10. Run evals for demo-cloud-provision and demo-deploy
Both skills have `evals/evals.json` written but the eval loop (run → grade → benchmark) hasn't been executed. These evals require a real cluster to be meaningful.

**Action:** After completing items 1–3 above, run the evals for these two skills using the skill-creator eval loop.

### ~~11. Decide on workspace root convention~~ ✅ Resolved
Default engagement path: **`engagements/{slug}/` under the demobuilder repository** (see `docs/decisions.md` D-019). Per-demo assets only; pipeline code stays in `skills/`, `docs/`, etc.

---

### 12. Provide reference repos before any Workflow or Agent Builder build

Before any demo that includes Kibana Workflows or Agent Builder, load these repos into context:
- `https://github.com/elastic/workflows` — authoritative Workflow YAML examples
- `https://github.com/elastic/kibana-agent-builder-sdk` — Agent Builder tool/agent API schema

**How:** Either clone locally and reference by path, or provide the URL and ask Claude to fetch the README and examples before starting. Without these, Workflow builds will require multiple debugging cycles.

### 13. Validate `.ml-anomalies-*` field names on first Serverless ML build

On first use of ML anomaly detection on Serverless, run this before writing any query or dashboard:
```
GET .ml-anomalies-*/_mapping
```
The Serverless field names differ from documentation: use `record_score`, `timestamp`, `partition_field_value`, `by_field_value`. See `skills/demo-deploy/references/serverless-differences.md`.

---

## ✅ Resolved

- *(none yet — this is the initial list)*
