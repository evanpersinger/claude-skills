---
name: checkout-repo
description: Build background on a whole repo/project before starting work. Broader than checkout-branch (which only reviews git state), this reads the docs, tech stack, structure, and how to run/test the project, then summarizes. All read-only.
---

## When to Use

- The user says "checkout repo"
- Starting work in an unfamiliar repo and need background on tech stack, structure, and how to run/test it

# Checkout repo (project onboarding)

When the user says **"checkout repo"**, build a mental model of the project in the current working directory so we're ready to work. Everything here is read-only.

Distinct from `checkout-branch`: that skill only orients on git state. This one
orients on the *whole project*. Do the git step here too, but the focus is
understanding what the project is and how it works.

All steps run in the current working directory.

## Fast path: repo.md

Check for `repo.md` at the repo root first (it's gitignored globally, so its absence is normal for a repo you haven't checked out before). If it exists, read it, that covers "What is this project?", "Tech stack and how it's run", and "Structure" below in one read. Skip straight to the **Git state** step, then summarize.

If `repo.md` doesn't exist, fall back to normal behavior: read through the repo yourself (README, manifest, structure, below) to get a sense of what kind of repo this is, same as you would with no skill at all. Then write `repo.md` (see **Writing repo.md** at the end) so the next checkout is instant.

## What is this project?

Read these if present (skip silently if absent): `README.md`, `docs/`, `.gitignore`, `things_to_know.md`. These give purpose and any project-specific working rules, follow those rules for the rest of the session. Note: `things_to_know.md` is the user's own freeform personal notes, distinct from `repo.md`, don't copy it verbatim into `repo.md`.

## Tech stack and how it's run

Find and read the manifest for the language:
- JS/TS: `package.json` (note `scripts`, deps; package manager via lockfile:
  `pnpm-lock.yaml` / `yarn.lock` / `package-lock.json`)
- Python: `pyproject.toml`: check for `uv.lock`, which means the project uses
  **uv** (`uv sync` to install, `uv run` to run). Also `requirements.txt` /
  `setup.py` on older projects.
- Also check: `Makefile`, `docker-compose*.yml`, `.github/workflows/` for
  build/test/run commands.

## Structure

`git ls-files | head -100`, or `ls` the top level plus one or two levels of the main source dir, to learn the layout and entry points. Don't read every file, sample the important ones.

## Git state

`git branch --show-current`, `git log --oneline -10`, `git status -sb`. Note the current branch, recent work, uncommitted changes.

## Summarize back

Concise:
- What the project is / does (one or two sentences)
- Language, framework, package manager
- How it's structured (the dirs that matter)
- How to run it and how to test it
- Current git state (branch, uncommitted changes)
- Anything unusual or worth flagging

Then ask what we're working on.

## Writing repo.md

Only when `repo.md` didn't already exist and you did the full exploration. Write `repo.md` at the repo root, no intro/meta paragraph, start straight with `# repo.md (<repo-name>)` as the title, then these sections:

- **What this is**: one or two sentences, purpose only
- **Tech stack**: language, framework, package manager, notable deps
- **Structure**: key dirs and what lives in each, entry points
- **How to run / build**: dev/build/start/test commands
- **Anything unusual**: architecture quirks, missing test suite, non-standard layout, etc.

Leave out git state (branch, commits, uncommitted changes), that's always live and would go stale. Leave out personal/freeform content from `things_to_know.md`, `repo.md` is structural orientation for Claude, not the user's notes. When summarizing back, mention that `repo.md` was created so they know it exists and is gitignored globally.

## Rules
- Read-only only, except writing `repo.md` itself when it didn't already exist. Never edit other files, run builds, or install anything during checkout.
- Keep the summary tight, a briefing, not a file dump. One concept at a time.
- If `repo.md` looks stale against what you're seeing (e.g. structure changed, deps don't match), say so and offer to regenerate it, don't silently trust it.
