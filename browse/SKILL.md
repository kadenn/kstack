---
name: browse
version: 2.0.0
description: |
  Fast headless browser for QA testing and site dogfooding. Navigate any URL, interact with
  elements, verify page state, diff before/after actions, take annotated screenshots, check
  responsive layouts, test forms and uploads, handle dialogs, and assert element states.
  Powered by agent-browser (native Rust CLI, ~7 MB, ~8 MB RAM). Use when you need to test a
  feature, verify a deployment, dogfood a user flow, or file a bug with evidence.
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

# browse: QA Testing & Dogfooding

Powered by [agent-browser](https://github.com/vercel-labs/agent-browser) — native Rust headless browser CLI.
First call auto-starts the daemon, then ~100ms per command.
State persists between calls (cookies, tabs, login sessions).

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

## Selectors

### Refs (recommended for AI)
Snapshot gives you @e refs — use them everywhere:
```bash
$B snapshot -i                        # get refs
$B click @e2                          # click by ref
$B fill @e3 "test@example.com"        # fill by ref
$B get text @e1                       # read by ref
```

### Other selector types
```bash
$B click "#id"                        # CSS selector
$B click ".class"                     # CSS class
$B click "text=Submit"                # text content
$B click "xpath=//button"             # XPath
$B find role button click --name "Submit"  # semantic locator
$B find label "Email" fill "test@test.com" # by label
```

## Core QA Patterns

### 1. Verify a page loads correctly
```bash
$B open https://yourapp.com
$B snapshot -i -c                     # interactive elements, compact
$B console                            # JS errors?
$B network requests                   # failed requests?
$B is visible ".main-content"         # key elements present?
```

### 2. Test a user flow
```bash
$B open https://app.com/login
$B snapshot -i                        # see all interactive elements
$B fill @e3 "user@test.com"
$B fill @e4 "password"
$B click @e5                          # submit
$B wait --load networkidle            # wait for page to settle
$B diff snapshot                      # what changed after submit?
$B is visible ".dashboard"            # success state present?
```

### 3. Verify an action worked
```bash
$B snapshot                           # baseline
$B click @e3                          # do something
$B diff snapshot                      # unified diff shows exactly what changed
```

### 4. Visual evidence for bug reports
```bash
$B screenshot --annotate /tmp/annotated.png  # labeled screenshot with [N] refs
$B screenshot /tmp/bug.png                   # plain screenshot
$B screenshot --full /tmp/full.png           # full page screenshot
$B console                                   # error log
```

### 5. Assert element states
```bash
$B is visible ".modal"
$B is enabled "#submit-btn"
$B is checked "#agree-checkbox"
```

### 6. Test responsive layouts
```bash
$B set viewport 375 812              # mobile
$B screenshot /tmp/mobile.png
$B set viewport 768 1024             # tablet
$B screenshot /tmp/tablet.png
$B set viewport 1440 900             # desktop
$B screenshot /tmp/desktop.png
```

### 7. Test file uploads
```bash
$B upload "#file-input" /path/to/file.pdf
$B is visible ".upload-success"
```

### 8. Test dialogs
```bash
$B dialog accept "yes"               # set up handler
$B click "#delete-button"            # trigger dialog
$B diff snapshot                     # verify deletion happened
```

### 9. Compare snapshots or URLs
```bash
$B snapshot                          # before
$B click @e3                         # action
$B diff snapshot                     # what changed

# Compare two different URLs
$B diff url https://v1.com https://v2.com
$B diff url https://v1.com https://v2.com --screenshot  # visual diff too
```

### 10. Run JavaScript
```bash
$B eval "document.title"
$B eval "window.scrollY"
$B eval "document.querySelectorAll('a').length"
```

## Snapshot Flags

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

## Authentication & Sessions

### Self-signed certs (local dev)
```bash
$B close                              # must close first to change launch options
# agent-browser --ignore-https-errors open https://local.myapp.com:3000
```

### Persistent profile (full browser state across restarts)
```bash
# agent-browser --profile ~/.myapp-profile open https://myapp.com
# Login once — cookies, IndexedDB, cache persist in the profile dir
# agent-browser --profile ~/.myapp-profile open https://myapp.com/dashboard
```

### Session persistence (auto-save/restore cookies + localStorage)
```bash
# agent-browser --session-name myapp open https://myapp.com
# State auto-saved to ~/.agent-browser/sessions/
```

### Save/load auth state
```bash
$B state save ./auth.json             # save cookies + storage to file
# agent-browser --state ./auth.json open https://myapp.com  (load on launch)
$B state list                          # list saved state files
```

### Connect to your real Chrome
```bash
# 1. Launch Chrome with: /Applications/Google\ Chrome.app/Contents/MacOS/Google\ Chrome --remote-debugging-port=9222
# 2. Connect:
$B connect 9222
$B snapshot                            # you're in your logged-in Chrome!

# Or auto-discover running Chrome:
# agent-browser --auto-connect snapshot
```

### HTTP auth / headers
```bash
$B set credentials admin secret123     # HTTP basic auth
$B open https://api.example.com --headers '{"Authorization": "Bearer <token>"}'
```

### Isolated sessions
```bash
# agent-browser --session agent1 open site-a.com  (own browser, cookies, history)
# agent-browser --session agent2 open site-b.com
$B session list                        # list active sessions
```

## Global Options

Key flags (pass before the command):

| Flag | Description |
|------|-------------|
| `--ignore-https-errors` | Accept self-signed certs (must `close` first to take effect) |
| `--profile <path>` | Persistent browser profile dir (cookies, IndexedDB, cache) |
| `--session-name <name>` | Auto-save/restore cookies + localStorage by name |
| `--session <name>` | Isolated browser session |
| `--state <path>` | Load auth state JSON on launch |
| `--headed` | Show browser window (not headless) |
| `--cdp <port\|url>` | Connect via Chrome DevTools Protocol |
| `--auto-connect` | Auto-discover running Chrome |
| `--json` | Machine-readable JSON output |
| `--args <args>` | Extra Chrome launch args (comma-separated) |
| `--user-agent <ua>` | Custom User-Agent string |
| `--proxy <url>` | Proxy server URL |
| `--extension <path>` | Load browser extension (repeatable) |
| `--allowed-domains <list>` | Restrict navigation to trusted domains |
| `--content-boundaries` | Wrap output in boundary markers (LLM safety) |
| `--max-output <chars>` | Truncate page output |
| `--annotate` | Annotated screenshots with numbered element labels |

## Full Command List

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
