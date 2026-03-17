# kstack development

## Commands

```bash
./setup              # one-time setup: install agent-browser + symlink skills
```

## Project structure

```
kstack/
├── browse/          # Browser automation (powered by agent-browser)
│   ├── bin/         # find-browse shim
│   └── SKILL.md     # Browse skill docs (edit directly)
├── ship/            # Ship workflow skill
├── review/          # PR review skill
├── plan-ceo-review/ # /plan-ceo-review skill
├── plan-eng-review/ # /plan-eng-review skill
├── retro/           # Retrospective skill
├── qa/              # QA testing skill
├── setup            # One-time setup: install agent-browser + symlink skills
└── SKILL.md         # Main skill docs (edit directly)
```

## Browser engine

The browser is powered by [agent-browser](https://github.com/vercel-labs/agent-browser),
a native Rust headless browser CLI optimized for AI agents. It provides:
- ~99x smaller install vs Playwright (7 MB vs 710 MB)
- ~18x less memory (8 MB vs 143 MB)
- Snapshot with @ref element selection (same pattern as before)
- Native CDP (Chrome DevTools Protocol) — no Node.js runtime needed for the daemon

## SKILL.md workflow

SKILL.md files are edited directly. No generation or templates.

## Browser interaction

When you need to interact with a browser (QA, dogfooding, auth state), use the
`/browse` skill or run `agent-browser` directly via `$B <command>`. NEVER use
`mcp__claude-in-chrome__*` tools — they are slow, unreliable, and not what this
project uses.

## Deploying to the active skill

The active skill lives at `~/.claude/skills/kstack/`. After making changes:

1. Push your branch
2. Fetch and reset in the skill directory: `cd ~/.claude/skills/kstack && git fetch origin && git reset --hard origin/main`
