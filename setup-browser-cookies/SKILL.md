---
name: setup-browser-cookies
version: 1.1.0
description: |
  Save and load authentication state (cookies + storage) for the headless browser session.
  Use before QA testing authenticated pages.
allowed-tools:
  - Bash
  - Read
  - AskUserQuestion
---

## Update Check (run first)

```bash
_UPD=$(~/.claude/skills/kstack/bin/kstack-update-check 2>/dev/null || .claude/skills/kstack/bin/kstack-update-check 2>/dev/null || true)
[ -n "$_UPD" ] && echo "$_UPD" || true
```

If output shows `UPGRADE_AVAILABLE <old> <new>`: read `~/.claude/skills/kstack/kstack-upgrade/SKILL.md` and follow the "Inline upgrade flow" (AskUserQuestion → upgrade if yes, `touch ~/.kstack/last-update-check` if no). If `JUST_UPGRADED <from> <to>`: tell user "Running kstack v{to} (just updated!)" and continue.

# Setup Browser Cookies / Auth State

Save and load authenticated sessions for the headless browser.

## How it works

1. Find the agent-browser binary
2. Log in to the target site
3. Save auth state (cookies + localStorage + sessionStorage)
4. Load it in future sessions to skip login

## Steps

### 1. Find the browse binary

## SETUP (run this check BEFORE any browse command)

```bash
B=""
if command -v agent-browser &>/dev/null; then
  B="agent-browser"
else
  _ROOT=$(git rev-parse --show-toplevel 2>/dev/null)
  [ -n "$_ROOT" ] && [ -x "$_ROOT/.claude/skills/kstack/node_modules/.bin/agent-browser" ] && B="$_ROOT/.claude/skills/kstack/node_modules/.bin/agent-browser"
  [ -z "$B" ] && [ -x ~/.claude/skills/kstack/node_modules/.bin/agent-browser ] && B=~/.claude/skills/kstack/node_modules/.bin/agent-browser
fi
if [ -n "$B" ]; then
  echo "READY: $B"
else
  echo "NEEDS_SETUP"
fi
```

If `NEEDS_SETUP`:
1. Tell the user: "kstack needs agent-browser installed. OK to proceed?" Then STOP and wait.
2. Run: `cd <SKILL_DIR> && ./setup`
3. Or install globally: `npm install -g agent-browser && agent-browser install`

### 2. Log in and save state

```bash
$B open https://app.example.com/login
$B snapshot -i
$B fill @e3 "user@example.com"
$B fill @e4 "password"
$B click @e5
$B state save /tmp/auth-state.json
```

Tell the user: **"Auth state saved. You can load this in future sessions to skip login."**

### 3. Load saved state

```bash
$B state load /tmp/auth-state.json
$B open https://app.example.com/dashboard
```

### 4. Direct cookie management

```bash
# View current cookies
$B cookies

# Set a specific cookie
$B cookies set session_token "abc123"

# Clear all cookies
$B cookies clear
```

### 5. Verify

After loading state:

```bash
$B cookies
```

Show the user a summary of loaded cookies (domain counts).

## Notes

- Auth state includes cookies, localStorage, and sessionStorage
- The browser session persists state between commands, so loaded cookies work immediately
- State files can be shared between team members for testing authenticated flows
- Use `--session-name` flag on `open` for automatic cookie persistence per session
