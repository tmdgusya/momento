---
name: memento-reflect
description: Use after completing any significant task to self-evaluate what worked or failed, update learned skill metrics, and discover reusable patterns worth capturing as new skills
---

# Memento Reflect

Post-task self-evaluation that closes the learning loop. This is the Write phase of Read-Write Reflective Learning.

## Protocol

### 1. Trace Analysis

Review the current conversation and identify:
- What task was performed?
- Which tools and approaches were used?
- Were any learned skills referenced or applied?
- How many iterations/retries were needed?

### 2. Self-Evaluation

Judge the outcome using these criteria:
- **User accepted result without corrections?** — strong success signal
- **Completed in one pass?** — success. Multiple retries — partial or failure.
- **Errors encountered during execution?** — note which phase failed
- **Final result meets the original request?** — ultimate success criterion

Verdict: `success` or `failure` with one-sentence reasoning.

### 3. Update Metrics

If a learned skill was used during this task, update its metrics file.

Read the appropriate metrics.json (`~/.claude/memento/metrics.json` for global skills, `.claude/memento/metrics.json` for project skills). Then update the skill entry:

- `usage_count`: increment by 1
- `success_count`: increment if success
- `failure_count`: increment if failure
- `last_used`: today's date
- `utility`: success_count / (success_count + failure_count)
- `trigger_log`: append {"task": "short description", "result": "success|failure", "date": "today"} — keep max 5 most recent

Write the updated JSON back. Create the directory and file if they don't exist yet — initialize with:

```json
{
  "version": 1,
  "config": {
    "utility_threshold": 0.4,
    "min_samples_for_judgment": 3,
    "prune_after_days_unused": 60
  },
  "skills": {}
}
```

### 4. Pattern Discovery

Ask yourself: "Did I do something reusable that no existing skill covers?"

Criteria for a new skill:
- The pattern would apply to future tasks (not one-off)
- It required non-obvious steps or domain knowledge
- An agent without this experience would likely struggle

If yes — tell the user what pattern you found and suggest: "Want me to capture this as a learned skill? I'll use `/memento create`."

### 5. Failure-Driven Optimization

If a learned skill was used but the task failed:
- Was the skill's Procedure incomplete or wrong?
- Was it applied to the wrong type of task (routing problem)?

If the skill itself is at fault — suggest: "The `{skill-name}` skill may need improvement. Want me to run `/memento optimize {skill-name}`?"

## Important

- Always show the user what you're updating before writing
- Never silently modify metrics — announce changes
- One reflect per task — don't batch multiple tasks
