# JSON Schemas

This directory contains versioned contracts for machine-readable demobuilder outputs. Skills still explain how to reason about each artifact, but schemas define the stable fields downstream stages may rely on.

These schemas are intentionally lightweight at first. They validate shape and core enums without trying to encode every engagement-specific field.

| Schema | Produced by | Consumed by |
|---|---|---|
| `opportunity-profile.schema.json` | `demo-opportunity-review` | `demo-platform-audit`, `demo-script-template`, orchestrator |
| `platform-audit.schema.json` | `demo-platform-audit` | `demo-script-template`, `demo-data-modeler`, `demo-deploy`, `demo-validator` |
| `data-model.schema.json` | `demo-data-modeler` | `demo-ml-designer`, `demo-deploy`, `demo-validator` |

## Maintenance

When changing an output contract:

1. Update the relevant schema here.
2. Update the producing skill's `SKILL.md`.
3. Update any consuming skills that read the changed fields.
4. Add or update the skill eval that covers the changed contract.
