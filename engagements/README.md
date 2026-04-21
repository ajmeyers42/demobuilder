# Engagement workspaces (outside this repository)

Per-customer demo folders — credentials (`.env`), generated scripts, discovery notes, and pipeline outputs — **do not live under this git clone**. They live under your **user profile** by default.

## Default location

**`~/engagements/`** — one subfolder per engagement (e.g. `~/engagements/2026CitizensAI/`).

The environment variable **`DEMOBUILDER_ENGAGEMENTS_ROOT`** points at that parent directory. If it is **unset**, agents and docs assume:

```bash
export DEMOBUILDER_ENGAGEMENTS_ROOT="$HOME/engagements"
```

Add that line to `~/.zshrc` or `~/.bashrc` if you want it explicit in every shell.

To use a different root:

```bash
export DEMOBUILDER_ENGAGEMENTS_ROOT="/path/to/your/engagements-parent"
```

Each engagement is: `$DEMOBUILDER_ENGAGEMENTS_ROOT/{slug}/`

See **[`docs/engagements-path.md`](../docs/engagements-path.md)** for full detail.
