---
name: update-tools
description: Check for and apply updates to the user's machine-wide dev tools (Homebrew + uv + gh, Claude Code, Docker Desktop, tokensave, rtk, openclaw, npm, pnpm, node, VS Code, Ollama, MySQL Workbench). Handles each tool's install-method-specific update command and post-update restart gotchas so they don't have to memorize them.
---

## When to Use

- The user says "check for updates," "update my tools," "keep up to date," or "are my tools up to date"
- The user runs `/update-tools`
- The user names a specific tool (Homebrew, uv, gh, Claude Code, Docker Desktop, tokensave, rtk, openclaw, npm, pnpm, node, VS Code, Ollama, MySQL Workbench) to update

# update-tools

Keeps the user's dev tooling current. Their tools span several install methods (Homebrew,
native installers, GitHub-released binaries, nvm/npm, self-updating apps, manual-download
apps), so each updates differently, this skill remembers the right command per tool.

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

Worth a pass every couple of weeks. Homebrew is the one that goes wrong quietly, so run
its check even on a pass where the user doesn't want to upgrade anything (see below).

## Tool reference

### Homebrew group (`uv`, `gh`, Homebrew itself)
All Homebrew-managed (`/opt/homebrew/bin/...`). Do NOT use `uv self update`, it errors
on Homebrew installs. Update through brew.

- Check: `brew outdated`
- Update: `brew update` then (separate call) `brew upgrade`
  - To update just one: `brew upgrade uv` / `brew upgrade gh`
- **Run `brew update` on every pass, even one where nothing gets upgraded.** It refreshes
  Homebrew itself and the formula definitions and upgrades no packages, so it is always
  safe. Homebrew falling behind the formulae it pulls breaks things that do not look like a
  version problem: treat a crash inside `formulary.rb` (typically `undefined method` raised
  from a formula's `service` block) as "Homebrew is stale" and run `brew update` before
  investigating further.
- Cleanup: `brew cleanup` (removes the old Cellar versions `brew upgrade` leaves behind).
  Safe, run it after any upgrade. Note brew also auto-runs this every 30 days unless
  `HOMEBREW_NO_INSTALL_CLEANUP` is set, so it's a nice-to-have, not a must.
- **Shared-port gotcha, check BEFORE upgrading anything that runs as a service.** Upgrading
  a service formula restarts it. If the same service also runs in Docker for another
  project, two servers can hold one port at once: native binds the specific addresses
  `127.0.0.1`/`[::1]` while a Docker container binds the wildcard `0.0.0.0`, and the kernel
  accepts both as distinct sockets. Specific wins, so `localhost` reaches native right up
  until native restarts, at which point the container silently inherits the port with no
  error, and it just looks like the data changed. Run `lsof -nP -iTCP:<port> -sTCP:LISTEN`
  first and stop the container before upgrading.
  - Common collision points when a service runs both natively and in Docker: Postgres on
    5432, Redis on 6379.
  - Treat those as examples, not the whole list. Run the `lsof` check against any service
    formula rather than assuming only Postgres and Redis are affected.
  - Whether a container is currently up, and whether a given upgrade is safe right now, is
    live state. Get it from `lsof`, never from a status written into this skill.
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

### openclaw
Global npm package (`~/.nvm/.../bin/openclaw`), but has its own update command, don't use
plain `npm install -g` for this one. Runs as a persistent launchd service that handles live
messaging channels.

- Check: `openclaw update status` (shows current channel and whether an update is available)
- Update: `openclaw update --yes` (non-interactive; add `--no-restart` to defer the service
  bounce during a running conversation)
- Post-update: **restarts the service automatically** by default, no manual step needed.
  If something looks stranded after, `openclaw update repair` fixes post-update
  plugin/doctor state.

### rtk
Standalone binary at `~/.local/bin/rtk`. No self-update subcommand in `rtk --help`, but a
background version-check file shows the installed binary has already moved past the
version it last cached, so it appears to keep itself current without a manual step.

- Check: `rtk --version`
- Update: none, believed to self-update in the background. If a check ever shows it's
  fallen behind, flag it to the user rather than guessing at a reinstall command, the
  actual update mechanism hasn't been confirmed.
- Post-update: none known.

### VS Code
Not Homebrew-cask managed on this machine. Has its own built-in auto-updater (macOS
default `update.mode` is enabled), so this is mostly a sanity check, not a real update step.

- Check: `code --version | head -1`
- Update: none needed normally, it updates itself in the background. If it looks stale,
  manual fallback is the in-app "Code > Check for Updates…" menu item.
- Post-update: restart VS Code windows to load the new build (it usually prompts "Restart
  to Update" itself).

### Ollama
Not Homebrew-cask managed on this machine, standalone app. No CLI self-update subcommand.

- Check: `ollama --version` (or `/usr/libexec/PlistBuddy -c "Print :CFBundleShortVersionString" "/Applications/Ollama.app/Contents/Info.plist"`)
- Update: re-download the latest build from https://ollama.com/download and replace the
  app. (Could move this to `brew install --cask ollama` for CLI-managed updates going
  forward, confirm with the user before switching the install method.)
- Post-update: quit and relaunch Ollama (and `ollama serve` if it's running as a background
  process).

### MySQL Workbench
Not Homebrew-cask managed on this machine, standalone app.

- Check: `/usr/libexec/PlistBuddy -c "Print :CFBundleShortVersionString" "/Applications/MySQLWorkbench.app/Contents/Info.plist"`
- Update: download the latest `.dmg` from https://dev.mysql.com/downloads/workbench/ and
  reinstall, no CLI update path.
- Post-update: quit and reopen the app.

### node (REPORT ONLY, never auto-update)
Managed by **nvm**, which is a shell function, not a binary, so it can't be driven from a
tool call, and bumping node major can break nvm-pinned globals. Do NOT update it here.

- Check: compare `node -v` against the latest LTS.
- If behind: **note that node needs updating, and that the user should do it themselves, and only
  when actually necessary.** Hand them the command to run in their own shell:

      ! nvm install --lts --reinstall-packages-from=current
      ! nvm alias default lts/*

  Do not run these; just report and let them decide.

## Notes
- Run each command as its own Bash call. Chaining trips the user's permission allow-list.
- Report failures honestly (version output, error text); don't paper over a failed update.
