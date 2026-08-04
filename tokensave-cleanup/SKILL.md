---
name: tokensave-cleanup
description: Survey tokensave indexes across all repos and clean up the stale and dead ones. Use when the user says "clean up tokensave", "tokensave cleanup", "check my tokensave indexes", or asks which indexes are stale. Reports index age, drift against git, and branch DBs for branches that no longer exist, then syncs or removes them with approval. Never deletes anything before naming exactly what dies.
---

## When to Use

- The user says "clean up tokensave," "tokensave cleanup," or "check my tokensave indexes"
- Asking which tokensave indexes are stale or dead

# Tokensave cleanup

Survey first, act second. The survey is read-only; nothing is synced or deleted until the
report is in front of the user and they've said which repos to act on.

The problem this solves: a tokensave index is a cache derived from committed code, and
nothing keeps it fresh. `sync` is manual, `branch add` is additive, and neither cleans up
after itself. Indexes drift and branch DBs outlive their branches. Left alone, a stale
index answers structural questions confidently and wrongly, which is worse than having no
index at all, because with no index we just read the files and get the truth.

## Survey

Find every index. Not `tokensave list`, that only sees the current folder, its parents and
its children, so it silently misses siblings:

```bash
find ~/Desktop -maxdepth 5 -type d -name ".tokensave"
```

(Adjust the root to wherever your repos actually live.)

For each repo found, gather three things:

```bash
tokensave status --json <repo>                    # node/edge/file counts, db_size_bytes, last_sync_at
git -C <repo> log -1 --format=%ct                 # epoch of most recent commit
git -C <repo> status --porcelain                  # uncommitted work the index can't know about
```

Always use `--json` on status. The default output prints an ANSI-art logo, roughly 38 KB of
escape codes to convey about fifteen numbers.

### Drift, not age

Judge staleness by comparing `last_sync_at` against the last commit time, never by the
index's age alone. An index untouched for two weeks on a repo with no commits in three
weeks is perfectly current. Report a repo as behind when either:

- last commit epoch > `last_sync_at`, or
- `git status --porcelain` is non-empty (uncommitted changes are never in the index, and
  a sync won't fix that, only committing will)

The second case is worth stating plainly rather than folding into "stale": syncing does not
help there, so don't propose it as the fix.

### Dead branch DBs

Every tracked branch gets its own full DB copy, registered in `branch-meta.json`:

```bash
cat <repo>/.tokensave/branch-meta.json
```

For each key under `branches` other than `default_branch`, check whether the branch still
exists:

```bash
git -C <repo> rev-parse --verify --quiet refs/heads/<branch-name>
```

Exit 1 means the branch is gone and its DB is dead weight. These are usually the biggest
single win: btdash was carrying 62 MB for a branch merged and deleted weeks earlier.

Sizes live in `<repo>/.tokensave/branches/`.

## Report

One table, largest reclaimable space first:

| repo | index size | last synced | behind? | dead branch DBs |
|---|---|---|---|---|

Then a one-line total of what's reclaimable, and a recommendation per repo. Three outcomes
only:

- **sync**: repo is behind and the user still works there
- **gc**: has dead branch DBs (independent of sync; a repo can need both)
- **remove**: the whole index isn't worth keeping

For that last one, remember the value test: an index earns its keep when the repo is too
large to hold in your head. That's btdash. It isn't an 11-file project, where reading every
relevant file costs less than a couple of graph queries. Recommend removal freely for the
small ones rather than defaulting to keeping everything.

## Act

**One repo at a time.** The survey covers everything at once; acting does not. Pick the
single repo the user named, or if they said "all of them", take the top of the table and work
down one by one. Finish that repo completely, report what changed, and wait for them to say
continue before touching the next. Never batch, never loop over the table, never fix a
second repo because you were already in there.

This is deliberate. Each action is aimed at exactly one repo, and handling them serially
means a wrong call costs one index instead of six, and the user gets a checkpoint to stop after
seeing what the first one actually did.

Each action is independently aimable:

```bash
tokensave sync <repo>                    # allow-listed, no prompt
tokensave branch gc --path <repo>        # prompts; removes DBs for branches gone from git
rm -rf <repo>/.tokensave                 # removes one index, exactly
```

A single repo may need more than one of these (behind *and* carrying dead branch DBs).
That's still one repo's turn, do both, then stop and report.

Re-verify with `tokensave status --json <repo>` afterward, except where the index was
removed entirely.

### Inspect before removing

Never delete an index you haven't looked inside. Before any `rm -rf` or `branch gc`, read
the actual contents of that repo's `.tokensave/`:

```bash
ls -la <repo>/.tokensave/
ls -la <repo>/.tokensave/branches/          # only exists when extra branches are tracked
cat <repo>/.tokensave/config.json
cat <repo>/.tokensave/branch-meta.json
git -C <repo> check-ignore -v .tokensave    # confirm it's an untracked local cache
```

Four things have to hold before deleting. If any fails, stop and ask the user:

1. **`config.json`'s `root_dir` matches the repo you think you're deleting.** If it points
   somewhere else, you are not standing where you think you are.
2. **The directory contains only tokensave's own files**: `tokensave.db`, `config.json`,
   `branch-meta.json`, a `branches/` directory, and possibly SQLite `-wal` / `-shm`
   sidecars. Anything else does not belong to tokensave and is not yours to remove.
3. **`check-ignore` confirms it's ignored.** The user's `~/.gitignore_global` has `.tokensave/`,
   so a tracked one means something is off.
4. **Every branch DB in `branches/` is accounted for** in `branch-meta.json` and has been
   checked against `git rev-parse`. Don't delete a branch DB you haven't confirmed is dead.

Then state plainly what is about to be destroyed, by path and size, and get a yes. "Removing
the index" is not specific enough; "removing `typing_practice/.tokensave/`, 536 KB, main
only, no branch DBs" is.

### Why `rm -rf` and not `wipe`

`tokensave wipe` takes no `PATH` argument. It always operates on the current directory
*plus its parents and its children*, so what it destroys depends on where you happen to be
standing, not on what you meant. From `~/Desktop/Projects` it takes all six indexes there.
There is no invocation that means "this one index."

`rm -rf <repo>/.tokensave` removes exactly one index regardless of cwd, and the directory
is a gitignored local cache, so nothing of the user's is at risk. Prefer it always.

`wipe` is set to `ask` in settings.json and `wipe --all` / `wipe -a` are denied outright.
Don't route around that. If a situation seems to call for `wipe`, say so and let the user run
it themselves.

## Guardrails

- One repo per turn. Finish it, report, wait. Never batch.
- Read before deleting, always. Run the Inspect before removing checks on the actual
  directory, don't infer its contents from the survey table. A survey row is a summary; the
  four checks are the thing that authorizes deletion.
- Never delete anything before printing which specific indexes and branch DBs die, with
  sizes, and getting a yes. The permission prompt shows the command, not the blast radius,
  so it is not the safety check here.
- Never `tokensave wipe --all` or `-a`. Denied in settings, and it empties every tracked
  project on the machine.
- Never run `tokensave init` here. Creating an index is a judgment call about whether a
  repo is worth indexing; this skill cleans up, it doesn't provision.
- Uncommitted changes can't be indexed. Don't propose a sync as the fix for them.
- Don't touch a repo the user didn't pick, even if the survey says it's the worst offender.
