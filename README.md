# Memento-Skills

A Claude Code plugin that lets your agent **learn from experience**. Based on the Read-Write Reflective Learning framework from the [Memento-Skills paper](https://arxiv.org/abs/2603.18743).

> *"If the model weights are frozen, all adaptation must come from the input — the prompt, the context, or in our case, the memory."*

## What This Does

Every time you complete a task with Claude Code, patterns emerge — ways of handling API errors, debugging strategies, project-specific workflows. Normally, these patterns vanish when the session ends.

Memento-Skills captures them as **learned skills** — structured markdown files that persist across sessions. Each skill tracks its own utility score, so the system knows which patterns actually work and which need improvement.

```
Session Start                          Session End
    │                                      │
    ▼                                      ▼
 ┌──────────┐    ┌─────────┐    ┌──────────────────┐
 │ Read     │───▶│  Act    │───▶│ Reflect (Write)  │
 │ (hook)   │    │ (task)  │    │ /memento reflect │
 └──────────┘    └─────────┘    └────────┬─────────┘
 Load metrics                            │
 + skill table                    ┌──────┴──────┐
                                  │             │
                              Update        Create new
                              metrics       learned skill
                                  │             │
                                  ▼             ▼
                              metrics.json   SKILL.md
                                  │             │
                                  └──────┬──────┘
                                         │
                                  Next session's
                                  hook loads these
```

## The Paper

**[Memento-Skills: Let Agents Design Agents](https://arxiv.org/abs/2603.18743)** (Zhou et al., 2026)

The paper introduces a **self-evolving agent system** built on the Stateful Reflective Decision Process (SRDP). The key insight: if you treat reusable skills (stored as structured markdown) as external memory, a frozen LLM can continuously improve without parameter updates.

### Core Concepts

| Concept | Paper | This Plugin |
|---------|-------|-------------|
| **Skill Memory** | Markdown files encoding behavior + context | `learned-*/SKILL.md` files |
| **Read Phase** | Retrieve relevant skill via trained router | SessionStart hook injects skill table with utility scores |
| **Write Phase** | Reflect on outcome, update skill library | `/memento reflect` — self-evaluate + update metrics |
| **Skill Router** | Contrastive embedding model (InfoNCE + offline RL) | Claude's natural language matching + utility scores |
| **Utility Tracking** | `U(skill) = n_success / (n_success + n_failure)` | `metrics.json` with per-skill scoring |
| **Skill Discovery** | Auto-create when no skill matches | `/memento create` |
| **Skill Optimization** | Failure attribution + file-level rewriting | `/memento optimize` |
| **Convergence** | Proven under KL-regularised soft policy iteration | Heuristic (utility trends over time) |

### Results from the Paper

Starting from just **5 atomic skills**, the system:
- Grew to **235 learned skills** on Humanity's Last Exam
- Achieved **26.2%** relative improvement on GAIA (General AI Assistants)
- Achieved **116.2%** relative improvement on HLE (Humanity's Last Exam)
- All without updating a single model parameter

## Installation

### From GitHub (recommended)

```bash
# 1. Add the marketplace
claude plugins marketplace add tmdgusya/momento

# 2. Install the plugin
claude plugins install memento-skills@momento

# 3. Restart Claude Code to activate
```

### Verify installation

```bash
claude plugins list
# Should show: memento-skills@momento  Version: 0.1.0  Status: ✔ enabled
```

### Uninstall

```bash
claude plugins uninstall memento-skills@momento
claude plugins marketplace remove momento
```

### Requirements

- [Claude Code CLI](https://docs.anthropic.com/en/docs/claude-code)
- `jq` recommended for full metrics table in session hook (falls back to basic mode without it)
- `bc` for utility threshold comparisons

## Usage

### `/memento reflect` — Learn from What You Just Did

After completing a task, reflect on what worked:

```
> /memento reflect

Analyzing this session...

Task: Implemented retry logic for API timeout handling
Self-evaluation: SUCCESS — completed in one pass, user accepted without corrections

Pattern discovered: "API retry with exponential backoff + circuit breaker"
This pattern required non-obvious domain knowledge and would help future tasks.

Want me to capture this as a learned skill? I'll use /memento create.
```

### `/memento create` — Capture a Reusable Pattern

```
> /memento create

Should this be a global skill (all projects) or project skill (this project only)?
> global

Created: ~/.claude/skills/learned-api-retry-with-backoff/SKILL.md
Registered in: ~/.claude/memento/metrics.json (utility: null, 0 uses)
```

### `/memento stats` — See Your Skill Dashboard

```
> /memento stats

Overview: 4 global, 2 project skills | Avg utility: 0.76

| Skill                        | Utility    | Uses | Scope   |
|------------------------------|------------|------|---------|
| learned-api-retry-with-backoff | 0.90 (9/10) | 10 | global  |
| learned-db-migration-safety  | 0.83 (5/6)  | 6  | project |
| learned-react-form-validation | 0.67 (4/6) | 6  | global  |
| learned-css-grid-layout      | ⚠ 0.33 (1/3) | 3 | global  |

⚠ learned-css-grid-layout: utility below threshold (0.33 < 0.40)
  → /memento optimize learned-css-grid-layout
```

### `/memento optimize` — Improve a Failing Skill

```
> /memento optimize learned-css-grid-layout

Analyzing failure log...
- Failed on: "responsive dashboard layout" (used flexbox where grid was needed)
- Failed on: "nested grid with auto-fill" (missing minmax() in template)

Revising SKILL.md:
- Added step for choosing grid vs flexbox
- Added Known Failure Mode: "nested grids need explicit minmax()"
- Updated utility summary

Show diff? [y/n]
```

### `/memento prune` — Clean Up Low-Value Skills

```
> /memento prune

Prune candidates:

1. learned-old-workaround (global)
   Utility: 0.25 (1/4) — below threshold 0.40
   Last used: 2026-01-15 — 69 days ago

Delete? [y/n/choose individually]
```

## How It Works

### Metrics System

Each learned skill tracks its performance in `metrics.json`:

```json
{
  "learned-api-retry-with-backoff": {
    "scope": "global",
    "usage_count": 10,
    "success_count": 9,
    "failure_count": 1,
    "utility": 0.9,
    "trigger_log": [
      {"task": "retry logic for payment API", "result": "success", "date": "2026-03-25"}
    ]
  }
}
```

**Utility formula:** `U(skill) = success / (success + failure)`

| Condition | Action |
|-----------|--------|
| `utility = null` (< 3 uses) | Keep using, gather data |
| `utility >= 0.4` | Healthy — keep |
| `utility < 0.4` (3+ uses) | Optimize candidate |
| Unused for 60+ days | Prune candidate |

### SessionStart Hook (Read Phase)

Every session, the hook automatically:
1. Reads `~/.claude/memento/metrics.json` (global) and `.claude/memento/metrics.json` (project)
2. Generates a summary table with utility scores
3. Injects into the system prompt so Claude knows which learned skills are available

### Self-Evaluation (Judge)

Claude evaluates its own performance using these signals:
- Did the user accept the result without corrections?
- Was it completed in one pass or with retries?
- Were there errors during execution?
- Does the final result meet the original request?

## Plugin Structure

```
memento-skills/
├── .claude-plugin/
│   └── plugin.json              # Plugin metadata
├── hooks/
│   ├── hooks.json               # SessionStart hook registration
│   ├── run-hook.cmd             # Cross-platform runner
│   └── session-start            # Read phase implementation
├── skills/
│   ├── memento-reflect/SKILL.md # Self-evaluate + update metrics
│   ├── memento-create/SKILL.md  # Generate new learned skill
│   ├── memento-optimize/SKILL.md # Refine failing skill
│   ├── memento-stats/SKILL.md   # Metrics dashboard
│   └── memento-prune/SKILL.md   # Remove low-value skills
└── templates/
    └── learned-skill.md         # Template for new skills
```

## Differences from the Paper

| Aspect | Paper | This Plugin |
|--------|-------|-------------|
| Learning loop | Fully automatic | Semi-automatic (user triggers `/memento reflect`) |
| Router | Trained embedding model (InfoNCE + RL) | Claude's semantic matching + utility scores |
| Judge | Automatic with ground-truth answers | Self-evaluation by Claude |
| Convergence | Mathematically proven (Theorem 1.3) | Heuristic (utility trends) |
| Scale | Hundreds of skills | Tens of skills (practical for individual use) |

**Advantages of this approach:**
- Human-in-the-loop quality control
- Zero training cost — works immediately
- Leverages Claude's reasoning for richer reflection than automated judges
- Integrates with existing Claude Code skill ecosystem

## Roadmap

- [ ] Embedding-based skill router (MCP server)
- [ ] Subagent auto-reflection after task completion
- [ ] Cross-project skill sync
- [ ] Skill composition (chaining multiple skills)
- [ ] Community skill marketplace integration

## References

- **Paper:** [Memento-Skills: Let Agents Design Agents](https://arxiv.org/abs/2603.18743) — Zhou et al., 2026
- **Original Implementation:** [Memento-Teams/Memento-Skills](https://github.com/Memento-Teams/Memento-Skills)
- **Theory:** [Memento 2 — SRDP Framework](https://arxiv.org/abs/2603.15566)

## License

MIT
