# Elastic Deployment — Behavioral Differences and Known Gotchas

Loaded by: demo-deploy, demo-ml-designer, demo-platform-audit, demo-validator

Source: Lowe's "Store That Knows" demo first-gen postmortem. Every item below caused real rework.

Most sections apply to Serverless. Feature flags (first section) apply to **both Serverless and ECH** until those features reach GA.

---

## Feature Flags — Serverless AND ECH (not enabled by default)

**Agent Builder and Kibana Workflows require feature flag activation on both Serverless and ECH deployments** — not just Serverless. Workflows is expected to reach GA with Elastic 9.4; until then, always verify.

Verify immediately after project/deployment creation, before any build work:

```
GET /api/agent_builder/agents   → 404 means not enabled, stop
GET /api/workflows              → 404 means not enabled, stop
```

Use `KIBANA_API_KEY` for these checks. Do not write any build code against these APIs until they return 200. How to enable depends on deployment type — surface this to the user if 404 is returned.

---

## ML Anomaly Detection — Field Names

`.ml-anomalies-*` on Serverless uses different field names than self-managed documentation:

| Self-managed docs say | Serverless actual |
|---|---|
| `anomaly_score` | `record_score` |
| `@timestamp` | `timestamp` |
| `store_id` (partition) | `partition_field_value` |
| `sku` (by field) | `by_field_value` |

Before writing any query against `.ml-anomalies-*`, run `GET .ml-anomalies-*/_mapping` to confirm.

---

## ML Anomaly Explorer UI

Not available on Serverless. Use `.ml-anomalies-*` index directly via ES|QL queries and Kibana dashboards instead. Update demo script accordingly — any scene referencing the ML Swimlane UI must use a custom dashboard panel instead.

---

## ML Datafeed — geo_point Fields

If an index used as a datafeed source contains a `geo_point` field, the ML datafeed will fail unless the field is excluded or shadowed via `runtime_mappings`. Add this to the datafeed config:

```json
"runtime_mappings": {
  "store_location": {
    "type": "keyword",
    "script": "emit('')"
  }
}
```

Or exclude the field from the datafeed's `_source` config.

---

## Kibana Workflows — Liquid Array Syntax

In Workflow YAML, array access uses Liquid `| first` filter, NOT JavaScript-style `[0]`:

```yaml
# WRONG — rejected by validator:
product_name: "{{ steps.geo_search.output.hits.hits[0]._source.product_name }}"

# CORRECT:
product_name: "{% assign h = steps.geo_search.output.hits.hits | first %}{{ h._source.product_name }}"
```

---

## Kibana Workflows — Sort with Template Variables

`_geo_distance` sort with template variable coordinates is rejected by the Workflow validator. Use a geo_distance filter to constrain results (with `size: 1`) instead of sorting by distance:

```yaml
# WORKS — geo filter constraint:
- geo_distance:
    distance: "{{ inputs.radius_miles }}mi"
    store_location:
      lat: ${{ inputs.origin_lat }}
      lon: ${{ inputs.origin_lon }}

# REJECTED — sort with template vars:
sort:
  - _geo_distance:
      store_location:
        lat: "{{ inputs.origin_lat }}"   # template var in sort = invalid
```

---

## Kibana Workflows — Condition/If Structure (YAML)

The YAML uses `if` step type with nested steps (not `condition` type with `runIf`):

```yaml
- name: check_found
  type: if
  condition: "${{ steps.search.output.hits.total.value > 0 }}"
  steps:
    - name: create_record
      type: elasticsearch.index
      with:
        ...
```

---

## ELSER on Serverless

Use `"service": "elser"` (managed endpoint). Do NOT specify `model_id` or use `"service": "elasticsearch"`:

```json
{
  "service": "elser",
  "service_settings": {
    "num_allocations": 1,
    "num_threads": 1
  }
}
```

Cold ELSER inference on Serverless can take 30+ seconds on first call. Always warm before demo.

---

## Agent Builder — Tool Config: `pattern` not `index`

For `index_search` tool type, the config field is `pattern`, NOT `index`:

```json
{
  "type": "index_search",
  "configuration": {
    "pattern": "inventory-positions"   // correct
    // "index": "inventory-positions"  // wrong — API rejects this
  }
}
```

---

## Kibana Saved Objects — Strip migrationVersion Before Import

Objects exported from Serverless include `migrationVersion` and `coreMigrationVersion` fields that cause import errors. Strip them before committing to repo or re-importing:

```python
import json
with open("dashboard.ndjson") as f:
    objects = [json.loads(line) for line in f if line.strip()]
for obj in objects:
    obj.pop("migrationVersion", None)
    obj.pop("coreMigrationVersion", None)
```

---

## ILM

ILM is not supported on Serverless. Use Data Stream Lifecycle (DSL) via the `lifecycle` block in the index template. bootstrap.py detects this via `DEPLOYMENT_TYPE` and skips the ILM API call.

---

## Kibana API Authentication

Always use `KIBANA_API_KEY` for all Kibana asset operations: Agent Builder, Workflows,
Dashboards, Connectors, and Saved Objects import. This is a separate key from `ES_API_KEY`.

API key privilege requirements for Kibana vs. Elasticsearch are under active product change.
`KIBANA_API_KEY` remains a required field in `.env` until product confirms a unified approach.
Do not fall back to `ES_API_KEY` for Kibana API calls — keep the keys separate.

---

## Reference Repositories

Always read these BEFORE writing Workflow YAML or Agent Builder API calls:

- `https://github.com/elastic/workflows` — authoritative Workflow YAML examples, Liquid syntax
- `https://github.com/elastic/kibana-agent-builder-sdk` — Agent Builder tool/agent API schema

If these repos are not in context, ask the user to provide them before writing any Workflow or Agent Builder code. Do not iterate past 30 minutes on undocumented API behavior without surfacing this as a blocker.
