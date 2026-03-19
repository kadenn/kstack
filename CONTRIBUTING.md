# Contributing to kstack

Thanks for wanting to make kstack better. Whether you're fixing a typo in a skill prompt or building an entirely new workflow, this guide will get you up and running fast.

## Quick start

kstack skills are Markdown files that Claude Code discovers from a `skills/` directory. Normally they live at `~/.claude/skills/kstack/` (your global install). But when you're developing kstack itself, you want Claude Code to use the skills *in your working tree* — so edits take effect instantly without copying or deploying anything.

That's what dev mode does. It symlinks your repo into the local `.claude/skills/` directory so Claude Code reads skills straight from your checkout.

```bash
git clone <repo> && cd kstack
bin/dev-setup                  # activate dev mode
```

Now edit any `SKILL.md`, invoke it in Claude Code (e.g. `/review`), and see your changes live. When you're done developing:

```bash
bin/dev-teardown               # deactivate — back to your global install
```

## How dev mode works

`bin/dev-setup` creates a `.claude/skills/` directory inside the repo (gitignored) and fills it with symlinks pointing back to your working tree. Claude Code sees the local `skills/` first, so your edits win over the global install.

```
kstack/                          <- your working tree
├── .claude/skills/              <- created by dev-setup (gitignored)
│   ├── kstack -> ../../         <- symlink back to repo root
│   ├── review -> kstack/review
│   ├── ship -> kstack/ship
│   └── ...                      <- one symlink per skill
├── review/
│   └── SKILL.md                 <- edit this, test with /review
├── ship/
│   └── SKILL.md
├── browse/
│   └── SKILL.md
└── ...
```

## Day-to-day workflow

```bash
# 1. Enter dev mode
bin/dev-setup

# 2. Edit a skill
vim review/SKILL.md

# 3. Test it in Claude Code — changes are live
#    > /review

# 4. Done for the day? Tear down
bin/dev-teardown
```

## Editing SKILL.md files

SKILL.md files are edited directly. No templates or generation step.

## Things to know

- **Dev mode shadows your global install.** Project-local skills take priority over `~/.claude/skills/kstack`. `bin/dev-teardown` restores the global one.
- **`.claude/skills/` is gitignored.** The symlinks never get committed.

## Testing a branch in another repo

When you're developing kstack in one workspace and want to test your branch in a
different project (e.g. testing browse changes against your real app), there are
two cases depending on how kstack is installed in that project.

### Global install only (no `.claude/skills/kstack/` in the project)

Point your global install at the branch:

```bash
cd ~/.claude/skills/kstack
git fetch origin
git checkout origin/<branch>
```

Now open Claude Code in the other project — it picks up skills from
`~/.claude/skills/` automatically. To go back to main when you're done:

```bash
cd ~/.claude/skills/kstack
git checkout main && git pull
```

### Vendored project copy (`.claude/skills/kstack/` checked into the project)

Some projects vendor kstack by copying it into the repo (no `.git` inside the
copy). Project-local skills take priority over global, so you need to update
the vendored copy too:

1. **Update your global install to the branch**:
   ```bash
   cd ~/.claude/skills/kstack
   git fetch origin
   git checkout origin/<branch>
   ```

2. **Replace the vendored copy** in the other project:
   ```bash
   cd /path/to/other-project

   # Remove old skill symlinks and vendored copy
   for s in browse plan-ceo-review plan-eng-review review ship retro qa socratic learn produce; do
     rm -f .claude/skills/$s
   done
   rm -rf .claude/skills/kstack

   # Copy from global install (strips .git so it stays vendored)
   cp -Rf ~/.claude/skills/kstack .claude/skills/kstack
   rm -rf .claude/skills/kstack/.git

   # Re-create skill symlinks
   cd .claude/skills/kstack && ./setup
   ```

3. **Test your changes** — open Claude Code in that project and use the skills.

To revert to main later, repeat steps 1-2 with `git checkout main && git pull`
instead of `git checkout origin/<branch>`.

## Shipping your changes

When you're happy with your skill edits:

```bash
/ship
```

This reviews the diff, bumps the version, and opens a PR. See `ship/SKILL.md` for the full workflow.
