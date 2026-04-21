---
name: 2026CitizensAI engagement setup
overview: Engagement folder `2026CitizensAI/` lives under `$DEMOBUILDER_ENGAGEMENTS_ROOT` (default `$HOME/engagements`), not in the git repo — see [docs/engagements-path.md](../../docs/engagements-path.md).
todos:
  - id: mkdir-layout
    content: Ensure $HOME/engagements/2026CitizensAI/ and discovery/ exist
    status: pending
  - id: readme-engagement
    content: README in engagement folder — slug, paths, default ~/engagements
    status: pending
  - id: gitignore-policy
    content: Engagement data outside git — no repo .gitignore for discovery
    status: pending
isProject: false
---

# 2026CitizensAI engagement setup

## Decisions locked in

- **Engagement root:** `$DEMOBUILDER_ENGAGEMENTS_ROOT/2026CitizensAI/` — default **`~/engagements/2026CitizensAI/`** when the env var is unset.
- **Repo:** Demobuilder clone contains only [`engagements/README.md`](../../engagements/README.md) as a pointer — not customer files.
- **Pipeline slug:** **`2026citizens-ai`** for generated artifacts. Folder name can stay `2026CitizensAI`.

## Layout

1. **`~/engagements/2026CitizensAI/`** — workspace; `discovery/` for raw notes.

2. **Engagement README** — slug `2026citizens-ai`, default path note.

## Orchestrator note

Workspace = **`${DEMOBUILDER_ENGAGEMENTS_ROOT:-$HOME/engagements}/2026CitizensAI`**. See [skills/demobuilder/SKILL.md](../../skills/demobuilder/SKILL.md).

## Scope not included unless you ask

- No cluster `.env` / `bootstrap.py` deploy unless you request provisioning / deploy.
