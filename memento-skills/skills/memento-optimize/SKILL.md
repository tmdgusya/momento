---
name: memento-optimize
description: Use when a learned skill has low utility score, has been failing, or the user wants to improve an existing learned skill based on failure analysis
---

# Memento Optimize

Refine a learned skill based on failure analysis. Corresponds to the paper's skill-level reflective update.

## Protocol

### 1. Identify Target

If not specified by user, find candidates:
- Read metrics.json (global and project)
- List skills where `utility < utility_threshold` AND `usage_count >= min_samples_for_judgment`
- Present list to user, let them choose

### 2. Analyze Failures

Read the target skill's `trigger_log` for failure entries. Also read the SKILL.md itself. Identify:
- **Procedure gaps**: steps that are missing or unclear
- **Wrong scope**: skill is being triggered for tasks it shouldn't handle
- **Outdated approach**: the procedure worked before but conditions changed

### 3. Revise SKILL.md

Make targeted edits:
- Fix Procedure steps that caused failures
- Add failure modes to "Known Failure Modes" section
- Tighten "When to Use" if the skill is being misapplied
- Update the description if trigger conditions need refinement
- Update the utility summary line in the blockquote header

### 4. Update Metrics

Update the skill's metrics entry:
- `last_optimized`: today's date
- `optimization_count`: increment by 1

Do NOT reset usage/success/failure counts. The history is valuable.

### 5. Show Changes

Present a before/after diff of the SKILL.md changes to the user.
