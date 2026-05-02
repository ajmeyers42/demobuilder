# demo-fleet-integrations

**Status: BACKLOG — not yet implemented**
**Scoped: 2026-05-01 | Originated from: 2026lenovoGAIaaS engagement**

## Purpose

Installs and configures Elastic Fleet integration packages (EPM) for demo environments. Provides managed assets — ingest pipelines, index templates, ILM policies, dashboards — from the Integrations catalog rather than hand-rolled equivalents. Supports both agent-based collection and synthetic bulk ingest into integration-managed indices.

## When to Use

- Demo scope requires showing how data flows in from real infrastructure agents
- Customer is already using Fleet Agent or evaluating it
- You want pre-built dashboards from the integration without building them manually
- Scenario mixes agent-managed sources (system, kubernetes, nginx) with custom app logs

## Trigger Phrases

"install the kubernetes integration", "use Fleet for log collection", "EPM package install", "I want the out-of-the-box dashboards", "set up the integration", "install integration packages for the demo"

## Inputs

- `discovery.json` — for the list of data sources identified in the engagement
- `data-model.json` — for index patterns and field schemas to cross-reference against integration schemas
- `.env` — cluster credentials (`ELASTICSEARCH_URL`, `KIBANA_URL`, `ES_API_KEY`, `KIBANA_API_KEY`)
- Optional: explicit list of package names and versions to install

## Outputs

- `deploy/integrations-manifest.json` — list of installed packages, versions, agent policy IDs, integration policy IDs, managed index patterns
- Updates `bootstrap.py` or generates a companion `bootstrap-integrations.py` with idempotent install steps
- Prints pre/post-install summary: packages installed, dashboards available, index patterns created

## Key Capabilities to Implement

### 1. Package resolution
- Query `GET /api/fleet/epm/packages` to list available packages and versions
- Match package names from the data model against available catalog entries
- Warn if requested version is not available for cluster version

### 2. Package installation
```
POST /api/fleet/epm/packages/{name}/{version}
{ "force": true }
```
- Handle already-installed packages (200 response, skip or upgrade based on flag)
- Capture managed index patterns and ingest pipeline names from install response
- Add installed package to manifest

### 3. Agent policy + integration policy creation
```
POST /api/fleet/agent_policies
POST /api/fleet/package_policies
```
- Create a demo-scoped agent policy per engagement (`{SLUG}-demo-policy`)
- Attach integration policies for each installed package with demo-appropriate config
- Output the enrollment token for documentation purposes (no actual agent enrollment in bootstrap)

### 4. Synthetic data compatibility
- After package install, resolve the managed write index for each integration
- Allow `bootstrap.py` to bulk-index synthetic documents directly into those indices
- Validate that synthetic doc schema matches integration ECS mappings (field type check)

### 5. Teardown registration
- Register all installed packages in `demobuilder-manifests` under `assets.fleet_integrations`
- `teardown.py` should call `DELETE /api/fleet/epm/packages/{name}/{version}` for each

## Packages to Support (Initial Scope)

| Package | Provides | Demo use case |
|---|---|---|
| `kubernetes` | Pod logs, node metrics, events | K8s infrastructure visibility |
| `system` | Host logs, CPU/memory metrics | Node-level health |
| `nvidia_gpu` (if available) | GPU metrics via DCGM | GPU utilization — may conflict with custom TSDB templates |
| `apm` | APM traces and metrics | LLM/agent latency tracing |
| `synthetics` | Uptime/availability | Endpoint health checks |

## Constraints and Risks

- **Fleet Server required** for full activation; demo clusters may not have Fleet Server enrolled agents. Skill must handle "packages installed, no agents enrolled" gracefully.
- **Schema conflicts**: If custom component templates are already applied to the same index pattern, package installation may conflict. Skill must detect and warn.
- **Version pinning**: Package versions are tied to stack version. The skill must resolve the correct version at bootstrap time, not hardcode it.
- **Serverless**: EPM is available on Serverless but Fleet Agent enrollment is not. Package install for asset provisioning (dashboards, pipelines) is still valid.

## Pipeline Position

```
demo-data-modeler → demo-fleet-integrations → bootstrap.py (step 4 or step 6+)
```

Runs after data model is defined so package selection can be cross-referenced against modeled indices. Generates additional steps in bootstrap for package install before index template creation (packages may create their own templates).

## Acceptance Criteria

- [ ] `python3 bootstrap.py --step integrations` installs all scoped packages idempotently
- [ ] Re-run does not reinstall already-installed packages (checks version match)
- [ ] Manifest records all installed package names, versions, and managed index patterns
- [ ] `teardown.py` successfully removes all fleet integration assets
- [ ] Works on ECH 9.x and Serverless Elasticsearch (asset-only mode for serverless)
- [ ] Warns clearly when a package is unavailable or version-incompatible
