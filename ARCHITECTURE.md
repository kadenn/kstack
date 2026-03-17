# Architecture

This document explains **why** kstack is built the way it is. For setup and commands, see CLAUDE.md. For contributing, see CONTRIBUTING.md.

## The core idea

kstack gives Claude Code a persistent browser and a set of opinionated workflow skills. The browser is powered by [agent-browser](https://github.com/vercel-labs/agent-browser), a native Rust headless browser CLI optimized for AI agents.

The key insight: an AI agent interacting with a browser needs **sub-second latency** and **persistent state**. If every command cold-starts a browser, you're waiting 3-5 seconds per tool call. If the browser dies between commands, you lose cookies, tabs, and login sessions. agent-browser solves this with a Rust daemon that keeps Chromium alive between commands.

```
Claude Code                     kstack
─────────                      ──────
                               ┌──────────────────────┐
  Tool call: $B snapshot -i    │  agent-browser CLI    │
  ─────────────────────────→   │  • Rust native binary │
                               │  • sends to daemon    │
                               └──────────┬───────────┘
                                          │ IPC/CDP
                               ┌──────────▼───────────┐
                               │  Rust daemon          │
                               │  • Chrome DevTools    │
                               │    Protocol (CDP)     │
                               │  • returns text/json  │
                               └──────────┬───────────┘
                                          │ CDP
                               ┌──────────▼───────────┐
                               │  Chromium (headless)   │
                               │  • persistent tabs     │
                               │  • cookies carry over  │
                               │  • auto idle timeout   │
                               └───────────────────────┘
```

First call starts everything (~2s). Every call after: ~100ms.

## Why agent-browser

We previously built a custom Playwright-based browser server. We replaced it with agent-browser for several reasons:

1. **99x smaller install.** 7 MB vs 710 MB with Playwright. Less disk, faster setup, fewer moving parts.

2. **18x less memory.** 8 MB vs 143 MB. Matters in CI and constrained environments.

3. **Native Rust.** No Node.js runtime needed for the daemon. The CLI is a compiled Rust binary with ~1ms startup time.

4. **Direct CDP.** Communicates with Chrome via the DevTools Protocol directly, skipping the Node.js → Playwright → CDP indirection layer.

5. **AI-optimized.** Built specifically for LLM agent workflows: snapshot with @refs, semantic element finders (`find role`, `find text`, `find label`), content boundaries for safety, domain allowlists.

6. **Active maintenance.** Maintained by Vercel Labs with a clear focus on the AI agent use case.

## The daemon model

### Why not start a browser per command?

Launching Chromium takes ~2-3 seconds. For a single screenshot, that's fine. For a QA session with 20+ commands, it's 40+ seconds of browser startup overhead. Worse: you lose all state between commands. Cookies, localStorage, login sessions, open tabs — all gone.

The daemon model means:

- **Persistent state.** Log in once, stay logged in. Open a tab, it stays open. localStorage persists across commands.
- **Sub-second commands.** After the first call, every command is ~100ms round-trip.
- **Automatic lifecycle.** The daemon auto-starts on first use, auto-shuts down after idle timeout. No process management needed.
- **Session isolation.** Use `--session <name>` for multiple parallel browser instances.

## Security model

agent-browser provides built-in security features:

- **Domain allowlists** (`--allowed-domains`): restrict which URLs the browser can navigate to
- **Content boundaries** (`--content-boundaries`): mark untrusted page content for LLM safety
- **Action policies** (`--action-policy`): gate destructive browser operations
- **Session isolation**: multiple parallel sessions with `--session` flag
- **Encrypted state**: auth state files with AES-GCM encryption

## The ref system

Refs (`@e1`, `@e2`) are how the agent addresses page elements without writing CSS selectors or XPath.

### How it works

```
1. Agent runs: $B snapshot -i
2. agent-browser builds accessibility tree from CDP
3. Parser walks the tree, assigns sequential refs: @e1, @e2, @e3...
4. Returns the annotated tree as plain text
5. Refs are deterministic — same page state = same refs

Later:
6. Agent runs: $B click @e3
7. agent-browser resolves @e3 → element → click
```

### Semantic finders

In addition to @refs from snapshot, agent-browser provides semantic element finders:

- `find role <role> <action>` — find by ARIA role
- `find text <text> <action>` — find by visible text
- `find label <label> <action>` — find by label
- `find placeholder <ph> <action>` — find by placeholder text
- `find testid <id> <action>` — find by data-testid

These are useful when you know what you're looking for without needing a full snapshot.

### Ref lifecycle

Refs are cleared on navigation. After navigating to a new page, run `snapshot` again to get fresh refs. Stale refs fail loudly rather than clicking the wrong element.

## Command dispatch

Commands in the registry are categorized by side effects:

- **READ** (get, snapshot, cookies, console, errors, ...): No mutations. Safe to retry. Returns page state.
- **WRITE** (open, click, fill, press, ...): Mutates page state. Not idempotent.
- **META** (screenshot, tab, diff, ...): Visual/structural operations.

This categorization helps structure the SKILL.md documentation and command reference.

## Error philosophy

Errors are for AI agents, not humans. agent-browser provides clear, actionable error messages. The agent should be able to read the error and know what to do next without human intervention.

## What's intentionally not here

- **No custom browser engine.** We use agent-browser rather than building our own. Focus on the skill layer, not browser plumbing.
- **No MCP protocol.** Plain CLI + plain text output is lighter on tokens and easier to debug than MCP's JSON schema overhead.
- **No multi-user support.** One session per workspace. Session isolation via `--session` flag handles parallel use cases.
