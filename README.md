# demobuilder

Elastic demobuilder — a reusable agent pipeline for turning customer discovery into working demos.

## What This Is

A collection of skills that take a sales engagement from raw discovery notes through to a built, deployed, and validated Elastic demo. Each skill is a discrete step in the pipeline; they're designed to be used independently or chained together.

## Pipeline (planned)

| Skill | Input | Output | Status |
|-------|-------|--------|--------|
| `demo-discovery-parser` | Discovery notes (PDF, markdown, raw text) | `{slug}-discovery.json`, `{slug}-confirmation.md`, `{slug}-gaps.md` | ✅ v1 |
| `demo-diagnostic-analyzer` | Elastic diagnostic files | Current-state architecture, gap findings | Planned |
| `demo-platform-audit` | Discovery JSON + diagnostic output | Feature feasibility report, version/tier gaps | Planned |
| `demo-script-template` | Discovery JSON | Structured demo script with scenes, timing, talking points | Planned |
| `demo-data-modeler` | Demo script | Index mappings, sample data, ingest pipelines | Planned |
| `demo-ml-designer` | Demo script + data model | ML job configs, anomaly injection plan | Planned |
| `demo-validator` | Built demo artifacts | Pre-demo checklist, known gaps, fallback plan | Planned |

## Skills

### `demo-discovery-parser`

Parses sales discovery notes — in any format or quality — into three structured outputs:
1. **`{slug}-discovery.json`** — structured customer profile for downstream pipeline skills
2. **`{slug}-confirmation.md`** — customer-facing confirmation document (no internal language, no competitor mentions)
3. **`{slug}-gaps.md`** — internal gap report with specific follow-up questions

Handles polished post-call reports, raw live meeting notes with typos, technical spec dumps, and mixed prep+live notes. Tested against Citizens Bank, IHG Club Vacations, Thermo Fisher, and Lowe's discovery artifacts.

## Usage

Skills are loaded into Claude Code via the skills plugin. Each `SKILL.md` documents its own trigger phrases and workflow.

## Repo Structure

```
demobuilder/
├── README.md
└── skills/
    └── demo-discovery-parser/
        ├── SKILL.md
        └── evals/
            └── evals.json
```
