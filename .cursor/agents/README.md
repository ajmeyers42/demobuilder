# Cursor subagent wrappers (loom)

Thin project agents under this directory **point at** portable loom skills — they do not
duplicate procedure text. Skills under `skills/*/SKILL.md` remain the source of truth for
Cursor, Claude, and Gem.

| Agent file | Skill | Use when |
|------------|-------|----------|
| `loom-warp-listen.md` | `skills/warp-listen` | Parse discovery notes |
| `loom-warp-scan.md` | `skills/warp-scan` | Parse diagnostic ZIP |
| `loom-thread-audit.md` | `skills/thread-audit` | Platform feasibility matrix |
| `loom-weave-fleet.md` | `skills/weave-fleet` | Fleet/EPM integrations manifest |
| `loom-weave-train.md` | `skills/weave-train` | ML anomaly / NLP config |
| `loom-finish-check.md` | `skills/finish-check` | Pre-demo checklist |
| `loom-finish-verify.md` | `skills/finish-verify` | Asset bundle / Agent A |
| `loom-bolt-bootstrap.md` | `skills/bolt-bootstrap` | Terraform + bootstrap-data codegen / Agent B |

Full tiers and when **not** to use subagents: [`docs/agent-patterns.md`](../../docs/agent-patterns.md).

**wind-pulse:** prefer `skills/wind-pulse/wind_pulse.py` (CLI) before any AI worker.
