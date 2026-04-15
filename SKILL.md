---
name: preflight
description: "Run pre-ship checklist for the current branch. Use when the user says /preflight, asks 'is this ready to ship?', 'run preflight', 'pre-flight checks', 'ready to merge?', or wants to validate their work before creating a PR."
allowed-tools: Bash, Read, Grep, Glob, Agent, Edit, Write, AskUserQuestion
argument-hint: "[--fix] [--section=testing,quality,...]"
---

# Preflight

Run a configurable pre-ship checklist against the current branch, comparing it to `main`. Output a structured pass/fail/warn report so the developer knows exactly what's left before opening a PR.

## Arguments

Parse `$ARGUMENTS` for:
- `--fix` flag: attempt to auto-fix issues where possible (run formatters, add missing lockfile entries, etc.)
- `--ship` flag: after all checks pass, push branch and create PR (full ship flow). If GSD planning artifacts exist (`.planning/`), auto-generate a rich PR body from ROADMAP.md, SUMMARY.md, VERIFICATION.md, and STATE.md.
- `--section=X,Y`: run only specific sections (comma-separated). Valid sections: `vcs`, `issues`, `testing`, `quality`, `dependencies`, `database`, `environment`, `documentation`, `api_collection`, `pr_hygiene`, `frontend`
- If no arguments, run all enabled sections

## Step 1: Load Configuration

Read `.claude/preflight.yaml` from the project root.

If the file does not exist, inform the user:

```
No .claude/preflight.yaml found in this project.

To get started, create one with your project's checks. Run:
  /preflight --init

Or see the schema reference: ~/.claude/skills/preflight/references/schema.yaml
```

Then stop (unless `--init` was passed, in which case generate a starter config by asking the user about their project).

## Step 2: Gather Context

Run these in parallel to build a picture of the current state:

1. **Git state**
   - `git branch --show-current` — current branch name
   - `git log main..HEAD --oneline` — commits on this branch
   - `git diff main...HEAD --stat` — files changed vs main
   - `git diff main...HEAD` — full diff for content analysis
   - `git status` — working tree status (unstaged/untracked)

2. **Changed file categorization**
   - Collect the list of all changed files (vs main)
   - Categorize by type: source, test, docs, config, migration, API, frontend

## Step 3: Run Checks

Execute each enabled section from the config. For each check, record one of:
- **PASS** — requirement met
- **FAIL** — requirement not met, blocks shipping
- **WARN** — potential issue, doesn't block but should be reviewed
- **SKIP** — check not applicable to this changeset

### Section: vcs (Version Control)

- Branch name matches `branch_pattern` regex
- Current branch is not in `protected_branches`
- All changes are committed (clean working tree)

### Section: issues (Issue Tracking)

The goal is to ensure the branch is linked to the correct existing ticket — **never create a new ticket** unless the user explicitly asks for one. Stale or orphaned tickets are worse than no ticket at all.

**Discovery**: find existing tickets using these strategies, in order of reliability:

1. **Branch name** — extract issue key using the configured `project_key` pattern (e.g., `AXN-123` from `axn-123/feature-name`)
2. **Commit messages** — scan `git log main..HEAD --oneline` for issue key references
3. **Planning files** — if GSD planning files exist (`.planning/`), search for `linear_issues`, issue keys, or milestone references in PLAN.md, ROADMAP.md, and SUMMARY.md frontmatter/content
4. **Tracker query** (if MCP tools for the configured tracker are available) — search for tickets matching the branch topic or phase name in the configured project. For Linear, use the `list_issues` or `search_issues` MCP tool filtered by project key and/or milestone name derived from the branch or planning context

**Checks**:

- PASS if an existing ticket is found and linked (via branch name or commits)
- WARN if a ticket was found via planning files or tracker query but is not referenced in the branch name or commits — suggest linking it
- WARN if the found ticket's status doesn't match the branch state (e.g., ticket is "Backlog" but branch has commits, or ticket is "Done" but branch isn't merged)
- FAIL if `require_linked_issue` is true and no ticket was found via any strategy

**Critical rule**: if a ticket is discovered through any strategy above, it MUST be surfaced in the report. The action plan must propose linking/updating that specific ticket — never propose creating a new one unless zero tickets were found across all strategies and the user confirms creation is desired

### Section: testing

- Run the configured `command` (e.g., `make check`) and verify it passes
- **Coverage enforcement** (if `coverage` is configured):
  - Run `coverage.command` to get current coverage percentage
  - Compare against `coverage.min_threshold` — FAIL if below
  - If `coverage.enforce_non_decrease` is true:
    - Get baseline coverage from main branch (stash changes, run coverage on main, restore)
    - FAIL if current coverage < baseline coverage
    - Report delta: "+X.X%" or "-X.X%"
  - If coverage command is not configured but `enforce_non_decrease` is true, WARN that coverage tracking is not set up
- **New tests required** (if `require_new_tests` is true):
  - Check if any test files were added or modified in the diff
  - If source files changed but no test files changed, WARN

### Section: quality (Code Quality)

- Run `lint_command` if configured — FAIL on non-zero exit
- Run `typecheck_command` if configured — FAIL on non-zero exit
- **Forbidden patterns**: scan the diff (added lines only, `git diff main...HEAD`) for each pattern in `forbidden_patterns`
  - For each match found, report the file, line, and configured message
  - Use the configured `severity` (default: `error` which means FAIL, `warn` means WARN)
- **Secrets detection** (if `no_secrets` is true):
  - Scan added lines in the diff for patterns that look like secrets:
    - API keys: strings matching common key formats (AWS, Stripe, etc.)
    - Connection strings with credentials
    - Private keys (BEGIN RSA/EC/PRIVATE KEY)
    - Hardcoded passwords in config-like assignments
  - FAIL if any potential secrets found

### Section: dependencies

- **Lockfile check** (if `lockfile_check` is true):
  - Check if any dependency manifest files changed (package.json, pyproject.toml, requirements.txt, Cargo.toml, go.mod, etc.)
  - If yes, verify that the corresponding file from `lockfiles` list was also modified
  - FAIL if deps changed but lockfile didn't
- **License check** (if `license_allowlist` is configured):
  - Check for newly added dependencies in the diff
  - WARN to verify licenses are compatible (list the new deps for manual review)

### Section: database

- **Migration check** (if `migration_check` is true):
  - Check if any files matching `model_paths` patterns changed
  - If yes, check if any new files were added under `migration_path`
  - WARN if models changed but no migration was added (it might be a non-schema change)
- **Reversible check** (if `reversible_check` is true):
  - For any new migration files, read them and check for a downgrade/reverse function
  - WARN if no downgrade path found

### Section: environment

- **Env example sync** (if `check_new_env_vars` is true):
  - Scan the diff for references to new environment variables (os.environ, process.env, env(), etc.)
  - Check if those variables are documented in `env_example_file`
  - WARN for any undocumented env vars
- **No hardcoded env values** (if `no_hardcoded_env` is true):
  - Scan added lines for patterns that look like hardcoded environment-specific values:
    - localhost URLs with ports
    - IP addresses
    - Hardcoded database connection strings
  - WARN for any matches (with file and line reference)

### Section: documentation

- **Required doc updates** (if `require_updates` is true):
  - For each entry in `files`, check if any changed files match the `when` glob pattern
  - If matched, check if the corresponding doc `path` was also modified
  - WARN if trigger files changed but docs weren't updated (list which docs may need updates)
- **Site config** (if `site_config` is configured):
  - If any new doc pages were added, check if they're referenced in the site config
  - WARN if new docs exist but aren't in the nav

### Section: api_collection

- Check if any files matching `trigger_paths` changed
- If yes, check if any files under `path` were modified
- FAIL if API source files changed but collection wasn't updated (configurable severity)

### Section: pr_hygiene

- **Diff size** (if `max_files_changed` is configured):
  - Count files changed vs main
  - WARN if over the threshold
- **Focused changes**: examine the diff categories — if changes span many unrelated areas, WARN
- **Description reminder**: remind user to write a meaningful PR description

### Section: frontend

Only run if `frontend.enabled` is true AND frontend files are in the diff.

- **Bundle size** (if `bundle_size_check` is true):
  - WARN to verify bundle size impact (suggest running build and comparing)
- **Accessibility** (if `accessibility` is configured):
  - If `check_alt_text`: scan added JSX/HTML for `<img` tags without `alt` attributes — WARN
  - If `check_keyboard_nav`: scan for new interactive elements (onClick on non-button/anchor) without keyboard handlers — WARN

## Step 4: Generate Report

Output a clean, scannable report grouped by section:

```
## Preflight Report: <branch-name>
<commits-count> commits, <files-count> files changed vs main

### Version Control
  PASS  Branch matches pattern: axn-123/feature-name
  PASS  Not on protected branch
  PASS  Working tree clean

### Testing
  PASS  make check passed
  PASS  Coverage: 87.2% (min: 80%) [+1.3% vs main]
  WARN  No new test files in diff — verify coverage is adequate

### Code Quality
  PASS  Lint passed
  PASS  Typecheck passed
  FAIL  Forbidden pattern found:
        src/api/handler.py:42 — console.log (Remove debug statements)
  PASS  No secrets detected

### Documentation
  WARN  API files changed but docs/api/endpoints.md not updated
  SKIP  No new doc pages added

---
Summary: 11 PASS | 2 WARN | 1 FAIL

FAIL items must be resolved before shipping.
WARN items should be reviewed — dismiss with justification if intentional.
```

## Step 5: Propose Action Plan

If there are any FAIL or WARN items, **do not stop at the report**. Immediately analyze the failures and build a concrete action plan.

### 5a: Classify each issue

For every FAIL and WARN, classify it as one of:

- **Auto-fixable**: you can fix it right now without changing business logic. Examples:
  - Forbidden patterns like `console.log` or `debugger` — remove them
  - Lint/format errors — run the configured formatter/linter with `--fix`
  - Missing lockfile update — run the package manager's install/lock command
  - Missing `alt` attributes on `<img>` tags — add empty `alt=""` or descriptive text based on context
  - Uncommitted changes — stage and commit them

- **Assistable**: you can do meaningful work toward a fix, but it requires judgment or the user should review. Examples:
  - Missing tests — generate test stubs or full tests for changed code
  - Missing documentation updates — draft updates to the relevant doc files
  - Missing migration — generate a migration skeleton
  - Undocumented env vars — add entries to `.env.example`
  - Coverage decrease — identify uncovered lines and write tests
  - Issue ticket found but not linked — rename branch to include the issue key, or update ticket status via MCP tools
  - Issue ticket status stale — update to match branch state (e.g., move to "In Progress" or "Done") via MCP tools

- **Manual**: you cannot fix it, only the user can. Examples:
  - Secrets detected in code — the user must decide the right approach (env var, vault, etc.)
  - Branch naming violation (non-issue-related) — the user must rename the branch
  - PR too large — the user must decide how to split it
  - No existing ticket found and user wants to create one — confirm scope and details before creating

### 5b: Present the plan

Output the action plan grouped by type, with specific details:

```
## Action Plan

### Auto-fix (I'll handle these)
1. Remove `console.log` at src/api/handler.py:42
2. Run `npm run lint -- --fix` to fix formatting issues
3. Run `npm install` to update package-lock.json

### Assist (I'll draft, you review)
4. Write tests for src/api/handler.py (new endpoint, no test coverage)
5. Update docs/api/endpoints.md with new POST /users route

### Manual (you need to handle these)
6. Rename branch from `fix-stuff` to match pattern `^feature/`
7. Remove hardcoded API key at src/config.ts:15 — move to environment variable

Shall I proceed with items 1-5? (y/n, or specify item numbers)
```

### 5c: Wait for confirmation

**Always ask the user before executing any action.** Use `AskUserQuestion` to get explicit confirmation. The user can:
- Confirm all proposed actions: proceed with everything
- Select specific items by number: only execute those
- Decline: stop, the report is informational only
- Ask questions about any item before deciding

### 5d: Execute approved actions

Once confirmed, execute the approved actions:

1. **Auto-fix items**: execute them directly, one at a time, showing what changed
2. **Assistable items**: generate the content (tests, docs, migrations, etc.) and present a brief summary of what was created/modified
3. After all actions are complete, **re-run only the checks that had failures** to verify the fixes worked
4. Present an updated mini-report showing the before/after status:

```
## Fix Results

  FIXED  Removed console.log at src/api/handler.py:42
  FIXED  Lint auto-fix applied (3 files formatted)
  FIXED  package-lock.json updated
  DONE   Tests written for src/api/handler.py (2 test cases)
  DONE   docs/api/endpoints.md updated with POST /users

Re-check: 4 PASS | 0 WARN | 0 FAIL (was: 2 WARN | 1 FAIL)

Remaining manual items:
  6. Rename branch from `fix-stuff` to match pattern `^feature/`
```

## Step 6: All Clear — Next Steps

If all checks pass (either initially or after fixes):

**If `--ship` flag was passed:** Proceed directly to Step 7 (Ship Flow) without asking.

**Otherwise:** Proactively suggest the logical next step:

```
All preflight checks passed! Ready to ship.

Next: shall I create the PR?
```

If the user confirms, proceed to Step 7. If the user declines, stop here.

## Step 7: Ship Flow

Push branch and create PR. This step runs when `--ship` was passed or the user confirmed PR creation.

### 7a: Verify shipping prerequisites

1. **On correct branch?**
   ```bash
   CURRENT_BRANCH=$(git branch --show-current)
   BASE_BRANCH=$(git symbolic-ref refs/remotes/origin/HEAD 2>/dev/null | sed 's|^refs/remotes/origin/||')
   BASE_BRANCH="${BASE_BRANCH:-main}"
   ```
   If on `${BASE_BRANCH}`: warn — should be on a feature branch. Ask user to confirm.

2. **Remote configured?**
   ```bash
   git remote -v | head -2
   ```
   If no remote: error — can't create PR.

3. **`gh` CLI available?**
   ```bash
   which gh && gh auth status 2>&1
   ```
   If `gh` not found or not authenticated: provide setup instructions and exit.

### 7b: Push branch

```bash
git push --set-upstream origin ${CURRENT_BRANCH} 2>&1
```

Report: "Pushed `{branch}` to origin ({commit_count} commits ahead of ${BASE_BRANCH})"

### 7c: Generate PR body

Check for GSD planning artifacts to generate a rich PR body:
```bash
ls .planning/ROADMAP.md 2>/dev/null
```

**If GSD planning artifacts exist (`.planning/` directory):**

Read ROADMAP.md for phase goal, VERIFICATION.md for verification status, SUMMARY.md files for change descriptions. Generate a rich body:

```markdown
## Summary

**Goal:** {goal from ROADMAP.md}
**Status:** {Verified / Gaps found from VERIFICATION.md}

{One paragraph synthesized from SUMMARY.md files — what was built}

## Changes

{For each SUMMARY.md: plan name, one-liner, key files}

## Verification

- [x] Preflight checks passed
- {verification items from VERIFICATION.md, if available}
```

**If no GSD planning artifacts:** Draft a simpler body from the branch diff and commit messages gathered in Step 2.

### 7d: Create PR

```bash
gh pr create \
  --title "${PR_TITLE}" \
  --body "${PR_BODY}" \
  --base ${BASE_BRANCH}
```

Report: "PR #{number} created: {url}"

## Important Behavioral Notes

- **Be proactive, not passive**. The goal is to get the branch ready to ship, not just to report problems. Always propose fixes and offer to execute them.
- **Never run destructive commands**. Auto-fixes must be safe and reversible — formatting, removing debug statements, updating lockfiles. Never modify business logic without explicit user approval.
- **Always confirm before acting**. Present the plan, wait for user approval, then execute.
- **Coverage comparison**: if checking coverage on main requires switching branches, use `git stash` + `git checkout main` + run coverage + `git checkout -` + `git stash pop`. Always restore the working tree.
- **Speed**: run independent checks in parallel where possible (e.g., lint and typecheck can run simultaneously).
- **--fix mode**: when passed explicitly, skip the confirmation step for auto-fixable items and execute them immediately. Still ask for confirmation on assistable items.
- **Respect .gitignore**: don't flag patterns in ignored files.
- **Only scan added lines**: when checking forbidden patterns and secrets, only look at lines added in the diff (lines starting with `+`), not removed lines or context.
- **Never create new tracker tickets unless explicitly asked**. When the issues check finds no ticket, exhaust all discovery strategies (branch name, commits, planning files, tracker MCP query) before concluding there is no ticket. If no ticket is found, report it and ask the user — do not auto-create. Orphaned tickets from unnecessary creation cause more harm than a missing link.
