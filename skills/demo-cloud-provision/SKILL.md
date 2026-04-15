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

## Step 2: Provision the Cluster

### Serverless (Elasticsearch project type)

Use the existing `cloud-create-project` skill to create the project. The project type
must be `elasticsearch` (not `observability` or `security`) for Agent Builder and
Workflows to be available.

Required inputs for the API call:
- `name`: `demobuilder-{slug}-{date}` (e.g., `demobuilder-citizens-bank-20260415`)
- `region_id`: as determined above
- `type`: `elasticsearch`

Capture from the response:
- `elasticsearch.endpoints[0]` → `ELASTICSEARCH_URL`
- `kibana.endpoints[0]` → `KIBANA_URL`  
- The project API key (or create one via `POST /{project_id}/keys`) → `ES_API_KEY`

### ECH (Elastic Cloud Hosted)

Use the `cloud-setup` skill to configure org credentials, then create a deployment via
the EC API. Reference `references/ech-config.md` for the deployment template and hardware
profile options.

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

Write `.env` to `{workspace}/{slug}/.env`. This file is the single source of truth for
all cluster credentials for this engagement. Every subsequent skill (demo-deploy,
bootstrap.sh) sources this file.

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

# Index namespace (blank = no prefix, indices use default names from data model)
# Set this if running multiple demos on the same cluster to avoid collisions
INDEX_PREFIX=

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
ELASTIC_VERSION=<e.g., 9.3.1>
ELASTICSEARCH_URL=<https://your-cluster.es.io:443>
KIBANA_URL=<https://your-kibana.kb.io:443>
ES_API_KEY=<your-api-key>
INDEX_PREFIX=<optional-e.g., cb->
```

**To reuse a cluster for a new engagement:** Copy the `.env` from an existing workspace
to the new workspace and update `DEMO_SLUG`, `ENGAGEMENT`, and optionally `INDEX_PREFIX`.
No re-provisioning needed. The new demo's indices will coexist with existing ones, namespaced
by prefix if set.

## Step 4: Validate Connectivity

After writing the `.env`, confirm the cluster is reachable and ready:

```bash
# Test connectivity
curl -s -u "elastic:${ES_API_KEY}" "${ELASTICSEARCH_URL}/_cluster/health?pretty" \
  | python3 -c "import sys,json; h=json.load(sys.stdin); print(f'Status: {h[\"status\"]}, Nodes: {h[\"number_of_nodes\"]}')"

# Confirm Kibana is reachable
curl -s "${KIBANA_URL}/api/status" | python3 -c "import sys,json; s=json.load(sys.stdin); print(f'Kibana: {s[\"version\"][\"number\"]} - {s[\"status\"][\"overall\"][\"level\"]}')"
```

If connectivity fails: surface the specific error, check that the API key has the right
permissions (`cluster:monitor/main`, `indices:admin/create`, `indices:data/write/*`),
and provide a remediation step before writing the `.env`.

## Step 5: Write the Provision Log

`{slug}-provision-log.md`:

```
# Provision Log — {Company}
**Date:** {date}
**Type:** {type}
**Region:** {region}
**Cluster/Project:** {name}

## Credentials
Written to: `{workspace}/{slug}/.env`
ES URL: {url} ✅
Kibana URL: {url} ✅
API Key: configured ✅

## To reuse this cluster for another demo:
cp {workspace}/{slug}/.env {workspace}/{other-slug}/.env
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
