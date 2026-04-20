---
name: 2026CitizensAI engagement setup
overview: Create an engagement directory inside the demobuilder git repo at [engagements/2026CitizensAI/](engagements/2026CitizensAI/), add a clear place for raw discovery inputs, align file-prefix slug with pipeline skills, and document how you (or the agent) load notes and run `demo-discovery-parser` / `demobuilder` against that folder.
todos:
  - id: mkdir-layout
    content: Create engagements/2026CitizensAI/ and discovery/ subfolder
    status: pending
  - id: readme-engagement
    content: Add engagements/2026CitizensAI/README.md with slug, paths, and run instructions
    status: pending
  - id: gitignore-policy
    content: Confirm with user or apply .gitignore rule for raw discovery if confidential
    status: pending
isProject: false
---

# 2026CitizensAI engagement setup (Option B)

## Decisions locked in

- **User confirmed (Option B):** Engagement lives **inside this git repo** under `engagements/2026CitizensAI/`, not under `~/demobuilder-workspace/`.
- **Location:** [`engagements/2026CitizensAI/`](engagements/2026CitizensAI/) under the demobuilder repo root.
- **Pipeline slug:** Use lowercase hyphenated slug **`2026citizens-ai`** for all generated artifacts (`2026citizens-ai-discovery.json`, etc.), per [skills/demo-discovery-parser/SKILL.md](skills/demo-discovery-parser/SKILL.md) and [README.md](README.md) — the display folder name can stay `2026CitizensAI` while outputs follow slug convention.

## What will be created (on execution)

1. **Directory layout**
   - `engagements/2026CitizensAI/` — engagement root (this is the “workspace” you point the agent at for this project).
   - `engagements/2026CitizensAI/discovery/` — drop **raw** discovery here (PDF, `.md`, `.txt`). Keeps inputs separate from generated `{slug}-*.json` / `.md` at the engagement root (optional but recommended).

2. **Small engagement README** (inside `engagements/2026CitizensAI/README.md`)
   - States slug `2026citizens-ai`, where to put raw files, and the exact next commands / prompts (e.g. “parse discovery from `discovery/`” or “run demobuilder for this engagement with workspace `…/engagements/2026CitizensAI`”).

3. **Git hygiene**
   - If discovery may contain **customer-confidential** material, ensure those patterns are covered by [`.gitignore`](.gitignore) (e.g. ignore `engagements/*/discovery/**` or specific file types) **only if** you want raw notes never committed; if you intend to version sample/sanitized notes, we skip ignoring or ignore only certain globs. This is a one-line policy choice at execution time.

## Loading discovery documents

- **You:** Copy or save discovery files into `engagements/2026CitizensAI/discovery/` (or tell the agent absolute paths).
- **Agent (later):** Read everything in `discovery/`, run [skills/demo-discovery-parser/SKILL.md](skills/demo-discovery-parser/SKILL.md), write:
  - `2026citizens-ai-discovery.json`
  - `2026citizens-ai-confirmation.md`
  - `2026citizens-ai-gaps.md`  
  in **`engagements/2026CitizensAI/`** (engagement root), matching [README quick-start layout](README.md).

## Orchestrator note

[skills/demobuilder/SKILL.md](skills/demobuilder/SKILL.md) defaults to `~/demobuilder-workspace/{slug}/`; for repo-local engagements, explicitly **set workspace** to `…/demobuilder/engagements/2026CitizensAI` whenever you run the orchestrator so inventory and outputs land in the right folder.

## Scope not included unless you ask

- No cluster `.env`, `bootstrap.py`, or deploy — that is later (`demo-cloud-provision` / `demo-deploy`).
- No automatic parsing in this setup step unless you request “run discovery parser” after files exist.
