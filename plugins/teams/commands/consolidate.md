---
description: Generate unified view of all team members' milestones and current status
allowed-tools:
  - Read
  - Write
  - Bash
  - Glob
  - Grep
  - Task
---

<objective>
Generate consolidated views of all team members' planning activity.

**What this does:**
- Reads all member directories from `.planning-shared/team/*/`
- Discovers session subdirectories (`.planning-shared/team/*/sessions/*/`)
- Creates `.planning-shared/MILESTONES.md` with completed work ordered by date
- Creates `.planning-shared/STATUS.md` with current work in progress per member/session
- Updates local `.planning/MILESTONES.md` with reference to team view

**When to run:**
- After `/gsd-teams:share` to see consolidated team state
- Before team syncs for visibility
- Periodically to track team progress

**What happens:**
1. Discovers flat member directories and session subdirectories
2. Collects milestones from all entries (flat members + sessions)
3. Sorts by completion date (earliest first)
4. Generates unified MILESTONES.md with attribution
5. Generates STATUS.md with current phase per member/session
6. Updates local MILESTONES.md with team reference
</objective>

<context>
**Arguments:** None

**State:**
- `@.planning/config.json` (team.member, team.max_session_age_days)

**Input:**
- `.planning-shared/team/*/MILESTONES.md` - Each flat member's completed work
- `.planning-shared/team/*/STATE.md` - Each flat member's current state
- `.planning-shared/team/*/sessions/*/MILESTONES.md` - Each session's completed work
- `.planning-shared/team/*/sessions/*/STATE.md` - Each session's current state

**Output:**
- `.planning-shared/MILESTONES.md` - Consolidated milestones by date
- `.planning-shared/STATUS.md` - Current work per member/session
- `.planning/MILESTONES.md` - Updated with team reference section
- Stale session directories removed (if max_session_age_days configured)
</context>

<process>

<step name="check_prerequisites">
Verify GSD planning state exists and team directory is populated.

```bash
# Check for required GSD planning files
MISSING=""
[ ! -f .planning/STATE.md ] && MISSING="${MISSING} STATE.md"
[ ! -f .planning/ROADMAP.md ] && MISSING="${MISSING} ROADMAP.md"
[ ! -f .planning/MILESTONES.md ] && MISSING="${MISSING} MILESTONES.md"

if [ -n "$MISSING" ]; then
  echo "Error: GSD planning state incomplete. Missing files in .planning/:${MISSING}"
  echo "Run /gsd:new-project first to initialize your planning state."
  exit 1
fi
```

Verify .planning-shared/team/ exists with at least one member or session.

**Check for team directory:**
```bash
if [ ! -d .planning-shared/team ]; then
  echo "Error: No .planning-shared/team/ directory found."
  echo "Run /gsd-teams:share first to share your planning state."
  exit 1
fi
```

**Count all entries (flat members + sessions):**
```bash
ENTRY_COUNT=0
for MEMBER_DIR in .planning-shared/team/*/; do
  MEMBER_NAME=$(basename "$MEMBER_DIR")

  # Check for session subdirectories
  if [ -d "${MEMBER_DIR}sessions/" ]; then
    for SESSION_DIR in "${MEMBER_DIR}sessions/"*/; do
      [ -d "$SESSION_DIR" ] && ENTRY_COUNT=$((ENTRY_COUNT + 1))
    done
  fi

  # Check for flat member files
  if [ -f "${MEMBER_DIR}STATE.md" ] || [ -f "${MEMBER_DIR}MILESTONES.md" ]; then
    ENTRY_COUNT=$((ENTRY_COUNT + 1))
  fi
done

if [ "$ENTRY_COUNT" -eq 0 ]; then
  echo "Error: No team members or sessions found in .planning-shared/team/"
  echo "Run /gsd-teams:share first to share your planning state."
  exit 1
fi

echo "Found ${ENTRY_COUNT} member/session entries"
```
</step>

<step name="load_last_version">
Load per-member/session version tracking for incremental extraction. Handles migration from old global format.

**Check for old format and migrate:**
```bash
LAST_VERSION_FILE=".planning-shared/last_consolidated.json"

# Check for old format and migrate
if [ -f "$LAST_VERSION_FILE" ] && grep -q '"last_version"' "$LAST_VERSION_FILE"; then
  # Old format: {"last_version": "v1"}
  OLD_VERSION=$(grep -o '"last_version"[[:space:]]*:[[:space:]]*"[^"]*"' "$LAST_VERSION_FILE" | sed 's/.*"\([^"]*\)"$/\1/')

  # Migrate: assign old version to all current flat members
  TMPFILE="${LAST_VERSION_FILE}.tmp"
  echo "{" > "$TMPFILE"
  FIRST=true
  for MEMBER_DIR in .planning-shared/team/*/; do
    MEMBER=$(basename "$MEMBER_DIR")
    if [ -f "${MEMBER_DIR}MILESTONES.md" ]; then
      [ "$FIRST" = true ] && FIRST=false || echo "," >> "$TMPFILE"
      printf '  "%s": "%s"' "$MEMBER" "$OLD_VERSION" >> "$TMPFILE"
    fi
  done
  echo "" >> "$TMPFILE"
  echo "}" >> "$TMPFILE"
  mv "$TMPFILE" "$LAST_VERSION_FILE"
  echo "Migrated last_consolidated.json from global to per-member format"
fi

# Now read per-member/session versions
# Version lookup happens per-entry in spawn_agents step using MEMBER_KEY
echo "Version tracking loaded from ${LAST_VERSION_FILE}"
```

**Helper function for per-entry version lookup (used by spawn_agents):**
```bash
# To look up version for a member/session key:
get_last_version() {
  local KEY="$1"
  local FILE=".planning-shared/last_consolidated.json"
  if [ -f "$FILE" ]; then
    local ESCAPED_KEY="${KEY//\//\\/}"
    grep -o "\"${ESCAPED_KEY}\"[[:space:]]*:[[:space:]]*\"[^\"]*\"" "$FILE" | sed 's/.*"\([^"]*\)"$/\1/'
  fi
}
# Returns empty string if key not found (triggers full extraction)
```
</step>

<step name="cleanup_stale_sessions">
Check for and remove stale session directories based on `max_session_age_days` configuration.

If `max_session_age_days` is set in `.planning/config.json`, sessions whose last git commit is older than the configured threshold are deleted. This only affects session directories under `.../sessions/` -- flat member directories are never cleaned.

Age detection uses git commit timestamp (reflects last share time), falling back to file modification time. CHANGELOG entries and shared milestone data are preserved -- only the session directory is removed.

If `max_session_age_days` is not configured (default), this step is a no-op.

```bash
# Read max age from config (key is under team.* object, but grep matches by key name regardless of nesting)
MAX_AGE=$(cat .planning/config.json 2>/dev/null | grep -o '"max_session_age_days"[[:space:]]*:[[:space:]]*[0-9]*' | grep -o '[0-9]*$')

if [ -n "$MAX_AGE" ] && [ "$MAX_AGE" -gt 0 ]; then
  echo "Checking for stale sessions (max age: ${MAX_AGE} days)"
  NOW_EPOCH=$(date +%s)
  MAX_AGE_SECONDS=$((MAX_AGE * 86400))
  CLEANED=0

  for MEMBER_DIR in .planning-shared/team/*/; do
    if [ -d "${MEMBER_DIR}sessions/" ]; then
      for SESSION_DIR in "${MEMBER_DIR}sessions/"*/; do
        [ -d "$SESSION_DIR" ] || continue

        # Get age using git timestamp (preferred - reflects last share time)
        SESSION_EPOCH=$(git log -1 --format="%ct" -- "$SESSION_DIR" 2>/dev/null)
        if [ -z "$SESSION_EPOCH" ]; then
          # Fallback to file mtime
          SESSION_EPOCH=$(stat -c '%Y' "$SESSION_DIR" 2>/dev/null)
        fi

        if [ -n "$SESSION_EPOCH" ]; then
          AGE_SECONDS=$((NOW_EPOCH - SESSION_EPOCH))
          if [ "$AGE_SECONDS" -gt "$MAX_AGE_SECONDS" ]; then
            MEMBER_NAME=$(basename "$(dirname "$(dirname "$SESSION_DIR")")")
            SESSION_NAME=$(basename "$SESSION_DIR")
            AGE_DAYS=$((AGE_SECONDS / 86400))
            echo "Removing stale session: ${MEMBER_NAME}/${SESSION_NAME} (${AGE_DAYS} days old)"
            rm -rf "$SESSION_DIR"
            CLEANED=$((CLEANED + 1))
          fi
        fi
      done

      # Clean up empty sessions/ directory
      if [ -d "${MEMBER_DIR}sessions/" ] && [ -z "$(ls -A "${MEMBER_DIR}sessions/" 2>/dev/null)" ]; then
        rmdir "${MEMBER_DIR}sessions/"
      fi
    fi
  done

  if [ "$CLEANED" -gt 0 ]; then
    echo "Cleaned ${CLEANED} stale session(s)"
  else
    echo "No stale sessions found"
  fi
else
  echo "Stale session cleanup: disabled (no max_session_age_days configured)"
fi
```
</step>

<step name="spawn_agents">
Spawn gsd-team-reporter agent for each member/session entry in parallel batches.

**Discover member directories and session subdirectories (two-level loop):**
```bash
MEMBER_ENTRIES=()

for DIR in .planning-shared/team/*/; do
  NAME=$(basename "$DIR")

  # Check for session subdirectories
  if [ -d "${DIR}sessions/" ]; then
    for SDIR in "${DIR}sessions/"*/; do
      [ -d "$SDIR" ] || continue
      SNAME=$(basename "$SDIR")
      MEMBER_ENTRIES+=("${SDIR}|${NAME}|${SNAME}")
    done
  fi

  # Check for flat member files
  if [ -f "${DIR}STATE.md" ] || [ -f "${DIR}MILESTONES.md" ]; then
    MEMBER_ENTRIES+=("${DIR}|${NAME}|")
  fi
done

echo "Spawning agents for ${#MEMBER_ENTRIES[@]} entries"
```

**For each entry, compute MEMBER_KEY and look up version:**
```bash
# For entry "dir|member|session":
IFS='|' read -r ENTRY_DIR MEMBER_NAME SESSION_NAME <<< "$ENTRY"
if [ -n "$SESSION_NAME" ]; then
  MEMBER_KEY="${MEMBER_NAME}/${SESSION_NAME}"
else
  MEMBER_KEY="${MEMBER_NAME}"
fi
ENTRY_LAST_VERSION=$(get_last_version "$MEMBER_KEY")
[ -z "$ENTRY_LAST_VERSION" ] && ENTRY_LAST_VERSION="none"
```

**For each entry, invoke gsd-team-reporter:**
Use Task tool to spawn agents. All Task() calls in a single message run in parallel.

For batches of up to 8 entries, spawn agents with:
- `subagent_type: "gsd-team-reporter"`
- `prompt`: Contains member directory, member_name, session_name, and last_version
- `description`: "Extract: {member_key}"

**Prompt template for each agent:**
```
Task(
  subagent_type: "gsd-team-reporter",
  prompt: "<objective>Extract milestones and status for: ${MEMBER_KEY}</objective>
<context>
Member directory: ${ENTRY_DIR}
Member name: ${MEMBER_NAME}
Session name: ${SESSION_NAME}
Last consolidated version: ${ENTRY_LAST_VERSION}
</context>
Read the member's MILESTONES.md and STATE.md files, extract structured data, and return in the specified format.",
  description: "Extract: ${MEMBER_KEY}"
)
```

**Batch processing:**
- Group entries into batches of 8
- Spawn all agents in a batch simultaneously
- Collect TaskOutput for each agent before moving to next batch
- Store outputs in AGENT_RESULTS array

The agents will return structured output with `---MILESTONE_START---` / `---MILESTONE_END---` and `---STATUS_START---` / `---STATUS_END---` delimiters for parsing, plus `**Session:**` field for session attribution.
</step>

<step name="collect_results">
Collect agent outputs and parse structured data.

**Parse each agent output:**
For each AGENT_RESULT:

1. **Extract member name:**
   - Parse `**Member:** {name}` line

2. **Extract session name:**
   - Parse `**Session:** {name}` line
   - Value "none" normalized to empty string

3. **Extract last updated timestamp:**
   - Parse `**Last Updated:** {timestamp}` line

4. **Extract milestones:**
   - Find all content between `---MILESTONE_START---` and `---MILESTONE_END---`
   - For each milestone block, extract:
     - version: line starting with `version:`
     - name: line starting with `name:`
     - date: line starting with `date:`
     - body: everything after `body:` line

5. **Extract status:**
   - Find content between `---STATUS_START---` and `---STATUS_END---`
   - Extract phase, phase_name, plan, status fields

6. **Handle errors:**
   - Check for `---ERROR---` markers
   - If MILESTONES section has error: note member has no completed milestones
   - If STATUS section has error: mark status as "unknown"

**Store parsed data:**
```bash
# Create temp files and directory for parsed data
PARSED_MILESTONES=$(mktemp)
PARSED_STATUS=$(mktemp)
MILESTONE_BODIES=$(mktemp -d)

# For each agent result, parse and append to temp files
for AGENT_OUTPUT in "${AGENT_RESULTS[@]}"; do
  # Extract member name
  MEMBER=$(echo "$AGENT_OUTPUT" | grep '^\*\*Member:\*\*' | sed 's/\*\*Member:\*\* //')

  # Extract session name
  SESSION=$(echo "$AGENT_OUTPUT" | grep '^\*\*Session:\*\*' | sed 's/\*\*Session:\*\* //')
  [ "$SESSION" = "none" ] && SESSION=""

  # Extract last updated timestamp
  LAST_UPDATED=$(echo "$AGENT_OUTPUT" | grep '^\*\*Last Updated:\*\*' | sed 's/\*\*Last Updated:\*\* //')

  # Check for milestone errors
  if echo "$AGENT_OUTPUT" | grep -q '### Milestones' && echo "$AGENT_OUTPUT" | sed -n '/### Milestones/,/### Status/p' | grep -q '\-\-\-ERROR\-\-\-'; then
    echo "Note: ${MEMBER} has no completed milestones" >> "$MILESTONE_NOTES"
  else
    # Extract each milestone block
    echo "$AGENT_OUTPUT" | awk '
      /---MILESTONE_START---/ { in_milestone=1; next }
      /---MILESTONE_END---/ {
        in_milestone=0
        # Output: DATE|MEMBER|VERSION|NAME|SESSION
        print date "|" member "|" version "|" name "|" session
        # Reset for next milestone
        version=""; name=""; date=""; body=""
        next
      }
      in_milestone && /^version:/ { version=$2 }
      in_milestone && /^name:/ { name=substr($0, 7) }
      in_milestone && /^date:/ { date=$2 }
      in_milestone && /^body:/ { in_body=1; body=""; next }
      in_milestone && in_body { body=body $0 "\n" }
    ' member="$MEMBER" session="$SESSION" >> "$PARSED_MILESTONES"

    # Extract and save milestone bodies
    echo "$AGENT_OUTPUT" | awk -v member="$MEMBER" -v session="$SESSION" -v bodydir="$MILESTONE_BODIES" '
      /---MILESTONE_START---/ { in_milestone=1; body=""; next }
      /---MILESTONE_END---/ {
        in_milestone=0
        # Save body to file keyed by DATE_MEMBER_SESSION_VERSION
        key=date "_" member "_" session "_" version
        gsub(/[ .]/, "_", key)
        if (body != "") {
          print body > (bodydir "/" key)
        }
        version=""; name=""; date=""; body=""
        next
      }
      in_milestone && /^version:/ { version=$2 }
      in_milestone && /^date:/ { date=$2 }
      in_milestone && /^body:/ { in_body=1; next }
      in_milestone && in_body { body=body $0 "\n" }
    '
  fi

  # Check for status errors
  if echo "$AGENT_OUTPUT" | grep -q '### Status' && echo "$AGENT_OUTPUT" | sed -n '/### Status/,/$/p' | grep -q '\-\-\-ERROR\-\-\-'; then
    echo "${LAST_UPDATED}|${MEMBER}|${SESSION}|Status unknown" >> "$PARSED_STATUS"
  else
    # Extract status block
    STATUS_BLOCK=$(echo "$AGENT_OUTPUT" | sed -n '/---STATUS_START---/,/---STATUS_END---/p')
    PHASE=$(echo "$STATUS_BLOCK" | grep '^phase:' | sed 's/phase: //')
    PHASE_NAME=$(echo "$STATUS_BLOCK" | grep '^phase_name:' | sed 's/phase_name: //')
    PLAN=$(echo "$STATUS_BLOCK" | grep '^plan:' | sed 's/plan: //')
    STATUS=$(echo "$STATUS_BLOCK" | grep '^status:' | sed 's/status: //')

    # Build status string
    if [ -n "$PHASE" ]; then
      STATUS_STRING="Phase ${PHASE}"
      [ -n "$PHASE_NAME" ] && STATUS_STRING="${STATUS_STRING} (${PHASE_NAME})"
      [ -n "$PLAN" ] && STATUS_STRING="${STATUS_STRING}, Plan ${PLAN}"
      [ -n "$STATUS" ] && STATUS_STRING="${STATUS_STRING} - ${STATUS}"
    else
      STATUS_STRING="Status unknown"
    fi

    echo "${LAST_UPDATED}|${MEMBER}|${SESSION}|${STATUS_STRING}" >> "$PARSED_STATUS"
  fi
done
```

**Data formats:**
- `PARSED_MILESTONES`: One line per milestone - `DATE|MEMBER|VERSION|NAME|SESSION`
- `PARSED_STATUS`: One line per member/session - `TIMESTAMP|MEMBER|SESSION|STATUS_STRING`
- `MILESTONE_BODIES`: Directory with body content files keyed by `DATE_MEMBER_SESSION_VERSION`
- `MILESTONE_NOTES`: File with notes about members with no milestones
</step>

<step name="generate_milestones_md">
Sort milestones by date and generate consolidated MILESTONES.md.

Uses PARSED_MILESTONES and MILESTONE_BODIES from collect_results step.

**Get metadata:**
```bash
# Count entries (not just top-level directories)
ENTRY_COUNT=0
for DIR in .planning-shared/team/*/; do
  [ -d "${DIR}sessions/" ] && for S in "${DIR}sessions/"*/; do [ -d "$S" ] && ENTRY_COUNT=$((ENTRY_COUNT + 1)); done
  { [ -f "${DIR}STATE.md" ] || [ -f "${DIR}MILESTONES.md" ]; } && ENTRY_COUNT=$((ENTRY_COUNT + 1))
done
GENERATION_TIME=$(date '+%Y-%m-%d %H:%M')
```

**Generate MILESTONES.md atomically:**
```bash
{
  echo "# Team Milestones"
  echo ""
  echo "*Consolidated from ${ENTRY_COUNT} member/session entries on ${GENERATION_TIME}*"
  echo ""

  # Sort by date (col 1), then member name (col 2) for same-date tiebreaker
  if [ -s "$PARSED_MILESTONES" ]; then
    sort -t'|' -k1,1 -k2,2 "$PARSED_MILESTONES" | while IFS='|' read -r DATE MEMBER VERSION NAME SESSION; do
      # Build attribution: "member" for flat, "member/session" for session entries
      if [ -n "$SESSION" ]; then
        ATTRIBUTION="${MEMBER}/${SESSION}"
      else
        ATTRIBUTION="${MEMBER}"
      fi

      # Format header: "## vX: Name (attribution)"
      echo "## ${VERSION}: ${NAME} (${ATTRIBUTION})"
      echo ""
      echo "**Shipped:** ${DATE}"
      echo ""

      # Include body content if available
      if [ -n "$SESSION" ]; then
        KEY=$(echo "${DATE}_${MEMBER}_${SESSION}_${VERSION}" | tr ' .' '__')
      else
        KEY=$(echo "${DATE}_${MEMBER}_${VERSION}" | tr ' .' '__')
      fi
      if [ -f "${MILESTONE_BODIES}/${KEY}" ]; then
        cat "${MILESTONE_BODIES}/${KEY}"
        echo ""
      fi

      # Add separator between milestones
      echo "---"
      echo ""
    done
  else
    echo "*No milestones found.*"
    echo ""
  fi

  # Include any notes/warnings from collect_results
  if [ -s "$MILESTONE_NOTES" ]; then
    echo "---"
    echo ""
    echo "### Notes"
    echo ""
    cat "$MILESTONE_NOTES"
    echo ""
  fi

} > .planning-shared/MILESTONES.tmp && mv .planning-shared/MILESTONES.tmp .planning-shared/MILESTONES.md
```

**Cleanup temp files:**
```bash
rm -f "$PARSED_MILESTONES" "$MILESTONE_NOTES"
rm -rf "$MILESTONE_BODIES"
```
</step>

<step name="generate_status_md">
Generate consolidated STATUS.md from agent-parsed status data.

Uses PARSED_STATUS from collect_results step (format: TIMESTAMP|MEMBER|SESSION|STATUS_STRING).

**Generate STATUS.md with one row per entry:**
```bash
GENERATION_TIME=$(date '+%Y-%m-%d %H:%M')

{
  echo "# Team Status"
  echo ""
  echo "*Consolidated from ${ENTRY_COUNT} member/session entries on ${GENERATION_TIME}*"
  echo ""
  echo "| Member | Current Work | Last Shared |"
  echo "|--------|--------------|-------------|"

  if [ -s "$PARSED_STATUS" ]; then
    # Sort by member name then session for consistent ordering
    sort -t'|' -k2,2 -k3,3 "$PARSED_STATUS" | while IFS='|' read -r TIME MEMBER SESSION STATUS; do
      if [ -n "$SESSION" ]; then
        echo "| ${MEMBER}/${SESSION} | ${STATUS} | ${TIME} |"
      else
        echo "| ${MEMBER} | ${STATUS} | ${TIME} |"
      fi
    done
  fi

} > .planning-shared/STATUS.tmp && mv .planning-shared/STATUS.tmp .planning-shared/STATUS.md
```

**Cleanup:**
```bash
rm -f "$PARSED_STATUS"
```
</step>

<step name="update_local_milestones">
Add team reference section to local MILESTONES.md if not already present.

**Check for local MILESTONES.md:**
```bash
LOCAL_MILESTONES=".planning/MILESTONES.md"

if [ ! -f "$LOCAL_MILESTONES" ]; then
  echo "Note: No local MILESTONES.md to update"
  # Skip the rest of this step
fi
```

**Check if team reference already exists (idempotent):**
```bash
if grep -q "## Team View" "$LOCAL_MILESTONES" 2>/dev/null; then
  echo "Team reference already exists in local MILESTONES.md"
  # Skip adding section
fi
```

**Add reference section atomically:**
```bash
{
  cat "$LOCAL_MILESTONES"
  echo ""
  echo "## Team View"
  echo ""
  echo "See consolidated team milestones: \`.planning-shared/MILESTONES.md\`"
  echo "See current team status: \`.planning-shared/STATUS.md\`"
  echo ""
} > "${LOCAL_MILESTONES}.tmp" && mv "${LOCAL_MILESTONES}.tmp" "$LOCAL_MILESTONES"

echo "Added team reference to local MILESTONES.md"
```
</step>

<step name="save_last_version">
Save per-member/session version map for future incremental extraction. Auto-cleans stale entries.

**Find highest version per member/session from consolidated milestones:**
```bash
LAST_VERSION_FILE=".planning-shared/last_consolidated.json"
TMPFILE="${LAST_VERSION_FILE}.tmp"

echo "{" > "$TMPFILE"
FIRST=true

# For each currently existing member/session entry, save their version
for DIR in .planning-shared/team/*/; do
  NAME=$(basename "$DIR")

  # Flat member
  if [ -f "${DIR}STATE.md" ] || [ -f "${DIR}MILESTONES.md" ]; then
    # Find highest version for this flat member from MILESTONES.md
    HIGHEST=$(grep "^## v" .planning-shared/MILESTONES.md 2>/dev/null | grep "(${NAME})" | grep -o "v[0-9][^ :]*" | sort -t'.' -k1,1n -k2,2n | tail -1)
    if [ -n "$HIGHEST" ]; then
      [ "$FIRST" = true ] && FIRST=false || echo "," >> "$TMPFILE"
      printf '  "%s": "%s"' "$NAME" "$HIGHEST" >> "$TMPFILE"
    fi
  fi

  # Sessions
  if [ -d "${DIR}sessions/" ]; then
    for SDIR in "${DIR}sessions/"*/; do
      [ -d "$SDIR" ] || continue
      SNAME=$(basename "$SDIR")
      SKEY="${NAME}/${SNAME}"
      HIGHEST=$(grep "^## v" .planning-shared/MILESTONES.md 2>/dev/null | grep "(${SKEY})" | grep -o "v[0-9][^ :]*" | sort -t'.' -k1,1n -k2,2n | tail -1)
      if [ -n "$HIGHEST" ]; then
        [ "$FIRST" = true ] && FIRST=false || echo "," >> "$TMPFILE"
        printf '  "%s": "%s"' "$SKEY" "$HIGHEST" >> "$TMPFILE"
      fi
    done
  fi
done

echo "" >> "$TMPFILE"
echo "}" >> "$TMPFILE"
mv "$TMPFILE" "$LAST_VERSION_FILE"
echo "Saved per-member/session versions to ${LAST_VERSION_FILE}"
```

This enables incremental extraction on future runs - agents will only extract milestones newer than their entry's version. Auto-clean: only directories that still exist get entries; stale entries for deleted sessions are automatically dropped.
</step>

<step name="git_commit">
Commit consolidated files to git. Note: .planning-shared/ is ALWAYS committed regardless of commit_docs setting.

**Check if .planning-shared is gitignored (rare edge case):**
```bash
if git check-ignore -q .planning-shared 2>/dev/null; then
  echo "Skipping git commit (.planning-shared is gitignored)"
  GIT_SKIPPED=true
else
  GIT_SKIPPED=false
fi
```

**Stage and commit (unless gitignored):**
```bash
if [ "$GIT_SKIPPED" = "false" ]; then
  # Always stage .planning-shared/ consolidated files and version tracking
  git add .planning-shared/MILESTONES.md .planning-shared/STATUS.md .planning-shared/last_consolidated.json

  # Check commit_docs setting for local MILESTONES.md
  COMMIT_DOCS=$(cat .planning/config.json 2>/dev/null | grep -o '"commit_docs"[[:space:]]*:[[:space:]]*[^,}]*' | grep -o 'true\|false' || echo "true")
  git check-ignore -q .planning 2>/dev/null && COMMIT_DOCS=false

  # If commit_docs=true, also stage local MILESTONES.md
  if [ "$COMMIT_DOCS" = "true" ]; then
    git add .planning/MILESTONES.md 2>/dev/null || true
  fi

  # Only commit if there are staged changes
  if ! git diff --cached --quiet; then
    git commit -m "$(cat <<'EOF'
docs(team): consolidate team milestones and status

Generated:
- .planning-shared/MILESTONES.md (completed work by date)
- .planning-shared/STATUS.md (current work per member)
EOF
)"
  else
    echo "No changes to commit (consolidation unchanged)"
  fi
fi
```

Handle "nothing to commit" gracefully - this happens when consolidation is re-run without changes.
</step>

<step name="show_summary">
Display completion summary with counts and status.

**Count milestones and entries:**
```bash
MILESTONE_COUNT=$(grep -c "^## v" .planning-shared/MILESTONES.md 2>/dev/null || echo "0")
```

**Success output:**
```
Consolidation complete.

Files generated:
- .planning-shared/MILESTONES.md (${MILESTONE_COUNT} milestones from ${ENTRY_COUNT} entries)
- .planning-shared/STATUS.md (${ENTRY_COUNT} entries)
```

**If local MILESTONES.md was updated:**
```
Local update:
- .planning/MILESTONES.md (team reference added)
```

**Git commit status:**
- If committed: `Changes committed to git`
- If skipped due to gitignore: `(Git commit skipped - .planning-shared is gitignored)`
- If no changes: `(No changes to commit - consolidation unchanged)`
</step>

</process>

<output>
**Files created/modified:**
- `.planning-shared/MILESTONES.md` - All team milestones ordered by date (with member/session attribution)
- `.planning-shared/STATUS.md` - Current status per member/session
- `.planning-shared/last_consolidated.json` - Per-member/session version tracking map
- `.planning/MILESTONES.md` - Team reference section (if exists)

**Git commit:**
- Commit message: `docs(team): consolidate team milestones and status`
</output>

<success_criteria>
- [ ] GSD planning state validated (STATE.md, ROADMAP.md, MILESTONES.md)
- [ ] .planning-shared/team/ directory exists with members or sessions
- [ ] Session directories discovered alongside flat member directories
- [ ] MILESTONES.md generated with date-ordered milestones and member/session attribution
- [ ] STATUS.md generated with member/session status table
- [ ] Per-member/session version tracking saved with auto-clean
- [ ] Local MILESTONES.md updated (if exists)
- [ ] Changes committed to git
</success_criteria>
