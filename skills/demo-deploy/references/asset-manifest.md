# Asset Manifest Reference

Per **`docs/decisions.md` D-031** — `bootstrap.py` writes a manifest document to the
target cluster so `teardown.py` has a fresh, authoritative inventory of every created
resource. The manifest is **cluster-resident** and survives local file loss or re-generation.

---

## Manifest index

| Field | Value |
|---|---|
| Index name | `demobuilder-manifests` |
| Document ID | normalized engagement ID (same as `demobuilder:<id>` tag value) |
| Prefix applied? | No — shared registry index; never prefixed, never deleted by teardown |

---

## Document schema

```json
{
  "engagement_id":      "cbfraud",
  "slug":               "2026citizens-ai",
  "bootstrap_version":  "1.0.0",
  "deployed_at":        "2026-04-22T14:30:00Z",
  "es_version":         "9.4.0",
  "es_url":             "https://demo-447f06.es.us-west2.gcp.elastic-cloud.com",
  "assets": {
    "ilm_policies":        ["cb-fraud-claims-ilm"],
    "ingest_pipelines":    ["cb-fraud-enrich"],
    "component_templates": ["cb-fraud-claims-mappings", "cb-fraud-claims-settings"],
    "index_templates":     ["cb-fraud-claims-template"],
    "indices":             ["cb-fraud-claims", "cb-fraud-escalations"],
    "data_streams":        [],
    "inference_endpoints": [
      {"task_type": "sparse_embedding", "id": "cb-elser"}
    ],
    "ml_jobs":             ["cb-fraud-volume-anomaly"],
    "ml_datafeeds":        ["datafeed-cb-fraud-volume-anomaly"],
    "enrich_policies":     [],
    "fleet_integrations": {
      "packages": [
        {"name": "kubernetes", "version": "1.62.0"}
      ],
      "agent_policy_ids": ["abc123-policy-id"]
    },
    "kibana": {
      "space_id":      "2026citizens-ai",
      "data_views":    ["cb-fraud-claims-*"],
      "slos": [
        {"id": "a1b2c3d4-...", "name": "cb-slo-debit-ack-rate"}
      ],
      "alerting_rules": [
        {"id": "e5f6a7b8-...", "name": "Citizens — Debit SLO Burn Rate"}
      ],
      "dashboards": [
        {"id": "cb-fraud-ops-dashboard", "title": "Citizens Fraud Operations"}
      ],
      "connectors": [
        {"id": "f9g0h1i2-...", "name": "cb-fraud-index-connector"}
      ],
      "tags": [
        {"id": "j3k4l5m6-...", "name": "demobuilder:cbfraud"}
      ],
      "workflows": [
        {"id": "n7o8p9q0-...", "name": "citizens-open-fraud-case"}
      ],
      "agent_tools": [
        {"id": "citizens-claims-search",   "name": "citizens-claims-search"},
        {"id": "citizens-esql-debit-at-risk", "name": "citizens-esql-debit-at-risk"}
      ],
      "agents": [
        {"id": "citizens-fraud-assistant-poc", "name": "Fraud Assistant"}
      ],
      "siem_rules": [
        {"rule_id": "demo-citizens-wire-transfer-ml-anomaly", "name": "Citizens POC — Wire Transfer Volume Anomaly"}
      ]
    }
  }
}
```

---

## Bootstrap — writing the manifest

### Helper functions in `bootstrap.py`

```python
MANIFEST_INDEX = "demobuilder-manifests"
_manifest: dict = {}   # in-memory accumulator — written to cluster after each step

def _manifest_init():
    """Seed the in-memory manifest with identity fields."""
    global _manifest
    _manifest = {
        "engagement_id":     _engagement_id_for_tag(),
        "slug":               SLUG,
        "bootstrap_version":  BOOTSTRAP_VERSION,
        "deployed_at":        __import__("datetime").datetime.utcnow().isoformat() + "Z",
        "es_version":         "",    # filled in step 1
        "es_url":             ES_URL,
        "assets": {
            "ilm_policies": [], "ingest_pipelines": [], "component_templates": [],
            "index_templates": [], "indices": [], "data_streams": [],
            "inference_endpoints": [], "ml_jobs": [], "ml_datafeeds": [],
            "enrich_policies": [],
            "fleet_integrations": {"packages": [], "agent_policy_ids": []},
            "kibana": {
                "space_id": "", "data_views": [], "slos": [],
                "alerting_rules": [], "dashboards": [], "connectors": [],
                "tags": [], "workflows": [], "agent_tools": [],
                "agents": [], "siem_rules": []
            }
        }
    }

def _manifest_push():
    """Upsert the current in-memory manifest to the cluster."""
    eng_id = _engagement_id_for_tag()
    try:
        es("POST", f"/{MANIFEST_INDEX}/_doc/{eng_id}",
           _manifest, ok=(200, 201))
    except Exception as exc:
        print(f"  ⚠  manifest write failed (non-fatal): {exc}")

def _manifest_add(section: str, value):
    """Append value to an assets list and push to cluster."""
    target = _manifest["assets"]
    if section in target:
        lst = target[section]
        if value not in lst:
            lst.append(value)
    elif section in target.get("kibana", {}):
        lst = target["kibana"][section]
        if value not in lst:
            lst.append(value)
    _manifest_push()
```

### Usage pattern — call `_manifest_add` after each successful resource creation

```python
# After step 2 ILM:
_manifest_add("ilm_policies", p("fraud-claims-ilm"))

# After step 8 ELSER:
_manifest_add("inference_endpoints", {"task_type": "sparse_embedding", "id": p("elser")})

# After Kibana workflow creation:
_manifest_add("workflows", {"id": wf_id, "name": workflow_name})

# After Agent Builder tool creation:
_manifest_add("agent_tools", {"id": tool_id, "name": tool_name})
```

### Ensure `demobuilder-manifests` index exists (idempotent — call in step 1)

```python
def _ensure_manifest_index():
    try:
        es("HEAD", f"/{MANIFEST_INDEX}", ok=(200,))
    except RuntimeError:
        es("PUT", f"/{MANIFEST_INDEX}", {
            "settings": {"number_of_shards": 1, "number_of_replicas": 1},
            "mappings": {
                "dynamic": "true",
                "_meta": {"description": "Demobuilder engagement asset manifests. Not a demo index — do not delete."}
            }
        }, ok=(200,))
```

---

## Teardown — reading the manifest

```python
MANIFEST_INDEX = "demobuilder-manifests"

def _load_manifest() -> dict | None:
    """Try to read the asset manifest from the cluster. Returns None if not found."""
    eng_id = _engagement_id_for_tag()
    try:
        resp = es("GET", f"/{MANIFEST_INDEX}/_doc/{eng_id}", ok=(200,))
        manifest = resp.get("_source", {})
        if manifest:
            print(f"  Manifest found (deployed {manifest.get('deployed_at','?')}, "
                  f"bootstrap v{manifest.get('bootstrap_version','?')})")
        return manifest
    except RuntimeError as e:
        if "404" in str(e):
            print("  ⚠  No manifest found — falling back to hardcoded inventory")
        else:
            print(f"  ⚠  Manifest read error ({e}) — falling back to hardcoded inventory")
        return None

def _build_inventory(manifest: dict | None):
    """Build the teardown resource lists from the manifest, or fall back to hardcoded."""
    if not manifest:
        return _hardcoded_inventory()  # defined at module level as a backstop
    assets = manifest.get("assets", {})
    kb_assets = assets.get("kibana", {})
    return {
        "ilm_policies":        assets.get("ilm_policies", []),
        "ingest_pipelines":    assets.get("ingest_pipelines", []),
        "component_templates": assets.get("component_templates", []),
        "index_templates":     assets.get("index_templates", []),
        "indices":             assets.get("indices", []),
        "data_streams":        assets.get("data_streams", []),
        "inference_endpoints": assets.get("inference_endpoints", []),
        "ml_jobs":             assets.get("ml_jobs", []),
        "ml_datafeeds":        assets.get("ml_datafeeds", []),
        "enrich_policies":     assets.get("enrich_policies", []),
        "fleet_integrations":  assets.get("fleet_integrations", {"packages": [], "agent_policy_ids": []}),
        "kibana_space_id":     kb_assets.get("space_id", ""),
        "kibana_data_views":   kb_assets.get("data_views", []),
        "slos":                kb_assets.get("slos", []),             # list of {"id":..,"name":..}
        "alerting_rules":      kb_assets.get("alerting_rules", []),   # list of {"id":..,"name":..}
        "dashboards":          kb_assets.get("dashboards", []),
        "connectors":          kb_assets.get("connectors", []),
        "tags":                kb_assets.get("tags", []),
        "workflows":           kb_assets.get("workflows", []),
        "agent_tools":         kb_assets.get("agent_tools", []),
        "agents":              kb_assets.get("agents", []),
        "siem_rules":          kb_assets.get("siem_rules", []),
    }
```

---

## What the manifest does NOT replace

- **`INDEX_PREFIX`** safety gate on shared clusters — still required in teardown
- **`DEMO_SLUG`** and `.env` credential fields — still required for connectivity
- **`demobuilder:<id>` tag** — still applied to all tagged assets; the manifest complements
  it by providing IDs, not tag-only discovery

---

## Notes

- The `demobuilder-manifests` index should be excluded from ILM and snapshot policies.
- Multiple engagements on the same cluster each have their own document (document ID =
  engagement_id). The index accumulates a record of every engagement deployed there.
- If bootstrap is re-run (idempotent re-deploy), the manifest is overwritten with the
  latest state. Old IDs from a previous partial run are replaced cleanly.
