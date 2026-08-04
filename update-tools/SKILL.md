---
name: update-tools
description: Check for and apply updates to the user's machine-wide dev tools (Homebrew + uv + gh, Claude Code, Docker Desktop, tokensave, npm, pnpm, node). Use when they say "check for updates", "update my tools", "keep up to date", "are my tools up to date", "/update-tools", or name a specific tool to update. Handles each tool's install-method-specific update command and post-update restart gotchas so they don't have to memorize them.
---

## When to Use

- The user says "check for updates," "update my tools," "keep up to date," or "are my tools up to date"
- The user runs `/update-tools`
- The user names a specific tool (Homebrew, uv, gh, Claude Code, Docker Desktop, tokensave, npm, pnpm, node) to update

# update-tools

Keeps the user's dev tooling current. Their tools split across four install methods, so
each updates differently, this skill remembers the right command per tool.

## Flow (always check first, then confirm, then update)

1. **Check phase.** Run every tool's *check* command (all read-only or self-checking).
   Never update in this phase.
2. **Report.** Print one table: tool, current version, latest (or "up to date" /
   "update available" / "unknown"), and any post-update gotcha.
3. **Confirm.** Ask the user which to update. Do NOT blind-update. Wait for their go.
4. **Update phase.** Run only the update commands the user approved, one at a time (their rule:
   no chaining with `&&`). Report each result plainly, including failures.
5. **Post-update.** Surface the restart reminders (app / MCP session) that apply.
6. **Cleanup.** Run the `Cleanup:` command for anything that actually upgraded. Skip
   whatever didn't upgrade, there's nothing to clear. Some cleanups are marked
   "occasional only", don't run those every pass.

Do not touch `node` in the update phase, see its section below.

## Tool reference

### Homebrew group (`uv`, `gh`, Homebrew itself)
All Homebrew-managed (`/opt/homebrew/bin/...`). Do NOT use `uv self update`, it errors
on Homebrew installs. Update through brew.

- Check: `brew outdated`
- Update: `brew update` then (separate call) `brew upgrade`
  - To update just one: `brew upgrade uv` / `brew upgrade gh`
- Cleanup: `brew cleanup` (removes the old Cellar versions `brew upgrade` leaves behind).
  Safe, run it after any upgrade. Note brew also auto-runs this every 30 days unless
  `HOMEBREW_NO_INSTALL_CLEANUP` is set, so it's a nice-to-have, not a must.
- Post-update: none

### Claude Code
Native installer at `~/.local/bin/claude`.

- Check: `claude --version` (it also self-notifies when a newer build exists)
- Update: `claude update`
- Post-update: **restart the running Claude Code session** so the new binary loads.

### Docker Desktop
Standalone app, not Homebrew-managed on this machine (`brew list --cask` doesn't show it).
Has its own CLI update path via the `docker desktop` plugin.

- Check: `docker desktop update --check-only`
- Update: `docker desktop update`
- Post-update: applying an update restarts Docker Desktop to load the new version. If
  containers are running, **ask the user before applying** rather than assuming it's fine to
  interrupt them.
- Cleanup: **none. Never run `docker system prune` here**, and never anything with
  `--volumes`. A local project database must never be wiped, it may hold config
  that isn't auto-seeded and is painful to restore. If it's running in Docker, a prune
  can take its volume with it. The disk space is not worth that risk, don't even offer it.

### tokensave
GitHub-binary at `~/.local/bin/tokensave`.

- Check: `tokensave upgrade` self-checks; prints "up to date" if current.
- Update: `tokensave upgrade`
- Post-update: **restart the Claude Code session / MCP server**, the tokensave MCP
  server launched at session start keeps running the old binary until reconnected.
  A point release does NOT need a re-`init`; `doctor`/`sync` migrate the index if needed.

### npm
Bundled with the nvm-managed node (`~/.nvm/.../bin/npm`).

- Check: `npm outdated -g --depth=0`
- Update npm itself: `npm install -g npm@latest`
- Post-update: none. (Tied to the current node version, see node below.)

### pnpm
Lives in the nvm node bin; project pins its own version via `packageManager` in
`package.json` (corepack). Updating the global pnpm does not change what a repo uses.

- Check: `pnpm --version`
- Update (global): `corepack use pnpm@latest`
- Cleanup (OCCASIONAL ONLY): `pnpm store prune` (drops packages nothing references from
  the global store). Do not run this every update pass. Pruned packages get re-downloaded
  on the next install in any project pnpm didn't have registered, so routine pruning just
  buys the same downloads twice. Run it when disk is actually tight, not on schedule.
- Post-update: none. Per-project pnpm is controlled by that repo's `packageManager` field,
  not this skill.

### node (REPORT ONLY, never auto-update)
Managed by **nvm**, which is a shell function, not a binary, so it can't be driven from a
tool call, and bumping node major can break nvm-pinned globals. Do NOT update it here.

- Check: compare `node -v` against the latest LTS.
- If behind: **note that node needs updating, and that the user should do it themselves, and only
  when actually necessary.** Hand them the command to run in their own shell:

      ! nvm install --lts --reinstall-packages-from=current
      ! nvm alias default lts/*

  Do not run these; just report and let him decide.

## Notes
- Run each command as its own Bash call. Chaining trips the user's permission allow-list.
- Report failures honestly (version output, error text); don't paper over a failed update.
