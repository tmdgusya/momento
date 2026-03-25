---
name: memento-stats
description: Use when the user wants to see an overview of their learned skills, check utility scores, or identify skills that need optimization or pruning
---

# Memento Stats

Display a dashboard of learned skill metrics.

## Protocol

### 1. Load Metrics

Read both:
- `~/.claude/memento/metrics.json` (global)
- `.claude/memento/metrics.json` (project, if exists)

### 2. Generate Report

Present a formatted summary:

**Overview:**
- Total skills: {N} global, {M} project
- Average utility: {avg} (excluding null)
- Skills with insufficient data: {count with utility=null}

**Full Table (sorted by utility descending):**

| Skill | Utility | Uses | Success | Fail | Scope | Created | Last Used | Optimized |
|-------|---------|------|---------|------|-------|---------|-----------|-----------|

**Action Items:**
- Optimize candidates: skills with `utility < threshold` and sufficient samples
- Prune candidates: skills unused for `prune_after_days_unused` days
- New skills: skills with `utility = null` (need more usage data)

### 3. Suggest Actions

If optimize candidates exist — suggest `/memento optimize {name}`
If prune candidates exist — suggest `/memento prune`
