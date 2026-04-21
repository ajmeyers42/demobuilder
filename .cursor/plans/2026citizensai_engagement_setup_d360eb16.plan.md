---
name: 2026CitizensAI engagement setup
overview: Engagement folder `2026CitizensAI/` lives under `$DEMOBUILDER_ENGAGEMENTS_ROOT` (e.g. Google Drive My Drive `demobuilder-engagements/`), not in the git repo — see [docs/engagements-path.md](../../docs/engagements-path.md). Raw discovery, pipeline outputs, and `.env` stay there; the repo holds skills only.
todos:
  - id: mkdir-layout
    content: Ensure $DEMOBUILDER_ENGAGEMENTS_ROOT/2026CitizensAI/ and discovery/ exist
    status: pending
  - id: readme-engagement
    content: README in engagement folder — slug, paths, DEMOBUILDER_ENGAGEMENTS_ROOT
    status: pending
  - id: gitignore-policy
    content: Engagement data outside git — no .gitignore needed for discovery in repo
    status: pending
isProject: false
---

# 2026CitizensAI engagement setup

## Decisions locked in

- **Engagement root:** `$DEMOBUILDER_ENGAGEMENTS_ROOT/2026CitizensAI/` (absolute path; example — Google Drive **My Drive** › `demobuilder-engagements/2026CitizensAI/`).
- **Repo:** Demobuilder clone contains only [`engagements/README.md`](../../engagements/README.md) as a pointer — not customer files.
- **Pipeline slug:** Use lowercase hyphenated slug **`2026citizens-ai`** for generated artifacts (`2026citizens-ai-discovery.json`, etc.). The folder name can stay `2026CitizensAI`.

## Layout (on execution)

1. **Directory layout** under `$DEMOBUILDER_ENGAGEMENTS_ROOT/2026CitizensAI/`
   - Engagement root — point the agent here as workspace.
   - `discovery/` — raw discovery (PDF, `.md`, `.txt`).

2. **Engagement README** — slug `2026citizens-ai`, `export DEMOBUILDER_ENGAGEMENTS_ROOT=...`, run instructions.

3. **Confidentiality** — data is outside git; sync is via Google Drive (or chosen path).

## Loading discovery

- Copy discovery into `$DEMOBUILDER_ENGAGEMENTS_ROOT/2026CitizensAI/discovery/` or pass absolute paths.
- Parser writes `2026citizens-ai-*.json` / `.md` at the engagement root.

## Orchestrator note

Set **`DEMOBUILDER_ENGAGEMENTS_ROOT`** then use workspace **`$DEMOBUILDER_ENGAGEMENTS_ROOT/2026CitizensAI`**. See [skills/demobuilder/SKILL.md](../../skills/demobuilder/SKILL.md).

## Scope not included unless you ask

- No cluster `.env` / `bootstrap.py` deploy unless you request provisioning / deploy.
