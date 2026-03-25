---
name: memento-prune
description: Use when the user wants to remove low-value learned skills, clean up unused skills, or reduce the learned skill library size
---

# Memento Prune

Remove learned skills that are no longer useful. Never auto-deletes — always confirms with user.

## Protocol

### 1. Identify Candidates

Read metrics.json (global and project). Flag skills matching ANY:
- `utility < utility_threshold` AND `usage_count >= min_samples_for_judgment` (proven low value)
- `last_used` older than `prune_after_days_unused` days from today (abandoned)

### 2. Present Candidates

Show each candidate with reasoning:

```
Prune candidates:

1. learned-bad-pattern (global)
   Utility: 0.20 (1/5) — below threshold 0.40
   Last used: 2026-01-15 — 69 days ago
   Reason: low utility + unused

2. learned-old-approach (project)
   Utility: — (0/0) — never used
   Last used: 2026-01-20 — 64 days ago
   Reason: abandoned (never used, 64 days old)
```

### 3. User Confirms Each

Ask: "Delete these skills? You can choose individually (e.g., 'just 1') or all."

For each confirmed deletion:
- Remove the `learned-{name}/` skill directory
- Remove the entry from metrics.json
- Announce what was removed

### 4. Never Auto-Delete

Even if called programmatically, always show the list and wait for confirmation.
