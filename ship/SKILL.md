---
name: ship
version: 1.0.0
description: |
  Ship workflow: merge main, run tests, review diff, bump VERSION, update CHANGELOG, commit, push, create PR.
allowed-tools:
  - Bash
  - Read
  - Write
  - Edit
  - Grep
  - Glob
  - AskUserQuestion
---

## Update Check (run first)

```bash
_UPD=$(~/.claude/skills/kstack/bin/kstack-update-check 2>/dev/null || .claude/skills/kstack/bin/kstack-update-check 2>/dev/null || true)
[ -n "$_UPD" ] && echo "$_UPD"
```

If output shows `UPGRADE_AVAILABLE <old> <new>`: read `~/.claude/skills/kstack/kstack-upgrade/SKILL.md` and follow the "Inline upgrade flow" (AskUserQuestion → upgrade if yes, `touch ~/.kstack/last-update-check` if no). If `JUST_UPGRADED <from> <to>`: tell user "Running kstack v{to} (just updated!)" and continue.

# Ship: Fully Automated Ship Workflow

You are running the `/ship` workflow. This is a **non-interactive, fully automated** workflow. Do NOT ask for confirmation at any step. The user said `/ship` which means DO IT. Run straight through and output the PR URL at the end.

**Only stop for:**
- On `main` branch (abort)
- Merge conflicts that can't be auto-resolved (stop, show conflicts)
- Test failures (stop, show failures)
- Pre-landing review finds CRITICAL issues and user chooses to fix (not acknowledge or skip)
- MINOR or MAJOR version bump needed (ask — see Step 4)
- Greptile review comments that need user decision (complex fixes, false positives)

**Never stop for:**
- Uncommitted changes (always include them)
- Version bump choice (auto-pick MICRO or PATCH — see Step 4)
- CHANGELOG content (auto-generate from diff)
- Commit message approval (auto-commit)
- Multi-file changesets (auto-split into bisectable commits)

---

## Step 1: Pre-flight

1. Check the current branch. If on `main`, **abort**: "You're on main. Ship from a feature branch."

2. Run `git status` (never use `-uall`). Uncommitted changes are always included — no need to ask.

3. Run `git diff main...HEAD --stat` and `git log main..HEAD --oneline` to understand what's being shipped.

---

## Step 2: Merge origin/main (BEFORE tests)

Fetch and merge `origin/main` into the feature branch so tests run against the merged state:

```bash
git fetch origin main && git merge origin/main --no-edit
```

**If there are merge conflicts:** Try to auto-resolve if they are simple (VERSION, schema.rb, CHANGELOG ordering). If conflicts are complex or ambiguous, **STOP** and show them.

**If already up to date:** Continue silently.

---

## Step 3: Run tests (on merged code)

First, detect what test commands are available in this project:

```bash
# Check for project-level test script hints
cat package.json 2>/dev/null | grep -A5 '"scripts"'
cat Makefile 2>/dev/null | grep -E "^test"
ls bin/test* 2>/dev/null
ls pytest.ini setup.cfg pyproject.toml 2>/dev/null
```

**Detection rules (in priority order):**
- `CLAUDE.md` or `README.md` mentions a test command → use that
- `bin/test-lane` exists → run it
- `package.json` has a `"test"` script → run `npm test` (or `yarn test` / `bun test` based on lockfile)
- `pytest.ini` or `pyproject.toml` with `[tool.pytest]` → run `pytest`
- `go.mod` exists → run `go test ./...`
- `Makefile` has a `test` target → run `make test`
- Multiple detected → run all in parallel

Run detected test commands, capturing output:

```bash
# Example — adapt based on what's detected above
<detected-test-command> 2>&1 | tee /tmp/ship_tests.txt
```

After completing, read the output and check pass/fail.

**If any test fails:** Show the failures and **STOP**. Do not proceed.

**If no test command is found:** Note "No test command detected — skipping tests." and continue.

**If all pass:** Continue silently — just note the counts briefly.

---

## Step 3.25: Eval Suites (conditional)

Run LLM evals when AI-related files change. Skip entirely if no AI-related files are in the diff.

**1. Check CLAUDE.md for project-specific eval instructions:**

```bash
cat CLAUDE.md 2>/dev/null | grep -A 10 -i "eval"
```

If CLAUDE.md defines eval patterns, eval commands, or file triggers — **use those exclusively** and skip the auto-detection below.

**2. Detect changed AI-related files:**

```bash
git diff origin/main --name-only
```

A file is AI-related if it matches any of:
- Name contains: `prompt`, `eval`, `llm`, `ai`, `generation`, `completion`
- Extension is `.txt` or `.md` inside a directory named `prompts/`, `system_prompts/`, or `instructions/`
- Content contains LLM API calls — grep changed files:
  ```bash
  git diff origin/main --name-only | xargs grep -l "openai\|anthropic\|claude\|gpt\|completion\|ChatCompletion" 2>/dev/null
  ```

**If no AI-related files found:** Print "No AI-related files changed — skipping evals." and continue to Step 3.5.

**3. Find eval test files for changed files:**

Look for eval tests in this order:
```bash
# Find files named with eval patterns near the changed files
find . -name "*eval*" -o -name "*llm_test*" -o -name "*ai_test*" | grep -v node_modules | grep -v .git

# Check common eval directories
ls test/evals/ tests/evals/ evals/ eval/ 2>/dev/null
```

Match changed files to eval tests by name similarity (e.g. `tweet_prompt.txt` → `tweet_eval_test.*`).

**If no eval tests found:** Print "No eval tests found — skipping evals." and continue to Step 3.5.

**4. Detect eval run command:**

In priority order:
- CLAUDE.md mentions an eval command → use it
- `package.json` has an `"eval"` or `"test:eval"` script → `npm run eval`
- `Makefile` has an `eval` target → `make eval`
- Test files are `.rb` and `bin/test-lane` exists → `bin/test-lane --eval <file>`
- Test files are `.py` → `pytest <file> -v`
- Test files are `.ts`/`.js` → `npm test -- <file>`
- Fallback → run with the same test command detected in Step 3, scoped to eval files

**5. Run affected eval tests:**

```bash
<detected-eval-command> 2>&1 | tee /tmp/ship_evals.txt
```

Run sequentially. If the first suite fails, stop — don't burn API cost on remaining suites.

**6. Check results:**

- **If any eval fails:** Show the failures and **STOP**. Do not proceed.
- **If all pass:** Note pass counts. Continue to Step 3.5.
- **Save eval output** — include results in the PR body (Step 8).

**Tier reference (for context — /ship always uses `full`):**
| Tier | When | Speed (cached) | Cost |
|------|------|----------------|------|
| `fast` (Haiku) | Dev iteration, smoke tests | ~5s (14x faster) | ~$0.07/run |
| `standard` (Sonnet) | Default dev, `bin/test-lane --eval` | ~17s (4x faster) | ~$0.37/run |
| `full` (Opus persona) | **`/ship` and pre-merge** | ~72s (baseline) | ~$1.27/run |

---

## Step 3.5: Pre-Landing Review

Review the diff for structural issues that tests don't catch.

1. Read `.claude/skills/review/checklist.md`. If the file cannot be read, **STOP** and report the error.

2. Run `git diff origin/main` to get the full diff (scoped to feature changes against the freshly-fetched remote main).

3. Apply the review checklist in two passes:
   - **Pass 1 (CRITICAL):** SQL & Data Safety, LLM Output Trust Boundary
   - **Pass 2 (INFORMATIONAL):** All remaining categories

4. **Always output ALL findings** — both critical and informational. The user must see every issue found.

5. Output a summary header: `Pre-Landing Review: N issues (X critical, Y informational)`

6. **If CRITICAL issues found:** For EACH critical issue, use a separate AskUserQuestion with:
   - The problem (`file:line` + description)
   - Your recommended fix
   - Options: A) Fix it now (recommend), B) Acknowledge and ship anyway, C) It's a false positive — skip
   After resolving all critical issues: if the user chose A (fix) on any issue, apply the recommended fixes, then commit only the fixed files by name (`git add <fixed-files> && git commit -m "fix: apply pre-landing review fixes"`), then **STOP** and tell the user to run `/ship` again to re-test with the fixes applied. If the user chose only B (acknowledge) or C (false positive) on all issues, continue with Step 4.

7. **If only non-critical issues found:** Output them and continue. They will be included in the PR body at Step 8.

8. **If no issues found:** Output `Pre-Landing Review: No issues found.` and continue.

Save the review output — it goes into the PR body in Step 8.

---

## Step 3.75: Address Greptile review comments (if PR exists)

Read `.claude/skills/review/greptile-triage.md` and follow the fetch, filter, and classify steps.

**If no PR exists, `gh` fails, API returns an error, or there are zero Greptile comments:** Skip this step silently. Continue to Step 4.

**If Greptile comments are found:**

Include a Greptile summary in your output: `+ N Greptile comments (X valid, Y fixed, Z FP)`

For each classified comment:

**VALID & ACTIONABLE:** Use AskUserQuestion with:
- The comment (file:line or [top-level] + body summary + permalink URL)
- Your recommended fix
- Options: A) Fix now (recommended), B) Acknowledge and ship anyway, C) It's a false positive
- If user chooses A: apply the fix, commit the fixed files (`git add <fixed-files> && git commit -m "fix: address Greptile review — <brief description>"`), reply to the comment (`"Fixed in <commit-sha>."`), and save to `~/.kstack/greptile-history.md` (type: fix).
- If user chooses C: reply explaining the false positive, save to history (type: fp).

**VALID BUT ALREADY FIXED:** Reply acknowledging the catch — no AskUserQuestion needed:
- Post reply: `"Good catch — already fixed in <commit-sha>."`
- Save to `~/.kstack/greptile-history.md` (type: already-fixed)

**FALSE POSITIVE:** Use AskUserQuestion:
- Show the comment and why you think it's wrong (file:line or [top-level] + body summary + permalink URL)
- Options:
  - A) Reply to Greptile explaining the false positive (recommended if clearly wrong)
  - B) Fix it anyway (if trivial)
  - C) Ignore silently
- If user chooses A: post reply using the appropriate API from the triage doc, save to history (type: fp)

**SUPPRESSED:** Skip silently — these are known false positives from previous triage.

**After all comments are resolved:** If any fixes were applied, the tests from Step 3 are now stale. **Re-run tests** (Step 3) before continuing to Step 4. If no fixes were applied, continue to Step 4.

---

## Step 4: Version bump (auto-decide)

1. Read the current `VERSION` file (4-digit format: `MAJOR.MINOR.PATCH.MICRO`)

2. **Auto-decide the bump level based on the diff:**
   - Count lines changed (`git diff origin/main...HEAD --stat | tail -1`)
   - **MICRO** (4th digit): < 50 lines changed, trivial tweaks, typos, config
   - **PATCH** (3rd digit): 50+ lines changed, bug fixes, small-medium features
   - **MINOR** (2nd digit): **ASK the user** — only for major features or significant architectural changes
   - **MAJOR** (1st digit): **ASK the user** — only for milestones or breaking changes

3. Compute the new version:
   - Bumping a digit resets all digits to its right to 0
   - Example: `0.19.1.0` + PATCH → `0.19.2.0`

4. Write the new version to the `VERSION` file.

---

## Step 5: CHANGELOG (auto-generate)

1. Read `CHANGELOG.md` header to know the format.

2. Auto-generate the entry from **ALL commits on the branch** (not just recent ones):
   - Use `git log main..HEAD --oneline` to see every commit being shipped
   - Use `git diff main...HEAD` to see the full diff against main
   - The CHANGELOG entry must be comprehensive of ALL changes going into the PR
   - If existing CHANGELOG entries on the branch already cover some commits, replace them with one unified entry for the new version
   - Categorize changes into applicable sections:
     - `### Added` — new features
     - `### Changed` — changes to existing functionality
     - `### Fixed` — bug fixes
     - `### Removed` — removed features
   - Write concise, descriptive bullet points
   - Insert after the file header (line 5), dated today
   - Format: `## [X.Y.Z.W] - YYYY-MM-DD`

**Do NOT ask the user to describe changes.** Infer from the diff and commit history.

---

## Step 6: Commit (bisectable chunks)

**Goal:** Create small, logical commits that work well with `git bisect` and help LLMs understand what changed.

1. Analyze the diff and group changes into logical commits. Each commit should represent **one coherent change** — not one file, but one logical unit.

2. **Commit ordering** (earlier commits first):
   - **Infrastructure:** migrations, config changes, route additions
   - **Models & services:** new models, services, concerns (with their tests)
   - **Controllers & views:** controllers, views, JS/React components (with their tests)
   - **VERSION + CHANGELOG:** always in the final commit

3. **Rules for splitting:**
   - A model and its test file go in the same commit
   - A service and its test file go in the same commit
   - A controller, its views, and its test go in the same commit
   - Migrations are their own commit (or grouped with the model they support)
   - Config/route changes can group with the feature they enable
   - If the total diff is small (< 50 lines across < 4 files), a single commit is fine

4. **Each commit must be independently valid** — no broken imports, no references to code that doesn't exist yet. Order commits so dependencies come first.

5. Compose each commit message:
   - First line: `<type>: <summary>` (type = feat/fix/chore/refactor/docs)
   - Body: brief description of what this commit contains
   - Only the **final commit** (VERSION + CHANGELOG) gets the version tag and co-author trailer:

```bash
git commit -m "$(cat <<'EOF'
chore: bump version and changelog (vX.Y.Z.W)

Co-Authored-By: Claude Opus 4.6 <noreply@anthropic.com>
EOF
)"
```

---

## Step 7: Push

Push to the remote with upstream tracking:

```bash
git push -u origin <branch-name>
```

---

## Step 8: Create PR

Create a pull request using `gh`:

```bash
gh pr create --title "<type>: <summary>" --body "$(cat <<'EOF'
## Summary
<bullet points from CHANGELOG>

## Pre-Landing Review
<findings from Step 3.5, or "No issues found.">

## Eval Results
<If evals ran: suite names, pass/fail counts, cost dashboard summary. If skipped: "No prompt-related files changed — evals skipped.">

## Greptile Review
<If Greptile comments were found: bullet list with [FIXED] / [FALSE POSITIVE] / [ALREADY FIXED] tag + one-line summary per comment>
<If no Greptile comments found: "No Greptile comments.">
<If no PR existed during Step 3.75: omit this section entirely>

## Test plan
- [x] All Rails tests pass (N runs, 0 failures)
- [x] All Vitest tests pass (N tests)

🤖 Generated with [Claude Code](https://claude.com/claude-code)
EOF
)"
```

**Output the PR URL** — this should be the final output the user sees.

---

## Important Rules

- **Never skip tests.** If tests fail, stop.
- **Never skip the pre-landing review.** If checklist.md is unreadable, stop.
- **Never force push.** Use regular `git push` only.
- **Never ask for confirmation** except for MINOR/MAJOR version bumps and CRITICAL review findings (one AskUserQuestion per critical issue with fix recommendation).
- **Always use the 4-digit version format** from the VERSION file.
- **Date format in CHANGELOG:** `YYYY-MM-DD`
- **Split commits for bisectability** — each commit = one logical change.
- **The goal is: user says `/ship`, next thing they see is the review + PR URL.**
