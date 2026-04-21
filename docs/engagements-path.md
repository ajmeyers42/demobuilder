# Engagement root: `DEMOBUILDER_ENGAGEMENTS_ROOT`

## Why

- **Portable repo:** The demobuilder git tree holds **pipeline skills** (`skills/`, `docs/`) only. Customer-specific files stay **out of version control** and can live on **local disk or cloud sync** (e.g. Google Drive “My Drive”).
- **Sharable:** You can push the repo without engagement data; teammates set their own root.

## Convention

| Variable | Meaning |
|----------|---------|
| `DEMOBUILDER_ENGAGEMENTS_ROOT` | Absolute path to a directory that **contains** one folder per engagement |

Engagement workspace for slug `{slug}`:

`$DEMOBUILDER_ENGAGEMENTS_ROOT/{slug}/`

Example: `$DEMOBUILDER_ENGAGEMENTS_ROOT/2026CitizensAI/`

## Setup

1. Create the root directory (e.g. `demobuilder-engagements` under Google Drive **My Drive**).
2. Export the variable in your shell profile:

```bash
export DEMOBUILDER_ENGAGEMENTS_ROOT="$HOME/Google Drive/demobuilder-engagements"
```

**This machine (example):** the Citizens AI engagement was moved to  
`~/Library/CloudStorage/GoogleDrive-andrew.meyers@elastic.co/My Drive/demobuilder-engagements`  
(same as `$HOME/Google Drive/demobuilder-engagements` if the Drive symlink is present).

3. Point the agent at **`$DEMOBUILDER_ENGAGEMENTS_ROOT/{slug}`** when running the orchestrator or a single skill.

## Agents

Skills and [`AGENTS.md`](../AGENTS.md) assume this variable is set when writing or reading engagement artifacts. If it is unset, resolve the path with the user before creating files.

## Decision record

See `docs/decisions.md` **D-019** and **D-023**.
