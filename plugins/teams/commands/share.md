---
description: Share planning state to isolated member directory with audit trail
argument-hint: "[--name member-name] [--shallow]"
allowed-tools:
  - Read
  - Write
  - Bash
  - Glob
  - Grep
  - Task
  - AskUserQuestion
---

<objective>
Share your GSD planning state to an isolated member directory with a full audit trail.

**Why:** Enables multiple team members (or parallel Claude sessions) to share planning state without merge conflicts. Each member's state is isolated in their own directory under `.planning-shared/team/`.

**When to run:**
- After completing a phase or significant milestone
- When switching to work on another machine/session
- Before team sync meetings to share current progress
- Periodically to maintain team visibility

**What happens:**
1. Copies `.planning/` contents to member's shared directory (flat or session-based)
2. Updates shared CHANGELOG.md with audit entry
3. Commits changes to git (if enabled)
</objective>

<context>
**Arguments:**
- `--name member-name` (optional) - Override member name for this share only
- `--shallow` (optional) - Sync only summary files, excluding WIP directories and draft plans

**State references:**
@.planning/config.json (member name)
@.planning/STATE.md (current phase)
@.planning/ROADMAP.md (milestone version)

**Outputs:**
- `${MEMBER_DIR}/` - Copy of planning state (flat: `.planning-shared/team/name/`, session: `.planning-shared/team/name/sessions/session/`)
- `.planning-shared/CHANGELOG.md` - Audit log entry
- `.planning/config.json` - Member name saved (if newly detected)
</context>

<process>

<step name="check_prerequisites">
Verify GSD planning directory exists:

```bash
if [ ! -d .planning ]; then
  echo "Error: No .planning/ directory found."
  echo "Run /gsd:new-project first to initialize your project."
  exit 1
fi

# Warn about missing files (non-blocking -- share still proceeds)
WARNINGS=""
[ ! -f .planning/STATE.md ] && WARNINGS="${WARNINGS}\n  - STATE.md missing (phase info will show as '?')"
[ ! -f .planning/ROADMAP.md ] && WARNINGS="${WARNINGS}\n  - ROADMAP.md missing (milestone version will show as 'v?')"

if [ -n "$WARNINGS" ]; then
  echo "Warning: Some planning files are missing:${WARNINGS}"
  echo "CHANGELOG entry will use placeholder values. Run /gsd:new-project to create these files."
fi
```

The .planning/ directory must exist, but missing STATE.md and ROADMAP.md only produce warnings since the extract_context_for_changelog step already has 2>/dev/null fallbacks with defaults ("?", "v?").
</step>

<step name="resolve_member_name">
Resolve member name using strict priority: argument > config > prompt.

```bash
MEMBER_NAME=""

# Priority 1: --name argument (if provided)
if [[ "$ARGUMENTS" == *"--name "* ]]; then
  MEMBER_NAME="${ARGUMENTS#*--name }"
  MEMBER_NAME="${MEMBER_NAME%% *}"
fi

# Priority 2: config.json (only if arg was empty)
if [ -z "$MEMBER_NAME" ]; then
  MEMBER_NAME=$(cat .planning/config.json 2>/dev/null | grep -o '"member"[[:space:]]*:[[:space:]]*"[^"]*"' | grep -o '"[^"]*"$' | tr -d '"' || echo "")
fi

# Priority 3: prompt user (only if both arg and config were empty)
if [ -z "$MEMBER_NAME" ]; then
  # Use AskUserQuestion - see below
  MEMBER_NAME="<from_prompt>"
fi
```

If MEMBER_NAME is still empty after priorities 1 and 2, use AskUserQuestion:
- header: "Member Identity"
- question: "Enter your member name for team sharing:"
- note: "This will be normalized to lowercase with hyphens (e.g., 'John Smith' becomes 'john-smith'). Use slash notation for parallel sessions (e.g., 'ian/green'). This name identifies you in .planning-shared/team/{name}/ and CHANGELOG.md."

Do NOT auto-detect from git config. The user must explicitly provide their name.

**After resolution, normalize and parse slash notation:**
```bash
# Normalize
MEMBER_NAME="${MEMBER_NAME,,}"
MEMBER_NAME="${MEMBER_NAME// /-}"

# Parse slash notation
if [[ "$MEMBER_NAME" == *"/"* ]]; then
  MEMBER_BASE="${MEMBER_NAME%%/*}"
  SESSION_RAW="${MEMBER_NAME#*/}"
  SESSION_NAME="${SESSION_RAW//\//-}"
  MEMBER_DIR=".planning-shared/team/${MEMBER_BASE}/sessions/${SESSION_NAME}"
  MEMBER_LABEL="${MEMBER_BASE}/${SESSION_NAME}"
  HAS_SESSION=true
else
  MEMBER_BASE="${MEMBER_NAME}"
  SESSION_NAME=""
  MEMBER_DIR=".planning-shared/team/${MEMBER_BASE}"
  MEMBER_LABEL="${MEMBER_BASE}"
  HAS_SESSION=false
fi
```

Display: "Using member name: ${MEMBER_LABEL}"
</step>

<step name="save_member_to_config">
If member was newly detected (not already in config), prompt for commit_docs preference and save to config.json.

**Check if config.json exists:**
```bash
if [ -f .planning/config.json ]; then
  CONFIG_CONTENT=$(cat .planning/config.json)
else
  CONFIG_CONTENT='{}'
fi
```

**Check if member key already exists (under team.* object, grep matches by key name regardless of nesting):**
```bash
EXISTING_MEMBER=$(echo "$CONFIG_CONTENT" | grep -o '"member"[[:space:]]*:[[:space:]]*"[^"]*"' | grep -o '"[^"]*"$' | tr -d '"' || echo "")
```

**Preserve existing team.* values (sync mode, max session age) when saving member name.**
This prevents config corruption:
```bash
# Extract existing team.* values before any write
EXISTING_SYNC=$(echo "$CONFIG_CONTENT" | grep -o '"sync"[[:space:]]*:[[:space:]]*"[^"]*"' | grep -o '"[^"]*"$' | tr -d '"' || echo "")
EXISTING_MAX_AGE=$(echo "$CONFIG_CONTENT" | grep -o '"max_session_age_days"[[:space:]]*:[[:space:]]*[0-9]*' | grep -o '[0-9]*' || echo "")

# Use existing values if present, defaults if not
SYNC_VALUE="${EXISTING_SYNC:-full}"
MAX_AGE_VALUE="${EXISTING_MAX_AGE:-1}"
```

**If no existing member, ask about commit_docs preference:**

Use AskUserQuestion:
- header: "Planning Docs in Git"
- question: "Should .planning/ files be committed to git?"
- options: ["no - keep planning docs local only (recommended)", "yes - commit planning docs"]
- note: "If 'no', .planning/ will be added to .gitignore (files kept locally). Note: .planning-shared/ is always committed regardless of this setting."

Parse response:
- If user selects "no" or response contains "no": COMMIT_DOCS=false
- Otherwise: COMMIT_DOCS=true

**If commit_docs=false, update .gitignore and untrack:**
```bash
if [ "$COMMIT_DOCS" = "false" ]; then
  # Add .planning/ to .gitignore if not already present
  if ! grep -q "^\.planning/$" .gitignore 2>/dev/null; then
    echo "" >> .gitignore
    echo "# GSD planning docs (local only)" >> .gitignore
    echo ".planning/" >> .gitignore
    echo "Added .planning/ to .gitignore"
  fi

  # Untrack .planning/ from git (keep files locally)
  if git ls-files --error-unmatch .planning/ >/dev/null 2>&1; then
    git rm -r --cached .planning/
    echo "Untracked .planning/ from git (files kept locally)"
  fi
fi
```

**Save member and commit_docs to config.json:**

For a simple config (few keys), can use string manipulation:
```bash
if [ -z "$EXISTING_MEMBER" ]; then
  # Add member and commit_docs to config
  if [ "$CONFIG_CONTENT" = "{}" ]; then
    # Empty config, create with commit_docs and team object (uses preserved values)
    echo "{
  \"commit_docs\": ${COMMIT_DOCS},
  \"team\": {
    \"member\": \"${MEMBER_NAME}\",
    \"sync\": \"${SYNC_VALUE}\",
    \"max_session_age_days\": ${MAX_AGE_VALUE}
  }
}" > .planning/config.tmp && mv .planning/config.tmp .planning/config.json
  else
    # Has content, insert commit_docs and team object before closing brace (uses preserved values)
    CONFIG_WITHOUT_CLOSE="${CONFIG_CONTENT%\}}"
    echo "${CONFIG_WITHOUT_CLOSE},
  \"commit_docs\": ${COMMIT_DOCS},
  \"team\": {
    \"member\": \"${MEMBER_NAME}\",
    \"sync\": \"${SYNC_VALUE}\",
    \"max_session_age_days\": ${MAX_AGE_VALUE}
  }
}" > .planning/config.tmp && mv .planning/config.tmp .planning/config.json
  fi
  echo "Saved member name and commit_docs preference to .planning/config.json"
fi
```

Use atomic write pattern (temp file then mv) for safety.
</step>

<step name="create_shared_directory">
Create the shared directory structure:

```bash
mkdir -p "${MEMBER_DIR}"
```

This is idempotent - safe to run multiple times. For session members, `mkdir -p` creates intermediate directories automatically (e.g., `.planning-shared/team/ian/sessions/green/`).
</step>

<step name="check_uncommitted">
Check for uncommitted changes in .planning/:

```bash
UNCOMMITTED=false
if git status --porcelain .planning/ 2>/dev/null | grep -q .; then
  UNCOMMITTED=true
fi
```

If uncommitted changes exist, note for warning in summary (don't block the share).
</step>

<step name="resolve_sync_mode">
Resolve sync mode using strict priority: --shallow flag > config (team.sync) > default (full).

```bash
SYNC_MODE=""

# Priority 1: --shallow flag
if [[ "$ARGUMENTS" == *"--shallow"* ]]; then
  SYNC_MODE="shallow"
fi

# Priority 2: config team.sync (only if flag was empty)
if [ -z "$SYNC_MODE" ]; then
  SYNC_MODE=$(cat .planning/config.json 2>/dev/null | grep -o '"sync"[[:space:]]*:[[:space:]]*"[^"]*"' | grep -o '"[^"]*"$' | tr -d '"' || echo "")
fi

# Priority 3: default (only if both flag and config were empty)
if [ -z "$SYNC_MODE" ]; then
  SYNC_MODE="full"
fi
```

Display: "Sync mode: ${SYNC_MODE}"

If shallow, also display: "Shallow sync: excluding codebase/, debug/, research/, milestones/, WIP plans"
</step>

<step name="copy_planning_state">
Copy .planning/ to shared directory by delegating to bounded sync subagent.

**IMPORTANT:** Do NOT run rsync directly. Delegate to the gsd-teams:gsd-team-sync agent which has strict execution boundaries preventing scope bleed to sibling directories.

Spawn the sync subagent using the Task tool:

```
Task: gsd-teams:gsd-team-sync
Parameters:
  SOURCE_DIR: .planning/
  DEST_DIR: ${MEMBER_DIR}
  SYNC_MODE: ${SYNC_MODE}
```

Wait for the subagent to return. Check for `---SYNC_ERROR---` in the response.

- If error: Stop and display the error to the user.
- If success (`## SYNC COMPLETE`): Continue to next step.

This makes the sync idempotent - running multiple times safely overwrites previous share.
The sync subagent only touches ${MEMBER_DIR} and cannot affect sibling directories.
</step>

<step name="extract_context_for_changelog">
Extract current phase and milestone for the CHANGELOG entry:

**Get current phase from STATE.md:**
```bash
CURRENT_PHASE=$(grep "^Phase:" .planning/STATE.md 2>/dev/null | grep -o "[0-9]*" | head -1 || echo "?")
PHASE_NAME=$(grep "^Phase:" .planning/STATE.md 2>/dev/null | sed 's/.*(\(.*\))/\1/' | head -1 || echo "unknown")
```

**Get milestone version from ROADMAP.md:**
```bash
MILESTONE_VERSION=$(grep -m1 "^- .*v[0-9]" .planning/ROADMAP.md 2>/dev/null | grep -o "v[0-9][^ )]*" | head -1 || echo "v?")
```

If version not found, try alternate patterns:
```bash
if [ "$MILESTONE_VERSION" = "v?" ]; then
  MILESTONE_VERSION=$(grep -o "v[0-9][0-9.]*" .planning/ROADMAP.md 2>/dev/null | head -1 || echo "v?")
fi
```
</step>

<step name="update_changelog">
Create CHANGELOG.md if missing, remove any existing same-day entry for this member, then prepend new entry.

**Create CHANGELOG.md if it doesn't exist:**
```bash
if [ ! -f .planning-shared/CHANGELOG.md ]; then
  mkdir -p .planning-shared
  cat > .planning-shared/CHANGELOG.md << 'HEADER'
# Team Sharing Changelog

Tracks all planning state shares across team members.

HEADER
fi
```

**Build the entry with today's date (session-aware format):**
```bash
TODAY=$(date '+%Y-%m-%d')

if [ "$HAS_SESSION" = true ]; then
  # Session format: [date] member (session): milestone Phase N (name)
  ENTRY="[${TODAY}] ${MEMBER_BASE} (${SESSION_NAME}): ${MILESTONE_VERSION} Phase ${CURRENT_PHASE} (${PHASE_NAME})"
  DEDUP_PATTERN="[${TODAY}] ${MEMBER_BASE} (${SESSION_NAME}):"
else
  # Flat format (unchanged): [date] member: milestone Phase N (name)
  ENTRY="[${TODAY}] ${MEMBER_LABEL}: ${MILESTONE_VERSION} Phase ${CURRENT_PHASE} (${PHASE_NAME})"
  DEDUP_PATTERN="[${TODAY}] ${MEMBER_LABEL}:"
fi
```

**Remove existing entry for same member+date (deduplication):**
```bash
# Use fixed-string matching (-F) to avoid regex issues with parentheses in session names
grep -vF "$DEDUP_PATTERN" .planning-shared/CHANGELOG.md > .planning-shared/CHANGELOG.tmp 2>/dev/null || cp .planning-shared/CHANGELOG.md .planning-shared/CHANGELOG.tmp
mv .planning-shared/CHANGELOG.tmp .planning-shared/CHANGELOG.md
```

**Prepend new entry atomically:**
```bash
{
  head -4 .planning-shared/CHANGELOG.md
  echo "$ENTRY"
  tail -n +5 .planning-shared/CHANGELOG.md
} > .planning-shared/CHANGELOG.tmp && mv .planning-shared/CHANGELOG.tmp .planning-shared/CHANGELOG.md
```

This ensures:
- Running multiple times on same day updates the entry (removes old, adds new)
- Different days create separate entries (history preserved)
- Different members on same day each have their own entry
</step>

<step name="git_commit">
Commit the shared files to git. Note: .planning-shared/ is ALWAYS committed regardless of commit_docs setting.

**Check if .planning-shared is gitignored (rare edge case):**
```bash
# If user explicitly gitignored .planning-shared, respect that
if git check-ignore -q .planning-shared 2>/dev/null; then
  echo "Skipping git commit (.planning-shared is gitignored)"
  # Set flag for summary step
  GIT_SKIPPED=true
else
  GIT_SKIPPED=false
fi
```

**Stage and commit (unless gitignored):**
```bash
if [ "$GIT_SKIPPED" = "false" ]; then
  # Always stage .planning-shared/
  git add .planning-shared/

  # Optionally stage .planning/config.json based on commit_docs preference
  COMMIT_DOCS=$(cat .planning/config.json 2>/dev/null | grep -o '"commit_docs"[[:space:]]*:[[:space:]]*[^,}]*' | grep -o 'true\|false' || echo "true")
  if [ "$COMMIT_DOCS" = "true" ]; then
    git add .planning/config.json 2>/dev/null || true
  fi

  # Only commit if there are staged changes
  if ! git diff --cached --quiet; then
    git commit -m "$(cat <<EOF
docs(team): share planning state to ${MEMBER_LABEL}

${MILESTONE_VERSION} Phase ${CURRENT_PHASE}: ${PHASE_NAME}
EOF
)"
  else
    echo "No changes to commit (share is up to date)"
  fi
fi
```

Handle "nothing to commit" gracefully - this happens when share is re-run without changes.
</step>

<step name="show_summary">
Display confirmation and next steps.

**Success output:**
```
Shared to ${MEMBER_DIR}/

CHANGELOG entry:
  ${ENTRY}
```

**If uncommitted changes were detected:**
```
Note: .planning/ has uncommitted changes - share reflects uncommitted state
```

**Next steps:**
```
**Next steps:**
- Run `/gsd-teams:consolidate` to update team-wide milestones and status
```

**If git commit was skipped:**
```
(Git commit skipped - .planning-shared is gitignored)
```
</step>

</process>

<output>
**Files created/modified:**
- `${MEMBER_DIR}/` - Complete copy of planning state (excluding config.json and sessions/)
- `.planning-shared/CHANGELOG.md` - New entry prepended with share details
- `.planning/config.json` - Member name saved (if newly detected)

**Git commit (if enabled):**
- Commit message: `docs(team): share planning state to ${MEMBER_LABEL}`
- Files staged: `.planning-shared/`, `.planning/config.json`
</output>

<success_criteria>
- [ ] .planning/ directory exists (warns if STATE.md or ROADMAP.md missing)
- [ ] Member name resolved (from argument, config, or user prompt)
- [ ] Member name saved to config for future shares
- [ ] Directory ${MEMBER_DIR}/ created
- [ ] Sync mode resolved (from --shallow flag, config, or default)
- [ ] Shallow mode excludes WIP directories and files
- [ ] .planning/ contents copied (excluding config.json and sessions/)
- [ ] CHANGELOG.md updated with new entry
- [ ] Changes committed to git (if enabled)
</success_criteria>
