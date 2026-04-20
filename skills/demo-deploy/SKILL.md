---
name: demo-deploy
description: >
  Generates a `bootstrap.py` deployment script from the demobuilder pipeline outputs and
  executes it against the target cluster specified in the engagement's .env file. Creates
  all Elasticsearch resources, loads seed data, imports Kibana objects, configures ML jobs,
  and injects demo anomalies. Idempotent — safe to re-run if a step fails. Per-engagement
  .env file isolates credentials so multiple customer demos never interfere.

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

**Before building Workflows or Agent Builder configs** — read these reference files first:
- `references/serverless-differences.md` — Serverless-specific behaviors, feature flags,
  ML field names, Liquid array syntax, ELSER service names, saved objects quirks
- `references/workflow-patterns.md` — working Workflow YAML patterns, `| first` syntax,
  template variables, step types, validation checklist, deploy API commands

Do not iterate past 30 minutes on an undocumented Workflow or Agent Builder API without
surfacing it as a blocker and asking for the `elastic/workflows` or
`elastic/kibana-agent-builder-sdk` reference repos.

## Step 1: Load the Environment

Read `{workspace}/engagements/{slug}/.env`. All subsequent API calls use these credentials. Never
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

## Step 2: Read Pipeline Outputs

Load all available artifacts from the workspace:
- `{slug}-data-model.json` — required. Defines all indices, templates, pipelines, build order.
- `{slug}-ml-config.json` — optional. ML jobs, datafeeds, injection plan.
- `{slug}-platform-audit.json` — read `deployment_type` and feature availability to
  adapt the bootstrap to the specific platform.

Extract the build order from the data model — this is the sequence the script must follow.

## Step 3: Generate `bootstrap.py`

Write a complete, executable Python script to `{workspace}/engagements/{slug}/bootstrap.py`.

The script structure:

```python
#!/usr/bin/env python3
"""
Demobuilder bootstrap — {Company} ({slug})
Generated: {date}
Deployment: {type} at {ELASTICSEARCH_URL}

Usage:
  python3 bootstrap.py              # deploy everything
  python3 bootstrap.py --dry-run   # print what would be done, no API calls
  python3 bootstrap.py --skip-data # skip data loading (re-run after a config change)
  python3 bootstrap.py --step N    # resume from step N (see step list below)

Steps:
  1.  Connectivity check
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
  13. Kibana saved objects import
  14. Anomaly injection
  15. Warm ELSER endpoint
"""

import os, sys, json, time, argparse
import urllib.request, urllib.error

# ── Credentials (from .env) ─────────────────────────────────────────────────
ES_URL    = os.environ.get("ELASTICSEARCH_URL", "").rstrip("/")
KB_URL    = os.environ.get("KIBANA_URL", "").rstrip("/")
API_KEY   = os.environ.get("ES_API_KEY", "")
KB_KEY    = os.environ.get("KIBANA_API_KEY", "")  # used for all Kibana API calls
DEP_TYPE  = os.environ.get("DEPLOYMENT_TYPE", "ech")
PREFIX    = os.environ.get("INDEX_PREFIX", "")

def p(name): return f"{PREFIX}{name}" if PREFIX else name  # apply index prefix

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
```
Fail fast and clearly if connectivity fails — don't attempt subsequent steps.

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

**ELSER endpoint (step 8)**

The ELSER deployment body differs between Serverless and ECH/self-managed:

```python
# Check if already deployed
try:
    es("GET", f"/_inference/sparse_embedding/{p('elser-v2-endpoint')}", ok=(200,))
    print("  ELSER endpoint: already deployed — skipping")
except:
    if DEP_TYPE == "serverless":
        # Serverless: managed endpoint — use "elser" service, no model_id
        body = {
            "service": "elser",
            "service_settings": {"num_allocations": 1, "num_threads": 1}
        }
    else:
        # ECH / self-managed: deploy model explicitly
        body = {
            "service": "elasticsearch",
            "service_settings": {
                "model_id": ".elser_model_2_linux-x86_64",
                "num_allocations": 1,
                "num_threads": 1
            }
        }
    es("PUT", f"/_inference/sparse_embedding/{p('elser-v2-endpoint')}", body)
    print("  ELSER endpoint: deploying... (allow 60-90s for model to load)")
    # Poll until allocated
    for _ in range(60):
        status = es("GET", f"/_inference/sparse_embedding/{p('elser-v2-endpoint')}")
        if status.get("service_settings", {}).get("num_allocations", 0) > 0:
            break
        time.sleep(3)
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

**Kibana saved objects (step 13)**
Use the Kibana Saved Objects import API. Objects are in `.ndjson` format. Apply prefix
to index pattern references if `INDEX_PREFIX` is set. Prefer **ES|QL** in chart/query definitions
(Lens or Vega-Lite with ES|QL) per current Elastic guidance; avoid KQL or Query DSL in new assets
unless ES|QL cannot express the filter for that stack version.
```python
with open("kibana-objects/{slug}-dashboards.ndjson", "rb") as f:
    kb("POST", "/api/saved_objects/_import?overwrite=true", f, content_type="multipart/form-data")
```

If the `.ndjson` files don't exist yet (first-time deploy for a new demo), use the
`kibana-dashboards` skill (from `elastic/agent-skills`) to generate dashboard definitions
from the data model, and the `kibana-agent-builder` skill to create agent configurations.
These skills write `.ndjson` output that bootstrap.py then imports.

For demos using Workflows that send email or webhooks, configure connectors first using
the `kibana-connectors` skill (from `elastic/agent-skills`) before importing Workflow
definitions — connectors must exist before Workflows that reference them.

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

## Step 4: Execute the Script

Source the `.env` and run:

```bash
set -a && source {workspace}/engagements/{slug}/.env && set +a
python3 {workspace}/engagements/{slug}/bootstrap.py
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
source {workspace}/engagements/{slug}/.env && python3 {workspace}/engagements/{slug}/bootstrap.py --skip-data
```

## Platform-Specific Adaptations

The script auto-adapts based on `DEPLOYMENT_TYPE` in the `.env`:

| Behavior | Serverless | ECH | Self-managed | Docker |
|---|---|---|---|---|
| ILM | Skipped (DSL in template) | Created | Created | Created |
| ELSER | Managed endpoint (no model_id) | Deployed model | Deployed model | Deployed model |
| ML node check | Skipped (auto-scaled) | Checked | Checked (warn if none) | Warn |
| Kibana Workflows | Supported | 9.3+ only | Not available | Not available |
| Agent Builder | Supported | Cloud only | Not available | Not available |

## What Good Looks Like

**Clean first run:** All 15 steps succeed. No existing resources — everything created
fresh. ELSER warms in < 2s. ML anomaly visible within 2 bucket spans of injection.
Bootstrap completes in 4–8 minutes for a typical demo data model.

**Idempotent re-run:** Bootstrap run again after a partial failure. Steps 1–7 skip
(resources exist), step 8 picks up (ELSER wasn't deployed). Remaining steps complete.
No duplicate data, no errors on existing resources.

**Multi-customer, same cluster:** Citizens Bank bootstrap used `INDEX_PREFIX=cb-`. IHG
bootstrap uses `INDEX_PREFIX=ihg-`. Both sets of indices coexist. Each bootstrap only
touches its own prefixed resources.

**Prefix copy workflow:**
```bash
cp ~/demobuilder-workspace/citizens-bank/.env ~/demobuilder-workspace/ihg-club/.env
# Edit ihg-club/.env: DEMO_SLUG=ihg-club, ENGAGEMENT="IHG Club Vacations", INDEX_PREFIX=ihg-
source ~/demobuilder-workspace/ihg-club/.env
python3 ~/demobuilder-workspace/ihg-club/bootstrap.py
```
