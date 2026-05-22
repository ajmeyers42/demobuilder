---
name: bolt-launch
description: >
  DEPRECATED as of 2026-05-05. This skill has been split into two focused agents:
  finish-verify (Agent A) and bolt-bootstrap (Agent B). Do not execute this skill.
status: deprecated
---

# bolt-launch — DEPRECATED

**This skill has been replaced. Do not execute further.**

Use the two-stage replacement pipeline:

1. **`finish-verify`** — probes the target cluster, confirms schemas and managed assets, calls `elastic/agent-skills` to author verified asset definitions, and produces `asset-bundle/`.
2. **`bolt-bootstrap`** — reads `asset-bundle/` and generates `main.tf`, `providers.tf`, `{slug}.tfvars`, and `bootstrap-data.py`.

---

**To use the replacement:**

Read `skills/finish-verify/SKILL.md` now and follow it. Do not return to this file.

---

*Reference files that were previously under `skills/bolt-launch/references/` have been moved to `skills/references/`. All paths updated in finish-verify, bolt-ech, bolt-serverless, and bolt-bootstrap.*
