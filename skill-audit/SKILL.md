---
name: skill-audit
description: Survey Claude Code skills across settings.json, disk, and actual usage, and clean up the dead ones. Use when the user says "clean up my skills," "skill audit," "check for unused skills," or asks which skills they can delete. Reports which skills are disabled-and-unused, active-and-never-used, have dangling config (skillOverrides or .gitignore entries pointing at a deleted directory), or look mis-structured (nested too deep, name field mismatched). Never deletes anything before naming exactly what dies.
---

## When to Use

- The user says "clean up my skills," "skill audit," "check for unused skills"
- Asks which skills they can delete, or whether any skills are dead weight

# Skill audit

Survey first, act second. The survey is read-only; nothing is deleted or edited until the
report is in front of the user and they've said which skills to act on.

One thing to get right first: `~/.claude` is a symlink to `~/dotfiles/.claude` (see
`~/dotfiles/README.md`). There is one real copy of every skill, not two synced ones. Don't
go looking for drift between `~/.claude/skills/` and `~/dotfiles/.claude/skills/`, they're
the same file on disk.

## Survey

Gather four things.

**1. Disabled skills**, from `skillOverrides`:
```bash
jq '.skillOverrides' ~/.claude/settings.json
```

**2. What's actually on disk**:
```bash
ls ~/.claude/skills/
```

For each directory, check its own `name:` frontmatter and its nesting depth:
```bash
for d in ~/.claude/skills/*/; do
  echo "$(basename "$d") -> $(grep -m1 '^name:' "$d/SKILL.md" 2>/dev/null | cut -d' ' -f2-)"
done
find ~/.claude/skills -mindepth 3 -name SKILL.md   # anything found here is nested too deep
```

`skillOverrides` and Skill-tool invocations both key off the **directory name**, not the
frontmatter `name:` field, so a mismatch there doesn't break dispatch, it's just confusing
to read. Nesting depth is the thing that's actually broken a skill before (see
`~/dotfiles/.claude/ISSUES.md`, "a skill silently stopped loading after its directory
moved") — `<skills-root>/<name>/SKILL.md` must be exactly one level deep.

**3. The `.gitignore` whitelist**, since this repo tracks skills opt-in:
```bash
grep -n "!/.claude/skills/" ~/dotfiles/.gitignore
```
Use `Read` instead of Bash `grep` if this comes back garbled ("N matches in 0 files") —
that's the known rtk output bug, not an empty result.

**4. Real usage**, from every session transcript:
```bash
find ~/.claude/projects -name "*.jsonl" -print0 | xargs -0 jq -r \
  'select(.message.content != null) | .message.content[]? |
   select(.type=="tool_use" and .name=="Skill") | .input.skill' 2>/dev/null \
  | sort | uniq -c | sort -rn
```

## Report

One table: skill, on/off, use count, flags. Flags are: dangling `skillOverrides` entry (off,
no matching directory), dangling `.gitignore` entry (whitelisted, no matching directory),
nested too deep, name-field mismatch.

Classify each skill with a directory into one of:

- **Dead weight** — disabled *and* zero uses. The clear delete candidates.
- **Never used, but active** — zero uses and not disabled. Flag these separately, don't
  fold them into "dead weight": some are new, or meant to be referenced rather than
  invoked directly. Ask before assuming they're unused-and-unwanted.
- **Rarely used (1-2x)** — note the count, don't recommend deletion on your own. Low
  frequency isn't the same as dead; some things (migrations, evals) are rare by design.
- **Config-only cleanup** — a `skillOverrides` or `.gitignore` entry with no directory
  behind it. Nothing to delete, just stale config. Fine to clean without a separate
  approval round since there's no actual skill content at stake, but still say what you're
  removing.

## Act

Only after the user names which skills to remove (batching is fine here, this isn't
tokensave-cleanup's one-repo-at-a-time situation, there's no shared index that makes
partial cleanup risky).

For each skill directory being deleted, check it's not mid-edit first:
```bash
cd ~/dotfiles && git status --short -- .claude/skills/<name>
```
Uncommitted changes mean active work-in-progress, not an unused skill. Exclude it and say
why, don't delete over someone's in-flight edit just because usage looks like zero.

Then, for every skill actually removed, all three of these, not just the directory:
```bash
rm -rf ~/dotfiles/.claude/skills/<name>/
```
- Remove the matching key from `skillOverrides` in `~/.claude/settings.json`, if present.
- Remove the matching `!/.claude/skills/<name>/` line from `~/dotfiles/.gitignore`, if
  present.

Skipping the last two is exactly how the dangling-config flags in Survey get created in
the first place.

## Guardrails

- Read-only until the user says which skills to act on. Survey and report first, always.
- Never lump "rarely used" in with "dead weight" automatically. Name the count, let the
  user decide.
- Check `git status` on a skill's directory before deleting it. Uncommitted work means skip
  it and flag it, not delete it.
- When deleting a skill, always clean up its `skillOverrides` and `.gitignore` entries in
  the same pass. A directory-only delete just creates the next audit's dangling-config
  findings.
- Everything under `~/dotfiles/.claude/` is ignored by default (see the `/.claude/*` /
  `!/.claude/skills/<name>/` pattern in `~/dotfiles/.gitignore`); a skill only gets
  git-tracked if it has its own explicit `!` line. That means `.gitignore` has to move in
  step with the skills directory in both directions: add a skill without adding its line
  and it silently never gets tracked, no error, delete a skill without removing its line
  and the line just dangles. This applies generally, whenever a skill is added or removed
  anywhere in this repo, not only during an audit.
- Don't touch a skill the user didn't approve, even if it's the survey's clearest
  dead-weight case.
