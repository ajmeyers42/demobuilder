---
name: demo-cloud-provision
description: >
  Creates a new Elastic Cloud deployment (ECH or Serverless) or generates a Docker Compose
  configuration for local demo environments. Captures the cluster credentials and writes
  them to a per-engagement .env file in the workspace. Designed for disposable demo
  clusters — each engagement gets its own isolated environment and credential file.

  ALWAYS use this skill when the user says "create a new cluster for this demo", "spin up
  a serverless project", "provision an ECH deployment", "set up the demo environment",
  or "I need a cluster for the [company] demo". Also trigger when demo-deploy is about to
  run but no .env exists in the workspace. Run before demo-deploy — this is the
  provisioning step; demo-deploy is the deployment step.
---

# Demo Cloud Provision

You are provisioning a fresh Elastic cluster for a demo environment. The cluster is
disposable — it exists to run this demo, not to host production data. Credentials are
written to a per-engagement `.env` file so multiple customer demos can coexist using
different clusters (or the same cluster with isolated index namespaces).

## Step 1: Determine Deployment Target

Ask the user (or infer from context) which deployment type they want:

| Type | When to use | Feature notes |
|---|---|---|
| **Serverless — Elasticsearch** | Agentic demos, Agent Builder + Workflows in scope | Agent Builder ✅, Workflows ✅, ELSER managed ✅, ILM → DSL ⚠️, ML auto-scaled ✅ |
| **ECH (Elastic Cloud Hosted)** | Full control, specific version required, ML UI needed | Full feature set, version-pinnable, dedicated ML nodes |
| **Docker (local)** | No cloud access, offline demo, dev/test | Version-dependent, no Kibana workflows, manual setup |

**Default recommendation:** Serverless Elasticsearch for any demo that includes Agent
Builder, Workflows, or ELSER. ECH for any demo where the customer needs to see a specific
version, ML node configuration, or ILM.

Also determine:
- **Region** — default to `us-east-1` (AWS) or `us-central1` (GCP) unless the customer
  is in EMEA (use `eu-west-1`) or APAC (use `ap-southeast-1`)
- **Index prefix** — if the user will run multiple demos on the same cluster, ask for a
  short prefix to namespace the indices (e.g., `cb-` for Citizens Bank makes
  `fraud-claims` → `cb-fraud-claims`). Leave blank for isolated clusters.

**Version (critical):**
- **Creating a new deployment or Serverless project** — Use the **latest generally
  available** stack for that product line **unless the SA names a specific version**.
  Serverless tracks current GA; ECH deployments should select the **newest supported GA**
  stack version in the create API/UI. Record the resolved version in `.env` as
  `ELASTIC_VERSION` and in the provision log.
- **Connecting to an existing deployment or project** (reuse — `cloud-manage-project`,
  copied `.env`, or customer URLs) — **Do not assume latest.** Read `version.number`
  from `GET /` against `ELASTICSEARCH_URL` and Kibana’s version from `/api/status` after
  credentials work, and persist both in the engagement context before downstream skills
  build scripts or plans.

## Step 2: Provision the Cluster

> **Dependency:** All cloud operations require the `cloud-setup` skill (from
> `elastic/agent-skills`) to have been run at least once to configure `EC_API_KEY`.
> If `EC_API_KEY` is not set in the environment, run `cloud-setup` first and follow
> its prompts. **Never ask the user to paste an API key into chat.**

### Serverless (Elasticsearch project type)

Use the `cloud-create-project` skill (from `elastic/agent-skills`) to create the project.
The project type must be `elasticsearch` (not `observability` or `security`) for Agent
Builder and Workflows to be available.

Required inputs for the API call:
- `name`: `demobuilder-{slug}-{date}` (e.g., `demobuilder-citizens-bank-20260415`)
- `region_id`: as determined above
- `type`: `elasticsearch`

Capture from the response:
- `elasticsearch.endpoints[0]` → `ELASTICSEARCH_URL`
- `kibana.endpoints[0]` → `KIBANA_URL`  
- The project API key (or create one via `POST /{project_id}/keys`) → `ES_API_KEY`

To connect to an existing serverless project (reuse cluster scenario), use
`cloud-manage-project` instead — it resolves endpoints and acquires a scoped API key
without creating a new project.

### ECH (Elastic Cloud Hosted)

Use the `cloud-setup` skill (from `elastic/agent-skills`) to configure org credentials,
then create a deployment via the EC API. Reference `references/ech-config.md` for the
deployment template and hardware profile options.

Capture:
- `resources.elasticsearch[0].info.metadata.endpoint` → `ELASTICSEARCH_URL`
- `resources.kibana[0].info.metadata.endpoint` → `KIBANA_URL`
- Create an API key post-deployment → `ES_API_KEY`

### Docker (local)

Generate a `docker-compose.yml` in the workspace root. Reference
`references/docker-compose-template.yml`. The user runs `docker compose up -d` manually.

Capture (from the compose file, for the .env):
- `ELASTICSEARCH_URL=http://localhost:9200`
- `KIBANA_URL=http://localhost:5601`
- `ES_API_KEY=` ← user must create after starting (or use basic auth — see note)

## Step 3: Write the Per-Engagement .env

Write `.env` to `{engagement_dir}/.env` (at the **engagement root** — not in a subfolder). This file is the single source of truth for all cluster
credentials for this engagement. Every subsequent skill (demo-deploy, `bootstrap.py`)
sources this file from the root path.

```bash
# Demo environment — {company}
# Provisioned: {date}
# Deployment type: {type}
# ⚠️  Do not commit to version control — contains credentials

DEMO_SLUG={slug}
DEPLOYMENT_TYPE={serverless|ech|self_managed|docker}
ELASTIC_VERSION={version}       # from provisioning response or platform audit

ELASTICSEARCH_URL={url}
KIBANA_URL={url}
ES_API_KEY={api_key}
KIBANA_API_KEY={kibana_api_key}   # used for all Kibana asset operations

# Index namespace (blank = no prefix, indices use default names from data model)
# Set this if running multiple demos on the same cluster to avoid collisions
INDEX_PREFIX=

# Kibana Space — written by Step 4.1 (ensure_kibana_space); /s/{DEMO_SLUG} for isolation
KIBANA_SPACE_PATH=/s/{slug}

# Optional: override demobuilder:<id> tag suffix (D-026); see demo-deploy/references/demobuilder-tagging.md
# DEMO_ASSET_TAG=

# Cluster metadata (informational)
CLUSTER_NAME={name}
REGION={region}
PROVISIONED_BY=demobuilder
ENGAGEMENT={company}
```

Also write `.env.example` with the same keys but values replaced by `<your-value-here>`.
This is safe to share or commit — it documents what's needed without exposing credentials.

```bash
# .env.example — copy to .env and fill in values
DEMO_SLUG=<slug>
DEPLOYMENT_TYPE=<serverless|ech|self_managed|docker>
# D-020: must match live cluster version (run: curl ... GET / | grep '"number"')
ELASTIC_VERSION=<e.g., 9.3.1>
ELASTICSEARCH_URL=<https://your-cluster.es.io:443>
KIBANA_URL=<https://your-kibana.kb.io:443>
ES_API_KEY=<your-es-api-key>
KIBANA_API_KEY=<your-kibana-api-key>
KIBANA_SPACE_PATH=/s/<slug>
INDEX_PREFIX=<optional-e.g., cb->
# DEMO_ASSET_TAG=<optional-override-for-demobuilder-tags>
```

**To reuse a cluster for a new engagement:** Copy the `.env` from an existing workspace
to the new workspace and update `DEMO_SLUG`, `ENGAGEMENT`, and optionally `INDEX_PREFIX`.
No re-provisioning needed. The new demo's indices will coexist with existing ones, namespaced
by prefix if set.

## Step 4: Validate Connectivity

After writing the `.env`, confirm the cluster is reachable and ready:

```bash
# Test connectivity
curl -s -H "Authorization: ApiKey ${ES_API_KEY}" "${ELASTICSEARCH_URL}/_cluster/health?pretty" \
  | python3 -c "import sys,json; h=json.load(sys.stdin); print(f'Status: {h[\"status\"]}, Nodes: {h[\"number_of_nodes\"]}')"

# Confirm Kibana is reachable
curl -s "${KIBANA_URL}/api/status" | python3 -c "import sys,json; s=json.load(sys.stdin); print(f'Kibana: {s[\"version\"][\"number\"]} - {s[\"status\"][\"overall\"][\"level\"]}')"
```

If connectivity fails: surface the specific error, check that the API key has the right
permissions (`cluster:monitor/main`, `indices:admin/create`, `indices:data/write/*`),
and provide a remediation step before writing the `.env`.

## Step 4.1: Create Engagement Kibana Space

Every engagement gets a dedicated Kibana Space for asset isolation — saved objects, SLOs,
and Agent Builder entities created by bootstrap will live here, not in the shared default
Space. This makes teardown clean and prevents demo assets from appearing in the customer's
or team's default view.

**Space ID:** use `DEMO_SLUG` as-is (e.g. `2026citizens-ai`). The Kibana Spaces API
is always called on the **default-space base URL** (no `/s/{id}` prefix), even when
`KIBANA_SPACE_PATH` is set.

```python
import json, urllib.request, urllib.error

def ensure_kibana_space(kb_url, kb_api_key, demo_slug, engagement):
    """Create a Kibana Space for this engagement if it does not exist. Idempotent."""
    space_id = demo_slug.strip().lower()
    headers = {
        "Authorization": f"ApiKey {kb_api_key}",
        "Content-Type": "application/json",
        "kbn-xsrf": "demobuilder",
    }
    # Always use the default-space path for space management APIs
    url = f"{kb_url.rstrip('/')}/api/spaces/space"
    # Check if space already exists
    req = urllib.request.Request(f"{url}/{space_id}", method="GET", headers=headers)
    try:
        urllib.request.urlopen(req, timeout=30)
        print(f"  Kibana space '{space_id}' already exists")
        return  # 200 → exists, nothing to do
    except urllib.error.HTTPError as e:
        if e.code != 404:
            raise
    # Create the space
    body = json.dumps({
        "id": space_id,
        "name": engagement,
        "description": f"Demobuilder engagement {space_id}",
    }).encode()
    req = urllib.request.Request(url, data=body, method="POST", headers=headers)
    try:
        urllib.request.urlopen(req, timeout=30)
        print(f"  Kibana space '{space_id}' created")
    except urllib.error.HTTPError as e:
        if e.code == 409:
            print(f"  Kibana space '{space_id}' already exists (409)")
        else:
            raise
```

After creating the space, append to `.env`:
```
KIBANA_SPACE_PATH=/s/{slug}
```

All subsequent Kibana API calls in bootstrap use this path prefix via `kb()` helper.

## Step 4.5: Verify Feature Flags

**Agent Builder and Kibana Workflows require feature flag activation on both Serverless
and ECH deployments** — not just Serverless. Availability varies by stack version and
deployment type; always verify at provisioning time before writing any build code. Do not
write any build code against these APIs until this check passes.

Run immediately after connectivity is confirmed, for all deployment types except Docker:

```bash
# Agent Builder — 404 means the feature is not yet enabled
curl -s -o /dev/null -w "%{http_code}" \
  -H "Authorization: ApiKey ${KIBANA_API_KEY}" \
  "${KIBANA_URL}/api/agent_builder/agents"
# → 200: enabled ✅   → 404: NOT enabled — activate before building ❌

# Workflows — same check
curl -s -o /dev/null -w "%{http_code}" \
  -H "Authorization: ApiKey ${KIBANA_API_KEY}" \
  "${KIBANA_URL}/api/workflows"
# → 200: enabled ✅   → 404: NOT enabled — activate before building ❌
```

If either returns 404, stop and activate the feature flag before proceeding. The path
to enable varies by deployment type — surface this to the user with the specific steps
for their platform (Serverless: project settings UI; ECH: deployment configuration).
Attempting to build against a disabled feature requires full retesting after activation.

Record the feature flag state in the provision log.

## Step 5: Write the Provision Log

`deploy/{slug}-provision-log.md`:

```
# Provision Log — {Company}
**Date:** {date}
**Type:** {type}
**Region:** {region}
**Cluster/Project:** {name}

## Credentials
Written to: `{engagement_dir}/.env` (engagement root — not in a subfolder)
ES URL: {url} ✅
Kibana URL: {url} ✅
API Key: configured ✅
**ELASTIC_VERSION:** {version} (from API — new deploys: latest GA unless pinned)
**KIBANA_SPACE_PATH:** /s/{slug} (created in Step 4.1)

## To reuse this cluster for another demo:
ROOT="${DEMOBUILDER_ENGAGEMENTS_ROOT:-$HOME/engagements}"
cp "$ROOT/{slug}/.env" "$ROOT/{other-slug}/.env"
# Then update DEMO_SLUG, ENGAGEMENT, and INDEX_PREFIX in the copied file

## To teardown this cluster when the demo is complete:
[ECH] Delete deployment via cloud.elastic.co or EC API
[Serverless] Delete project via cloud.elastic.co or EC API
[Docker] docker compose down -v

## Connectivity verified: {timestamp}
Cluster health: {green/yellow/red}
Nodes: {N}
Kibana: {version}
```

## What Good Looks Like

**Serverless, new demo:** User says "create a serverless project for the Citizens Bank
demo." Skill creates project named `demobuilder-citizens-bank-20260415` in `us-east-1`,
captures endpoint URLs and API key, writes `citizens-bank/.env`, validates connectivity.
Provision log confirms green health. Ready for demo-deploy.

**ECH, reusing cluster:** User has an existing ECH cluster and says "use my existing
ECH cluster for the Thermo Fisher demo." Skill skips provisioning, asks for the URL and
API key, writes `thermo-fisher/.env` with `INDEX_PREFIX=tf-` to avoid collisions with
any existing Citizens Bank indices on the same cluster. Validates connectivity.

**Multi-customer, same cluster:** User says "reuse the Citizens Bank cluster for the
IHG demo." Skill copies `citizens-bank/.env` to `ihg-club/.env`, sets
`INDEX_PREFIX=ihg-`, updates `DEMO_SLUG=ihg-club` and `ENGAGEMENT=IHG Club Vacations`.
No new provisioning needed.
