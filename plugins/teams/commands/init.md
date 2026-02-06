---
description: Set up team configuration for first-time use (member name, sync mode)
argument-hint: "[--name member-name] [--sync full|shallow]"
allowed-tools:
  - Read
  - Write
  - Bash
  - AskUserQuestion
---

<objective>
Configure team settings in `.planning/config.json` for first-time use. Sets member name and default sync mode.

Can also be re-run to update settings.

**When to run:**
- First time setting up team sharing in a project
- When changing member name or sync mode
- After cloning a repo that uses gsd-teams
</objective>

<context>
**Arguments:**
- `--name member-name` (optional) - Set member name directly
- `--sync full|shallow` (optional) - Set default sync mode

**State references:**
@.planning/config.json (team configuration)
@.planning/ (validates GSD is initialized)

**Outputs:**
- `.planning/config.json` - Updated with team.member and team.sync settings
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
```

If .planning/ directory does not exist, stop with error.
</step>

<step name="load_existing_config">
Read current config if it exists:

```bash
if [ -f .planning/config.json ]; then
  CONFIG_CONTENT=$(cat .planning/config.json)
else
  CONFIG_CONTENT='{}'
fi

# Extract existing values
EXISTING_MEMBER=$(echo "$CONFIG_CONTENT" | grep -o '"member"[[:space:]]*:[[:space:]]*"[^"]*"' | grep -o '"[^"]*"$' | tr -d '"' || echo "")
EXISTING_SYNC=$(echo "$CONFIG_CONTENT" | grep -o '"sync"[[:space:]]*:[[:space:]]*"[^"]*"' | grep -o '"[^"]*"$' | tr -d '"' || echo "")
EXISTING_MAX_AGE=$(echo "$CONFIG_CONTENT" | grep -o '"max_session_age_days"[[:space:]]*:[[:space:]]*[0-9]*' | grep -o '[0-9]*' || echo "")
EXISTING_COMMIT_DOCS=$(echo "$CONFIG_CONTENT" | grep -o '"commit_docs"[[:space:]]*:[[:space:]]*[^,}]*' | grep -o 'true\|false' || echo "")
```

Display current settings if they exist:
```
Current team settings:
  Member: ${EXISTING_MEMBER:-not set}
  Sync mode: ${EXISTING_SYNC:-not set}
```
</step>

<step name="resolve_member_name">
Resolve member name using strict priority: argument > existing config > prompt.

```bash
MEMBER_NAME=""

# Priority 1: --name argument (if provided)
if [[ "$ARGUMENTS" == *"--name "* ]]; then
  MEMBER_NAME="${ARGUMENTS#*--name }"
  MEMBER_NAME="${MEMBER_NAME%% -*}"
  MEMBER_NAME="${MEMBER_NAME%% }"
fi

# Priority 2: existing config (only if arg was empty)
if [ -z "$MEMBER_NAME" ]; then
  MEMBER_NAME="$EXISTING_MEMBER"
fi

# Priority 3: prompt user (only if both arg and config were empty)
if [ -z "$MEMBER_NAME" ]; then
  # Use AskUserQuestion
  MEMBER_NAME="<from_prompt>"
fi
```

If MEMBER_NAME is still empty after priorities 1 and 2, use AskUserQuestion:
- header: "Member Identity"
- question: "Enter your member name for team sharing:"
- note: "This will be normalized to lowercase with hyphens (e.g., 'John Smith' becomes 'john-smith'). Use slash notation for parallel sessions (e.g., 'ian/green'). This name identifies you in .planning-shared/team/{name}/ and CHANGELOG.md."

Do NOT auto-detect from git config. The user must explicitly provide their name.

**Normalize after resolution:**
```bash
MEMBER_NAME="${MEMBER_NAME,,}"
MEMBER_NAME="${MEMBER_NAME// /-}"
```
</step>

<step name="resolve_sync_mode">
Resolve sync mode using strict priority: argument > existing config > prompt.

```bash
SYNC_MODE=""

# Priority 1: --sync argument (if provided)
if [[ "$ARGUMENTS" == *"--sync "* ]]; then
  SYNC_MODE="${ARGUMENTS#*--sync }"
  SYNC_MODE="${SYNC_MODE%% *}"
fi

# Priority 2: existing config (only if arg was empty)
if [ -z "$SYNC_MODE" ]; then
  SYNC_MODE="$EXISTING_SYNC"
fi

# Priority 3: prompt user (only if both arg and config were empty)
if [ -z "$SYNC_MODE" ]; then
  # Use AskUserQuestion
  SYNC_MODE="<from_prompt>"
fi
```

If SYNC_MODE is still empty after priorities 1 and 2, use AskUserQuestion:
- header: "Sync Mode"
- question: "Choose default sync mode for team sharing:"
- options: ["full - sync all planning files (recommended for most teams)", "shallow - sync only summaries, exclude WIP content"]
- note: "Full mode shares everything in .planning/. Shallow mode excludes codebase/, research/, debug/, milestones/, and draft plans. You can override per-share with the --shallow flag on /gsd-teams:share."

Parse response:
- If response contains "shallow": SYNC_MODE="shallow"
- Otherwise: SYNC_MODE="full"

Validate SYNC_MODE is either "full" or "shallow". If neither, default to "full".
</step>

<step name="save_config">
Write member name and sync mode to .planning/config.json under the team.* object. Preserve existing non-team keys.

```bash
MAX_AGE_VALUE="${EXISTING_MAX_AGE:-1}"
COMMIT_DOCS_VALUE="${EXISTING_COMMIT_DOCS:-false}"

# Build config with preserved values
echo "{
  \"commit_docs\": ${COMMIT_DOCS_VALUE},
  \"team\": {
    \"member\": \"${MEMBER_NAME}\",
    \"sync\": \"${SYNC_MODE}\",
    \"max_session_age_days\": ${MAX_AGE_VALUE}
  }
}" > .planning/config.tmp && mv .planning/config.tmp .planning/config.json
```

Use atomic write pattern (temp file then mv) for safety.

Note: This overwrites the full config but preserves all known keys. For projects with additional custom keys in config.json, those would need manual preservation. The standard GSD config only uses commit_docs and team.*.
</step>

<step name="show_summary">
Display saved settings and suggest next step.

```
Team configuration saved to .planning/config.json

  Member: ${MEMBER_NAME}
  Sync mode: ${SYNC_MODE}
  Max session age: ${MAX_AGE_VALUE} day(s)

Next step: Run `/gsd-teams:share` to share your planning state.
```

If settings were updated (not first-time):
```
Settings updated. Changes take effect on next /gsd-teams:share.
```
</step>

</process>

<output>
**Files modified:**
- `.planning/config.json` - Team settings saved (member, sync mode, max_session_age_days)
</output>

<success_criteria>
- [ ] GSD planning directory validated (.planning/ exists)
- [ ] Member name resolved (from argument, config, or prompt)
- [ ] Sync mode resolved (from argument, config, or prompt)
- [ ] Settings saved to .planning/config.json with atomic write
- [ ] Existing non-team config keys preserved
- [ ] Summary displayed with next step suggestion
</success_criteria>
