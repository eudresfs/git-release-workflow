---
allowed-tools: Bash(git:*), Bash(node:*), Bash(pnpm:*), Bash(npm:*), Bash(bun:*), Bash(cat:*), Bash(ls:*), Bash(mkdir:*), Read, Write, Edit
description: Configure semantic-release, commitlint and Husky in the current repository. Run once per repo before using /commit.
model: sonnet
---

# Setup Release

Configure automated versioning and release pipeline for this repository.

> **Companion command:** after setup, use `/commit` for daily commits and push.

## Current State

- Git status: !`git status --porcelain`
- Current branch: !`git branch --show-current`
- Root package.json: !`cat package.json 2>/dev/null || echo "NOT FOUND"`
- Lock files present: !`ls pnpm-lock.yaml package-lock.json bun.lockb yarn.lock 2>/dev/null || echo "none"`
- Existing releaserc: !`cat .releaserc.json 2>/dev/null || echo "NOT FOUND"`
- Existing commitlint config: !`cat commitlint.config.js 2>/dev/null || echo "NOT FOUND"`
- Existing husky hooks: !`ls .husky/ 2>/dev/null || echo "NOT FOUND"`
- Existing release workflow: !`cat .github/workflows/release.yml 2>/dev/null || echo "NOT FOUND"`

## What This Command Does

1. Runs pre-flight checks
2. Detects the package manager in use
3. Checks which artifacts already exist and skips them
4. Installs required devDependencies
5. Creates `commitlint.config.js`
6. Initializes Husky and sets the `commit-msg` hook
7. Creates `.releaserc.json`
8. Creates `.github/workflows/release.yml`
9. Commits only if at least one new file was created

## Pre-flight Checks

Abort with a clear message if any of these fail:

- This must be a git repository (`git status` succeeds)
- A `package.json` must exist at the root

If the user is not on `main`, present the following options before proceeding:

```
You are not on the main branch (current: <branch>). How would you like to proceed?

  1. Switch to main first (recommended)
  2. Continue on current branch
  3. Abort
```

Wait for the user's choice before continuing.

## Package Manager Detection

**In order of priority — lockfile presence wins:**

| Lockfile | Package manager |
|----------|----------------|
| `pnpm-lock.yaml` | pnpm |
| `bun.lockb` | bun |
| `package-lock.json` | npm |
| `yarn.lock` | → abort (see below) |
| none found | → ask user (see below) |

Do **not** rely on the `packageManager` field in `package.json` as the primary detection method — it is often absent. Use it only as a tiebreaker if multiple lockfiles somehow coexist.

**Yarn detected:**
```
Yarn detected. This command supports pnpm, bun, and npm only. Please configure manually.
```
Abort.

**No lockfile found** — present:
```
No lockfile detected. Which package manager are you using?

  1. pnpm (recommended)
  2. bun
  3. npm
```
Wait for the user's choice, then proceed accordingly.

## Package Manager Reference

All subsequent steps use values from this table based on the detected package manager:

| | pnpm | bun | npm |
|-|------|-----|-----|
| **Install deps** | `pnpm add -D` | `bun add -d` | `npm install -D` |
| **Run binary** | `pnpm exec` | `bunx` | `npx` |
| **Install all** | `pnpm install` | `bun install` | `npm ci` |
| **Husky init** | `pnpm exec husky init` | `bunx husky init` | `npx husky init` |
| **commitlint in hook** | `pnpm exec commitlint --edit $1` | `bunx commitlint --edit $1` | `npx commitlint --edit $1` |

Refer to this table throughout — do not hardcode any package manager.

## Skip Logic

Before creating each artifact, check if it already exists. If it does, print:
`⚠ <filename> already exists — skipping`

Track how many files were actually created. If zero files were created, skip the final commit step entirely and inform the user:
`No new files created — nothing to commit.`

## Dependencies to Install

```
husky
@commitlint/cli
@commitlint/config-conventional
semantic-release
@semantic-release/changelog
@semantic-release/git
@semantic-release/github
@semantic-release/commit-analyzer
@semantic-release/release-notes-generator
```

Install as devDependencies using the detected package manager's install command from the reference table.

## Files to Create

### commitlint.config.js

Check `"type"` in the **root** `package.json`:

- If `"type": "module"` → use ESM:
  ```js
  export default { extends: ['@commitlint/config-conventional'] }
  ```
- Otherwise (`"type": "commonjs"` or field absent) → use CJS:
  ```js
  module.exports = { extends: ['@commitlint/config-conventional'] }
  ```

> **Monorepo note:** this file lives at the root and applies globally. If a `packages/` or `apps/` directory is detected, warn the user:
> `Monorepo detected. Verify that individual packages do not have conflicting "type" settings relative to the root.`

### Husky

Initialize using the Husky init command from the reference table.

Then write `.husky/commit-msg` using the commitlint hook command from the reference table:

```sh
# example for pnpm — use the correct runner for the detected package manager
pnpm exec commitlint --edit $1
```

### .releaserc.json

```json
{
  "branches": ["main"],
  "plugins": [
    "@semantic-release/commit-analyzer",
    "@semantic-release/release-notes-generator",
    ["@semantic-release/changelog", { "changelogFile": "CHANGELOG.md" }],
    ["@semantic-release/git", {
      "assets": ["CHANGELOG.md", "package.json"],
      "message": "chore: release ${nextRelease.version} [skip ci]"
    }],
    "@semantic-release/github"
  ]
}
```

### .github/workflows/release.yml

Generate the workflow based on the detected package manager:

**pnpm:**
```yaml
name: release

on:
  push:
    branches:
      - main

permissions:
  contents: write

jobs:
  release:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0
          persist-credentials: false

      # version: read from "packageManager" field in package.json if present (e.g. "pnpm@9.1.0")
      # if absent, "latest" avoids hardcoding a version that diverges from the project
      - uses: pnpm/action-setup@v3
        with:
          version: latest

      - uses: actions/setup-node@v4
        with:
          node-version: 20
          cache: pnpm

      - run: pnpm install

      - run: npx semantic-release
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
```

**bun:**
```yaml
name: release

on:
  push:
    branches:
      - main

permissions:
  contents: write

jobs:
  release:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0
          persist-credentials: false

      - uses: oven-sh/setup-bun@v2

      - run: bun install

      - run: bunx semantic-release
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
```

**npm:**
```yaml
name: release

on:
  push:
    branches:
      - main

permissions:
  contents: write

jobs:
  release:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0
          persist-credentials: false

      - uses: actions/setup-node@v4
        with:
          node-version: 20
          cache: npm

      - run: npm ci

      - run: npx semantic-release
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
```

## Final Commit

Only run if at least one file was created:

```bash
git add .
git commit -m "chore: setup automated release pipeline"
```

Do not push. Inform the user:

```
Setup complete. Review the created files, then push manually to main.
On the next push containing feat/fix commits, semantic-release will create the release automatically.
```

## Version Bump Rules (inform the user after setup)

| Commit type | Result |
|-------------|--------|
| `fix:` | patch → `1.0.1` |
| `feat:` | minor → `1.1.0` |
| `feat!:` or `BREAKING CHANGE` in body | major → `2.0.0` |

Every push to `main` triggers the pipeline. If no qualifying commits are found since the last release, no release is created.

## Important Notes

- `fetch-depth: 0` is mandatory — semantic-release needs full git history to determine the next version
- `[skip ci]` in the release commit message prevents an infinite loop when semantic-release pushes back `package.json` and `CHANGELOG.md`
- `GITHUB_TOKEN` is automatically available in GitHub Actions — no manual secret needed for public or private repos (release creation only)
- If the repo has branch protection rules on `main`, the token may need explicit write permissions in repo Settings → Actions → Workflow permissions
- If you later add `@semantic-release/npm` to publish to npm, add `NPM_TOKEN` as a repository secret and expose it as `NPM_TOKEN: ${{ secrets.NPM_TOKEN }}` in the workflow env
