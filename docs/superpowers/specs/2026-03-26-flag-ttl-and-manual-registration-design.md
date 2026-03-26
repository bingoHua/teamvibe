# Flag TTL & Manual Registration Design

Date: 2026-03-26

## Problem

Two issues in the teamwork decision management system affect reliability and collaboration:

1. **Flag file persistence**: The commit gate (`pre-commit-decision.py`) uses a flag file in `/tmp/cursor-hooks/` to allow a retry commit after the user generates a decision record. If the session crashes before the decision record is written, the flag file persists. On the next session, any commit bypasses the decision gate — the entire enforcement mechanism is silently broken.

2. **Silent auto-registration**: When a new team member first uses the project, `resolve_current_user` in `compat.py` silently writes their git username into `.teamwork/config.json`. This causes: (a) the user has no chance to confirm their name or set their role, (b) git usernames may be wrong or inconsistent across machines, (c) two people registering simultaneously can cause git merge conflicts on `config.json`.

## Solution

### Change 1: Flag file TTL (10 minutes)

**Current behavior**: Flag file contains the conversation_id string. If the file exists, commit is allowed.

**New behavior**: Flag file contains a Unix timestamp (seconds since epoch). On check:
1. Read the timestamp from the flag file
2. If parsing fails (corrupt file, old-format conversation_id string, empty, etc.), treat as expired — delete and intercept
3. If current time minus timestamp > 600 seconds (10 minutes), treat as expired: delete the flag, proceed with normal interception
4. If within 10 minutes, allow the commit (normal retry flow)

**Why 10 minutes**: Normal flow (intercept -> generate decision -> user review -> retry commit) takes under 5 minutes. 10 minutes gives ample margin while catching stale flags from crashed sessions. Note: macOS `/tmp` is not reliably cleared on reboot, so TTL is necessary even across reboots.

**Backward compatibility**: Existing flag files from before this change contain conversation_id strings, not timestamps. These will fail float parsing and be treated as expired — safe default behavior.

**Files changed**: `pre-commit-decision.py` only.

**Specific changes**:
- Flag creation: write `str(time.time())` into flag file instead of conversation_id
- Flag check logic: read file content, try parse as float; on parse failure treat as expired; on success compare with `time.time()`, expire if > 600

### Change 2: AI-guided manual registration

**Current behavior**: `resolve_current_user()` in `compat.py` auto-registers unknown users into `config.json`.

**New behavior**: Unknown users are returned with `source: "unregistered"` and no write to `config.json`. Registration happens through explicit AI-guided interaction.

**Files changed**: `compat.py`, `session-init.py`, `pre-commit-decision.py`, `teamwork-decisions.md`, `teamwork-decisions.mdc`.

**Specific changes**:

#### compat.py — `resolve_current_user()`
- Remove auto-registration logic (the two branches that call `save_config` for git-name-only and email-only cases)
- When user not found in team_members, return user dict with `source: "unregistered"` and whatever identity info is available (git name, git email, account email) for display
- Handle edge case: if neither git name nor email is available, return `source: "unknown"` (same as current behavior)
- Add a new helper function `register_user(project_root, name, role, email)` that:
  - Reads current config.json (fresh read, not cached — avoids overwriting another user's recent registration)
  - Appends new member to `team_members`
  - Writes back to config.json
  - To be called by AI after user provides their info

#### session-init.py
- After resolving user, check `source` field:
  - `"unregistered"`: prepend registration prompt with available identity info:
    ```
    **[New member detected]** You are not registered in this project's team config.
    Detected git name: {git_name or "not set"}, git email: {git_email or "not set"}.
    Please tell me your preferred display name and team role (e.g., product manager, frontend developer, designer), and I will register you.
    ```
  - `"unknown"`: same prompt but note that no git identity was detected
- Still inject decision context (don't block the session entirely)

#### pre-commit-decision.py
- Note: this file uses `resolve_current_user_simple()` wrapper which returns only the user dict — the `source` field is accessible through it
- After resolving user, check if `source` is `"unregistered"` or `"unknown"`
- If so, deny the commit with message: "Please register as a team member first. Tell the AI your name and role."
- This ensures unregistered users cannot commit without identity

#### Rules files (teamwork-decisions.md, teamwork-decisions.mdc)
- Both files get identical content (`.mdc` is Cursor's markdown variant, same syntax)
- Add a section:
  ```
  ## New Member Registration
  When the session context indicates an unregistered or unknown member:
  1. Ask the user for their preferred display name and team role
  2. Write their info to .teamwork/config.json using a fresh read-then-append approach
  3. Confirm registration to the user
  4. The config change should be committed with [skip decision] tag
  ```

## Data flow

### Flag TTL flow
```
User runs git commit
  -> pre-commit-decision.py checks flag file
     -> No flag: intercept (create flag with timestamp, deny commit)
     -> Flag exists, age < 10 min: allow commit, delete flag
     -> Flag exists, age >= 10 min: delete stale flag, intercept as new
```

### Registration flow
```
New user opens project
  -> session-init.py detects unregistered user
  -> Injects prompt: "Please register with name and role"
  -> User tells AI their info
  -> AI does fresh read of config.json (picks up any recent registrations by others)
  -> AI appends new member and writes config.json
  -> AI commits config change with [skip decision] tag
  -> Next session: user is recognized
```

## Testing

- **Flag TTL**: Create a flag file with a timestamp from 15 minutes ago, verify commit is intercepted. Create one from 1 minute ago, verify commit is allowed.
- **Registration**: Start with empty `team_members`, verify session-init shows registration prompt. Verify commit is blocked for unregistered users. After manual registration, verify user is recognized.

## Not included

- Problem 3 (push validation false positives on non-code files) is deferred — user hasn't decided on approach yet.
- No changes to `pre-push-validate.py`, `post-pull-review.py`, or `pre-compact-reminder.py`.
