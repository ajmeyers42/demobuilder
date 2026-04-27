---
name: demo-deploy
description: >
  Generates a `bootstrap.py` deployment script from the demobuilder pipeline outputs and
  executes it against the target cluster specified in the engagement's .env file. Creates
  whatever Elasticsearch and Kibana-side resources the **demo script, data model, platform
  audit, and validator** scope for that engagement — search, Observability, Security,
  hybrid, ML, semantic search, Agent Builder, etc. Provisions Kibana and related APIs
  via automation (no default “finish in the UI”) for every asset class that is in scope.
  Idempotent — safe to re-run if a step fails. Per-engagement .env isolates credentials.
  Completion via manual UI is only acceptable when the platform audit documents a hard blocker.

  ALWAYS use this skill when the user says "deploy the demo", "run the bootstrap",
  "set up the cluster", "load the data", "execute the deployment", or "it's ready to
  build — deploy it". Also trigger when demo-validator returns go/conditional-go and
  the user wants to proceed to actual deployment. Requires a .env file in the workspace
  (run demo-cloud-provision first if one doesn't exist).
---

# Demo Deploy

You are deploying a demo environment to an Elastic cluster. You generate a Python
deployment script (`bootstrap.py`) from the pipeline outputs, then execute it. The script
is the deployment record — it documents exactly what was created, in what order, and why.

Every step is idempotent: if the script is interrupted and re-run, it picks up where it
left off without creating duplicates or overwriting data that's already correct.

## Automation contract (skills — not one-off scripts, not manual UI)

**Single deployment surface:** `bootstrap.py` in the engagement workspace is the only
required executable for “deploy the demo.” It must perform **every** automated provisioning
step that **this engagement’s** script, checklist, data model, and platform audit require —
using Elasticsearch APIs for cluster-side work and **Kibana / Kibana-adjacent APIs** for
anything the story needs in the UI (saved objects, Observability, Security, Stack Management,
etc.), with `KIBANA_API_KEY` where applicable (see `references/env-reference.md`). Do **not**
hand off with a separate `deploy_*.py` beside bootstrap, “run these scripts in order,” or
“click through Kibana to finish” unless the **platform audit** records a genuine blocker and
the deploy log documents the exception.

**Scenario-driven — not a fixed stack:** No engagement is required to use SLOs, Agent Builder,
Security rules, or any other specific feature. **Derive** bootstrap contents from:

- `{slug}-demo-script.md` and `{slug}-demo-checklist.md` — what scenes and clicks must work
- `{slug}-platform-audit.json` — what is licensed, version-supported, and enabled
- `{slug}-data-model.json` — indices, pipelines, seed data
- `{slug}-ml-config.json` (if any) — ML and anomaly injection
- Optional specs (`{slug}-*-dashboards-spec.md`, `{slug}-agent-builder-spec.md`, etc.)

Skip entire subsystems when out of scope (e.g. pure Elasticsearch relevance demos may only
need data views + saved searches + dashboards; a SIEM demo may emphasize detection rules
and timelines; an Observability demo may emphasize SLOs and APM — **implement only what is
scoped**).

**Skill catalog (`elastic/agent-skills`) — pick what matches the scenario:**

The **full** `elastic/agent-skills` install includes **Search, Observability, and Security**
specialists. Keep them **available** for every engagement; **invoke** only what the script
and audit require (a non-Security demo does not call Security APIs — but the skills must
still be present so hybrid and Sec-first demos are supported without a different install).

Use these specialists to **author** payloads, NDJSON, and API bodies before or while writing
`bootstrap.py`. This table is a **menu**, not a mandatory checklist:

| Typical need | Skill (plugin) | Notes |
|--------------|----------------|--------|
| Dashboards / Lens / saved objects | `kibana-dashboards` | Any vertical — viz as code |
| Observability SLOs | `observability-manage-slos` | When the script calls for SLOs |
| Alerting rules (including SLO burn rate) | `kibana-alerting-rules` | Rule types vary by use case |
| Connectors | `kibana-connectors` | Before Workflows that notify |
| **Security / SIEM** | `security-detection-rule-management`, `security-alert-triage`, `security-case-management`, `security-generate-security-sample-data`, other `security-*` | First-class — use when demo is Sec or hybrid; sample data for POCs |
| Agent Builder | `elastic/kibana-agent-builder-sdk` + `{slug}-agent-builder-spec.md` | When script includes agents |
| ES|QL / analytics | `elasticsearch-esql` | Query validation and panels |

The deploying agent **implements** chosen payloads inside `bootstrap.py` (or loads JSON
adjacent files). “Placeholder only” assets are **incomplete** unless `{slug}-demo-checklist.md`
or validator explicitly allowed thin shells for a pilot.

**Anti-pattern:** Copying a prior engagement’s `bootstrap.py` or Kibana steps wholesale.
Each demo gets a **generated** script aligned to its artifacts.

**Elastic deployability and datatypes (`docs/decisions.md` D-025):** Do not author assets that
cannot be applied with supported APIs. Mappings and documents use **Elasticsearch field types**;
Agent Builder ES|QL tool `params.*.type` values must match what the Kibana server validates
(typically ES-style: `keyword`, `text`, `integer`, `date`, … — verify per stack). Security
rules, SLOs, and saved objects follow **product** schemas for the target version.

**Before building Workflows or Agent Builder configs** — read these reference files first:
- `references/serverless-differences.md` — Serverless-specific behaviors, feature flags,
  ML field names, Liquid array syntax, ELSER service names, saved objects quirks
- `references/workflow-patterns.md` — working Workflow YAML patterns, `| first` syntax,
  template variables, step types, stale-read warning, DELETE endpoint, search-by-name, deploy API commands
- `references/asset-manifest.md` — manifest index schema, `_manifest_add()` helpers,
  teardown read pattern (D-031)
- `hive-mind/patterns/agent-builder/AGENT_BUILDER_API_MANAGEMENT.md` — API management guide
  for Agent Builder (CRUD, tool types, system prompts, agent cloning) — D-034
- `hive-mind/patterns/dashboards/DASHBOARD_NDJSON_FORMAT.md` — Kibana dashboard NDJSON reference
  (Lens panel types, stable UUIDs, common pitfalls) — D-034

Do not iterate past 30 minutes on an undocumented Workflow or Agent Builder API without
surfacing it as a blocker and asking for the `elastic/workflows` or
`elastic/kibana-agent-builder-sdk` reference repos.

## Step 0: Review gate (before any live cluster changes)

**Do not** execute `bootstrap.py` against a **live** cluster (or run `demo-cloud-provision`
to create resources) until the SA has:

1. **Reviewed** the generated **`bootstrap.py`** (what it will create or mutate).
2. **Reviewed** analysis outputs the deploy relies on: **`{slug}-platform-audit`**, **`{slug}-risks`**,
   **`{slug}-demo-checklist.md`**, and any committed **`kibana-objects/`**, **`kibana/`**, or
   **`elasticsearch/`** files the script imports.
3. **Explicitly approved** provision/deploy for this session (same as `AGENTS.md`).

Allowed without that approval: **author** `bootstrap.py`, **`--dry-run`**, local edits to NDJSON,
connectivity checks that do not mutate production. See **`docs/decisions.md` D-024**.

## Step 1: Load the Environment

Read `{engagement_dir}/.env` (engagement directory = `{engagement_dir}`). All subsequent API calls use these credentials. Never
hardcode credentials in the script itself — always read from the `.env`.

If `.env` doesn't exist: stop and tell the user to run `demo-cloud-provision` first, or
to create a `.env` from the `.env.example` template.

Verify the `.env` has all required fields:
- `ELASTICSEARCH_URL` — non-empty, starts with `https://` or `http://`
- `KIBANA_URL` — non-empty
- `ES_API_KEY` — non-empty
- `DEPLOYMENT_TYPE` — one of: `serverless`, `ech`, `self_managed`, `docker`

Read `INDEX_PREFIX` (may be blank). If set, prepend it to every index name, template name,
and pipeline name in the deployment. Apply consistently so a prefix of `cb-` makes
`fraud-claims` → `cb-fraud-claims` everywhere including query references.

Read optional **`DEMO_ASSET_TAG`** — overrides the engagement id used in **`demobuilder:<id>`**
tags when set (see **`references/demobuilder-tagging.md`**, **`docs/decisions.md` D-026**).

## Step 2: Read Pipeline Outputs

Load all available artifacts from the workspace:
- `{slug}-data-model.json` — required. Defines all indices, templates, pipelines, build order.
- `{slug}-ml-config.json` — optional. ML jobs, datafeeds, injection plan.
- `{slug}-platform-audit.json` — read `deployment_type` and feature availability to
  adapt the bootstrap to the specific platform.
- `{slug}-demo-script.md`, `{slug}-demo-checklist.md`, and any supplemental specs
  (`{slug}-*-dashboards-spec.md`, `{slug}-agent-builder-spec.md`, Security plans, etc.) —
  required context for **step 13** so every scene that depends on a Kibana, Security, or
  Observability asset has a matching API step when those features are in scope.
- Optional: `{slug}-kibana-*.json` or other sidecar JSON if earlier stages materialized
  them; otherwise derive payloads using the **skill catalog** that matches this scenario.

Extract the build order from the data model — this is the sequence the script must follow.

**Kibana and ES collateral as files:** If the engagement workspace includes
`kibana-objects/{slug}-*.ndjson`, `kibana/workflows/*`, `kibana/agent/*.json`, or declarative
`elasticsearch/**` JSON, **`bootstrap.py` must load and apply them** via APIs (saved objects
import, Workflows, Agent Builder, ES `PUT`s) — single script, no parallel `deploy_*.py`. Paths
are relative to `{engagement_dir}` (**D-024**).

## Step 3: Generate `bootstrap.py`

Write a complete, executable Python script to `{engagement_dir}/bootstrap.py`.

The script structure:

```python
#!/usr/bin/env python3
"""
Demobuilder bootstrap — {Company} ({slug})
Generated: {date}
Deployment: {type} at {ELASTICSEARCH_URL}

Engagement dir is `{engagement_dir}` (default parent `~/engagements` per docs/engagements-path.md).
After `GET /`, print cluster version and warn if `ELASTIC_VERSION` in `.env` disagrees (D-020).

Usage:
  python3 bootstrap.py                    # deploy everything
  python3 bootstrap.py --dry-run         # print what would be done, no API calls
  python3 bootstrap.py --skip-data       # skip data loading (re-run after a config change)
  python3 bootstrap.py --step N          # resume from step N (see step list below)
  python3 bootstrap.py --only GROUP      # deploy only a named group (e.g. --only kibana, --only ml, --only data)

Steps:
  1.  Connectivity check (includes version validation vs ELASTIC_VERSION)
  1b. Kibana Space — ensure /s/{DEMO_SLUG} exists (create if absent, idempotent)
  2.  ILM / Data Stream Lifecycle policies
  3.  Enrich policies (create + execute)
  4.  Ingest pipelines
  5.  Component templates
  6.  Index templates (data streams)
  7.  Static indices (create if not exists)
  8.  ELSER inference endpoint
  9.  Semantic indices (requires ELSER from step 8)
  10. Load seed data
  11. ML anomaly detection jobs
  12. ML datafeeds (create + start)
  13. Kibana & platform UI assets (all API — see “Step 13” below; implement only sub-steps
      required by this engagement’s script, audit, and validator)
  14. Anomaly injection
  15. Warm ELSER endpoint
"""

import os, sys, json, time, argparse, re
import urllib.request, urllib.error

# ── Credentials (from .env) ─────────────────────────────────────────────────
ES_URL       = os.environ.get("ELASTICSEARCH_URL", "").rstrip("/")
KB_URL       = os.environ.get("KIBANA_URL", "").rstrip("/")
API_KEY      = os.environ.get("ES_API_KEY", "")
KB_KEY       = os.environ.get("KIBANA_API_KEY", "")  # used for all Kibana asset operations
DEP_TYPE     = os.environ.get("DEPLOYMENT_TYPE", "ech")
PREFIX       = os.environ.get("INDEX_PREFIX", "")
SLUG         = os.environ.get("DEMO_SLUG", "demo")
SPACE_PATH   = os.environ.get("KIBANA_SPACE_PATH", "").strip()  # e.g. /s/2026citizens-ai

def p(name): return f"{PREFIX}{name}" if PREFIX else name  # apply index prefix

# ── Demobuilder engagement tag (D-026) — merge into every API payload that has "tags" ──
def _engagement_id_for_tag() -> str:
    override = os.environ.get("DEMO_ASSET_TAG", "").strip()
    raw = override or (PREFIX.strip() if PREFIX.strip() else SLUG)
    s = re.sub(r"[-_\s]+", "", raw).lower()
    return s or "demo"

def demobuilder_tags() -> list[str]:
    return [f"demobuilder:{_engagement_id_for_tag()}"]

def merge_tags(existing):
    return sorted(set((existing or []) + demobuilder_tags()))

# ── HTTP helpers ─────────────────────────────────────────────────────────────
def es(method, path, body=None, *, ok=(200,201)):
    """Call Elasticsearch API. Raises on unexpected status."""
    ...

def kb(method, path, body=None, *, ok=(200,)):
    """Call Kibana API. Uses KIBANA_API_KEY (KB_KEY) for all Kibana asset operations
    (Agent Builder, Workflows, Dashboards, Connectors, Saved Objects)."""
    ...

def step(n, label):
    """Print step header and check if we should skip."""
    ...

# ── Step implementations ──────────────────────────────────────────────────────
def check_connectivity():   ...

def ensure_kibana_space():
    """Step 1b — create /s/{DEMO_SLUG} if absent. Idempotent; 409 = already exists."""
    space_id = SLUG.strip().lower()
    if not space_id:
        return
    # Space management API MUST use the default-space base URL (no SPACE_PATH prefix)
    import urllib.error
    headers = {"Authorization": f"ApiKey {KB_KEY}", "Content-Type": "application/json", "kbn-xsrf": "demobuilder"}
    req = urllib.request.Request(f"{KB_URL}/api/spaces/space/{space_id}", method="GET", headers=headers)
    try:
        urllib.request.urlopen(req, timeout=30)
        print(f"  Kibana space '{space_id}' already exists"); return
    except urllib.error.HTTPError as e:
        if e.code != 404: raise
    body = json.dumps({
        "id": space_id,
        "name": os.environ.get("ENGAGEMENT", space_id),
        "description": f"Demobuilder engagement {space_id}",
        "disabledFeatures": [],                          # enable all Kibana features in this space
        "solution": os.environ.get("KIBANA_SOLUTION", "es")  # "es", "oblt", or "security" per demo
    }).encode()
    req = urllib.request.Request(f"{KB_URL}/api/spaces/space", data=body, method="POST", headers=headers)
    try:
        urllib.request.urlopen(req, timeout=30)
        print(f"  Kibana space '{space_id}' created")
    except urllib.error.HTTPError as e:
        if e.code == 409: print(f"  Kibana space '{space_id}' already exists (409)")
        else: raise

def create_ilm_policies():  ...  # skipped on serverless (uses DSL)
def create_dsl_policies():  ...  # serverless only
def create_enrich_policies():    ...
def execute_enrich_policies():   ...
def create_pipelines():     ...
def create_component_templates(): ...
def create_index_templates():    ...
def create_static_indices():     ...
def deploy_elser():         ...
def create_semantic_indices():   ...
def load_seed_data():       ...
def create_ml_jobs():       ...
def start_datafeeds():      ...
def import_kibana_objects():     ...
def inject_anomalies():     ...
def warm_elser():           ...

# ── Main ──────────────────────────────────────────────────────────────────────
if __name__ == "__main__":
    ...
```

### Critical implementation details for each step

**Connectivity check (step 1)**
```python
resp = es("GET", "/")
version = resp["version"]["number"]
print(f"  Connected: {ES_URL}")
print(f"  Version:   {version}")
print(f"  Prefix:    '{PREFIX}' ({'applied' if PREFIX else 'none — using default index names'})")
# D-020: warn if .env ELASTIC_VERSION disagrees with live cluster
env_ver = os.environ.get("ELASTIC_VERSION", "").strip()
if env_ver and env_ver != version:
    print(f"  ⚠  ELASTIC_VERSION in .env ({env_ver}) != cluster ({version}) — update .env (D-020)")
# D-033: 9.4+ baseline gate — halt on older clusters
major, minor = (int(x) for x in version.split(".")[:2])
if (major, minor) < (9, 4):
    skip_check = os.environ.get("SKIP_VERSION_CHECK", "").lower() == "true"
    if not skip_check:
        print(f"  ⛔ Cluster version {version} is below the 9.4 baseline (D-033).")
        print(f"     Set SKIP_VERSION_CHECK=true in .env to override (unsupported).")
        sys.exit(1)
    print(f"  ⚠  Version {version} below 9.4 baseline — SKIP_VERSION_CHECK override active")
# D-031: initialise and ensure manifest index exists
_manifest_init()
_ensure_manifest_index()
_manifest["es_version"] = version
_manifest_push()
```
Fail fast and clearly if connectivity fails — don't attempt subsequent steps.

**Kibana Space (step 1b)**

Run `ensure_kibana_space()` immediately after connectivity. This step uses the **default-space
base URL** (`KB_URL/api/spaces/space`) — never the `SPACE_PATH`-prefixed URL — because the
space does not yet exist when this call is first made. All subsequent Kibana calls (step 13
and beyond) use `SPACE_PATH` as the prefix through the `kb()` helper:

```python
def kb(method, path, body=None, *, ok=(200,)):
    """Kibana API with SPACE_PATH prefix (set by KIBANA_SPACE_PATH in .env)."""
    full_path = f"{SPACE_PATH}{path}" if SPACE_PATH else path
    ...
```

**ILM / DSL (step 2) — deployment-type-aware**
```python
if DEP_TYPE == "serverless":
    # Serverless uses Data Stream Lifecycle, not ILM
    # Apply DSL via index template `lifecycle` block — no separate ILM API call
    print("  Serverless detected — using Data Stream Lifecycle (skipping ILM API)")
else:
    # Create ILM policy via /_ilm/policy/{name}
    for policy in data_model["ilm_policies"]:
        resp = es("PUT", f"/_ilm/policy/{p(policy['name'])}", policy["definition"])
        print(f"  ILM policy: {p(policy['name'])} → {resp.get('acknowledged')}")
```

**Enrich policies (step 3)**
Enrich policies MUST be executed (not just created) before any pipeline that references
them. The execute call is synchronous but can take seconds on large lookup indices.
```python
es("PUT", f"/_enrich/policy/{p(policy['name'])}", policy["match"])
es("POST", f"/_enrich/policy/{p(policy['name'])}/_execute")
# Poll until complete:
while True:
    status = es("GET", f"/_enrich/policy/{p(policy['name'])}")
    if status["policies"][0]["match"].get("indices"):
        break
    time.sleep(1)
```

**Idempotent index creation (step 7)**
Check existence before creating — never overwrite an existing index's data:
```python
try:
    es("HEAD", f"/{p(index['name'])}", ok=(200,))
    print(f"  {p(index['name'])}: exists — skipping")
except:
    es("PUT", f"/{p(index['name'])}", index["mapping"])
    print(f"  {p(index['name'])}: created")
```

**ELSER endpoint (step 8) — EIS for ECH, managed for Serverless (D-028)**

```python
endpoint_id = p("elser")
task_type   = "sparse_embedding"

# GET response wraps endpoints in {"endpoints": [...]} on 9.x — unwrap before checking
try:
    resp      = es("GET", f"/_inference/{task_type}/{endpoint_id}", ok=(200,))
    endpoints = resp.get("endpoints") or []
    existing  = endpoints[0] if endpoints else resp
    svc       = existing.get("service", "")
    expected  = "elser" if DEP_TYPE == "serverless" else "elastic"
    if svc == expected:
        print(f"  ELSER endpoint '{endpoint_id}': exists on correct service='{svc}' — skipping")
        _manifest_add("inference_endpoints", {"task_type": task_type, "id": endpoint_id})
        return
    elif svc:
        print(f"  ELSER endpoint '{endpoint_id}': found but service='{svc}' != expected '{expected}' — deleting to recreate")
        es("DELETE", f"/_inference/{task_type}/{endpoint_id}", ok=(200, 204))
except RuntimeError as e:
    if "404" not in str(e):
        raise

if DEP_TYPE == "serverless":
    # Serverless: managed endpoint — use "elser" service, no model_id (D-028)
    body = {"service": "elser", "service_settings": {}}
else:
    # ECH 9.4+: use Elastic Inference Service — do NOT deploy on ML node (D-028)
    body = {"service": "elastic", "service_settings": {"model_id": ".elser-2"}}

es("PUT", f"/_inference/{task_type}/{endpoint_id}", body)
print(f"  ELSER endpoint '{endpoint_id}': deploying via {'Serverless managed' if DEP_TYPE == 'serverless' else 'EIS'}...")
_manifest_add("inference_endpoints", {"task_type": task_type, "id": endpoint_id})
# EIS endpoints are ready immediately; Serverless managed may need a warm-up call
if DEP_TYPE != "serverless":
    print("  EIS endpoint ready (no model allocation needed)")
```

Cold ELSER inference on Serverless can take 30+ seconds on the first call. The warm-up
step (step 15) handles this, but allow extra time on first deploy.

**Seed data loading (step 10)**
Generate realistic synthetic data from the sample data spec in the data model. Use
`_bulk` API with batches of 500 documents. Check existing doc count first:
```python
count = es("GET", f"/{p(ds['index'])}/_count")["count"]
if count >= ds["seed_document_count"] * 0.9:  # 90% threshold — already loaded
    print(f"  {p(ds['index'])}: {count} docs exist — skipping load")
    continue
# Generate and bulk-index
```

**Data generation** — synthesize realistic values from the sample data spec:
- Use the `key_entities` distribution weights to sample realistic field values
- Generate `demo_critical_docs` first and index them individually (verified)
- Fill remaining doc count with randomized but realistic values
- For `days_since_filed`, `@timestamp`, etc. — use realistic date arithmetic from "now"

**ML jobs (step 11)**
```python
try:
    es("GET", f"/_ml/anomaly_detectors/{job['job_id']}", ok=(200,))
    print(f"  ML job {job['job_id']}: exists — skipping")
except:
    es("PUT", f"/_ml/anomaly_detectors/{job['job_id']}", job["analysis_config_etc"])
```

**Step 13 — Kibana & platform UI assets (complete everything in scope)**

**Not** “import NDJSON only.” Implement **every** Kibana API and related call that this
engagement’s **script + audit + validator** require — **omit** sub-bullets that are out of
scope. Honor `KIBANA_SPACE_PATH` in `.env` by prefixing paths (`/s/{space}/api/...`). Use
`KIBANA_API_KEY` for Kibana calls.

**13a — Data views** *(when the demo uses Discover, Lens, or dashboards bound to indices)*  
`POST /api/data_views/data_view` for each index pattern the script references (count and
names come from the data model + script — not a fixed pair). Idempotent: list existing
titles, skip if present.

**13b — Observability SLOs** *(when the script / checklist explicitly includes SLOs)*  
`POST /api/observability/slos` per `observability-manage-slos`. Indicator types vary (KQL,
APM, Synthetics, etc.). Treat `409` as already created.

**13c — Alerting rules** *(when the script needs alerts — SLO burn, metric threshold, etc.)*  
`POST /api/alerting/rule/{id}` with the correct `rule_type_id` for that scenario (e.g. SLO
burn rate: `slo.rules.burnRate`, `consumer`: `slo`) — validate with `kibana-alerting-rules`
for the target version.

**SLO stack docs:** Elastic Guide (concepts, create SLO, burn-rate alerts, troubleshoot incl.
reset) plus the API hub — see **`docs/references-observability-slo.md`** in demobuilder. Use the **`current`** Guide branch for 9.x stacks (D-033/D-025) and re-check pages when the stack version changes.

**13d — Saved objects / Dashboards** *(when dashboards/visualizations are in scope)*  
`POST /api/saved_objects/_import?overwrite=true` (multipart NDJSON). Prefer definitions
authored via `kibana-dashboards` from the data model + script. Apply `INDEX_PREFIX` to
index titles in queries before import.

**Stable UUIDs for dashboards and saved objects** — always use deterministic UUIDs derived
from the engagement ID + object name so re-deploys produce the same IDs and avoid duplicates:

```python
import uuid
_NS = uuid.UUID("6ba7b810-9dad-11d1-80b4-00c04fd430c8")  # UUID namespace (URL)

def stable_id(name: str) -> str:
    """Generate a stable UUID from engagement slug + object name."""
    return str(uuid.uuid5(_NS, f"{SLUG}:{name}"))
```

Use `stable_id("dashboard-name")` whenever authoring dashboard or saved object IDs.
This makes teardown reliable (IDs are known at generation time) and prevents duplicate
panels when bootstrap is re-run. Reference: `hive-mind/patterns/dashboards/DASHBOARD_NDJSON_FORMAT.md`.

**Kibana APIs (rules, connectors, saved objects):** See **`docs/references-kibana-apis.md`**
— Kibana Guide `current` for **Saved Objects** and **Alerting** (rules and connectors). SLO
burn-rate rules are one `rule_type_id` under Alerting; the Observability SLO reference
**`docs/references-observability-slo.md`** stays focused on SLOs + SLO burn-rate behavior (**D-025**).

**13e — Agent Builder** *(when the agent spec exists and audit allows it)*  
`PUT/POST` under `/api/agent_builder/...` per `{slug}-agent-builder-spec.md` — not a
manual “Create agent” handoff.

**13f — Workflows** *(when in scope and supported)*  
After connectors if needed — follow `references/workflow-patterns.md`.

**13g — Security / SIEM** *(when demo is Sec or hybrid)*

Use `security-*` skills as appropriate. For detection rules:

- **Custom demo rules (authored from scratch):** Upsert by `rule_id` — check existence
  first, POST if absent, PUT if present (increment `version`). Include `demobuilder_tags()`
  and `BOOTSTRAP_VERSION` (D-026, D-030).

- **Elastic-managed / prebuilt rules (D-032):** Never modify the original. Clone instead:

```python
# 1. Fetch the prebuilt rule as a template
existing = kb("GET", f"/api/detection_engine/rules?rule_id={source_rule_id}", ok=(200,))
# 2. Strip immutable/read-only fields
READ_ONLY = {"id","created_at","updated_at","updated_by","created_by",
             "immutable","rule_source","revision","execution_summary"}
clone = {k: v for k, v in existing.items() if k not in READ_ONLY}
# 3. Assign demo identity
clone["rule_id"]     = f"demo-{source_rule_id}"
clone["name"]        = f"[{SLUG}] {existing['name']}"
clone["description"] = f"[v{BOOTSTRAP_VERSION}] Clone of '{existing['name']}' — {purpose}"
clone["tags"]        = merge_tags(clone.get("tags", []))
clone["version"]     = 1
# 4. Idempotent create — check if clone already exists
try:
    kb("GET", f"/api/detection_engine/rules?rule_id={clone['rule_id']}", ok=(200,))
    print(f"  SIEM rule '{clone['rule_id']}': already exists — skipping")
except RuntimeError as e:
    if "404" not in str(e): raise
    kb("POST", "/api/detection_engine/rules", clone, ok=(200, 201))
    _manifest_add("siem_rules", {"rule_id": clone["rule_id"], "name": clone["name"]})
    print(f"  SIEM rule '{clone['rule_id']}': created")
```

Teardown deletes cloned rules by `rule_id`. Elastic-managed originals are left untouched.

**13h — Cases configuration** *(when Security or SIEM is in scope)*  
Configure the Cases feature for the engagement:

```python
# GET current config (will 404 on first run → POST to create)
try:
    existing = kb("GET", "/api/cases/configure", ok=(200,))
    version = existing[0]["version"] if existing else None
    if version:
        kb("PATCH", "/api/cases/configure", {"closure_type": "close-by-user", "version": version})
    else:
        kb("POST", "/api/cases/configure", {"closure_type": "close-by-user"})
except Exception:
    kb("POST", "/api/cases/configure", {"closure_type": "close-by-user", "owner": "securitySolution"})
print("  Cases: configured")
```

**13i — Probe-based feature detection** *(for optional capabilities — applies alongside D-033 version gate)*

For capabilities that may or may not be available (Agent Builder, Workflows, ELSER, Cases),
use HTTP status probe before attempting deployment — do not only rely on deployment type:

```python
def attempt_or_skip(label: str, fn):
    """Try fn(); skip gracefully if API returns 404/403/400 (feature not available)."""
    try:
        fn()
    except RuntimeError as e:
        code = str(e)[:3]
        if code in ("404", "403", "400"):
            print(f"  {label}: not available on this cluster — skipping (probe: {code})")
        else:
            raise
```

Usage:
```python
attempt_or_skip("Agent Builder",  lambda: create_agent_builder())
attempt_or_skip("Workflows",      lambda: deploy_workflows())
attempt_or_skip("Cases config",   lambda: configure_cases())
```

This complements the version gate (D-033) — it catches feature-flag or license-level
unavailability that version numbers don't predict. Reference: `hive-mind/patterns/deployment/SERVERLESS_FEATURE_DETECTION.md`.

**13j — Token Visibility dashboard** *(when Agent Builder is in scope and INCLUDE_TOKEN_VISIBILITY=true)*

Deploy the AI Cost + Usage dashboard from `skills/token-visibility/SKILL.md`.
Create the `{prefix}agent-sessions` index, load 30-60 seed session documents, and build
the ES|QL dashboard panels. This is a standard deliverable for any Agent Builder demo (D-036).

*13k — Other** *(streams, Fleet, synthetics, etc.)*  
If the platform audit and script call for them, add the matching API steps — do not assume
this list is exhaustive.

**13l — Engagement tagging (`demobuilder:<id>`)** *(D-026 — whenever a create payload has `tags`)*  
Merge **`demobuilder_tags()`** from **`references/demobuilder-tagging.md`** into SLOs, alerting
rules, ML job bodies, Agent Builder agents/tools, Security rules (if tagged), and any other
scoped creates. Indices remain distinguished by **`p(name)`**; saved objects should carry the tag
in export or follow-up tagging when the stack supports it.

**Example — saved objects import (multipart for NDJSON):**
```python
with open("kibana-objects/{slug}-dashboards.ndjson", "rb") as f:
    kb("POST", "/api/saved_objects/_import?overwrite=true", f, content_type="multipart/form-data")
```

If a required artifact type is missing for **this** engagement, **stop** and run the
corresponding skills to generate it; **do not** complete `demo-deploy` with “TODO: add in
Kibana” for an in-scope asset.

**Anomaly injection (step 14)**
Run the injection spec from `{slug}-ml-config.json`. Sleep `2 × bucket_span` after
injection before verifying anomaly scores:
```python
for entity in injection_plan["target_entities"]:
    # Generate skewed events for this entity during the anomaly period
    # Index via _bulk
    ...
print(f"  Anomaly injection complete. Waiting {2 * bucket_span_minutes}m for ML to process...")
time.sleep(bucket_span_seconds * 2)
```

**ELSER warm-up (step 15)**
```python
resp = es("POST", f"/{p(semantic_index)}/_search", {
    "query": {"semantic": {"field": "body_semantic", "query": "warmup"}}
})
latency = resp["took"]
print(f"  ELSER warm: {latency}ms {'✅' if latency < 2000 else '⚠️ slow — run again'}")
```

**Asset manifest — final write (step 16)**

After all steps complete successfully, do a final upsert of the manifest so it reflects
the full deployed state. This is the definitive copy teardown will read (D-031).

```python
def write_final_manifest():
    _manifest["deployed_at"] = __import__("datetime").datetime.utcnow().isoformat() + "Z"
    _manifest_push()
    print(f"  Manifest written: demobuilder-manifests/{_engagement_id_for_tag()}")
```

The manifest is also written incrementally after each step, so a partial deploy leaves a
valid manifest for partial teardown. The final write closes out the `deployed_at` timestamp.

## Step 4: Execute the Script

Source the `.env` and run:

```bash
set -a && source {engagement_dir}/.env && set +a
python3 {engagement_dir}/bootstrap.py
```

Stream output to the terminal so the SE can watch progress. Each step prints:
```
[Step 1/15] Connectivity check
  Connected: https://abc123.es.io:443
  Version:   9.3.1
  Prefix:    '' (none — using default index names)
  ✅ Done (0.4s)

[Step 2/15] ILM / Data Stream Lifecycle
  Serverless detected — using Data Stream Lifecycle
  ✅ Done (skipped)

[Step 3/15] Enrich policies
  store-location-enrich: created
  store-location-enrich: executing...
  store-location-enrich: complete (1.2s)
  ✅ Done (1.6s)
...
```

On completion:
```
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
 BOOTSTRAP COMPLETE — {Company}
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
 Cluster:   {ES_URL}
 Duration:  {total_seconds}s
 Indices:   {N} created
 Documents: {N} loaded
 ML jobs:   {N} running
 Kibana:    {N} objects imported

 To re-run a step:     python3 bootstrap.py --step N
 To skip data reload:  python3 bootstrap.py --skip-data
 To verify:            python3 bootstrap.py --dry-run

 ⚠️  Pre-demo: clear test sessions 10 min before going live:
 POST /{slug}-sessions/_delete_by_query
   {"query":{"range":{"@timestamp":{"lt":"now-10m"}}}}
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
```

## Step 5: Write the Deploy Log

`{slug}-deploy-log.md`:

```
# Deploy Log — {Company}
**Date:** {date} | **Duration:** {N}s | **Status:** ✅ Complete

## Environment
Cluster: {ES_URL}
Deployment type: {type}
Index prefix: {PREFIX or 'none'}

## What Was Created
| Resource | Name | Status |
|---|---|---|
[one row per created artifact]

## Seed Data
| Index | Documents loaded | Demo-critical docs verified |
|---|---|---|

## ML Jobs
| Job | Status | Last bucket | Anomaly score on target |
|---|---|---|---|

## Kibana Objects
| Type | Name | Status |
|---|---|---|

## To re-run (if something changed):
source {engagement_dir}/.env && python3 {engagement_dir}/bootstrap.py --skip-data
```

## Platform-Specific Adaptations

The script auto-adapts based on `DEPLOYMENT_TYPE` in the `.env`. **Baseline: 9.4+ (D-033).**
Clusters below 9.4 are rejected at step 1 unless `SKIP_VERSION_CHECK=true` is set.

| Behavior | Serverless | ECH 9.4+ |
|---|---|---|
| ILM | Skipped (DSL in template) | Created (hot-only by default, D-027) |
| ELSER inference | `service: "elser"` (managed, no model_id) | `service: "elastic"` via EIS (D-028) |
| ML node check | Skipped (auto-scaled) | Checked — anomaly detection only |
| Kibana Workflows | Supported | Supported 9.4+ (feature-flag check, D-011) |
| Agent Builder | Supported | Supported (feature-flag check, D-011) |
| Asset manifest | Written to cluster (D-031) | Written to cluster (D-031) |

## What Good Looks Like

**Clean first run:** All 15 steps succeed (step 13 covers every **in-scope** Kibana/Security/
Observability API the story needs — no manual UI follow-up for those). No existing resources
— everything created fresh. ELSER warms in < 2s when semantic search is in scope. ML anomaly
visible within 2 bucket spans when ML is in scope. Duration varies by scenario (simple search
demo vs. full hybrid Sec + Obs + agents).

**Idempotent re-run:** Bootstrap run again after a partial failure. Steps 1–7 skip
(resources exist), step 8 picks up (ELSER wasn't deployed). Remaining steps complete.
No duplicate data, no errors on existing resources.

**Multi-customer, same cluster:** Engagement A uses `INDEX_PREFIX=cb-`; engagement B uses
`INDEX_PREFIX=acme-`. Both sets of indices coexist. Each bootstrap only touches its own
prefixed resources.

**Prefix copy workflow:** (see `docs/engagements-path.md` — default root `~/engagements`)
```bash
ROOT="${DEMOBUILDER_ENGAGEMENTS_ROOT:-$HOME/engagements}"
cp "$ROOT/engagement-a/.env" "$ROOT/engagement-b/.env"
# Edit engagement-b/.env: DEMO_SLUG, ENGAGEMENT, INDEX_PREFIX for the new customer
set -a && source "$ROOT/engagement-b/.env" && set +a
python3 "$ROOT/engagement-b/bootstrap.py"
```
