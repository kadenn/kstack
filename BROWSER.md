# Browser — technical details

This document covers the command reference and internals of kstack's headless browser, powered by [agent-browser](https://github.com/vercel-labs/agent-browser).

## Command reference

| Category | Commands | What for |
|----------|----------|----------|
| Navigate | `open`, `back`, `forward`, `reload` | Get to a page |
| Read | `get text`, `get html`, `get value`, `get attr`, `get title`, `get url`, `get count`, `get styles` | Extract content |
| Snapshot | `snapshot [-i] [--json]` | Get refs for element selection |
| Interact | `click`, `fill`, `select`, `hover`, `type`, `press`, `scroll`, `wait`, `upload`, `check`, `uncheck`, `drag` | Use the page |
| Inspect | `is visible/enabled/checked`, `console`, `errors`, `network requests`, `cookies`, `storage` | Debug and verify |
| Visual | `screenshot [--full] [--annotate]`, `pdf` | See what Claude sees |
| Compare | `diff snapshot`, `diff screenshot --baseline <file>` | Spot differences |
| Dialogs | `dialog accept [text]`, `dialog dismiss` | Control alert/confirm/prompt handling |
| Tabs | `tab`, `tab new`, `tab <n>`, `tab close` | Multi-page workflows |
| Auth | `state save <path>`, `state load <path>` | Save/load auth state (cookies + storage) |
| Config | `set viewport`, `set device`, `set geo`, `set headers` | Configure browser |
| Semantic | `find role`, `find text`, `find label`, `find placeholder`, `find testid` | AI-optimized element finding |

All selector arguments accept CSS selectors or `@e` refs after `snapshot`. 50+ commands total.

## How it works

agent-browser is a native Rust headless browser CLI that communicates with Chrome via the Chrome DevTools Protocol (CDP). It uses a client-daemon architecture for fast repeated commands.

```
┌─────────────────────────────────────────────────────────────────┐
│  Claude Code                                                    │
│                                                                 │
│  "agent-browser open https://staging.myapp.com"                 │
│       │                                                         │
│       ▼                                                         │
│  ┌──────────┐                    ┌──────────────┐               │
│  │ agent-   │    IPC / CDP       │ Rust daemon  │               │
│  │ browser  │ ──────────────── → │              │               │
│  │ CLI      │                    │  Chrome      │──── Chromium  │
│  │          │ ◄──────────────    │  DevTools    │    (headless) │
│  │ Rust     │  text/json         │  Protocol    │               │
│  │ native   │                    │              │               │
│  └──────────┘                    └──────────────┘               │
│   ~1ms startup                    persistent daemon             │
│                                   auto-starts on first call     │
└─────────────────────────────────────────────────────────────────┘
```

### Key advantages over Playwright

| Metric | agent-browser | Playwright |
|--------|--------------|------------|
| Install size | ~7 MB | ~710 MB |
| Memory usage | ~8 MB | ~143 MB |
| Cold start | 1.6x faster | Baseline |
| Runtime | Native Rust | Node.js |
| Protocol | Direct CDP | CDP via Node.js |

### Lifecycle

1. **First call**: CLI starts the Rust daemon which launches headless Chromium via CDP. This takes ~2 seconds.

2. **Subsequent calls**: CLI sends command to the running daemon. ~100ms round trip.

3. **Idle shutdown**: After configurable idle timeout, daemon shuts down. Next call restarts it.

4. **Sessions**: Use `--session <name>` for multiple parallel browser instances.

### The snapshot system

The browser's key feature for AI agents is ref-based element selection via the accessibility tree:

1. `snapshot` command returns the accessibility tree with `@e1`, `@e2`, ... refs
2. Use `snapshot -i` for interactive elements only (buttons, links, inputs)
3. Refs are deterministic IDs tied to accessibility tree nodes
4. Later commands like `click @e3` use these refs to target elements
5. No fragile CSS selectors needed — LLMs can reliably use @refs

**Diff support:**
- Run `snapshot` before an action, then `diff snapshot` after to see what changed

### Authentication

agent-browser supports:
- `state save <path>` / `state load <path>` — persist cookies + localStorage + sessionStorage
- `cookies set <name> <value>` — set individual cookies
- `--session-name <name>` — automatic cookie persistence per named session

### Environment variables

| Variable | Default | Description |
|----------|---------|-------------|
| `AGENT_BROWSER_IDLE_TIMEOUT_MS` | configurable | Idle shutdown timeout |
| `AGENT_BROWSER_STREAM_PORT` | none | WebSocket streaming port for visual output |

### Performance

| Tool | First call | Subsequent calls | Context overhead per call |
|------|-----------|-----------------|--------------------------|
| Chrome MCP | ~5s | ~2-5s | ~2000 tokens (schema + protocol) |
| Playwright MCP | ~3s | ~1-3s | ~1500 tokens (schema + protocol) |
| **agent-browser** | **~2s** | **~100ms** | **0 tokens** (plain text stdout) |

### Why CLI over MCP?

MCP (Model Context Protocol) works well for remote services, but for local browser automation it adds pure overhead:

- **Context bloat**: every MCP call includes full JSON schemas and protocol framing
- **Connection fragility**: persistent WebSocket/stdio connections drop and fail to reconnect
- **Unnecessary abstraction**: Claude Code already has a Bash tool. A CLI that prints to stdout is the simplest possible interface

agent-browser skips all of this. Native Rust binary. Plain text in, plain text out. No protocol overhead.

## Acknowledgments

The browser automation is powered by [agent-browser](https://github.com/vercel-labs/agent-browser) by Vercel Labs. It provides native Rust CDP-based browser control optimized specifically for AI agent workflows. Thank you to the Vercel team for building such a performant and well-designed tool.

## Development

### Prerequisites

- agent-browser (`npm install -g agent-browser && agent-browser install`)

### Source map

| File | Role |
|------|------|
| `browse/bin/find-browse` | Shell shim to locate the agent-browser binary. |
| `browse/SKILL.md` | Browse skill docs — edit directly. |

### Deploying to the active skill

The active skill lives at `~/.claude/skills/kstack/`. After making changes:

1. Push your branch
2. Pull in the skill directory: `cd ~/.claude/skills/kstack && git pull`
