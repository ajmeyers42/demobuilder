# Kibana Workflow Patterns — Practical Reference (9.3)

Patterns derived from the Lowe's "Store That Knows" demo. Every snippet here
was validated against a live Serverless cluster. Use this as a lookup, not a
tutorial.

---

## Complete Working Workflow: Geo-Search + Conditional Index

`nearby-store-transfer` is the most complex pattern used in demos. It runs a
geo-distance search and, if stock is found, writes a transfer request document.
Copy this as a starting point for any workflow that needs conditional indexing.

```yaml
version: "1"
name: nearby-store-transfer
description: >
  Searches inventory-positions for a SKU within the given radius.
  If stock is found, creates a transfer request document.
enabled: true
tags: [inventory, associate-agent, demo]

inputs:
  - name: sku
    type: string
    required: true
  - name: origin_lat
    type: number
    required: true
  - name: origin_lon
    type: number
    required: true
  - name: dest_store_id
    type: string
    required: true
  - name: session_id
    type: string
    required: true
  - name: radius_miles
    type: number
    default: 10

triggers:
  - type: manual

steps:
  - name: geo_search
    type: elasticsearch.search
    with:
      index: inventory-positions
      size: 1
      query:
        bool:
          filter:
            - term:
                sku: "{{ inputs.sku }}"
            - range:
                stock_on_hand:
                  gt: 0
            - geo_distance:
                distance: "{{ inputs.radius_miles }}mi"
                store_location:
                  lat: ${{ inputs.origin_lat }}
                  lon: ${{ inputs.origin_lon }}
      _source: [store_id, stock_on_hand, product_name]

  - name: check_found
    type: if
    condition: "${{ steps.geo_search.output.hits.total.value > 0 }}"
    steps:
      - name: create_transfer_request
        type: elasticsearch.index
        with:
          index: fulfillment-requests
          document:
            request_id: "TXF-{{ execution.id }}"
            session_id: "{{ inputs.session_id }}"
            sku: "{{ inputs.sku }}"
            product_name: "{% assign h = steps.geo_search.output.hits.hits | first %}{{ h._source.product_name }}"
            origin_store_id: "{% assign h = steps.geo_search.output.hits.hits | first %}{{ h._source.store_id }}"
            dest_store_id: "{{ inputs.dest_store_id }}"
            type: transfer
            status: pending
            created_at: "{{ execution.startedAt }}"
```

**Things to notice:**

- `geo_distance` with `size: 1` — always limit to 1 hit when you only need the
  nearest result. Keeps the response small and makes `| first` unambiguous.
- `lat`/`lon` use `${{ }}` (expression syntax), not `{{ }}` (string
  interpolation). Numbers passed as inputs must go through expression syntax or
  they arrive as strings and geo queries reject them.
- `condition:` on the `if` step also uses `${{ }}` because it evaluates an
  expression, not a string.
- `TXF-{{ execution.id }}` gives every transfer request a stable, unique ID
  tied to the execution — safe to replay.

---

## Accessing Array Values — The `| first` Rule

This is the #1 cause of Workflow validation failures.

**The problem:** Elasticsearch returns hits as an array.
`steps.geo_search.output.hits.hits` is a list. You cannot index into it with
`[0]` in Liquid — the `[0]` bracket syntax is not supported in Kibana's Liquid
engine and causes a validation error that surfaces as `"valid": false` with a
cryptic or empty `validationErrors` array.

**The fix:** Use the `| first` filter to get the first element.

```yaml
# WRONG — causes validation failure
product_name: "{{ steps.geo_search.output.hits.hits[0]._source.product_name }}"

# CORRECT — assign first element, then access its fields
product_name: "{% assign h = steps.geo_search.output.hits.hits | first %}{{ h._source.product_name }}"
```

**Accessing nested fields after `| first`:**

```yaml
# Assign once, use multiple times in the same field value
origin_store_id: "{% assign hit = steps.search_step.output.hits.hits | first %}{{ hit._source.store_id }}"
stock_count:     "{% assign hit = steps.search_step.output.hits.hits | first %}{{ hit._source.stock_on_hand }}"
```

Each field value in `document:` is a separate template string — you must
re-assign `| first` in each one. There is no cross-field variable scope.

**Checking whether results exist before accessing:**

Always guard with an `if` step before accessing `hits.hits`. If the search
returns zero hits and you try to render `| first`, the value is `nil` and the
document will index with a null field rather than failing loudly.

```yaml
- name: guard
  type: if
  condition: "${{ steps.my_search.output.hits.total.value > 0 }}"
  steps:
    - name: do_something
      type: elasticsearch.index
      with:
        # ... safe to use | first here
```

---

## Template Variable Reference

| Variable | Type | Notes |
|---|---|---|
| `inputs.{name}` | any | Values provided by the caller (agent tool or manual trigger). Type coercion follows the input's declared `type`. |
| `steps.{step_name}.output` | object | Full response body from the previous step. For `elasticsearch.search`, this is the standard ES response object (`hits.hits`, `hits.total.value`, `_shards`, etc.). |
| `execution.id` | string | Unique ID for this workflow execution. Use it to construct deterministic document IDs like `TXF-{{ execution.id }}`. |
| `execution.startedAt` | string | ISO 8601 timestamp of when this execution began. Safe to write directly into date fields. |

**Syntax rules:**

| Syntax | When to use |
|---|---|
| `{{ variable }}` | String interpolation — renders the value as text inside a string field. |
| `${{ expression }}` | Expression evaluation — use in `condition:`, and for numeric inputs passed into numeric positions (lat/lon, sizes, etc.). |
| `{% assign x = ... %}` | Liquid tag — used to capture intermediate values like the first array element. Must appear in the same string as the `{{ x }}` that uses it. |

---

## Step Types Quick Reference

| Step type | What it does | Key `with:` fields |
|---|---|---|
| `elasticsearch.search` | Runs an ES query and returns the full response body as `output`. | `index` (required), `query` (required), `size`, `_source`, `sort` |
| `elasticsearch.index` | Indexes a document, returns `_id` and `result` in `output`. | `index` (required), `document` (required), `id` (optional — omit to auto-generate) |
| `if` | Conditionally executes a block of child steps. Child steps are nested under `steps:`. | `condition` (required, expression syntax `${{ }}`), `steps` (required) |
| `kibana.email` | Sends an email via a configured email connector. | `connectorId` (required), `to` (list), `subject`, `message` |

**Notes:**

- `if` steps do not produce `output`. They are control flow only. Reference the
  output of child steps directly by their name:
  `steps.create_transfer_request.output`.
- `elasticsearch.index` auto-generates `_id` when `id:` is omitted. If you
  need a stable, idempotent ID (e.g., one transfer per execution), set
  `id: "TXF-{{ execution.id }}"` explicitly.
- `sort:` in `elasticsearch.search` does not support template variables.
  Hardcode sort fields. Attempting `sort: { "{{ inputs.sort_field }}": "asc" }`
  will fail validation.
- `kibana.email` requires the connector to already exist in Kibana. Get the
  `connectorId` from `GET /api/actions/connectors`.

---

## Validation Checklist

Run this before wiring a Workflow to an agent:

```
GET /api/workflows/{workflow-id}
→ "valid": true          ✅ proceed — safe to add to Agent Builder tool config
→ "valid": false         ❌ check validationErrors array, fix Liquid syntax
→ 404                    ❌ feature flag not enabled — activate Workflows before building
```

**If `valid: false` with empty `validationErrors`:**

This is a known quirk. The error is almost always one of:
1. `[0]` array indexing instead of `| first`
2. Template variable in a `sort:` field
3. Expression syntax (`${{ }}`) used where string syntax (`{{ }}`) is required
   or vice versa

Re-read every template field carefully — the validation API does not always
pinpoint the offending line.

**If 404:**

The Workflows feature flag is not enabled on the project. Go to
**Management → Advanced Settings** and enable it, or provision the project with
`kibana.workflows.enabled: true` in the project config before building.

---

## Deploying via API

All commands use the `.env` credentials. Set these before running:

```bash
source .env
```

**1. Create a workflow from YAML:**

```bash
curl -s -X POST "${KIBANA_URL}/api/workflows" \
  -H "Authorization: ApiKey ${ES_API_KEY}" \
  -H "Content-Type: application/yaml" \
  -H "kbn-xsrf: true" \
  --data-binary @nearby-store-transfer.yaml
```

Response includes `id` — this is the `workflow_id` you wire into Agent Builder.

```json
{
  "id": "wf-a1b2c3d4",
  "name": "nearby-store-transfer",
  "valid": true
}
```

**2. List existing workflows to get IDs:**

```bash
curl -s -X GET "${KIBANA_URL}/api/workflows" \
  -H "Authorization: ApiKey ${ES_API_KEY}" \
  -H "kbn-xsrf: true" \
  | jq '.items[] | {id, name, valid}'
```

**3. Get a specific workflow to verify `valid: true`:**

```bash
WORKFLOW_ID="wf-a1b2c3d4"

curl -s -X GET "${KIBANA_URL}/api/workflows/${WORKFLOW_ID}" \
  -H "Authorization: ApiKey ${ES_API_KEY}" \
  -H "kbn-xsrf: true" \
  | jq '{id, name, valid, validationErrors}'
```

The `id` returned from creation is what goes into the Agent Builder tool config
under `workflow_id`. Do not confuse it with the workflow `name` — the agent
runtime resolves by ID only.

---

## Common Errors and Fixes

| Error | Cause | Fix |
|---|---|---|
| `"valid": false` — `validationErrors` references `[0]` syntax | Array index bracket notation is not supported in Kibana's Liquid engine | Replace `hits.hits[0]._source.field` with `{% assign h = hits.hits \| first %}{{ h._source.field }}` |
| `"valid": false` with empty `validationErrors` array | Template engine fails silently on unsupported syntax (often `[0]`, or `${{ }}` in a string position) | Audit every template field; switch array access to `\| first`; confirm expression vs. string syntax per field |
| Template variable in `sort:` causes validation failure | `sort:` fields are not templated — Kibana parses sort at definition time | Hardcode the sort field name. Move dynamic sort logic to separate search steps if needed |
| `workflow_id not found` error in Agent Builder at runtime | The agent tool config references the workflow `name` instead of its API `id` | Run `GET /api/workflows` to get the `id` field (e.g. `wf-a1b2c3d4`) and update the tool config |
| `404` on `GET /api/workflows` | Workflows feature flag is not enabled | Enable via Management → Advanced Settings or re-provision with `kibana.workflows.enabled: true` |
| Geo query rejects `lat`/`lon` as wrong type | Input values passed through `{{ }}` string interpolation arrive as strings | Use `${{ inputs.origin_lat }}` (expression syntax) for all numeric input positions |
| `nil` field written to index document | Search returned 0 hits; `\| first` on an empty array returns nil; no guard in place | Wrap the index step in an `if` step that checks `steps.search.output.hits.total.value > 0` |
