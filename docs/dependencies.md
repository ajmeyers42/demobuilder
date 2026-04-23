# Dependencies

Demobuilder has two required external repositories and one optional environment variable.

## elastic/agent-skills

**https://github.com/elastic/agent-skills**

The full plugin must be installed — not a Search-only subset. Security and hybrid demos require the Security skills; Observability scenes require the Observability skills. Install everything once and all engagements work.

```bash
npx skills add elastic/agent-skills
```

Representative skills used by demobuilder (non-exhaustive):

| Area | Skills | Purpose |
|---|---|---|
| Cloud | `cloud/setup`, `cloud/create-project`, `cloud/manage-project`, `cloud/network-security` | EC_API_KEY setup, provision, teardown, traffic filters |
| Kibana | `kibana/kibana-dashboards`, `kibana/kibana-connectors`, `kibana/kibana-alerting-rules`, `kibana/streams` | Deploy, dashboards, Workflows connectors, alerting |
| Observability | `observability/manage-slos`, `observability/logs-search`, `observability/service-health` | SLOs, log analysis, APM health when in scope |
| **Security** | `security/detection-rule-management`, `security/alert-triage`, `security/case-management`, `security/generate-security-sample-data` | Detection rules, triage, cases, sample data |
| Elasticsearch | `elasticsearch/elasticsearch-esql`, `elasticsearch/elasticsearch-authz` | ES\|QL queries, RBAC |

Run `cloud-setup` once to configure `EC_API_KEY` before using `demo-cloud-provision`.

If `elastic/agent-skills` is not installed, the assistant will say so clearly rather than fail silently. See [docs/todo.md](todo.md) for the full one-time setup checklist.

---

## elastic/hive-mind

**https://github.com/elastic/hive-mind**

A reference library for Elastic integration patterns, SA coaching frameworks, and demo tooling. Several demobuilder pipeline stages read hive-mind patterns directly — SA ideation archetypes, data fidelity guidelines, token optimization strategies, Kibana Workflows references, and dashboard construction patterns.

### Install

Clone as a **sibling of demobuilder** (same parent directory):

```bash
git clone https://github.com/elastic/hive-mind ../hive-mind
```

The skill symlinks in `.cursor/skills/`, `.claude/skills/`, and `.agents/skills/` point to `../hive-mind/skills/` using relative paths. As long as hive-mind is a sibling, they resolve automatically.

### Keeping it current

```bash
cd ../hive-mind && git pull
```

No re-linking needed. The orchestrator's Step 0 currency check will warn if either repo is behind its remote.

### What's linked

Seven hive-mind skills are pre-symlinked into demobuilder's agent skill directories:

| Skill | Used for |
|---|---|
| `hive-sa-coaching` | SA ideation (`demo-ideation` stage) — Demo Archetypes, COACHING_CONVERSATION framework |
| `hive-token-optimization` | Token visibility dashboard, AI spend tracking (`token-visibility` skill) |
| `hive-workflows` | Kibana Workflows API reference, YAML step types, troubleshooting |
| `hive-dashboards` | Dashboard construction patterns, NDJSON format, Lens panel reference |
| `hive-demo-data` | Data fidelity guidelines, dataset generation, LLM data quality |
| `hive-demo-recipes` | Composite demo guides for search, Agent Builder, and e-commerce scenarios |
| `hive-elastic-agent-skills` | Elastic Agent Skills integration (`npx skills`, agentskills.io) |

### If you move hive-mind

If you need hive-mind at a different path, re-run the symlink setup from the demobuilder root:

```bash
# Remove old links
for dir in .cursor/skills .claude/skills .agents/skills; do
  for skill in hive-sa-coaching hive-token-optimization hive-workflows hive-dashboards hive-demo-data hive-demo-recipes hive-elastic-agent-skills; do
    rm -f "$dir/$skill"
  done
done

# Re-link from new location (adjust path as needed)
HIVE_MIND=/path/to/hive-mind
for skill in hive-sa-coaching hive-token-optimization hive-workflows hive-dashboards hive-demo-data hive-demo-recipes hive-elastic-agent-skills; do
  for dir in .cursor/skills .claude/skills .agents/skills; do
    ln -s "$(python3 -c "import os; print(os.path.relpath('$HIVE_MIND/skills/$skill', '$dir'))")" "$dir/$skill"
  done
done
```

---

## DEMOBUILDER_ENGAGEMENTS_ROOT (optional)

By default, engagement workspaces are written to `~/engagements/{slug}/`. Set this env var in your shell profile to use a different parent directory:

```bash
export DEMOBUILDER_ENGAGEMENTS_ROOT="/Volumes/work/engagements"
```

See [docs/engagements-path.md](engagements-path.md) for details and multi-customer isolation patterns.
