# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Repository status

This repository currently defines a **development workflow, not an application**. It contains:

- `git-rule-draft.md` — the Japanese source draft of the rules.
- `SKILL.md` — the workflow packaged as a Claude Code Skill (the authoritative, most detailed version). **When guiding branching/commit/PR/release work, follow `SKILL.md`.**
- `README.md` — project overview.

There is no source code, build system, or committed history yet. When code is added, follow the conventions below and update this file with the concrete build/lint/test commands introduced.

## Git-Flow branching model

- `main` and `dev` are permanent — created once, never deleted.
- `dev` is the integration branch; `main` holds released code.
- **feature branches** (`feature/名前` or `feature/タスク番号_名前`): branch from `dev`, one branch per feature, PR back into `dev` when done.
- **release branches** (`release/v1.0.0`): branch from `dev` for final pre-release testing. When testing passes, open **two** PRs — one into `main` and one back into `dev`.
- **hotfix branches** (`hotfix/issue番号_名前`): branch from `dev` to fix a reported bug. Same as release — merge back into **both** `main` and `dev` via two PRs.

## Parallel development (git worktree)

When several sessions/agents edit the same repository at once, use **git worktree**, not `git checkout` — one branch per worktree, placed *outside* the repository (`<repo>-worktrees/<slug>/`).

A bare `git worktree add` only brings across git-tracked content, so gitignored working resources (submodule contents, `.env`, virtualenvs, local editor/agent settings) are missing and fail at runtime. Each repository must therefore provide a worktree-creation script (fetch → `worktree add` → `submodule update --init --recursive` → copy ignored files from the main worktree → install dependencies), and worktrees must be created only through it. Record that in the consuming project's `CLAUDE.md`, since skills are not always loaded.

Committed resources that programs rewrite (SQLite DBs, generated datasets, snapshots) must never be written from two worktrees concurrently — git cannot merge them.

## Versioning / tags

Version format is `major.minor.build`:
- **major** — large changes that alter the architecture.
- **minor** — feature additions or bug fixes.
- **build** — incremented on every small fix.

The version is a single source of truth per language, bumped in the PR:
- **C++** — `CMakeLists.txt` (`project(<name> VERSION X.Y.Z)`); `version.h.in` is expanded to `version.h` via `configure_file()` and consumed from source (`version.h` is gitignored).
- **Python** — `pyproject.toml` (`[project] version`), read at runtime via `importlib.metadata`; never hardcode the version string.

Do **not** tag manually — the release automation creates the `vX.Y.Z` tag after merge to `main`. PR descriptions should record what was implemented at each build version so the development history stays traceable.

## Commits

- Commit in small increments and keep them local (do not push each one).
- When pushing to the remote, **squash** the local commits into a consolidated set.

## GitHub Actions (to be set up)

**CI** runs on Pull Requests — both jobs must pass to merge; run them locally first:
- **Static analysis** — C++: **clang-tidy** (reads `compile_commands.json`, config in `.clang-tidy`); Python: **PyLint**.
- **Tests** — C++: **GoogleTest** (registered via CMake `gtest_discover_tests`, run with `ctest`); Python: **pytest**.

**Release automation** runs when a PR is *merged into* `main` (`pull_request` `closed` + `merged == true` + `base_ref == main`): it reads the version from the source-of-truth file, pushes a `vX.Y.Z` tag, and creates a GitHub Release. Requires `permissions: contents: write`.

## Pull Request format

Every PR must include (so reviewers understand both the change and its motivation):

1. **Feature/bug overview** — background (why the change was made) and a high-level summary, explained in plain language a layperson could follow, with no technical detail.
2. **Root cause** (bug fixes only) — why the bug occurred; technical detail belongs here.
3. **Main changes** — detailed, technical description; if the build version changed during the work, describe what was implemented per build version.
4. **Verification checklist** — planned tests. New features need both success-path and error-path tests; bug fixes need a regression test that reproduces the bug.
5. **Tests** — added/modified tests and what each verifies, as a list.
6. **Documentation** — docs added or changed alongside the code.

## Language

`git-rule-draft.md` and the intended project documentation are in Japanese. Match the existing language when editing docs.
