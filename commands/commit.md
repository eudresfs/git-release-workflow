---
description: Create safe, conventional commits from the current staged snapshot
allowed-tools: Bash(git add:*), Bash(git status:*), Bash(git commit:*), Bash(git diff:*), Bash(git log:*), Bash(git push:*), Bash(git reset:*), Bash(git rev-parse:*), Bash(git grep:*), Bash(node:*), Bash(pnpm:*), Bash(npm:*), Bash(bun:*), Bash(cat:*), Bash(diff:*), Bash(rg:*), Bash(mktemp:*), AskUserQuestion
---

# /commit

Create one or more **safe, conventional commits** from the **current staged snapshot**.

This command is designed to avoid accidental inclusion of unrelated or unstaged changes. It plans commits from what is staged, validates risk conditions, optionally runs project lint, and only then creates commits.

## Behavior Summary

* Uses the **staged snapshot** as the source of truth
* Never silently pulls additional unstaged changes into scope
* Supports:

  * single commit mode
  * safe file-level split into multiple commits
  * limited hunk-level splitting only when it is actually safe
* Uses **Conventional Commits**
* Can optionally push after commit creation
* Refuses unsafe automation when staged and unstaged changes overlap in the same file

## Flags

* `--skip-lint`: skip command-level lint
* `--no-verify`: alias for `--skip-lint` for backward compatibility, but do **not** pass `--no-verify` to `git commit`
* `--all-in-one`: create a single commit from the full staged set
* `--push`: ask to push after successful commit creation

## Pre-flight Checks

Abort with a clear message if any of these fail:

* current directory is inside a git repository
* repository root can be resolved with `git rev-parse --show-toplevel`
* repository has at least one staged change before commit creation begins

Determine:

```bash
REPO_ROOT="$(git rev-parse --show-toplevel)"
CURRENT_BRANCH="$(git -C "$REPO_ROOT" branch --show-current)"
```

## Setup Prerequisite Check

This command expects release/commit tooling to already exist.

Check for these files at the repository root:

* `commitlint.config.js`
* `.husky/commit-msg`

If either is missing, stop and present:

```text
This repository is missing commit hook setup.
Run /setup-release first, then re-run /commit.
```

## Package Manager Detection

Detect the package manager once and use it consistently for this run.

Detection order:

1. `pnpm-lock.yaml` → `pnpm`
2. `bun.lockb` → `bun`
3. `package-lock.json` → `npm`
4. `package.json.packageManager`, if present
5. otherwise: unknown

If package manager is unknown, skip command-level lint and continue with commit planning.

## Command-level Lint

Unless `--skip-lint` or `--no-verify` is passed, run exactly one lint command based on detected package manager:

* pnpm:

  ```bash
  pnpm lint
  ```
* bun:

  ```bash
  bun run lint
  ```
* npm:

  ```bash
  npm run lint
  ```

If `package.json` has no `lint` script, print:

```text
No lint script found — skipping command-level lint.
```

Do not use fallback chains like `pnpm lint || npm run lint`.

If lint fails, use `AskUserQuestion` with header "Lint" and options:

* **Fix and re-run** — "Fix issues and re-run /commit"
* **Continue anyway** — "Proceed despite lint failures"
* **Abort** — "Cancel the commit"

Only continue if the user explicitly chooses to continue.

## Scope of Changes

This command treats the **staged snapshot** as the exact content in scope.

### If files are already staged

* Use only the staged snapshot
* Do not auto-stage additional files
* Do not expand scope silently

### If nothing is staged

List modified and untracked files:

```bash
git -C "$REPO_ROOT" status --short
```

Then use `AskUserQuestion` with header "Staging" and options:

* **Stage all** — "Stage all listed files and continue"
* **Stage manually** — "Let me stage files manually, then re-run"
* **Abort** — "Cancel the commit"

If the user chooses "Stage all", run:

```bash
git -C "$REPO_ROOT" add -A
```

Then re-read the staged diff and continue.

If, after this step, there is still no staged content, stop.

## Mixed Staged/Unstaged Overlap Gate

Before planning a split, compare:

```bash
git -C "$REPO_ROOT" diff --cached --name-only
git -C "$REPO_ROOT" diff --name-only
```

If any file path appears in both outputs, that file contains both staged and unstaged changes.

In that case, automatic split is unsafe because re-staging from the working tree could accidentally include unstaged hunks.

Stop and use `AskUserQuestion` with header "Overlap" and options:

* **Fix manually** — "Finish staging/discarding changes in those files, then re-run"
* **Single commit** — "Commit the current staged snapshot as a single commit"
* **Abort** — "Cancel the commit"

If the user chooses "Single commit", force `--all-in-one` behavior for this run.

## Secrets Check

Run the secrets check after the final staged set is known and before any commit is created.

### Filename-based risk check

Inspect staged file paths:

```bash
git -C "$REPO_ROOT" diff --cached --name-only -z
```

Flag paths matching patterns such as:

* `.env`
* `.env.*` except `.env.example` and `.env.template`
* `*credentials*`
* `*secret*`
* `*private_key*`

### Content-based risk check

Inspect staged content from the index only:

```bash
git -C "$REPO_ROOT" grep --cached -n -I -E 'sk-|ghp_|xoxb-|AKIA' -- .
```

If a risky file or token-like pattern is detected, stop and use `AskUserQuestion` with header "Secret" and question mentioning the filename. Options:

* **Exclude file** — "Exclude this file and continue"
* **Review first** — "Review the file content before deciding"
* **Abort** — "Cancel the commit"

Do not continue until the risk is explicitly resolved.

## Commit Planning

Build the plan from the staged diff only:

```bash
git -C "$REPO_ROOT" diff --cached --stat
git -C "$REPO_ROOT" diff --cached
```

### Planning Rules

Split into multiple commits only when the staged snapshot can be separated safely.

Use **automatic file-level splitting** when changes are clearly independent by file or by tightly related file groups.

Use **hunk-level splitting** only when all of the following are true:

* the file is fully in scope for this run
* the file has no additional unstaged hunks
* interactive patch mode is actually supported
* the split is simple enough to perform reliably

If those conditions are not true, do not pretend to split by hunk.

Fall back in this order:

1. safe file-level split
2. `--all-in-one`
3. manual staging and re-run

### `--all-in-one`

If `--all-in-one` is passed, skip split planning and create exactly one commit from the full staged snapshot.

## Commit Message Rules

Generate messages using Conventional Commits:

```text
<type>(<scope>): <description>
```

Scope is optional when unnecessary:

```text
<type>: <description>
```

### Allowed Types

* `feat`
* `fix`
* `docs`
* `style`
* `refactor`
* `perf`
* `test`
* `chore`
* `ci`
* `revert`
* `db`

### Message Constraints

* imperative mood
* concise and specific
* no trailing period
* header should fit within 72 characters
* do not invent scope when no meaningful scope exists

### Examples

* `feat(auth): add refresh token rotation`
* `fix(api): handle null customer id`
* `docs: clarify release flow`
* `chore(eslint): align rules with ci`
* `db(migration): add user last_login column`

## Confirmation

Present the full commit plan as a text summary, then use `AskUserQuestion` with header "Commit" and options:

* **Confirm** — "Execute the commit plan as shown"
* **Edit plan** — "Let me adjust the plan before committing"
* **Abort** — "Cancel the commit"

Use the `preview` field on the "Confirm" option to show the commit plan (messages + files).

Wait for explicit confirmation before creating any commit.

## Commit Execution Workflow

For each planned commit:

### Step 1 — Unstage current index without discarding working tree changes

```bash
git -C "$REPO_ROOT" reset
```

### Step 2 — Stage only the current logical group

For file-level grouping:

```bash
git -C "$REPO_ROOT" add -- <files...>
```

For hunk-level grouping, only when safe:

```bash
cd "$REPO_ROOT" && git add -p -- <file>
```

### Step 3 — Refuse empty staged content

```bash
if git -C "$REPO_ROOT" diff --cached --quiet; then
  echo "Planned commit has no staged content. Stop and adjust the plan."
  exit 1
fi
```

### Step 4 — Create the commit

```bash
git -C "$REPO_ROOT" commit -m "<type>(<scope>): <description>"
```

### Step 5 — Repeat for remaining planned commits

Repeat until all planned commits are created.

### Step 6 — Show leftovers

After the last commit, show any remaining unstaged or untracked files:

```bash
git -C "$REPO_ROOT" status --short
```

Explain that these files were intentionally left out of scope.

## Push Behavior

If `--push` is passed, do not push automatically.

After successful commit creation, use `AskUserQuestion` with header "Push" and options:

* **Push** — "Push current branch to origin"
* **Skip** — "Do not push"

If the user chooses "Push", run:

```bash
git -C "$REPO_ROOT" push origin "$CURRENT_BRANCH"
```

## Safety Rules

* Never use unstaged content as implicit input to commit planning
* Never silently add unrelated files
* Never auto-split a file that contains both staged and unstaged changes
* Never claim hunk-level split succeeded unless the actual staged result matches the intended group
* Never create a commit without explicit confirmation of the plan
* Never pass `--no-verify` to `git commit`
* Never push automatically without explicit user approval

## Notes

* This command assumes commit hooks already exist
* Repository hooks validate actual commit acceptance at commit time
* This command generates commit plans and messages, but should not claim that every repository policy is enforced unless the repository hook truly enforces it
