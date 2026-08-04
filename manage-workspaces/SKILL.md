---
name: manage-workspaces
description: Reference for the user's VS Code .code-workspace files, what each one is for and where it lives. Invoke when discussing VS Code workspace setup, deciding which workspace a new project or repo belongs in, or when asked to add/remove/update folders in a workspace file.
---

## When to Use

- Discussing VS Code workspace setup
- Deciding which `.code-workspace` file a new project or repo belongs in
- Asked to add, remove, or update folders in a workspace file
- The user mentions `claude-skills`, `dotfiles/`, global Claude settings, or OpenClaw settings

# VS Code workspaces reference

The user groups folders in VS Code using two `.code-workspace` files (plain JSON, `folders` array). Edit them directly to add/remove entries, there's no CLI for this.

## `~/config.code-workspace`
- Purpose: Claude Code and OpenClaw settings, not project code.
- Folders: `dotfiles`, `.openclaw`, `claude-skills`
- `dotfiles` is `~/dotfiles`, which `~/.claude` (and other dotfiles) symlink back to.
- `claude-skills` is `~/Desktop/Projects/claude-skills`, a standalone repo holding only the user's own personal skills (nothing from coworkers or elsewhere), it's just a copy: the live skills stay in `dotfiles/.claude/skills`. It won't be updated often, just synced occasionally so there's always a repo with the personal ones up to date.

## `~/Desktop/Projects/projects.code-workspace`
- Purpose: personal projects and open source repos the user clones or contributes to.
- Folders live under `~/Desktop/Projects/`, one entry per project/repo.
- When a new project/repo is added under `~/Desktop/Projects/`, add a matching entry here.

## Notes
- The folder list in each workspace JSON is the source of truth, don't rely on this doc for the current list, read the file if you need it.
- A third file, `~/discordgpt/VS Code.code-workspace`, exists but is stray/unused clutter, out of scope, ignore it.
