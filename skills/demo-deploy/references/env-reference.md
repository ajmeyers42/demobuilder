# Demo Environment Reference

## .env File — Required Fields

Every engagement workspace needs a `.env` file before `demo-deploy` can run.
All fields are read by `bootstrap.py` at runtime — never hardcoded.

```bash
# ── Identity ──────────────────────────────────────────────
DEMO_SLUG=citizens-bank          # slug used for file naming throughout pipeline
ENGAGEMENT=Citizens Bank         # human-readable company name
DEPLOYMENT_TYPE=serverless       # serverless | ech | self_managed | docker

# ── Cluster Credentials ───────────────────────────────────
ELASTICSEARCH_URL=https://abc123.es.io:443
KIBANA_URL=https://abc123.kb.io:443
ES_API_KEY=VuaCfGcBCdbkQm...     # base64-encoded API key or ApiKey header value

# ── Kibana API Key ────────────────────────────────────────────────────────
# Use KIBANA_API_KEY for all Kibana asset operations: Agent Builder, Workflows,
# Dashboards, Connectors, Saved Objects import. API key privilege requirements
# for Kibana vs Elasticsearch are under active product change — keeping separate
# keys is the safe default until product confirms a unified approach.
# Set this at provisioning time alongside ES_API_KEY.
KIBANA_API_KEY=

# ── Version (informational, set by demo-cloud-provision) ──
ELASTIC_VERSION=9.3.1

# ── Index Namespace ───────────────────────────────────────
# Leave blank for isolated clusters (default — recommended)
# Set a short prefix when sharing a cluster across multiple demos
# Example: INDEX_PREFIX=cb-  →  fraud-claims becomes cb-fraud-claims
INDEX_PREFIX=

# ── Cluster Metadata (informational) ──────────────────────
CLUSTER_NAME=demobuilder-citizens-bank-20260415
REGION=us-east-1
PROVISIONED_BY=demobuilder
```

## Multi-Customer Workflow

### New cluster per demo (recommended for isolation)

```bash
# Each demo gets its own cluster — no prefix needed
~/demobuilder-workspace/engagements/
├── {slug-A}/.env    → https://cluster-A.es.io  INDEX_PREFIX=
├── {slug-B}/.env    → https://cluster-B.es.io  INDEX_PREFIX=
└── {slug-C}/.env    → https://cluster-C.es.io  INDEX_PREFIX=
```

### Shared cluster (when you want to conserve cloud spend)

```bash
# One cluster, all demos on it — prefix separates namespaces
~/demobuilder-workspace/engagements/
├── {slug-A}/.env    → https://shared.es.io  INDEX_PREFIX=a-
├── {slug-B}/.env    → https://shared.es.io  INDEX_PREFIX=b-
└── {slug-C}/.env    → https://shared.es.io  INDEX_PREFIX=c-

# Copy workflow for a new demo on the same cluster:
cp ~/demobuilder-workspace/engagements/{slug-A}/.env ~/demobuilder-workspace/engagements/{slug-B}/.env
# Then edit {slug-B}/.env:
#   DEMO_SLUG={slug-B}
#   ENGAGEMENT={Company B}
#   INDEX_PREFIX=b-
```

### Prefix behavior

When `INDEX_PREFIX=cb-` is set, bootstrap.py applies it everywhere:
- Index names: `fraud-claims` → `cb-fraud-claims`
- Pipeline names: `fraud-ingest-pipeline` → `cb-fraud-ingest-pipeline`
- Template names: `fraud-template` → `cb-fraud-template`
- ML job IDs: `fraud-sla-monitor` → `cb-fraud-sla-monitor`
- Kibana index patterns: automatically updated in saved objects on import

The demo script ES|QL queries also need updating when a prefix is in use — bootstrap.py
patches the query strings in the Kibana saved objects before import.

## .env.example Template

This file is safe to commit — it documents requirements without exposing credentials.
Every workspace should have one alongside the `.env`:

```bash
# .env.example — copy to .env and fill in values
# See: skills/demo-deploy/references/env-reference.md

DEMO_SLUG=<slug-e.g.-citizens-bank>
ENGAGEMENT=<company-name>
DEPLOYMENT_TYPE=<serverless|ech|self_managed|docker>
ELASTIC_VERSION=<e.g.-9.3.1>

ELASTICSEARCH_URL=<https://your-cluster.es.io:443>
KIBANA_URL=<https://your-kibana.kb.io:443>
ES_API_KEY=<your-api-key>

INDEX_PREFIX=<optional-e.g.-cb->

CLUSTER_NAME=<demobuilder-slug-YYYYMMDD>
REGION=<e.g.-us-east-1>
PROVISIONED_BY=demobuilder
ENGAGEMENT=<company-name>
```

## API Key Permissions Required

The `ES_API_KEY` must have at minimum:

```json
{
  "cluster": ["monitor", "manage_ilm", "manage_ingest_pipelines", "manage_ml",
              "manage_enrich", "manage_index_templates"],
  "indices": [{ "names": ["*"], "privileges": ["all"] }],
  "applications": [{ "application": "kibana-.kibana", "privileges": ["all"],
                     "resources": ["*"] }]
}
```

For serverless: the project API key created at provisioning time has sufficient scope.
For ECH/self-managed: create a dedicated key with the above privileges — do not use
the `elastic` superuser key for bootstrap scripts.
