# Engagement root: `DEMOBUILDER_ENGAGEMENTS_ROOT`

## Why

- **Portable repo:** The demobuilder git tree holds **pipeline skills** (`skills/`, `docs/`) only. Customer-specific files stay **out of version control** on **local disk** under your user profile.
- **Sharable:** You can push the repo without engagement data; each machine uses the same default layout.

## Convention

| Variable | Meaning |
|----------|---------|
| `DEMOBUILDER_ENGAGEMENTS_ROOT` | Absolute path to a directory that **contains** one folder per engagement |

**Default (when unset):** **`$HOME/engagements`** — a normal folder in your home directory (not a cloud symlink). Agents and scripts should resolve engagement paths as:

`"${DEMOBUILDER_ENGAGEMENTS_ROOT:-$HOME/engagements}/{slug}/"`

Override only if you want engagements somewhere else:

```bash
export DEMOBUILDER_ENGAGEMENTS_ROOT="/custom/path"
```

Engagement workspace for slug `{slug}`:

`$DEMOBUILDER_ENGAGEMENTS_ROOT/{slug}/`

Example: `~/engagements/2026CitizensAI/`

## Setup

1. Create the default root once (optional — tools can create it when needed):

```bash
mkdir -p "$HOME/engagements"
```

2. Optionally pin the variable in `~/.zshrc` / `~/.bashrc` (same default as above, explicit):

```bash
export DEMOBUILDER_ENGAGEMENTS_ROOT="$HOME/engagements"
```

3. Point the agent at **`$DEMOBUILDER_ENGAGEMENTS_ROOT/{slug}`** when running the orchestrator or a single skill.

## Agents

Skills and [`AGENTS.md`](../AGENTS.md): if `DEMOBUILDER_ENGAGEMENTS_ROOT` is unset, use **`$HOME/engagements`** before asking the SA.

## Decision record

See `docs/decisions.md` **D-019** and **D-023**.
