---
name: git-release-workflow
description: >
  Automated git versioning and release workflow for Node.js/TypeScript repositories.
  Use this skill whenever the user wants to set up automated releases, configure semantic-release,
  create conventional commits, manage changelogs, or establish a CI/CD release pipeline on GitHub.
  Also triggers for questions about commitlint, Husky, semantic versioning, or commit message
  formatting in any project context — even if the user doesn't mention "release" or "CI/CD" explicitly.
  This skill provides two ready-to-use Claude Code slash commands that work together as a complete workflow.
---

# Git Release Workflow

A complete automated versioning and release pipeline for Node.js/TypeScript repositories,
built around **Conventional Commits**, **semantic-release**, and **GitHub Actions**.

## What This Skill Provides

Two Claude Code slash commands that work together:

| Command | Purpose | When to run |
|---------|---------|-------------|
| `/setup-release` | One-time repo configuration | Once per repository |
| `/commit` | Daily commit workflow | Every commit |

## How the Pipeline Works

```text
dev writes code
  → /commit: lint → plan commits from staged snapshot → conventional commit
    → GitHub Actions: semantic-release analyzes commits
      → creates tag → generates changelog → publishes GitHub Release
````

Versioning is fully automatic. Do not bump versions manually. Do not edit `CHANGELOG.md` manually.

## Version Bump Rules

| Commit type                   | Bump  | Example           |
| ----------------------------- | ----- | ----------------- |
| `fix:`                        | patch | `1.0.0` → `1.0.1` |
| `feat:`                       | minor | `1.0.1` → `1.1.0` |
| `feat!:` or `BREAKING CHANGE` | major | `1.1.0` → `2.0.0` |

If a push to the configured release branch contains no release-relevant commits since the last release, no release is created.

## Interaction Model

This skill must prefer **interactive prompts with selectable options** over free-form text input whenever a decision can be expressed as a finite set of choices.

### Prompting Rules

* Prefer **selection prompts** for confirmations, branch choices, conflict handling, staging decisions, lint failures, push decisions, and any other bounded decision
* Use free-form text input only when the user must provide information that cannot be represented as a small fixed option set
* When presenting options, always number them clearly
* Default to an **interactive prompt** even if free-form input would also work
* Do not ask the user to type an exact value when the command can instead present valid choices directly
* If an operation can be safely continued, edited, skipped, retried, or aborted, present those as selectable options

### Preferred Format

Use prompts like:

```text
How would you like to proceed?

  1. Confirm and continue
  2. Edit the plan
  3. Abort
```

Instead of prompts like:

```text
Type the commit type:
Type the branch name:
Type yes to continue:
```

### Examples of When to Use Selection Prompts

Use selection prompts for:

* continue / abort decisions
* choosing whether to stage all files
* choosing whether to proceed after lint failure
* choosing whether to push after commit creation
* choosing between single-commit mode and split mode
* handling detected secrets or incompatible setup files
* choosing whether to continue on a non-release branch

### Free-form Input Exceptions

Use free-form input only for information such as:

* a custom commit message when the command explicitly allows manual override
* a manually edited commit plan
* content that cannot be predicted or enumerated safely

When both approaches are possible, prefer the selection-based interactive prompt.

## Command Reference

Both commands are bundled in `commands/`. Read the relevant file before executing:

* **`commands/setup-release.md`** — full instructions for `/setup-release`
* **`commands/commit.md`** — full instructions for `/commit`

### When to read which file

* User runs `/setup-release` or asks to configure a repo → read `commands/setup-release.md`
* User runs `/commit` or asks to commit changes safely → read `commands/commit.md`
* User asks how the pipeline works or what to set up first → explain the overview above, then suggest `/setup-release`

## Compatibility

* **Package managers:** pnpm, bun, npm
* **CI:** GitHub Actions
* **Node.js:** 18+
* **Yarn:** not supported — instruct the user to configure manually if detected

## Key Constraints (enforce always)

* Prefer interactive selection prompts over free-form text input whenever possible
* Commit descriptions follow **the user's language** as inferred from the conversation
* The `type` and optional `scope` structure follow Conventional Commits syntax
* No emojis in commit messages
* No signatures in commit messages
* Do not claim trailers or footer rules are enforced unless the repository hook actually enforces them
* `wip` is not a valid commit type
* Never bump versions manually
* Never edit `CHANGELOG.md` manually
* Never push before all local commits are finalized
* Never assume the release branch is `main`
* `/commit` must use the **staged snapshot** as the source of truth
* `/commit` must never silently include unrelated unstaged changes
* `/setup-release` must never silently overwrite incompatible managed files
* `/setup-release` must never use `git add .`

## Execution Guidance

### `/setup-release`

Use `/setup-release` when the user wants to:

* configure `semantic-release`
* add `commitlint`
* add Husky commit hooks
* create a GitHub Actions release workflow
* standardize release automation in a repository

Before executing, read `commands/setup-release.md` and follow it exactly.

### `/commit`

Use `/commit` when the user wants to:

* create one or more safe commits
* split staged changes into logical commits
* generate Conventional Commit messages
* optionally push after commits are finalized

Before executing, read `commands/commit.md` and follow it exactly.

## Expected Workflow Order

1. Run `/setup-release` once per repository
2. Use `/commit` for normal day-to-day commit creation
3. Let CI run semantic-release on the configured release branch
4. Do not manually manage versions or changelog entries

## Notes

* This skill is optimized for repositories that want a predictable release workflow with minimal manual versioning work
* The slash command documents are the source of truth for command behavior
* If the repository already contains release tooling, validate compatibility instead of overwriting files blindly
