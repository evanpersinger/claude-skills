---
name: checkout-repo
description: Build background on a whole repo/project before starting work, when the user says "checkout repo". Broader than checkout-branch (which only reviews git state), this reads the docs, tech stack, structure, and how to run/test the project, then summarizes. All read-only.
---

## When to Use

- The user says "checkout repo"
- Starting work in an unfamiliar repo and need background on tech stack, structure, and how to run/test it

# Checkout repo (project onboarding)

When the user says **"checkout repo"**, build a mental model of the project in the current working directory so we're ready to work. Everything here is read-only.

Distinct from `checkout-branch`: that skill only orients on git state. This one
orients on the *whole project*. Do the git step here too, but the focus is
understanding what the project is and how it works.

## Steps (run in the current working directory)

1. **What is this project?** Read these if present (skip silently if absent):
   `README.md`, `docs/`, `.gitignore`, `things_to_know`.
   These give purpose and any project-specific working rules, follow those rules
   for the rest of the session.
2. **Tech stack & how it's run.** Find and read the manifest for the language:
   - JS/TS: `package.json` (note `scripts`, deps; package manager via lockfile:
     `pnpm-lock.yaml` / `yarn.lock` / `package-lock.json`)
   - Python: `pyproject.toml`: check for `uv.lock`,
     which means the project uses **uv** (`uv sync` to install, `uv run` to run).
     Also `requirements.txt` / `setup.py` on older projects.
   - Also check: `Makefile`, `docker-compose*.yml`, `.github/workflows/` for
     build/test/run commands.
3. **Structure.** `git ls-files | head -100`, or `ls` the top level plus one or two
   levels of the main source dir, to learn the layout and entry points. Don't read
   every file, sample the important ones.
4. **Git state.** `git branch --show-current`, `git log --oneline -10`,
   `git status -sb`. Note the current branch, recent work, uncommitted changes.
5. **Summarize back, concise:**
   - What the project is / does (one or two sentences)
   - Language, framework, package manager
   - How it's structured (the dirs that matter)
   - How to run it and how to test it
   - Current git state (branch, uncommitted changes)
   - Anything unusual or worth flagging
   Then ask what we're working on.

## Rules
- Read-only only. Never edit, run builds, or install anything during checkout.
- Keep the summary tight, a briefing, not a file dump. One concept at a time.
