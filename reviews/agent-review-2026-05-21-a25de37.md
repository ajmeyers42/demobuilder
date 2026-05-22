# Agent Codebase Review — loom
**Date:** 2026-05-21 · **Agent version:** a25de37 · **Review version:** v1.0  
**Frameworks detected:** Cursor (high), Claude Code (high), Kibana Agent Builder (high)  
**Overall:** 🟡 Amber

---

## Dimension scores

| # | Dimension | Score | Findings |
|---|---|---|---|
| 1 | Token efficiency | 🟡 A | F006 |
| 2 | Cost & model selection | 🟡 A | F003, F010 |
| 3 | Architecture & decomposition | 🟡 A | F002, F004, F009, F013 |
| 4 | Subagent / task spawning | 🟢 G | — |
| 5 | UX / DX | 🟡 A | F008, F012 |
| 6 | Interoperability | 🟡 A | F005, F015 |
| 7 | Telemetry & observability | 🔴 R | F001, F007 |
| 8 | Security & privacy | 🟡 A | F011, F014 |
| 9 | Self-improvement | 🟡 A | F001, F007 |

---

## P1 findings (high)

### F001 — No eval runner: 17 evals.json files, zero automated execution
**Dimension:** Telemetry & observability / Self-improvement  
**Risk:** `safe`

Every planning skill (warp-listen, warp-spark, weave-script, weave-model, etc.) has a well-structured `evals.json` with expected outputs and assertions. None of them run automatically. A SKILL.md edit — including today's changes to thread-qualify, weave-script, weave-model, and weave-agent — could regress behavior with no detection signal.

**Citations:** `skills/*/evals/evals.json` (17 files)

**Recommendation:** Add `scripts/run_evals.py` that iterates every `skills/*/evals/evals.json`, runs the prompts against the skill (or logs them for LLM-eval review), and reports pass/fail against the assertions array. Even a dry-run mode that confirms eval JSON schema validity would catch structural breaks immediately.

---

### F002 — bolt-launch deprecated but still registered as an active skill
**Dimension:** Architecture & decomposition  
**Risk:** `safe`

`skills/bolt-launch/SKILL.md` is marked "DEPRECATED as of 2026-05-05. Use finish-verify → bolt-bootstrap instead." However, it remains symlinked into `.claude/skills/`, appears in the agent skills list, and the loom orchestrator still references it by name in the deploy approval gate text ("Stage 8b (finish-verify)… Stage 9 (bolt-bootstrap)" alongside older "bolt-launch" language). An SA triggering it directly would get the deprecated behavior.

**Citations:** `skills/bolt-launch/SKILL.md:1-10`, `skills/loom/SKILL.md:55-60`

**Recommendation:** Update the bolt-launch SKILL.md header to immediately redirect: "This skill is deprecated — read `skills/finish-verify/SKILL.md` instead. Do not execute further." Remove its symlink from `.claude/skills/` (or keep it as a redirect-only stub). Update loom/SKILL.md Stage 8 language to remove all bolt-launch references.

---

### F003 — No token/cost tracking on loom pipeline runs
**Dimension:** Cost & model selection  
**Risk:** `safe`

`weave-cost` provides excellent AI cost visibility for the *customer's* demo environment (Agent Builder sessions). There is no equivalent for the loom pipeline itself. An SA running the full pipeline (warp-listen → weave-script → weave-model → finish-verify → bolt-bootstrap) has no idea whether the session cost $0.40 or $4.00. The hive-mind token optimization skill is listed as a routing target in AGENTS.md but is never invoked by the pipeline.

**Citations:** `AGENTS.md:hive-token-optimization routing entry`, `skills/weave-cost/SKILL.md`

**Recommendation:** Add a Step 0 note to `skills/loom/SKILL.md` directing the SA to `hive-token-optimization` for pipeline cost tracking setup — or integrate the Group B ES Analytics pattern so pipeline session costs are indexed alongside demo session costs. This is a one-time SA tooling setup, not per-engagement.

---

### F004 — bolt-eck silently falls back to ECH without SA warning
**Dimension:** Architecture & decomposition  
**Risk:** `risky`

`skills/bolt-eck/SKILL.md` is marked "PLACEHOLDER — ECK support is not yet implemented. This skill will route ECK deployments to the ECH variant with documented exceptions." bolt-bootstrap routes to it when `DEPLOYMENT_TYPE=eck`. An SA who sets `DEPLOYMENT_TYPE=eck` (for a genuine Kubernetes on-prem requirement) will get an ECH deployment silently — potentially deploying to the wrong cluster type with the wrong API shapes.

**Citations:** `skills/bolt-eck/SKILL.md:1-8`, `skills/bolt-bootstrap/SKILL.md` routing table

**Recommendation:** Update bolt-eck/SKILL.md to halt execution immediately with a visible warning block:
```
⛔ ECK deployment type is not yet implemented.
   DEPLOYMENT_TYPE=eck was set in .env.
   If you intended ECH, update .env to DEPLOYMENT_TYPE=ech and re-run.
   If you genuinely need ECK, this is a known gap — contact the loom maintainers.
```
Never silently fall through to ECH for a different deployment type.

---

### F005 — warp-discovery sync has no drift detection
**Dimension:** Interoperability  
**Risk:** `safe`

`skills/warp-discovery/components.md` documents a manual re-sync checklist: after thread-qualify or warp-listen changes, update warp-discovery/SKILL.md and recompile the Gem system prompt. Today's session caught the drift manually and fixed it. But there's no automated signal — a thread-qualify change in a future session could go unsynced for weeks with no indication.

**Citations:** `skills/warp-discovery/components.md:32-56`

**Recommendation:** Add a Step 0 check to `skills/loom/SKILL.md` (alongside the existing `git fetch` currency check) that reads the `Last synced` dates from `warp-discovery/components.md` and compares them to the `git log` dates of the upstream skills. If a synced skill was modified after its last-sync date, surface:
```
⚠️ warp-discovery may be out of sync with thread-qualify (last synced: 2026-05-21, last modified: {date}).
   Run the re-sync checklist in skills/warp-discovery/components.md before delivering to Gem.
```

---

## P2 findings (medium)

### F006 — Orchestrator preamble duplicates decisions.md prose verbatim
**Dimension:** Token efficiency  
**Risk:** `safe`

`skills/loom/SKILL.md` contains ~80 lines of inline policy prose that duplicates content from `docs/decisions.md` (D-024, D-025, D-026, D-033, D-036, D-043, D-044, etc.) and AGENTS.md. Every full pipeline invocation loads all of it, even when the SA is only running a single stage. The decisions.md rule (D-042) says reference files win over SKILL.md prose — but the prose is also in SKILL.md.

**Citations:** `skills/loom/SKILL.md:55-90` (deploy approval, tagging, version scope, deployability — all verbatim from docs/decisions.md)

**Recommendation:** Replace inline policy summaries in loom/SKILL.md with one-line pointers: "Deploy approval: see D-024. Tagging: see D-026. Deployability: see D-025." Keep the behavioral instruction (what the orchestrator *does*), remove the rationale (why the policy exists — that lives in decisions.md). Estimated reduction: ~50 lines from the 772-line orchestrator.

---

### F007 — Evals for thread-qualify, weave-script, weave-model don't cover today's additions
**Dimension:** Self-improvement  
**Risk:** `safe`

Today's changes added Customer Overview and Opportunity Overview (with pain_points and success_goals tables) to thread-qualify, added opportunity-profile.json as an optional input to weave-script with a new close-scene template, and added Step 1b KPI field pass to weave-model. None of the existing evals assert these new behaviors.

**Citations:** `skills/thread-qualify/evals/evals.json`, `skills/weave-script/evals/evals.json`, `skills/weave-model/evals/evals.json`

**Recommendation:** Add at least one eval case per modified skill asserting: (a) thread-qualify produces `customer_overview` and `opportunity_overview.success_goals` in the profile JSON; (b) weave-script value confirmation close references a `success_goals` entry when the profile is present; (c) weave-model Step 1b adds `success_metric_fields` when a quantified goal exists.

---

### F008 — wind_pulse.py has no configurable request timeout
**Dimension:** UX / DX  
**Risk:** `safe`

`wind_pulse.py` uses `urllib.request` for all HTTP calls with no explicit timeout argument. On a slow or unreachable cluster, each call will block at the OS TCP timeout (often 60–120 seconds). A pre-demo pulse check that takes 3 minutes to fail is worse than one that fails in 10 seconds with a clear error.

**Citations:** `skills/wind-pulse/wind_pulse.py` — all `urllib.request.urlopen()` calls

**Recommendation:** Add `PULSE_TIMEOUT_SECS = int(os.environ.get("PULSE_TIMEOUT_SECS", "10"))` near the top of the file, and pass `timeout=PULSE_TIMEOUT_SECS` to every `urllib.request.urlopen()` call. Catch `urllib.error.URLError` with a timeout message that names the endpoint.

---

### F009 — Pipeline-wide reference files are namespaced under a deprecated skill
**Dimension:** Architecture & decomposition  
**Risk:** `needs-review`

`skills/bolt-launch/references/` contains pipeline-wide constants used by finish-verify, bolt-ech, bolt-serverless, and bolt-bootstrap: `inference-config.md`, `env-reference.md`, `terraform-patterns.md`, `feature-compatibility.md`, `pipeline-constants.md`, etc. These are not bolt-launch-specific — they're shared infrastructure. Having them under a deprecated skill's directory is confusing and creates hesitation about whether they're still maintained.

**Citations:** `skills/bolt-launch/references/` (10 reference files), cross-referenced from `skills/finish-verify/SKILL.md`, `skills/bolt-ech/SKILL.md`, `skills/bolt-serverless/SKILL.md`

**Recommendation:** Move `skills/bolt-launch/references/` to `skills/references/` and update all `../bolt-launch/references/` paths in finish-verify, bolt-ech, bolt-serverless, bolt-bootstrap, and weave-model. The bolt-launch SKILL.md can stay as a deprecated redirect. This is a path rename only — no content changes needed.

---

### F010 — No model-tiering guidance across pipeline stages
**Dimension:** Cost & model selection  
**Risk:** `needs-review`

The pipeline spans tasks with very different complexity and quality requirements. Some stages (warp-listen structured extraction, wind-pulse health formatting, bolt-bootstrap template generation) could run on a faster/cheaper model with no quality loss. Others (weave-script narrative generation, warp-spark ideation, thread-qualify MEDDPIC analysis) benefit from the best available model. There's no guidance — every stage uses whatever model the SA happens to have open.

**Citations:** `skills/loom/SKILL.md` stage table — no model annotations

**Recommendation:** Add a `Model tier` column to the loom orchestrator stage table:

| Stage | Skills | Model tier |
|---|---|---|
| 0 | warp-spark | Best — creative, high-stakes |
| 1 | warp-listen | Fast — structured extraction |
| 2a | warp-scan | Fast — structured extraction |
| 2b | thread-qualify | Best — analytical, judgment calls |
| 3 | thread-audit | Fast — lookup and comparison |
| 4 | thread-suggest | Fast — catalog lookup |
| 5 | weave-script | Best — narrative, customer-facing |
| 6 | weave-model | Best — technical accuracy critical |
| 7 | weave-train / weave-agent | Best — API shape correctness critical |
| 8 | finish-verify | Fast — probe and template |
| 9 | bolt-bootstrap | Fast — template generation |

---

### F011 — Eval fixtures use real-sounding customer names
**Dimension:** Security & privacy  
**Risk:** `needs-review`

`skills/warp-listen/evals/evals.json` references engagement names including Citizens Bank, IHG Club Vacations, Thermo Fisher Scientific, and Lowe's — with detailed pain points, contact names, infrastructure specs, and meeting context. If these are based on real engagements, this data is in the public loom git repo (or at minimum in any fork/clone). If they're illustrative composites, they should be clearly labeled as such.

**Citations:** `skills/warp-listen/evals/evals.json:1-75`

**Recommendation:** Add a comment at the top of each eval file: `// All customer names, contacts, and details in this file are illustrative composites — not real engagements.` If any eval data is based on a real engagement, anonymize it.

---

## P3 findings (nice-to-have)

### F012 — No mid-pipeline progress report from loom orchestrator
**Dimension:** UX / DX  
**Risk:** `safe`

After completing stages 1–4, the SA has no "you're here in the pipeline" status output from the orchestrator. warp-scout has an excellent terminal summary format. The full loom pipeline ends each stage silently before proceeding.

**Recommendation:** Add a one-line stage-completion banner to loom/SKILL.md after each stage: `✅ Stage N complete — {artifact(s) written}. Proceeding to Stage N+1.`

---

### F013 — weave-pipe and wind-streams are undeveloped stubs
**Dimension:** Architecture & decomposition  
**Risk:** `safe`

Both skills have skeleton SKILL.md files (< 30 lines, no step structure). They appear in the skills listing and could be triggered by the orchestrator under the right conditions.

**Recommendation:** Add `status: planned` to the YAML frontmatter and a clear "Not yet implemented" header to both SKILL.md files so an SA who triggers them gets an explicit message rather than a confusing empty result.

---

### F014 — No data retention guidance for engagement directories
**Dimension:** Security & privacy  
**Risk:** `safe`

Customer discovery notes (discovery.json, opportunity-summary.md, confirmation.md) are written to `~/engagements/{slug}/` with no guidance on how long to retain them, whether to encrypt at rest, or how to clean up after the engagement ends. There's no `.gitignore` guidance to prevent accidental commits if engagements/ ends up inside a tracked directory.

**Recommendation:** Add a "Engagement data hygiene" section to AGENTS.md covering: (1) keep engagements/ outside any git-tracked directory, (2) delete engagement directories after deal closure per your organization's data retention policy, (3) never commit .env files.

---

### F015 — External dependencies (hive-mind, elastic/agent-skills) are unpinned
**Dimension:** Interoperability  
**Risk:** `needs-review`

Both `../hive-mind` and `elastic/agent-skills` are referenced by local path / latest-clone with no version pinning. A breaking change in either (renamed pattern file, changed skill API, removed tool) could break loom skills silently on the next pull.

**Recommendation:** Add a `docs/dependency-pins.md` recording the git SHA or version of each external dependency at the time it was last validated against loom. Reference it from `docs/decisions.md` as a new D-NNN entry.

---

## Best practices applied this run

- `elastic/agent-skills` source: last checked 2026-04-30 (21 days ago, below 30-day threshold — no refresh needed)
- Cursor skill architecture patterns applied: skill body length, reference file separation, frontmatter contract
- GenAI semconv applied in telemetry recommendations (Step 9)

---

## Telemetry plan

See Step 9 output: `reviews/telemetry-scaffold/` — OTel-first scaffold with Elastic OTLP endpoint.

---

## Changes applied this session

| Finding | Status | What was done |
|---|---|---|
| F001 | ✅ Applied | `scripts/run_evals.py` created — schema validation + inventory of all 45 evals |
| F002 | ✅ Applied | `bolt-launch/SKILL.md` made redirect-only stub |
| F003 | ✅ Applied | Step 0 cost-tracking pointer added to loom/SKILL.md |
| F004 | ✅ Applied (modified) | bolt-eck halts with explicit message; README documents ECH/Serverless support, ECK on roadmap |
| F005 | ✅ Applied | warp-discovery drift detection check added to loom Step 0 |
| F006 | ✅ Applied | 60 lines of duplicate policy prose removed from loom/SKILL.md |
| F007 | ✅ Applied | New eval cases added to thread-qualify (3), weave-script (4), weave-model (3) |
| F008 | ✅ Applied | `PULSE_TIMEOUT_SECS` env var added to wind_pulse.py (default 10s, was hardcoded 90s) |
| F009 | ✅ Applied | `bolt-launch/references/` copied to `skills/references/`; all 9 SKILL.md files updated |
| F010 | ✅ Applied | Model tier guidance table added to loom/SKILL.md stage table |
| F011 | ✅ Applied (modified) | All evals sanitized: customer names → synthetic composites; `_comment` field added |
| F012 | ✅ Applied | Stage completion banner pattern updated in loom/SKILL.md Step 4 |
| F013 | ✅ Applied | `weave-pipe` and `wind-streams` now have YAML frontmatter with `status: planned` |
| F014 | ✅ Applied | "Engagement data hygiene" section added to AGENTS.md (5 rules) |
| F015 | ⏳ Design proposed | `ext-registry` pattern described; awaiting SA approval to implement |

## Telemetry scaffold note

Pipeline-level telemetry: `weave-cost/SKILL.md` already documents the Group B ES Analytics pattern
(SA-side cost tracking into `agent-sessions` index). No additional scaffold needed in the loom repo.
SA can invoke `weave-cost` for their own pipeline session cost visibility.

## Comparison to last review

No prior `agent-review-*.json` found in `reviews/` — this is the first review run.
