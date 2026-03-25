---

description: Set up commit hooks and semantic-release for this repository
allowed-tools: Bash(git add:*), Bash(git status:*), Bash(git commit:*), Bash(git diff:*), Bash(git rev-parse:*), Bash(git branch:*), Bash(git symbolic-ref:*), Bash(cat:*), Bash(node:*), Bash(pnpm:*), Bash(npm:*), Bash(bun:*), Bash(mkdir:*), Bash(chmod:*), Bash(test:*), Bash(rg:*), Bash(ls:*)
----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------

# /setup-release

Set up **commit message validation** and a **semantic-release pipeline** for the current repository.

This command installs and configures:

* `commitlint`
* `husky`
* `semantic-release`
* a GitHub Actions release workflow
* release configuration files

It is designed to be **safe**, **idempotent**, and **compatibility-aware**.

## Behavior Summary

* Detects the package manager and uses it consistently
* Detects the repository release branch instead of assuming `main`
* Creates missing setup files
* Skips compatible files
* Refuses to silently overwrite incompatible files
* Stages only files touched by this setup
* Does not push automatically

## Pre-flight Checks

Abort with a clear message if any of these fail:

* current directory is inside a git repository
* repository root can be resolved with `git rev-parse --show-toplevel`
* `package.json` exists at the repository root

Determine:

```bash id="bvhvg4"
REPO_ROOT="$(git rev-parse --show-toplevel)"
CURRENT_BRANCH="$(git -C "$REPO_ROOT" branch --show-current)"
REMOTE_HEAD="$(git -C "$REPO_ROOT" symbolic-ref refs/remotes/origin/HEAD 2>/dev/null || true)"
RELEASE_BRANCH="${REMOTE_HEAD#refs/remotes/origin/}"
[ -n "$RELEASE_BRANCH" ] || RELEASE_BRANCH="$CURRENT_BRANCH"
```

If the current branch is not the detected release branch, present:

```text id="pc33zg"
You are not on the release branch.

Current branch: <current-branch>
Release branch: <release-branch>

How would you like to proceed?

  1. Continue on the current branch
  2. Abort
```

Only continue if the user explicitly chooses to continue.

## Package Manager Detection

Detect the package manager once and use it consistently for the entire run.

Detection order:

1. `pnpm-lock.yaml` → `pnpm`
2. `bun.lockb` → `bun`
3. `package-lock.json` → `npm`
4. `package.json.packageManager`, if present
5. otherwise: stop and ask the user to choose a package manager manually outside this command

## Dependencies to Install

Install these dev dependencies:

* `@commitlint/cli`
* `@commitlint/config-conventional`
* `husky`
* `semantic-release`
* `@semantic-release/commit-analyzer`
* `@semantic-release/release-notes-generator`
* `@semantic-release/changelog`
* `@semantic-release/git`
* `@semantic-release/github`

Use exactly one command based on the detected package manager.

### pnpm

```bash id="i9jw6o"
pnpm add -D @commitlint/cli @commitlint/config-conventional husky semantic-release @semantic-release/commit-analyzer @semantic-release/release-notes-generator @semantic-release/changelog @semantic-release/git @semantic-release/github
```

### bun

```bash id="9j9ri6"
bun add -d @commitlint/cli @commitlint/config-conventional husky semantic-release @semantic-release/commit-analyzer @semantic-release/release-notes-generator @semantic-release/changelog @semantic-release/git @semantic-release/github
```

### npm

```bash id="0dg9zt"
npm install -D @commitlint/cli @commitlint/config-conventional husky semantic-release @semantic-release/commit-analyzer @semantic-release/release-notes-generator @semantic-release/changelog @semantic-release/git @semantic-release/github
```

## Managed Artifacts

This command manages these files:

* `commitlint.config.js`
* `.husky/commit-msg`
* `.releaserc.json`
* `.github/workflows/release.yml`

## Compatibility-based Skip Logic

Do not skip files purely because they exist.

For each managed artifact, classify it as one of:

* **missing** → create it
* **compatible** → keep it and print a skip notice
* **incompatible** → stop and present a summary of the incompatibility

If a managed file already exists and is compatible, print:

```text id="3jg0se"
<filename> already exists and is compatible — skipping
```

If a managed file already exists and is incompatible, stop and present:

```text id="94xucw"
<filename> already exists but is incompatible with this setup.

How would you like to proceed?

  1. Abort and review manually
  2. Show the expected content
```

Do not overwrite incompatible files silently.

## commitlint Configuration

Create `commitlint.config.js` only if it is missing.

Expected content:

```js id="19up2s"
module.exports = {
  extends: ['@commitlint/config-conventional'],
  rules: {
    'type-enum': [
      2,
      'always',
      ['feat', 'fix', 'docs', 'style', 'refactor', 'perf', 'test', 'chore', 'ci', 'revert', 'db'],
    ],
    'header-max-length': [2, 'always', 72],
  },
}
```

This aligns the repository hook with the `/commit` command.

## Husky Setup

Do not rely on `husky init` as the default path.

### Step 1 — Ensure `prepare` script exists

Ensure `package.json` contains:

```json id="snrjzb"
{
  "scripts": {
    "prepare": "husky"
  }
}
```

If `scripts.prepare` already exists and does not include `husky`, stop and present:

```text id="hy4ck0"
package.json already has a prepare script that does not match this setup.

How would you like to proceed?

  1. Abort and update package.json manually
  2. Show the expected change
```

Do not silently overwrite unrelated prepare logic.

### Step 2 — Run prepare once

Based on the detected package manager:

* pnpm:

  ```bash
  pnpm run prepare
  ```
* bun:

  ```bash
  bun run prepare
  ```
* npm:

  ```bash
  npm run prepare
  ```

### Step 3 — Create `.husky/commit-msg`

Create `.husky/commit-msg` only if it is missing.

Use the correct runner for the package manager.

#### pnpm

```sh id="z5d7xv"
pnpm exec commitlint --edit "$1"
```

#### bun

```sh id="7n4hvm"
bunx commitlint --edit "$1"
```

#### npm

```sh id="vjyu7k"
npx commitlint --edit "$1"
```

The file should contain:

```sh id="r6es5n"
#!/usr/bin/env sh
. "$(dirname -- "$0")/_/husky.sh"

<runner command here>
```

Then make it executable.

Do not create unrelated sample hooks.

## semantic-release Configuration

Create `.releaserc.json` only if it is missing.

Expected content:

```json id="n6q4n8"
{
  "branches": ["<RELEASE_BRANCH>"],
  "plugins": [
    "@semantic-release/commit-analyzer",
    "@semantic-release/release-notes-generator",
    ["@semantic-release/changelog", { "changelogFile": "CHANGELOG.md" }],
    ["@semantic-release/git", {
      "assets": ["CHANGELOG.md"],
      "message": "chore(release): ${nextRelease.version} [skip ci]"
    }],
    "@semantic-release/github"
  ]
}
```

Notes:

* `<RELEASE_BRANCH>` must be replaced with the detected release branch
* this setup commits `CHANGELOG.md`
* this setup does **not** update `package.json` version

## GitHub Actions Workflow

Create `.github/workflows/release.yml` only if it is missing.

Use the detected package manager variant.

### pnpm workflow

```yaml id="hsvwwa"
name: release

on:
  push:
    branches:
      - <RELEASE_BRANCH>

permissions:
  contents: read

jobs:
  release:
    runs-on: ubuntu-latest
    permissions:
      contents: write
      issues: write
      pull-requests: write
    env:
      HUSKY: 0
    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0

      - uses: pnpm/action-setup@v3
        with:
          version: latest

      - uses: actions/setup-node@v4
        with:
          node-version: "lts/*"
          cache: pnpm

      - run: pnpm install --frozen-lockfile

      - run: pnpm exec semantic-release
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
```

### bun workflow

```yaml id="qay35v"
name: release

on:
  push:
    branches:
      - <RELEASE_BRANCH>

permissions:
  contents: read

jobs:
  release:
    runs-on: ubuntu-latest
    permissions:
      contents: write
      issues: write
      pull-requests: write
    env:
      HUSKY: 0
    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0

      - uses: actions/setup-node@v4
        with:
          node-version: "lts/*"

      - uses: oven-sh/setup-bun@v2

      - run: bun install

      - run: bunx semantic-release
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
```

### npm workflow

```yaml id="9qv7z0"
name: release

on:
  push:
    branches:
      - <RELEASE_BRANCH>

permissions:
  contents: read

jobs:
  release:
    runs-on: ubuntu-latest
    permissions:
      contents: write
      issues: write
      pull-requests: write
    env:
      HUSKY: 0
    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0

      - uses: actions/setup-node@v4
        with:
          node-version: "lts/*"
          cache: npm

      - run: npm ci

      - run: npx semantic-release
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
```

Replace `<RELEASE_BRANCH>` with the detected release branch.

## File Creation Rules

When creating files:

* create parent directories if needed
* do not overwrite incompatible existing files
* preserve existing compatible files
* use exact managed content for newly created files

## Final Staging

Stage only the files touched by this setup.

Allowed staged files:

* `package.json`
* exactly one lockfile:

  * `pnpm-lock.yaml`
  * `bun.lockb`
  * `package-lock.json`
* `commitlint.config.js`
* `.husky/commit-msg`
* `.releaserc.json`
* `.github/workflows/release.yml`

Do **not** use `git add .`.

Use explicit staging:

```bash id="6azjqm"
git -C "$REPO_ROOT" add -- package.json commitlint.config.js .releaserc.json .github/workflows/release.yml .husky/commit-msg pnpm-lock.yaml bun.lockb package-lock.json 2>/dev/null || true
```

## Final Commit

After staging, present:

```text id="kkocd7"
Release setup is ready.

How would you like to proceed?

  1. Create commit now
  2. Review changes manually
  3. Abort
```

If the user chooses to commit, run:

```bash id="rz92o6"
git -C "$REPO_ROOT" commit -m "chore: setup release pipeline"
```

Do not push automatically.

## Safety Rules

* Never assume the release branch is `main`
* Never overwrite incompatible managed files silently
* Never use `git add .`
* Never promise `package.json` version updates unless `@semantic-release/npm` is explicitly added
* Never push automatically
* Never install or configure unrelated tooling outside the managed artifacts

## Notes

* `fetch-depth: 0` is required for semantic-release
* this setup creates GitHub releases and tags
* this setup may commit `CHANGELOG.md` during release runs
* if branch protection blocks pushes to the release branch, release commits may fail
* in CI, `HUSKY=0` prevents hook installation during workflow execution
