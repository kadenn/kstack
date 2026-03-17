---
name: kstack
version: 1.2.0
description: |
  Fast headless browser for QA testing and site dogfooding. Navigate any URL, interact with
  elements, verify page state, diff before/after actions, take annotated screenshots, check
  responsive layouts, test forms and uploads, handle dialogs, and assert element states.
  Powered by agent-browser (native Rust CLI). Use when you need to test a feature, verify a
  deployment, dogfood a user flow, or file a bug with evidence.
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

# kstack browse: QA Testing & Dogfooding

Powered by [agent-browser](https://github.com/vercel-labs/agent-browser) — native Rust headless browser CLI for AI agents.
First call auto-starts the daemon, then ~100ms per command. Auto-shuts down after idle timeout.
State persists between calls (cookies, tabs, sessions).

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

## IMPORTANT

- Use the agent-browser binary via Bash: `$B <command>`
- NEVER use `mcp__claude-in-chrome__*` tools. They are slow and unreliable.
- Browser persists between calls — cookies, login sessions, and tabs carry over.
- Dialogs (alert/confirm/prompt) are auto-accepted by default — no browser lockup.

## QA Workflows

### Test a user flow (login, signup, checkout, etc.)

```bash
# 1. Go to the page
$B open https://app.example.com/login

# 2. See what's interactive
$B snapshot -i

# 3. Fill the form using refs
$B fill @e3 "test@example.com"
$B fill @e4 "password123"
$B click @e5

# 4. Verify it worked
$B diff snapshot                 # diff shows what changed after clicking
$B is visible ".dashboard"       # assert the dashboard appeared
$B screenshot /tmp/after-login.png
```

### Verify a deployment / check prod

```bash
$B open https://yourapp.com
$B get text                          # read the page — does it load?
$B console                           # any JS errors?
$B network requests                  # any failed requests?
$B get title                         # correct title?
$B is visible ".hero-section"        # key elements present?
$B screenshot /tmp/prod-check.png
```

### Dogfood a feature end-to-end

```bash
# Navigate to the feature
$B open https://app.example.com/new-feature

# Take annotated screenshot — shows every interactive element with labels
$B screenshot --annotate /tmp/feature-annotated.png

# Walk through the flow
$B snapshot -i          # baseline
$B click @e3            # interact
$B diff snapshot        # what changed? (unified diff)

# Check element states
$B is visible ".success-toast"
$B is enabled "#next-step-btn"
$B is checked "#agree-checkbox"

# Check console for errors after interactions
$B console
```

### Test responsive layouts

```bash
# Set specific viewport sizes
$B open https://yourapp.com
$B set viewport 375 812     # iPhone
$B screenshot /tmp/mobile.png
$B set viewport 768 1024    # Tablet
$B screenshot /tmp/tablet.png
$B set viewport 1440 900    # Desktop
$B screenshot /tmp/desktop.png
```

### Test file upload

```bash
$B open https://app.example.com/upload
$B snapshot -i
$B upload @e3 /path/to/test-file.pdf
$B is visible ".upload-success"
$B screenshot /tmp/upload-result.png
```

### Test forms with validation

```bash
$B open https://app.example.com/form
$B snapshot -i

# Submit empty — check validation errors appear
$B click @e10                        # submit button
$B diff snapshot                     # diff shows error messages appeared
$B is visible ".error-message"

# Fill and resubmit
$B fill @e3 "valid input"
$B click @e10
$B diff snapshot                     # diff shows errors gone, success state
```

### Test dialogs (delete confirmations, prompts)

```bash
# Set up dialog handling BEFORE triggering
$B dialog accept              # will auto-accept next alert/confirm
$B click "#delete-button"     # triggers confirmation dialog
$B diff snapshot              # verify the item was deleted

# For prompts that need input
$B dialog accept "my answer"  # accept with text
$B click "#rename-button"     # triggers prompt
```

### Test authenticated pages (save/load auth state)

```bash
# Save auth state after logging in
$B open https://app.example.com/login
$B snapshot -i
$B fill @e3 "user@example.com"
$B fill @e4 "password"
$B click @e5
$B state save /tmp/auth-state.json

# Load auth state in a fresh session
$B state load /tmp/auth-state.json
$B open https://app.example.com/dashboard
$B screenshot /tmp/dashboard.png
```

### Compare snapshots

```bash
$B open https://staging.app.com
$B snapshot -i
# ... perform action ...
$B diff snapshot
```

## Quick Assertion Patterns

```bash
# Element exists and is visible
$B is visible ".modal"

# Button is enabled/disabled
$B is enabled "#submit-btn"

# Checkbox state
$B is checked "#agree"

# Page contains text
$B get text "#message"

# Element count
$B get count ".list-item"

# Specific attribute value
$B get attr "#logo" "src"

# CSS property
$B get styles ".button"
```

## Snapshot System

The snapshot is your primary tool for understanding and interacting with pages.

```
-i          --interactive           Only interactive elements (buttons, links, inputs) — cleaner @e refs for forms/navigation
-C          --cursor                Include cursor-interactive elements (divs with onclick, cursor:pointer, tabindex)
-c          --compact               Remove empty structural elements — reduces noise in large pages
-d <n>      --depth                 Limit tree depth (e.g. -d 3)
-s <sel>    --selector              Scope snapshot to a CSS selector (e.g. -s "#main")
--json      --json                  Output as structured JSON instead of text tree
```

**Ref numbering:** @e refs are assigned sequentially (@e1, @e2, ...) in tree order.

After snapshot, use @refs as selectors in any command:
```bash
$B click @e3       $B fill @e4 "value"     $B hover @e1
$B get text @e2    $B get html @e5
```

**Output format:** indented accessibility tree with @ref IDs, one element per line.
```
  @e1 [heading] "Welcome" [level=1]
  @e2 [textbox] "Email"
  @e3 [button] "Submit"
```

Refs are invalidated on navigation — run `snapshot` again after `open`.

## Command Reference

### Navigation
| Command | Description |
|---------|-------------|
| `back` | History back |
| `forward` | History forward |
| `open <url>` | Navigate to URL (aliases: goto, navigate) |
| `reload` | Reload page |

### Reading
| Command | Description |
|---------|-------------|
| `find <type> <value> <action> [text]` | Semantic element finder: role, text, label, placeholder, alt, title, testid, first, last, nth — then act |
| `get <prop> [sel]` | Get element data: text, html, value, attr, title, url, cdp-url, count, box, styles |

### Interaction
| Command | Description |
|---------|-------------|
| `check <sel>` | Check a checkbox |
| `click <sel>` | Click element (--new-tab to open in new tab) |
| `dblclick <sel>` | Double-click element |
| `dialog <accept [text]|dismiss>` | Accept or dismiss dialogs (accept with optional prompt text) |
| `download <sel> <path>` | Download file by clicking element |
| `drag <src> <tgt>` | Drag and drop between elements |
| `eval <js>` | Run JavaScript in page context (-b for base64, --stdin for piped input) |
| `fill <sel> <val>` | Clear and fill input |
| `focus <sel>` | Focus element |
| `highlight <sel>` | Highlight element on page |
| `hover <sel>` | Hover over element |
| `keyboard <type|inserttext> <text>` | Type with real keystrokes or inserttext (no selector, current focus) |
| `keydown <key>` | Hold key down |
| `keyup <key>` | Release key |
| `mouse <move x y|down [btn]|up [btn]|wheel dy [dx]>` | Low-level mouse control: move, down, up, wheel |
| `press <key>` | Press key — Enter, Tab, Escape, ArrowUp/Down/Left/Right, Backspace, Delete (alias: key) |
| `scroll <dir> [px]` | Scroll direction (up/down/left/right) or pixels (--selector to scope) |
| `scrollintoview <sel>` | Scroll element into view (alias: scrollinto) |
| `select <sel> <val>` | Select dropdown option |
| `set <setting> [value]` | Set viewport, device, geolocation, offline, headers, credentials, media (dark/light) |
| `state <save|load|list|show|rename|clear> [path]` | Save/load/list/show/rename/clear auth state (cookies + storage) |
| `type <sel> <text>` | Type into element |
| `uncheck <sel>` | Uncheck a checkbox |
| `upload <sel> <file> [file2...]` | Upload file(s) to input |
| `wait <sel|ms|--text|--url|--load|--fn>` | Wait for element, text, URL pattern, load state, JS condition, or milliseconds |

### Inspection
| Command | Description |
|---------|-------------|
| `clipboard <read|write <text>|copy|paste>` | Read/write clipboard, copy selection, paste |
| `console [--clear]` | View console messages (--clear to reset) |
| `cookies [get|set <name> <val>|clear]` | Get/set/clear cookies (set supports --url, --domain, --path, --httpOnly, --secure, --sameSite, --expires) |
| `errors [--clear]` | View page errors (--clear to reset) |
| `is <prop> <sel>` | State check (visible, enabled, checked) |
| `network <subcommand>` | Network requests, route interception, mocking (route, unroute, requests --filter) |
| `storage <local|session> [key|set <k> <v>|clear]` | Read/write localStorage and sessionStorage |

### Visual
| Command | Description |
|---------|-------------|
| `diff <snapshot|screenshot|url> [--baseline file]` | Compare snapshots, screenshots, or two URLs |
| `pdf <path>` | Save page as PDF |
| `screenshot [path]` | Capture screenshot (--full, --annotate for numbered labels, --screenshot-dir, --screenshot-format, --screenshot-quality) |

### Snapshot
| Command | Description |
|---------|-------------|
| `snapshot [flags]` | Accessibility tree with @e refs. Flags: -i interactive, -c compact, -C cursor, -d depth, -s selector, --json |

### Debugging
| Command | Description |
|---------|-------------|
| `inspect` | Open Chrome DevTools for the active page |
| `profiler <start|stop [path]>` | Chrome DevTools profiling |
| `trace <start [path]|stop>` | Start/stop trace recording |

### Tabs
| Command | Description |
|---------|-------------|
| `frame <sel|main>` | Switch to iframe or return to main frame |
| `session [list]` | List active sessions or show current session |
| `tab [new [url] | <n> | close [n]]` | List tabs, open new tab (optionally with URL), switch tab, or close tab |
| `window new` | Open new browser window |

### System
| Command | Description |
|---------|-------------|
| `close` | Close browser (aliases: quit, exit) |
| `connect <port|url>` | Connect to existing browser via CDP (port or WebSocket URL) |
| `install [--with-deps]` | Download Chrome for Testing (--with-deps for Linux system deps) |

## Tips

1. **Navigate once, query many times.** `open` loads the page; then `get text`, `screenshot` all hit the loaded page instantly.
2. **Use `snapshot -i` first.** See all interactive elements, then click/fill by ref. No CSS selector guessing.
3. **Use `diff snapshot` to verify.** Baseline → action → diff. See exactly what changed.
4. **Use `is` for assertions.** `is visible .modal` is faster and more reliable than parsing page text.
5. **Use `screenshot --annotate` for evidence.** Annotated screenshots are great for bug reports.
6. **Check `console` after actions.** Catch JS errors that don't surface visually.
7. **Use `--json` for structured output.** Parse snapshot results programmatically when needed.
