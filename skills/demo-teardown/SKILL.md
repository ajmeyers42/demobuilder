---
name: demo-teardown
description: >
  Generates and optionally executes a `teardown.py` script that deletes every Elastic
  resource created by `bootstrap.py` for a specific demo engagement — indices, data
  streams, ML jobs, ELSER endpoint, Kibana objects, ingest pipelines, index templates,
  component templates, ILM/DSL policies, and enrich policies. Uses INDEX_PREFIX to
  scope all deletes so only this engagement's resources are removed, even on a shared
  cluster. Writes a teardown log confirming what was deleted and when.

  ALWAYS use this skill when the user says "tear down the demo", "clean up the cluster",
  "delete the demo resources", "remove the [company] demo", "decommission [slug]",
  "post-demo cleanup", "we're done with the demo", "wipe the cluster", "delete everything
  for [company]", "the demo is over — clean up", "shut down the demo environment", or
  any mention of cleanup, decommission, or removing resources after a demo. Also trigger
  when the user asks what teardown would do, wants a dry run, or asks how to undo what
  bootstrap created.
---

# Demo Teardown

You are cleaning up a demo environment after the engagement is complete. You generate a
`teardown.py` script from the `.env` credentials and execute it. The script deletes only
what `bootstrap.py` created — scoped to this engagement's `INDEX_PREFIX` — and confirms
each resource is gone with a follow-up check.

Safety is the priority. The script never touches system indices, `.kibana`, or anything
not explicitly named in the data model's resource list. Shared clusters are protected by
the prefix requirement.

**Discovery:** Deployments should tag demo resources with **`demobuilder:<engagement_id>`**
(**`docs/decisions.md` D-026**) so operators can filter in Kibana; teardown remains driven by
**`INDEX_PREFIX`** and the data model inventory, not tag-only deletes.

## Step 1: Load the Environment

Read `{workspace}/.env`. Verify all required fields are present:
- `ELASTICSEARCH_URL` — non-empty, starts with `https://` or `http://`
- `KIBANA_URL` — non-empty
- `ES_API_KEY` — non-empty
- `DEPLOYMENT_TYPE` — one of: `serverless`, `ech`, `self_managed`, `docker`
- `INDEX_PREFIX` — read and note whether it is set or blank
- `DEMO_SLUG` — used for log file naming and confirmation prompts

If `.env` doesn't exist: stop. Tell the user the workspace doesn't appear to be a
demobuilder engagement, or that the cluster was provisioned outside of demobuilder.

**Shared cluster safety gate:**
If `INDEX_PREFIX` is blank AND `DEPLOYMENT_TYPE` is `ech` or `self_managed` (signals a
shared cluster), emit a prominent warning and require `--confirm` to proceed:

```
⚠️  WARNING: INDEX_PREFIX is not set on a shared cluster type.
    Teardown will delete resources by their bare names from the data model.
    If another demo uses the same names on this cluster, those resources will be deleted.

    Are you certain this cluster is dedicated to this engagement?
    Pass --confirm to proceed, or set INDEX_PREFIX in .env to scope deletes safely.
```

If `INDEX_PREFIX` is blank AND `DEPLOYMENT_TYPE` is `serverless`: the cluster is always
isolated per-project; proceed without warning.

## Step 2: Build the Resource Inventory

**Primary source — cluster-resident manifest (D-031):**

Try to read the asset manifest from the cluster before falling back to local files. The
manifest is written by `bootstrap.py` after each step and is the most accurate record of
what was actually created (including Kibana-assigned IDs that are unknown at generation
time). See `skills/demo-deploy/references/asset-manifest.md` for the full schema.

```python
MANIFEST_INDEX = "demobuilder-manifests"

def _load_manifest() -> dict | None:
    """Read the cluster-resident asset manifest. Returns None if not found."""
    eng_id = _engagement_id_for_tag()
    try:
        resp = es("GET", f"/{MANIFEST_INDEX}/_doc/{eng_id}", ok=(200,))
        manifest = resp.get("_source", {})
        if manifest:
            print(f"  Manifest found — deployed {manifest.get('deployed_at','?')}, "
                  f"bootstrap v{manifest.get('bootstrap_version','?')}")
        return manifest
    except RuntimeError as e:
        if "404" in str(e):
            print("  No manifest found — falling back to hardcoded inventory")
        else:
            print(f"  Manifest read error ({e}) — falling back to hardcoded inventory")
        return None
```

**Fallback — local pipeline outputs** (only if manifest is absent):

- `{slug}-data-model.json` — indices, templates, pipelines, enrich policies
- `{slug}-ml-config.json` — ML job IDs and datafeed IDs
- `{slug}-deploy-log.md` — cross-reference to exclude resources the deploy log marks
  as skipped or failed

**Hardcoded inventory backstop:**

When neither the manifest nor local files are available, `teardown.py` includes a
hardcoded `_hardcoded_inventory()` function populated at generation time. This is the
least reliable option (IDs may be stale from a re-deploy) — always prefer the manifest.

Extract the full resource inventory — one flat list of every named artifact by type:
`ml_job`, `ml_datafeed`, `kibana_dashboard`, `kibana_connector`, `kibana_agent`,
`kibana_workflow`, `elser_endpoint`, `data_stream`, `index`, `index_template`,
`component_template`, `ingest_pipeline`, `enrich_policy`, `ilm_policy`.

Apply the prefix function to every name: `p(name) = f"{PREFIX}{name}"` where PREFIX is
the value of INDEX_PREFIX from `.env` (may be empty string).

## Step 3: Check for Snapshot Option

Before generating the teardown script, check if `SNAPSHOT_REPO` is set in `.env`. If it
is, or if the user said "snapshot before teardown", offer to snapshot all demo indices
first:

```
SNAPSHOT_REPO is set: s3-demobuilder-backups

Would you like to snapshot all demo indices before teardown?
This preserves the data for post-demo analysis or re-deployment.

  Snapshot name: {slug}-pre-teardown-{date}
  Indices: cb-fraud-claims, cb-fraud-escalations, cb-store-transactions, ...

  yes → snapshot now, then teardown
  no  → skip snapshot, proceed to teardown
  only → snapshot only, do not tear down yet
```

If the user confirms a snapshot:

```python
# Register repository (if not already registered)
es("PUT", f"/_snapshot/{SNAPSHOT_REPO}", {
    "type": "s3",
    "settings": {"bucket": repo_bucket, "base_path": f"demobuilder/{slug}"}
})

# Create snapshot (wait=true for < ~20 indices; otherwise fire and poll)
es("PUT", f"/_snapshot/{SNAPSHOT_REPO}/{slug}-pre-teardown-{date}?wait_for_completion=true", {
    "indices": ",".join(demo_indices),
    "ignore_unavailable": True,
    "include_global_state": False
})
```

Note in the teardown log how to restore:
```bash
# To restore from snapshot:
source {workspace}/.env
curl -X POST "${ELASTICSEARCH_URL}/_snapshot/{SNAPSHOT_REPO}/{slug}-pre-teardown-{date}/_restore" \
  -H "Authorization: ApiKey ${ES_API_KEY}" \
  -H "Content-Type: application/json" \
  -d '{"indices": "cb-*", "rename_pattern": "cb-(.+)", "rename_replacement": "restored-cb-$1"}'
```

## Step 4: Generate `teardown.py`

Write a complete, executable Python script to `{workspace}/teardown.py`.

The script structure:

```python
#!/usr/bin/env python3
"""
Demobuilder teardown — {Company} ({slug})
Generated: {date}
Cluster: {ELASTICSEARCH_URL}
Index prefix: '{PREFIX}' ('{PREFIX}*' resources only)

Usage:
  python3 teardown.py               # delete everything, confirm before proceeding
  python3 teardown.py --dry-run     # print what would be deleted, no API calls
  python3 teardown.py --keep-data   # delete config/ML/Kibana but keep index data
  python3 teardown.py --yes         # skip confirmation prompt (for automation)
  python3 teardown.py --confirm     # required when INDEX_PREFIX is blank on a shared cluster

Steps:
  1.  Connectivity check
  2.  Stop ML datafeeds
  3.  Delete ML jobs
  4.  Delete Kibana saved objects
  5.  Delete ELSER inference endpoint
  6.  Delete data streams (and backing indices)
  7.  Delete static indices
  8.  Delete index templates
  9.  Delete component templates
  10. Delete ingest pipelines
  11. Delete enrich policies
  12. Delete ILM / DSL policies
  13. Confirm all resources removed
"""

import os, sys, json, time, argparse
import urllib.request, urllib.error

# ── Credentials (from .env) ─────────────────────────────────────────────────
ES_URL    = os.environ.get("ELASTICSEARCH_URL", "").rstrip("/")
KB_URL    = os.environ.get("KIBANA_URL", "").rstrip("/")
API_KEY   = os.environ.get("ES_API_KEY", "")
DEP_TYPE  = os.environ.get("DEPLOYMENT_TYPE", "ech")
PREFIX    = os.environ.get("INDEX_PREFIX", "")
SLUG      = os.environ.get("DEMO_SLUG", "demo")

def p(name): return f"{PREFIX}{name}" if PREFIX else name  # apply index prefix

# ── HTTP helpers ─────────────────────────────────────────────────────────────
def es(method, path, body=None, *, ok=(200, 201)):
    """Call Elasticsearch API. Returns parsed JSON. Raises on unexpected status."""
    ...

def kb(method, path, body=None, *, ok=(200,)):
    """Call Kibana API. Returns parsed JSON."""
    ...

def exists_es(path):
    """Return True if a HEAD request to path returns 200, False on 404."""
    ...

def step(n, label):
    """Print step header."""
    print(f"\n[Step {n}/13] {label}")

def deleted(name):
    """Print deletion confirmation line."""
    print(f"  {name}: deleted")

def skipped(name, reason="not found"):
    """Print skip line — resource didn't exist or was already gone."""
    print(f"  {name}: {reason} — skipping")

def check_gone(resource_type, path):
    """Verify via HEAD/GET that a resource no longer exists. Warn if still present."""
    ...

# ── Resource inventory (D-031: cluster-resident manifest is the trusted source) ──────────
# Populated from the manifest at runtime; hardcoded values are a last-resort backstop.

MANIFEST_INDEX = "demobuilder-manifests"

def _engagement_id_for_tag() -> str:
    override = os.environ.get("DEMO_ASSET_TAG", "").strip()
    raw = override or (PREFIX.strip() if PREFIX.strip() else SLUG)
    import re
    s = re.sub(r"[-_\s]+", "", raw).lower()
    return s or "demo"

def _load_manifest() -> dict | None:
    eng_id = _engagement_id_for_tag()
    try:
        resp = es("GET", f"/{MANIFEST_INDEX}/_doc/{eng_id}", ok=(200,))
        m = resp.get("_source", {})
        if m:
            print(f"  Manifest found \u2014 deployed {m.get('deployed_at','?')}, "
                  f"bootstrap v{m.get('bootstrap_version','?')}")
        return m
    except RuntimeError as e:
        print(f"  \u26a0  Manifest not found ({e}) \u2014 falling back to hardcoded inventory")
        return None

def _hardcoded_inventory() -> dict:
    """Backstop: IDs hardcoded at teardown.py generation time. May be stale after re-deploy."""
    return {
        "ilm_policies":        [p(x) for x in [...]],   # fill from {slug}-data-model.json
        "ingest_pipelines":    [p(x) for x in [...]],
        "component_templates": [p(x) for x in [...]],
        "index_templates":     [p(x) for x in [...]],
        "indices":             [p(x) for x in [...]],
        "data_streams":        [p(x) for x in [...]],
        "inference_endpoints": [],                        # e.g. [{"task_type":"sparse_embedding","id":p("elser")}]
        "ml_jobs":             [p(x) for x in [...]],
        "ml_datafeeds":        [p(f"datafeed-{x}") for x in [...]],
        "enrich_policies":     [p(x) for x in [...]],
        "kibana_space_id":     SLUG,
        "kibana_data_views":   [...],
        "slos":                [],                        # [{"id":"...","name":"..."}]
        "alerting_rules":      [],
        "dashboards":          [],
        "connectors":          [],
        "tags":                [],
        "workflows":           [],
        "agent_tools":         [],
        "agents":              [],
        "siem_rules":          [],
    }

def _build_inventory(manifest: dict | None) -> dict:
    if not manifest:
        return _hardcoded_inventory()
    a  = manifest.get("assets", {})
    kb = a.get("kibana", {})
    return {
        "ilm_policies":        a.get("ilm_policies", []),
        "ingest_pipelines":    a.get("ingest_pipelines", []),
        "component_templates": a.get("component_templates", []),
        "index_templates":     a.get("index_templates", []),
        "indices":             a.get("indices", []),
        "data_streams":        a.get("data_streams", []),
        "inference_endpoints": a.get("inference_endpoints", []),
        "ml_jobs":             a.get("ml_jobs", []),
        "ml_datafeeds":        a.get("ml_datafeeds", []),
        "enrich_policies":     a.get("enrich_policies", []),
        "kibana_space_id":     kb.get("space_id", ""),
        "kibana_data_views":   kb.get("data_views", []),
        "slos":                kb.get("slos", []),
        "alerting_rules":      kb.get("alerting_rules", []),
        "dashboards":          kb.get("dashboards", []),
        "connectors":          kb.get("connectors", []),
        "tags":                kb.get("tags", []),
        "workflows":           kb.get("workflows", []),
        "agent_tools":         kb.get("agent_tools", []),
        "agents":              kb.get("agents", []),
        "siem_rules":          kb.get("siem_rules", []),
    }

# Populated at runtime in step 1 after manifest read:
INVENTORY: dict = {}   # set via INVENTORY = _build_inventory(_load_manifest())

# ── Step implementations ──────────────────────────────────────────────────────

def stop_datafeeds():   ...
def delete_ml_jobs():   ...
def delete_kibana_objects(): ...
def delete_elser():     ...
def delete_data_streams(): ...
def delete_static_indices(): ...
def delete_index_templates(): ...
def delete_component_templates(): ...
def delete_ingest_pipelines(): ...
def delete_enrich_policies(): ...
def delete_lifecycle_policies(): ...
def verify_clean():     ...

# ── Main ──────────────────────────────────────────────────────────────────────
if __name__ == "__main__":
    ...
```

### Critical implementation details for each step

**Connectivity check (step 1)**
```python
resp = es("GET", "/")
print(f"  Connected: {ES_URL}")
print(f"  Prefix:    '{PREFIX}' ({'scoped to ' + PREFIX + '*' if PREFIX else 'no prefix — deleting by bare name'})")
print(f"\n  Resources to be deleted:")
print(f"    ML jobs:             {len(ML_JOBS)}")
print(f"    Kibana objects:      {len(KIBANA_DASHBOARDS) + len(KIBANA_CONNECTORS) + len(KIBANA_AGENTS) + len(KIBANA_WORKFLOWS)}")
print(f"    Data streams:        {len(DATA_STREAMS)}")
print(f"    Static indices:      {len(STATIC_INDICES)}")
print(f"    Index templates:     {len(INDEX_TEMPLATES)}")
print(f"    Component templates: {len(COMPONENT_TEMPLATES)}")
print(f"    Ingest pipelines:    {len(INGEST_PIPELINES)}")
print(f"    Enrich policies:     {len(ENRICH_POLICIES)}")
print(f"    ILM policies:        {len(ILM_POLICIES)}")
```

Fail fast and clearly if connectivity fails — do not proceed to deletion steps.

**Confirmation prompt (pre-step)**
Unless `--yes` was passed:
```python
if not args.yes:
    print(f"\n  Teardown will permanently delete all {PREFIX}* resources listed above.")
    print(f"  This cannot be undone. Type 'yes' to continue: ", end="")
    answer = input().strip().lower()
    if answer != "yes":
        print("  Aborted.")
        sys.exit(0)
```

**Stop ML datafeeds → delete ML jobs (steps 2–3)**

Stop datafeeds first. A job cannot be deleted while its datafeed is running.
```python
for df in ML_DATAFEEDS:
    try:
        es("POST", f"/_ml/datafeeds/{df}/_stop?force=true")
        print(f"  {df}: datafeed stopped")
    except urllib.error.HTTPError as e:
        if e.code == 404:
            skipped(df, "datafeed not found")
        else:
            raise

for job in ML_JOBS:
    try:
        es("DELETE", f"/_ml/anomaly_detectors/{job}?force=true")
        deleted(job)
        # Verify gone
        if exists_es(f"/_ml/anomaly_detectors/{job}"):
            print(f"  ⚠️  {job}: still present after delete — manual check required")
    except urllib.error.HTTPError as e:
        if e.code == 404:
            skipped(job)
        else:
            raise
```

Do NOT use wildcard ML job deletes — only delete the specific job IDs from the data model.

**Delete Kibana saved objects (step 4)**

Each saved object type is deleted individually by ID using the Kibana Saved Objects API.
```python
# Delete in dependency order: workflows → agents → dashboards → connectors
# (agents may reference connectors; dashboards may reference agents)
for wf_id in KIBANA_WORKFLOWS:
    try:
        kb("DELETE", f"/api/saved_objects/workflow/{wf_id}")
        deleted(f"workflow/{wf_id}")
    except:
        skipped(f"workflow/{wf_id}")

for agent_id in KIBANA_AGENTS:
    try:
        kb("DELETE", f"/api/saved_objects/agent/{agent_id}")
        deleted(f"agent/{agent_id}")
    except:
        skipped(f"agent/{agent_id}")

for dash_id in KIBANA_DASHBOARDS:
    try:
        kb("DELETE", f"/api/saved_objects/dashboard/{dash_id}")
        deleted(f"dashboard/{dash_id}")
    except:
        skipped(f"dashboard/{dash_id}")

for conn_id in KIBANA_CONNECTORS:
    try:
        kb("DELETE", f"/api/actions/connector/{conn_id}")
        deleted(f"connector/{conn_id}")
    except:
        skipped(f"connector/{conn_id}")
```

Never use a wildcard or bulk Kibana delete — only the saved object IDs imported by bootstrap.

**Delete ELSER inference endpoint (step 5)**

Only delete inference endpoints listed in the manifest `inference_endpoints` array.
Use `GET` (not `HEAD`) to check existence — the inference API returns 404 on HEAD even
when the endpoint exists (a known 9.x behaviour):

```python
def _inference_exists(task_type: str, endpoint_id: str) -> bool:
    """Use GET, not HEAD — inference API returns 404 on HEAD even when endpoint exists."""
    try:
        resp = es("GET", f"/_inference/{task_type}/{endpoint_id}", ok=(200,))
        endpoints = resp.get("endpoints") or []
        return bool(endpoints or resp.get("service"))
    except RuntimeError:
        return False

for ep in INVENTORY.get("inference_endpoints", []):
    task_type   = ep["task_type"]
    endpoint_id = ep["id"]
    if _inference_exists(task_type, endpoint_id):
        es("DELETE", f"/_inference/{task_type}/{endpoint_id}", ok=(200, 204))
        deleted(endpoint_id)
        time.sleep(2)
        if _inference_exists(task_type, endpoint_id):
            print(f"  ⚠️  {endpoint_id}: still present — may be in use by another index")
    else:
        skipped(endpoint_id)
```

If `--keep-data` was passed, skip step 5 (ELSER endpoint may still be needed for queries
against kept indices). Skip data streams and static indices as well (steps 6–7).

**Delete data streams (step 6)**

Data stream deletion also removes all backing indices.
```python
if args.keep_data:
    print("  --keep-data: skipping data stream deletion")
else:
    for ds in DATA_STREAMS:
        try:
            es("DELETE", f"/_data_stream/{ds}")
            deleted(ds)
        except urllib.error.HTTPError as e:
            if e.code == 404:
                skipped(ds)
            else:
                raise
```

**Delete static indices (step 7)**

```python
if args.keep_data:
    print("  --keep-data: skipping static index deletion")
else:
    for idx in STATIC_INDICES:
        try:
            es("DELETE", f"/{idx}")
            deleted(idx)
        except urllib.error.HTTPError as e:
            if e.code == 404:
                skipped(idx)
            else:
                raise
```

Never use `DELETE /{PREFIX}*` — always delete specific index names from the resource list.
A wildcard delete is a footgun on any cluster with other users or data.

**Delete index templates (step 8)**

```python
for tmpl in INDEX_TEMPLATES:
    try:
        es("DELETE", f"/_index_template/{tmpl}")
        deleted(tmpl)
    except urllib.error.HTTPError as e:
        if e.code == 404:
            skipped(tmpl)
        else:
            raise
```

**Delete component templates (step 9)**

```python
for comp in COMPONENT_TEMPLATES:
    try:
        es("DELETE", f"/_component_template/{comp}")
        deleted(comp)
    except urllib.error.HTTPError as e:
        if e.code == 404:
            skipped(comp)
        else:
            raise
```

**Delete ingest pipelines (step 10)**

```python
for pipe in INGEST_PIPELINES:
    try:
        es("DELETE", f"/_ingest/pipeline/{pipe}")
        deleted(pipe)
    except urllib.error.HTTPError as e:
        if e.code == 404:
            skipped(pipe)
        else:
            raise
```

**Delete enrich policies (step 11)**

Enrich policies cannot be deleted while an enrich processor in an active pipeline
references them. Delete pipelines (step 10) before enrich policies.
```python
for ep in ENRICH_POLICIES:
    try:
        es("DELETE", f"/_enrich/policy/{ep}")
        deleted(ep)
    except urllib.error.HTTPError as e:
        if e.code == 404:
            skipped(ep)
        elif e.code == 409:
            print(f"  ⚠️  {ep}: in use by a pipeline — delete pipeline first")
        else:
            raise
```

**Delete ILM / DSL policies (step 12)**

ILM is only applicable on non-serverless deployments. DSL is embedded in index templates
and is removed when the template is deleted (step 8), so no separate DSL API call needed.
```python
if DEP_TYPE == "serverless":
    print("  Serverless — ILM not applicable (DSL removed with templates in step 8)")
else:
    for pol in ILM_POLICIES:
        try:
            es("DELETE", f"/_ilm/policy/{pol}")
            deleted(pol)
        except urllib.error.HTTPError as e:
            if e.code == 404:
                skipped(pol)
            elif e.code == 409:
                # Policy is still attached to a managed index
                print(f"  ⚠️  {pol}: still attached to an index — indices may not have been deleted")
            else:
                raise
```

**Verify all resources removed (step 13)**

Run existence checks for every resource that was targeted for deletion. Report anything
still present.
```python
remaining = []

for job in ML_JOBS:
    if exists_es(f"/_ml/anomaly_detectors/{job}"):
        remaining.append(f"ML job: {job}")

for ds in DATA_STREAMS:
    if not args.keep_data and exists_es(f"/_data_stream/{ds}"):
        remaining.append(f"Data stream: {ds}")

for idx in STATIC_INDICES:
    if not args.keep_data and exists_es(f"/{idx}"):
        remaining.append(f"Index: {idx}")

for tmpl in INDEX_TEMPLATES:
    if exists_es(f"/_index_template/{tmpl}"):
        remaining.append(f"Index template: {tmpl}")

# ... repeat for all resource types ...

if remaining:
    print("\n  ⚠️  The following resources could not be confirmed as deleted:")
    for r in remaining:
        print(f"    {r}")
    print("\n  Manual cleanup may be required. Check cluster logs for errors.")
else:
    print(f"\n  All {PREFIX}* resources confirmed removed.")
```

### Dry-run mode

When `--dry-run` is passed, no API calls are made. The script prints the full resource
list that would be deleted, organized by type:

```
DRY RUN — no changes will be made

Resources that would be deleted:
  ML jobs (2):
    cb-fraud-sla-monitor
    cb-claims-velocity

  ML datafeeds (2):
    datafeed-cb-fraud-sla-monitor
    datafeed-cb-claims-velocity

  Kibana objects (3):
    dashboard/cb-fraud-operations-dashboard
    agent/cb-fraud-agent
    connector/cb-fraud-email-connector

  ELSER endpoint (1):
    cb-elser-v2-endpoint

  Data streams (2):
    cb-fraud-claims
    cb-store-transactions

  [... all resource types ...]

To execute: python3 teardown.py
To keep index data: python3 teardown.py --keep-data
```

### Keep-data mode

When `--keep-data` is passed, steps 5 (ELSER endpoint), 6 (data streams), and 7
(static indices) are skipped. Everything else is deleted. This is useful when:
- The customer wants to query their own data after the demo
- The SA needs to run post-demo analysis on the event data
- A follow-up demo is planned and seeding the data again would be slow

Note in the teardown log that data was retained and which indices still exist.

## Step 5: Execute the Script

Source the `.env` and run with the appropriate flags:

```bash
set -a && source {workspace}/.env && set +a
python3 {workspace}/teardown.py
```

For dry-run first:
```bash
python3 {workspace}/teardown.py --dry-run
```

Stream output to the terminal so the SE can watch progress. Each step prints:

```
[Step 1/13] Connectivity check
  Connected: https://abc123.es.io:443
  Prefix:    'cb-' (scoped to cb-*)

  Resources to be deleted:
    ML jobs:             2
    Kibana objects:      3
    Data streams:        2
    Static indices:      1
    Index templates:     3
    Component templates: 2
    Ingest pipelines:    2
    Enrich policies:     1
    ILM policies:        1

  Teardown will permanently delete all cb-* resources listed above.
  This cannot be undone. Type 'yes' to continue: yes

[Step 2/13] Stop ML datafeeds
  datafeed-cb-fraud-sla-monitor: datafeed stopped
  datafeed-cb-claims-velocity: datafeed stopped
  ✅ Done (1.2s)

[Step 3/13] Delete ML jobs
  cb-fraud-sla-monitor: deleted
  cb-claims-velocity: deleted
  ✅ Done (0.8s)

[Step 4/13] Delete Kibana objects
  workflow/cb-fraud-triage-workflow: not found — skipping
  agent/cb-fraud-agent: deleted
  dashboard/cb-fraud-operations-dashboard: deleted
  connector/cb-fraud-email-connector: deleted
  ✅ Done (1.1s)

[Step 5/13] Delete ELSER endpoint
  cb-elser-v2-endpoint: deleted
  ✅ Done (2.3s)

[Step 6/13] Delete data streams
  cb-fraud-claims: deleted
  cb-store-transactions: deleted
  ✅ Done (0.6s)

[Step 7/13] Delete static indices
  cb-fraud-escalations: deleted
  ✅ Done (0.3s)

[Step 8/13] Delete index templates
  cb-fraud-claims-template: deleted
  cb-store-transactions-template: deleted
  cb-fraud-escalations-template: not found — skipping
  ✅ Done (0.5s)

[Step 9/13] Delete component templates
  cb-mappings-base: deleted
  cb-settings-base: deleted
  ✅ Done (0.4s)

[Step 10/13] Delete ingest pipelines
  cb-fraud-enrich-pipeline: deleted
  cb-store-transactions-pipeline: deleted
  ✅ Done (0.4s)

[Step 11/13] Delete enrich policies
  cb-store-location-enrich: deleted
  ✅ Done (0.3s)

[Step 12/13] Delete ILM policies
  cb-fraud-claims-ilm: deleted
  ✅ Done (0.3s)

[Step 13/13] Verify all resources removed
  All cb-* resources confirmed removed.
  ✅ Done (1.8s)

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
TEARDOWN COMPLETE — Citizens Bank
  Duration: 28s
  Resources removed: 14
  Resources skipped: 2 (not found)
  Cluster: https://abc123.es.io
  All cb-* resources deleted. Cluster is clean.
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
```

## Step 6: Optional — Delete the Cluster

If the cluster was provisioned specifically for this demo (`DEPLOYMENT_TYPE=serverless`
or the `.env` `PROVISIONED_BY=demobuilder` flag is set), offer to delete the entire
cluster or project after teardown completes:

```
Teardown complete. The cluster at https://abc123.es.io is now empty.

This cluster was provisioned by demobuilder for this engagement.
Would you like to delete the entire project?

  yes → delete the Elastic Cloud project (uses cloud-manage-project skill)
  no  → leave the empty cluster running (you will continue to be billed)
```

If the user says yes, invoke the `cloud-manage-project` skill to delete the project.
Do NOT delete the cluster automatically — always prompt first. A running empty cluster
is harmless; an accidentally deleted cluster may have other users or be needed for follow-up.

## Step 7: Write the Teardown Log

Write `{workspace}/{slug}-teardown-log.md`:

```
# Teardown Log — {Company}
**Date:** {date} | **Duration:** {N}s | **Status:** ✅ Complete

## Environment
Cluster: {ES_URL}
Deployment type: {type}
Index prefix: {PREFIX or 'none'}
Mode: {full teardown | --keep-data | --dry-run}

## What Was Deleted
| Resource type | Name | Status |
|---|---|---|
| ML job | cb-fraud-sla-monitor | deleted |
| ML job | cb-claims-velocity | deleted |
| Kibana dashboard | cb-fraud-operations-dashboard | deleted |
| Kibana agent | cb-fraud-agent | deleted |
| Kibana connector | cb-fraud-email-connector | deleted |
| Kibana workflow | cb-fraud-triage-workflow | not found (skipped) |
| ELSER endpoint | cb-elser-v2-endpoint | deleted |
| Data stream | cb-fraud-claims | deleted |
| Data stream | cb-store-transactions | deleted |
| Index | cb-fraud-escalations | deleted |
| Index template | cb-fraud-claims-template | deleted |
| Component template | cb-mappings-base | deleted |
| Ingest pipeline | cb-fraud-enrich-pipeline | deleted |
| Enrich policy | cb-store-location-enrich | deleted |
| ILM policy | cb-fraud-claims-ilm | deleted |

## Verification
All targeted cb-* resources confirmed absent from the cluster.

## Data Retention
{If --keep-data: "Index data retained per --keep-data flag: cb-fraud-claims, cb-store-transactions, cb-fraud-escalations"}
{If snapshot taken: "Pre-teardown snapshot: {slug}-pre-teardown-{date} in {SNAPSHOT_REPO}. To restore: [command]"}

## Cluster Status
{If cluster deleted: "Elastic Cloud project deleted. Cluster no longer exists."}
{If cluster retained: "Cluster at {ES_URL} is retained (now empty of {PREFIX}* resources)."}
```

## Safety Constraints (Non-Negotiable)

1. **Never use wildcard index deletes.** `DELETE /cb-*` would destroy the cluster if
   another pattern matched. Always delete specific names from the resource list.

2. **Never delete system indices.** `.kibana`, `.ml-*`, `.security-*`, `.fleet-*`, and
   any index starting with `.` that is not explicitly in the data model is off-limits.

3. **Prefix is mandatory on shared clusters.** If `INDEX_PREFIX` is blank and
   `DEPLOYMENT_TYPE` is not `serverless`, require explicit `--confirm` before proceeding.
   Print a clear warning explaining the risk.

4. **Always verify after delete.** Every deletion step is followed by an existence check.
   If a resource is still present after a delete call, warn — do not silently move on.

5. **Graceful on 404.** If a resource doesn't exist (was never created, was already
   deleted, or the deploy failed before creating it), skip it quietly. A 404 is not an
   error during teardown — it means the resource is already gone.

6. **Dry-run is always safe.** `--dry-run` must never make API calls. It is safe to run
   at any time as a preview of what teardown would do.

7. **Keep-data respects the user's post-demo needs.** `--keep-data` is not just a
   convenience flag — the SA may have promised the customer access to the cluster for
   a few days after the demo. Honor it.

## What Good Looks Like

**Clean teardown, prefixed cluster:**
Citizens Bank demo on a shared ECH cluster with `INDEX_PREFIX=cb-`. Teardown runs, all
14 `cb-*` resources deleted, 0 IHG or Thermo Fisher resources touched. Verification
confirms `cb-*` cluster-wide returns no matches. Log written. SE confirms cluster still
has other demo resources.

**Teardown with keep-data:**
Post-demo, the SA wants to show the customer their own transaction data in Discover for
a week. `teardown.py --keep-data` runs: ML jobs stopped and deleted, Kibana objects
deleted, templates and pipelines deleted, but `cb-fraud-claims` and `cb-store-transactions`
data streams are left intact with their data. Teardown log documents the retained indices.

**Dry-run before commitment:**
SE is unsure what teardown will touch on a shared cluster. Runs `teardown.py --dry-run`.
Gets a full list: 2 ML jobs, 3 Kibana objects, 2 data streams, 5 templates. No API calls
made. Confirms the list looks right, then runs without `--dry-run` to execute.

**Teardown + cluster delete:**
IHG serverless project was spun up just for the IHG demo (`PROVISIONED_BY=demobuilder`).
Teardown completes, all resources removed. Skill prompts: "Delete the serverless project?"
SA says yes. `cloud-manage-project` deletes the project. Billing stops. Log records both
the resource teardown and the project deletion.

**Resource not found (graceful):**
Bootstrap partially failed — the ELSER endpoint was never created because the deploy
timed out at step 8. Teardown step 5 checks, finds 404, prints "cb-elser-v2-endpoint:
not found — skipping". Continues without error. Teardown log records it as skipped.

**Shared cluster without prefix — safety gate fires:**
SE accidentally runs teardown with `INDEX_PREFIX` blank on an ECH cluster shared with
two other customer demos. The safety gate fires before any deletion occurs. SE is warned,
sets `INDEX_PREFIX=cb-` in the `.env`, reruns. Only `cb-*` resources are deleted.
