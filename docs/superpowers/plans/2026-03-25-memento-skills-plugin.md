# Memento-Skills Plugin Implementation Plan

> **For agentic workers:** REQUIRED: Use superpowers:subagent-driven-development (if subagents available) or superpowers:executing-plans to implement this plan. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Build a Claude Code plugin that implements Read-Write Reflective Learning — a SessionStart hook auto-injects learned skill metrics (Read phase), and five skills handle reflection, creation, optimization, stats, and pruning (Write phase).

**Architecture:** Plugin with SessionStart hook (bash script reads metrics JSON, generates context summary, injects into system prompt) + 5 SKILL.md files + 1 learned-skill template. Runtime data (metrics.json, learned-*/SKILL.md) lives outside the plugin, in user's `~/.claude/` and project `.claude/` directories.

**Tech Stack:** Bash (hook), jq (JSON parsing), Markdown (skills), JSON (metrics storage)

**Spec:** `docs/superpowers/specs/2026-03-25-memento-skills-plugin-design.md`

---

## File Structure

```
memento-skills/
├── .claude-plugin/
│   └── plugin.json                    # Plugin metadata
├── package.json                       # Package metadata
├── hooks/
│   ├── hooks.json                     # SessionStart hook registration
│   ├── run-hook.cmd                   # Cross-platform polyglot wrapper
│   └── session-start                  # Read phase: metrics → context injection
├── skills/
│   ├── memento-reflect/
│   │   └── SKILL.md                   # Write phase: self-evaluate + update metrics
│   ├── memento-create/
│   │   └── SKILL.md                   # Skill discovery: generate new learned skill
│   ├── memento-optimize/
│   │   └── SKILL.md                   # Skill refinement: improve failing skill
│   ├── memento-stats/
│   │   └── SKILL.md                   # Dashboard: show metrics summary
│   └── memento-prune/
│       └── SKILL.md                   # Cleanup: remove low-value skills
└── templates/
    └── learned-skill.md               # Template for new learned skills
```

All paths below are relative to `memento-skills/` (created at project root).

---

## Chunk 1: Plugin Scaffold and Hook Infrastructure

### Task 1: Create plugin metadata files

**Files:**
- Create: `memento-skills/.claude-plugin/plugin.json`
- Create: `memento-skills/package.json`

- [ ] **Step 1: Create plugin.json**

```json
{
  "name": "memento-skills",
  "description": "Self-evolving skill system — learn reusable patterns from experience via Read-Write Reflective Learning",
  "version": "0.1.0",
  "author": {
    "name": "roach"
  },
  "license": "MIT",
  "keywords": ["skills", "learning", "reflection", "metrics", "self-evolving", "memento"]
}
```

- [ ] **Step 2: Create package.json**

```json
{
  "name": "memento-skills",
  "version": "0.1.0",
  "description": "Self-evolving skill system for Claude Code",
  "license": "MIT"
}
```

- [ ] **Step 3: Commit scaffold**

```bash
git add memento-skills/.claude-plugin/plugin.json memento-skills/package.json
git commit -m "Scaffold memento-skills plugin metadata"
```

### Task 2: Create hook registration and cross-platform runner

**Files:**
- Create: `memento-skills/hooks/hooks.json`
- Create: `memento-skills/hooks/run-hook.cmd`

- [ ] **Step 1: Create hooks.json**

Registers the SessionStart hook, same pattern as superpowers:

```json
{
  "hooks": {
    "SessionStart": [
      {
        "matcher": "startup|clear|compact",
        "hooks": [
          {
            "type": "command",
            "command": "\"${CLAUDE_PLUGIN_ROOT}/hooks/run-hook.cmd\" session-start",
            "async": false
          }
        ]
      }
    ]
  }
}
```

- [ ] **Step 2: Create run-hook.cmd**

Copy the superpowers polyglot pattern verbatim from `~/.claude/plugins/cache/claude-plugins-official/superpowers/5.0.5/hooks/run-hook.cmd`. This handles Windows (batch finds bash) and Unix (exec bash directly).

```bash
: << 'CMDBLOCK'
@echo off
REM Cross-platform polyglot wrapper for hook scripts.
REM On Windows: cmd.exe runs the batch portion, which finds and calls bash.
REM On Unix: the shell interprets this as a script (: is a no-op in bash).

if "%~1"=="" (
    echo run-hook.cmd: missing script name >&2
    exit /b 1
)

set "HOOK_DIR=%~dp0"

if exist "C:\Program Files\Git\bin\bash.exe" (
    "C:\Program Files\Git\bin\bash.exe" "%HOOK_DIR%%~1" %2 %3 %4 %5 %6 %7 %8 %9
    exit /b %ERRORLEVEL%
)
if exist "C:\Program Files (x86)\Git\bin\bash.exe" (
    "C:\Program Files (x86)\Git\bin\bash.exe" "%HOOK_DIR%%~1" %2 %3 %4 %5 %6 %7 %8 %9
    exit /b %ERRORLEVEL%
)

where bash >nul 2>nul
if %ERRORLEVEL% equ 0 (
    bash "%HOOK_DIR%%~1" %2 %3 %4 %5 %6 %7 %8 %9
    exit /b %ERRORLEVEL%
)

exit /b 0
CMDBLOCK

# Unix: run the named script directly
SCRIPT_DIR="$(cd "$(dirname "$0")" && pwd)"
SCRIPT_NAME="$1"
shift
exec bash "${SCRIPT_DIR}/${SCRIPT_NAME}" "$@"
```

Make executable: `chmod +x memento-skills/hooks/run-hook.cmd`

- [ ] **Step 3: Commit hook infrastructure**

```bash
git add memento-skills/hooks/hooks.json memento-skills/hooks/run-hook.cmd
git commit -m "Add hook registration and cross-platform runner"
```

### Task 3: Implement session-start hook (Read Phase)

This is the core of the Read phase. The script reads metrics JSON files, merges them, generates a summary table, and injects it into the system prompt.

**Files:**
- Create: `memento-skills/hooks/session-start`

- [ ] **Step 1: Write the session-start script**

```bash
#!/usr/bin/env bash
# SessionStart hook for memento-skills plugin
# Read Phase: loads metrics and injects learned skill context

set -euo pipefail

SCRIPT_DIR="$(cd "$(dirname "$0")" && pwd)"
PLUGIN_ROOT="$(cd "${SCRIPT_DIR}/.." && pwd)"

GLOBAL_METRICS="${HOME}/.claude/memento/metrics.json"
PROJECT_METRICS=".claude/memento/metrics.json"

# --- helpers ---

has_jq() { command -v jq >/dev/null 2>&1; }

# Read a metrics.json, output tab-separated skill rows.
# Each row: name \t utility_display \t usage_count \t scope \t last_used \t raw_utility
# Falls back to a minimal bash JSON parser when jq is absent.
read_metrics() {
  local file="$1"
  [ -f "$file" ] || return 0

  if has_jq; then
    jq -r '
      .skills // {} | to_entries[] |
      .key as $name |
      .value |
      (if .utility == null then "\u2014"
       else (.utility * 100 | floor / 100 | tostring)
       end) as $util_raw |
      (if .utility == null then
         "\u2014 (\(.success_count)/\(.success_count + .failure_count))"
       else
         "\($util_raw) (\(.success_count)/\(.success_count + .failure_count))"
       end) as $util_display |
      "\($name)\t\($util_display)\t\(.usage_count)\t\(.scope)\t\(.last_used)\t\($util_raw)"
    ' "$file" 2>/dev/null || true
  else
    # Minimal fallback: just report that learned skills exist
    if grep -q '"skills"' "$file" 2>/dev/null; then
      local count
      count=$(grep -c '"scope"' "$file" 2>/dev/null || echo 0)
      echo "__fallback__${count}"
    fi
  fi
}

# --- main ---

global_rows=""
project_rows=""
global_count=0
project_count=0

if [ -f "$GLOBAL_METRICS" ]; then
  global_rows=$(read_metrics "$GLOBAL_METRICS")
  if [ -n "$global_rows" ]; then
    global_count=$(echo "$global_rows" | wc -l | tr -d ' ')
  fi
fi

if [ -f "$PROJECT_METRICS" ]; then
  project_rows=$(read_metrics "$PROJECT_METRICS")
  if [ -n "$project_rows" ]; then
    project_count=$(echo "$project_rows" | wc -l | tr -d ' ')
  fi
fi

total=$((global_count + project_count))

# Build context string
context=""

if [ "$total" -eq 0 ]; then
  context="<memento-skills-context>\\nYou have memento-skills installed. No learned skills yet.\\nAfter completing significant tasks, consider using /memento reflect to capture reusable patterns.\\n</memento-skills-context>"
else
  # Combine and sort by usage_count descending
  all_rows=""
  [ -n "$global_rows" ] && all_rows="$global_rows"
  if [ -n "$project_rows" ]; then
    [ -n "$all_rows" ] && all_rows="${all_rows}"$'\n'"${project_rows}" || all_rows="$project_rows"
  fi

  # Check for fallback mode
  if echo "$all_rows" | grep -q "^__fallback__"; then
    context="<memento-skills-context>\\nYou have learned skills from past experience. Use them when relevant.\\nAfter completing significant tasks, consider using /memento reflect.\\n\\nLearned skills available: ${total} (install jq for detailed metrics table).\\n</memento-skills-context>"
  else
    # Build markdown table
    table="| Skill | Utility | Uses | Scope | Last Used |\\n|-------|---------|------|-------|-----------|"
    warnings=""
    shown=0
    max_shown=20

    # Read threshold from global config
    threshold="0.4"
    if has_jq && [ -f "$GLOBAL_METRICS" ]; then
      threshold=$(jq -r '.config.utility_threshold // 0.4' "$GLOBAL_METRICS" 2>/dev/null || echo "0.4")
    fi
    min_samples="3"
    if has_jq && [ -f "$GLOBAL_METRICS" ]; then
      min_samples=$(jq -r '.config.min_samples_for_judgment // 3' "$GLOBAL_METRICS" 2>/dev/null || echo "3")
    fi

    while IFS=$'\t' read -r name util_display usage scope last_used raw_util; do
      [ -z "$name" ] && continue
      if [ "$shown" -ge "$max_shown" ]; then
        continue
      fi
      table="${table}\\n| ${name} | ${util_display} | ${usage} | ${scope} | ${last_used} |"
      shown=$((shown + 1))

      # Check for low-utility warning
      if [ "$raw_util" != "—" ] && [ "$raw_util" != "null" ] && [ -n "$raw_util" ]; then
        if has_jq; then
          is_low=$(echo "$raw_util $threshold $usage $min_samples" | jq -rn '
            [input | split(" ") | .[]] as $v |
            if ($v[0] | tonumber) < ($v[1] | tonumber) and ($v[2] | tonumber) >= ($v[3] | tonumber)
            then "yes" else "no" end
          ' 2>/dev/null || echo "no")
          if [ "$is_low" = "yes" ]; then
            warnings="${warnings}\\n\\u26a0 ${name}: utility below threshold (${raw_util} < ${threshold}). Consider /memento optimize or /memento prune."
          fi
        fi
      fi
    done <<< "$all_rows"

    remainder=$((total - shown))
    overflow=""
    if [ "$remainder" -gt 0 ]; then
      overflow="\\n... and ${remainder} more learned skills."
    fi

    context="<memento-skills-context>\\nYou have learned skills from past experience. Use them when relevant.\\nAfter completing significant tasks, consider using /memento reflect.\\n\\n## Learned Skills (${global_count} global, ${project_count} project)\\n\\n${table}${overflow}${warnings}\\n</memento-skills-context>"
  fi
fi

# --- output ---

escape_for_json() {
  local s="$1"
  s="${s//\\/\\\\}"
  s="${s//\"/\\\"}"
  s="${s//$'\n'/\\n}"
  s="${s//$'\r'/\\r}"
  s="${s//$'\t'/\\t}"
  printf '%s' "$s"
}

escaped=$(escape_for_json "$context")

if [ -n "${CURSOR_PLUGIN_ROOT:-}" ]; then
  printf '{\n  "additional_context": "%s"\n}\n' "$escaped"
elif [ -n "${CLAUDE_PLUGIN_ROOT:-}" ]; then
  printf '{\n  "hookSpecificOutput": {\n    "hookEventName": "SessionStart",\n    "additionalContext": "%s"\n  }\n}\n' "$escaped"
else
  printf '{\n  "additional_context": "%s"\n}\n' "$escaped"
fi

exit 0
```

Make executable: `chmod +x memento-skills/hooks/session-start`

- [ ] **Step 2: Test with no metrics files**

```bash
cd memento-skills && CLAUDE_PLUGIN_ROOT="$(pwd)" bash hooks/session-start
```

Expected: JSON output containing "No learned skills yet" in additionalContext.

- [ ] **Step 3: Test with mock global metrics**

Create a temporary test metrics file and verify the table is generated:

```bash
mkdir -p /tmp/memento-test/.claude/memento
cat > /tmp/memento-test/.claude/memento/metrics.json << 'TESTEOF'
{
  "version": 1,
  "config": {
    "utility_threshold": 0.4,
    "min_samples_for_judgment": 3,
    "prune_after_days_unused": 60
  },
  "skills": {
    "learned-api-retry": {
      "scope": "global",
      "created_at": "2026-03-25",
      "last_used": "2026-03-25",
      "last_optimized": null,
      "usage_count": 10,
      "success_count": 8,
      "failure_count": 2,
      "utility": 0.8,
      "optimization_count": 0,
      "trigger_log": []
    },
    "learned-bad-skill": {
      "scope": "global",
      "created_at": "2026-03-20",
      "last_used": "2026-03-22",
      "last_optimized": null,
      "usage_count": 5,
      "success_count": 1,
      "failure_count": 4,
      "utility": 0.2,
      "optimization_count": 0,
      "trigger_log": []
    }
  }
}
TESTEOF

HOME=/tmp/memento-test CLAUDE_PLUGIN_ROOT="$(pwd)" bash hooks/session-start
```

Expected: JSON with table showing both skills, warning for `learned-bad-skill` (utility 0.2 < 0.4).

- [ ] **Step 4: Clean up test files and commit**

```bash
rm -rf /tmp/memento-test
git add memento-skills/hooks/session-start
git commit -m "Implement SessionStart hook for Read phase

Reads global and project metrics.json, generates a summary table
of learned skills with utility scores, and injects into system
prompt. Supports jq with pure-bash fallback. Handles 0, 1-20,
and 20+ skill scenarios with context budget management."
```

---

## Chunk 2: Skills and Template

### Task 4: Create learned-skill template

**Files:**
- Create: `memento-skills/templates/learned-skill.md`

- [ ] **Step 1: Write the template**

This is the reference template that `memento-create` uses when generating new learned skills.

```markdown
---
name: learned-{name}
description: Use when {trigger conditions extracted from self-evaluation}
---

# {Skill Name}

> Learned skill -- auto-generated by memento-skills
> Utility: -- (0/0) | Last optimized: never

## When to Use
- {specific trigger situation 1}
- {specific trigger situation 2}

## Procedure
1. {step 1}
2. {step 2}
3. {step 3}

## Known Failure Modes
- None documented yet.

## Examples
{core pattern extracted from successful execution}
```

- [ ] **Step 2: Commit**

```bash
git add memento-skills/templates/learned-skill.md
git commit -m "Add learned skill template"
```

### Task 5: Write memento-reflect skill (Write Phase core)

This is the most important skill — it drives the learning loop.

**Files:**
- Create: `memento-skills/skills/memento-reflect/SKILL.md`

- [ ] **Step 1: Write the skill**

```markdown
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
- **User accepted result without corrections?** → strong success signal
- **Completed in one pass?** → success. Multiple retries → partial or failure.
- **Errors encountered during execution?** → note which phase failed
- **Final result meets the original request?** → ultimate success criterion

Verdict: `success` or `failure` with one-sentence reasoning.

### 3. Update Metrics

If a learned skill was used during this task, update its metrics file.

Read the appropriate metrics.json (`~/.claude/memento/metrics.json` for global skills, `.claude/memento/metrics.json` for project skills). Then update the skill entry:

```json
{
  "usage_count": "increment by 1",
  "success_count": "increment if success",
  "failure_count": "increment if failure",
  "last_used": "today's date",
  "utility": "success_count / (success_count + failure_count)",
  "trigger_log": "append {task, result, date} — keep max 5"
}
```

Write the updated JSON back. Create the directory and file if they don't exist yet (initialize with version 1 schema and default config).

### 4. Pattern Discovery

Ask yourself: **"Did I do something reusable that no existing skill covers?"**

Criteria for a new skill:
- The pattern would apply to future tasks (not one-off)
- It required non-obvious steps or domain knowledge
- An agent without this experience would likely struggle

If yes → tell the user what pattern you found and suggest: "Want me to capture this as a learned skill? I'll use `/memento create`."

### 5. Failure-Driven Optimization

If a learned skill was used but the task failed:
- Was the skill's Procedure incomplete or wrong?
- Was it applied to the wrong type of task (routing problem)?

If the skill itself is at fault → suggest: "The `{skill-name}` skill may need improvement. Want me to run `/memento optimize {skill-name}`?"

## Metrics File Initialization

When the metrics file doesn't exist yet, create it with:

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

## Important

- Always show the user what you're updating before writing
- Never silently modify metrics — announce changes
- One reflect per task — don't batch multiple tasks
```

- [ ] **Step 2: Verify frontmatter compliance**

```bash
head -3 memento-skills/skills/memento-reflect/SKILL.md
wc -w memento-skills/skills/memento-reflect/SKILL.md
```

Expected: valid YAML frontmatter with name and description. Word count under 500.

- [ ] **Step 3: Commit**

```bash
git add memento-skills/skills/memento-reflect/SKILL.md
git commit -m "Add memento-reflect skill (Write phase core)"
```

### Task 6: Write memento-create skill (Skill Discovery)

**Files:**
- Create: `memento-skills/skills/memento-create/SKILL.md`

- [ ] **Step 1: Write the skill**

```markdown
---
name: memento-create
description: Use when memento-reflect identifies a reusable pattern worth capturing, or when the user explicitly wants to create a new learned skill from a task execution pattern
---

# Memento Create

Generate a new learned skill from a task execution pattern and register it in the metrics system.

## Protocol

### 1. Scope Selection

Ask the user:
> "Should this be a **global** skill (available in all projects) or a **project** skill (only this project)?"

- Global → skill goes to `~/.claude/skills/learned-{name}/SKILL.md`, metrics to `~/.claude/memento/metrics.json`
- Project → skill goes to `.claude/skills/learned-{name}/SKILL.md`, metrics to `.claude/memento/metrics.json`

### 2. Name the Skill

Derive a name from the pattern. Rules:
- Prefix: `learned-`
- Lowercase letters, numbers, hyphens only
- Verb-first when possible: `learned-retry-api-on-timeout`
- Keep under 40 characters

### 3. Generate SKILL.md

Use the template at `templates/learned-skill.md` in this plugin. Fill in:
- **description**: "Use when..." with specific trigger conditions. Max 500 chars. No workflow summary.
- **When to Use**: concrete situations (symptoms, not solutions)
- **Procedure**: numbered steps extracted from the successful execution
- **Known Failure Modes**: any pitfalls discovered during the task
- **Examples**: the core pattern, minimal and concrete

Constraints: max 500 words total. CSO-optimized description.

### 4. Register in Metrics

Read the appropriate metrics.json (create if missing, using the initialization schema from memento-reflect). Add:

```json
"learned-{name}": {
  "scope": "global or project",
  "created_at": "today",
  "last_used": "today",
  "last_optimized": null,
  "usage_count": 0,
  "success_count": 0,
  "failure_count": 0,
  "utility": null,
  "optimization_count": 0,
  "trigger_log": []
}
```

### 5. Confirm with User

Show the generated SKILL.md content and the metrics entry. Write only after user confirms.
```

- [ ] **Step 2: Verify compliance**

```bash
head -3 memento-skills/skills/memento-create/SKILL.md
wc -w memento-skills/skills/memento-create/SKILL.md
```

- [ ] **Step 3: Commit**

```bash
git add memento-skills/skills/memento-create/SKILL.md
git commit -m "Add memento-create skill (Skill Discovery)"
```

### Task 7: Write memento-optimize skill (Skill Refinement)

**Files:**
- Create: `memento-skills/skills/memento-optimize/SKILL.md`

- [ ] **Step 1: Write the skill**

```markdown
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

```json
{
  "last_optimized": "today",
  "optimization_count": "increment by 1"
}
```

Do NOT reset usage/success/failure counts. The history is valuable.

### 5. Show Changes

Present a before/after diff of the SKILL.md changes to the user.
```

- [ ] **Step 2: Verify and commit**

```bash
wc -w memento-skills/skills/memento-optimize/SKILL.md
git add memento-skills/skills/memento-optimize/SKILL.md
git commit -m "Add memento-optimize skill (Skill Refinement)"
```

### Task 8: Write memento-stats skill (Dashboard)

**Files:**
- Create: `memento-skills/skills/memento-stats/SKILL.md`

- [ ] **Step 1: Write the skill**

```markdown
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

If optimize candidates exist → suggest `/memento optimize {name}`
If prune candidates exist → suggest `/memento prune`
```

- [ ] **Step 2: Verify and commit**

```bash
wc -w memento-skills/skills/memento-stats/SKILL.md
git add memento-skills/skills/memento-stats/SKILL.md
git commit -m "Add memento-stats skill (Dashboard)"
```

### Task 9: Write memento-prune skill (Cleanup)

**Files:**
- Create: `memento-skills/skills/memento-prune/SKILL.md`

- [ ] **Step 1: Write the skill**

```markdown
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
- Remove the `learned-{name}/SKILL.md` directory
- Remove the entry from metrics.json
- Announce what was removed

### 4. Never Auto-Delete

Even if called programmatically, always show the list and wait for confirmation.
```

- [ ] **Step 2: Verify and commit**

```bash
wc -w memento-skills/skills/memento-prune/SKILL.md
git add memento-skills/skills/memento-prune/SKILL.md
git commit -m "Add memento-prune skill (Cleanup)"
```

---

## Chunk 3: Integration and Verification

### Task 10: End-to-end verification

- [ ] **Step 1: Verify plugin directory structure**

```bash
find memento-skills -type f | sort
```

Expected output matching the file structure from spec:
```
memento-skills/.claude-plugin/plugin.json
memento-skills/hooks/hooks.json
memento-skills/hooks/run-hook.cmd
memento-skills/hooks/session-start
memento-skills/package.json
memento-skills/skills/memento-create/SKILL.md
memento-skills/skills/memento-optimize/SKILL.md
memento-skills/skills/memento-prune/SKILL.md
memento-skills/skills/memento-reflect/SKILL.md
memento-skills/skills/memento-stats/SKILL.md
memento-skills/templates/learned-skill.md
```

- [ ] **Step 2: Verify all SKILL.md files have valid frontmatter**

```bash
for f in memento-skills/skills/*/SKILL.md; do
  echo "--- $f ---"
  head -4 "$f"
  echo "Words: $(wc -w < "$f")"
  echo ""
done
```

Expected: each file has `---` / `name:` / `description:` / `---` and word count under 500.

- [ ] **Step 3: Verify hook runs cleanly with no metrics**

```bash
cd memento-skills && CLAUDE_PLUGIN_ROOT="$(pwd)" bash hooks/session-start | python3 -m json.tool
```

Expected: valid JSON with "No learned skills yet" message.

- [ ] **Step 4: Verify hook runs with mock data**

```bash
mkdir -p /tmp/memento-e2e/.claude/memento
cat > /tmp/memento-e2e/.claude/memento/metrics.json << 'EOF'
{
  "version": 1,
  "config": {"utility_threshold": 0.4, "min_samples_for_judgment": 3, "prune_after_days_unused": 60},
  "skills": {
    "learned-test-skill": {
      "scope": "global", "created_at": "2026-03-25", "last_used": "2026-03-25",
      "last_optimized": null, "usage_count": 5, "success_count": 4, "failure_count": 1,
      "utility": 0.8, "optimization_count": 0, "trigger_log": []
    }
  }
}
EOF

HOME=/tmp/memento-e2e CLAUDE_PLUGIN_ROOT="$(pwd)" bash hooks/session-start | python3 -m json.tool
rm -rf /tmp/memento-e2e
```

Expected: valid JSON with table showing `learned-test-skill` at utility 0.8.

- [ ] **Step 5: Final commit**

```bash
git add -A memento-skills/
git commit -m "Complete memento-skills plugin v0.1.0

Implements Read-Write Reflective Learning (arXiv:2603.18743):
- SessionStart hook for automatic Read phase (metrics injection)
- 5 skills: reflect, create, optimize, stats, prune
- Learned skill template
- JSON-based metrics with utility tracking

Constraint: No embedding router — uses description matching + utility scores
Confidence: high
Scope-risk: narrow
Tested: Hook with empty and populated metrics
Not-tested: Full learning loop across sessions"
```

---

## Post-Implementation

After all tasks are complete:

1. **Install the plugin** — symlink or register in `~/.claude/settings.json`
2. **First real test** — complete a task, run `/memento reflect`, create a skill, verify it appears in next session's hook output
3. **Iterate** — use the plugin on real work, let the learning loop prove itself
