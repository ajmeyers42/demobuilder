---
name: demo-data-modeler
description: >
  Reads a demo script and discovery profile to generate all Elasticsearch data artifacts
  needed to build the demo environment: index mappings, data stream templates, component
  templates, ILM policies, ingest pipelines, enrich policies, and sample data specifications.
  Produces a master build manifest and individual artifact files in correct build order.

  ALWAYS use this skill when the user says "generate the mappings", "build the data model",
  "create the index templates", "what indices do we need", or has a demo script and wants to
  start building the technical environment. Also trigger after demo-script-template completes
  when the user is ready to move from planning to building. Run before demo-ml-designer —
  ML jobs depend on the data model being defined first.
---

# Demo Data Modeler

You are generating the Elasticsearch data artifacts for a pre-sales demo environment.
Everything you produce must be valid, runnable Elasticsearch configuration — not pseudocode,
not placeholders. An SE should be able to take your output, run it against a cluster, and
have a working data layer.

## Step 1: Extract the Data Requirements

Read the demo script (`{slug}-demo-script.md`) and discovery JSON (`{slug}-discovery.json`).
From the script, identify every index, data stream, pipeline, and data element referenced:

- **Every index name** mentioned in queries (`FROM fraud-claims`, `GET associate-sessions/_doc/...`)
- **Every field** used in an ES|QL query, displayed in Kibana, or referenced in an agent tool
- **Every computed or derived field** (e.g., `discrepancy_pct = (system_on_hand - stock_on_hand) / system_on_hand * 100`)
- **Every ingest-time enrichment** (geo_point lookups, enrich processor joins)
- **Every feature-specific field type** (`semantic_text` for ELSER, `geo_point` for spatial, `nested` for multi-turn conversation history)
- **Every data relationship** (which index is written by an ingest pipeline processing another index)

Group them into:
1. **Data streams** — append-only, time-series (events, transactions, logs)
2. **Regular indices** — mutable documents (positions, sessions, metadata)
3. **System indices** — Kibana/agent artifacts (sessions, telemetry, fulfillment records)

## Step 2: Design Each Index

For each index, define:

**Field types** — use the most specific type that fits the query pattern:
- Identifiers (`store_id`, `sku`, `session_id`): `keyword`
- Free text for full-text search: `text` with `.keyword` sub-field for aggregations
- Semantic/ELSER search: `semantic_text` with `inference_id` pointing to the ELSER endpoint
- Counts and quantities: `integer` or `long`
- Rates and percentages: `float` or `double`
- Timestamps: `date`
- Locations: `geo_point`
- Computed at ingest: add as regular fields (populated by ingest pipeline script processor)
- Multi-value arrays with sub-structure: `nested`
- Booleans: `boolean`

**Do not use `dynamic: true` on production-intent indices.** Explicit mappings only.
On indices where dynamic mapping is acceptable (e.g., enrichment lookup tables), set
`dynamic: false` (accept but don't index unmapped fields) or `dynamic: strict` (reject).

**Shard count:**
- Data streams / high-volume indices: 1 primary shard per ~50GB expected data, minimum 1
- Small lookup/metadata indices (< 1GB): 1 primary shard
- Default replica count: 1 (adjust based on platform audit output)

**ILM:**
- Data streams always get an ILM policy (hot → warm → delete at minimum)
- Regular mutable indices that grow unboundedly also need ILM or retention strategy
- Lookup/seed data indices: no ILM needed

## Step 3: Design the Ingest Pipelines

For each pipeline, write the full processor chain. Common patterns:

**Enrich processor** (adds fields from a lookup index):
```json
{
  "enrich": {
    "policy_name": "store-location-enrich",
    "field": "store_id",
    "target_field": "store_meta",
    "max_matches": 1
  }
}
```
Enrich policies must be created and executed before the pipeline references them.

**Script processor** (computed fields):
```json
{
  "script": {
    "lang": "painless",
    "source": "ctx.discrepancy_pct = ctx.system_on_hand > 0 ? Math.round(((ctx.system_on_hand - ctx.stock_on_hand) / (float)ctx.system_on_hand) * 1000) / 10.0 : 0.0;"
  }
}
```

**Upsert via pipeline + update_by_query pattern:** When an ingest pipeline needs to update
a document in a different index (e.g., transactions updating inventory-positions), use a
script processor to emit to the secondary index via the `_index` rerouting or a subsequent
bulk action. For the scripted_upsert pattern:
```json
{
  "script": {
    "lang": "painless",
    "source": "...",
    "params": { ... }
  },
  "upsert": { ... }
}
```

**Date/timestamp normalization:**
```json
{ "date": { "field": "@timestamp", "formats": ["ISO8601", "epoch_millis"] } }
```

## Step 4: Define the Sample Data Specification

For each index, specify what the seed data should look like — not the actual data (that
is generated by a separate script), but the schema and value constraints:

```json
{
  "index": "inventory-positions",
  "seed_document_count": 500,
  "key_entities": [
    { "field": "store_id", "values": ["1842", "2051", "2089"], "distribution": "weighted_to_1842" },
    { "field": "sku", "range": "100000-999999", "count": 500 }
  ],
  "demo_critical_docs": [
    {
      "description": "SKU 174239 at store 1842 — low stock vs high system_on_hand (Scenario 1)",
      "fields": { "store_id": "1842", "sku": "174239", "stock_on_hand": 23, "system_on_hand": 47 }
    }
  ],
  "field_ranges": {
    "stock_on_hand": "0-200",
    "system_on_hand": "0-200",
    "discrepancy_pct": "computed"
  }
}
```

**Demo-critical documents are non-negotiable.** The scenarios in the script depend on
specific field values existing. Enumerate every document that must be present for a
scenario to work.

## Step 5: Write the Outputs

### Output 1: `{slug}-data-model.json`

Master manifest — one document describing everything that needs to be built:

```json
{
  "slug": "",
  "build_order": [
    { "step": 1, "artifact": "elser-inference-endpoint", "type": "inference", "reason": "required before semantic_text mapping" },
    { "step": 2, "artifact": "store-location-enrich-policy", "type": "enrich_policy", "reason": "required before ingest pipeline" },
    { "step": 3, "artifact": "store-transactions-pipeline", "type": "ingest_pipeline", "reason": "required before data stream template" },
    { "step": 4, "artifact": "store-transactions-template", "type": "index_template" },
    { "step": 5, "artifact": "inventory-positions", "type": "index" },
    { "step": 6, "artifact": "associate-knowledge-base", "type": "index", "reason": "semantic_text — ELSER endpoint must exist" }
  ],
  "indices": [ ... ],
  "data_streams": [ ... ],
  "ingest_pipelines": [ ... ],
  "enrich_policies": [ ... ],
  "ilm_policies": [ ... ],
  "inference_endpoints": [ ... ],
  "sample_data": [ ... ]
}
```

### Output 2: `{slug}-data-model.md`

Human-readable build overview for the SE:

```
# Data Model — [Company / Demo Name]

## What Gets Built
[Table: artifact name | type | purpose | build step]

## Build Order
[Numbered list with dependency notes]

## Index Designs
[For each index: purpose, key fields with types, shard count, ILM, special notes]

## Ingest Pipeline Logic
[For each pipeline: what it does, processor chain summary, any tricky logic]

## Sample Data Requirements
[For each index: doc count, key entities, demo-critical documents]

## Dependency Map
[Which artifacts block which — drawn in text if no diagram tool]
```

### Output 3: Individual artifact files in `mappings/` and `pipelines/`

For each index: a standalone JSON file with the complete mapping, settings, and aliases.
For each pipeline: a standalone JSON file with the complete processor chain.
These files should be directly passable to the Elasticsearch API:

```
# mapping file format
PUT /{index-name}
{
  "mappings": { ... },
  "settings": { "number_of_shards": 1, "number_of_replicas": 1 }
}
```

For data stream templates:
```
# component template + index template pattern
PUT /_component_template/{name}-mappings
{ "template": { "mappings": { ... } } }

PUT /_component_template/{name}-settings
{ "template": { "settings": { ... } } }

PUT /_index_template/{name}
{ "index_patterns": ["{name}-*"], "data_stream": {}, "composed_of": [...], "priority": 200 }
```

## What Good Looks Like

**Lowe's pattern** — complex interdependent model: `store-transactions` data stream (events
upsert into `inventory-positions` via pipeline), `inventory-positions` (computed
`discrepancy_pct` field, geo_point for nearby-store queries), `associate-knowledge-base`
(semantic_text with ELSER), `associate-sessions` (nested conversation_history), `fulfillment-requests`
(written by Workflows). Build order has 17 steps because ELSER must precede KB mapping.

**Citizens Bank pattern** — simpler unified model: `fraud-claims` index (ingested from
three sources via connectors — no custom pipeline, connector handles the ETL), `fraud-escalations`
(written by agent tool calls), `sla-monitoring-telemetry` (agent observability). No data
stream needed — claims are mutable documents. No ELSER endpoint if semantic scene is
de-scoped.

**Migration pattern** — read-heavy, current-state mirroring: `current-state-mirror` index
mirrors the key structure of the customer's existing indices (inferred from diagnostic).
Demo shows before/after on the same data model. Primary artifact is the mapping that
mirrors their schema — not a new one.

## Reference: Elasticsearch Mapping Quick Reference

See `references/mapping-patterns.md` for:
- Complete field type reference with demo use cases
- `semantic_text` exact syntax for ELSER v2
- Data stream template boilerplate
- ILM policy boilerplate (hot-warm-delete pattern)
- Ingest pipeline patterns (enrich, script, upsert)
- Nested vs object vs flattened tradeoffs
