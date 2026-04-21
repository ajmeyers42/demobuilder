# Engagement workspaces (outside this repository)

Per-customer demo folders — credentials (`.env`), generated scripts, discovery notes, and pipeline outputs — **do not live under this git clone**. They sit on a path you choose (for example **Google Drive › My Drive** so demos sync and stay off shared git remotes).

## Configure once per machine

Set an absolute path:

```bash
export DEMOBUILDER_ENGAGEMENTS_ROOT="/path/to/your/demobuilder-engagements"
```

**Example (Google Drive on macOS, “My Drive”):**

```bash
export DEMOBUILDER_ENGAGEMENTS_ROOT="$HOME/Library/CloudStorage/GoogleDrive-YOUR_ACCOUNT/My Drive/demobuilder-engagements"
```

If you use the Finder “Google Drive” shortcut:

```bash
export DEMOBUILDER_ENGAGEMENTS_ROOT="$HOME/Google Drive/demobuilder-engagements"
```

Add the same line to `~/.zshrc` or `~/.bashrc` so agents and terminals resolve engagement paths consistently.

Each engagement is a subfolder: `$DEMOBUILDER_ENGAGEMENTS_ROOT/{slug}/` (e.g. `2026CitizensAI/`).

See **[`docs/engagements-path.md`](../docs/engagements-path.md)** for rationale and agent behavior.
