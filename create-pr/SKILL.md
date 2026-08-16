---
name: create-pr
description: Open a brand new PR for the current branch. Gathers the branch/commit state and runs gh pr create with the standard title convention and body template.
---

## When to Use

- The user asks to "create a PR," "open a PR," "create pr," or "make a PR"
- The current branch has commits ready to open as a brand new PR

# Create PR

When the user asks to open a brand new PR for the current branch:

1. `git status -sb`, uncommitted changes and upstream tracking
2. `git branch -vv`, confirm the current branch (never open a PR from `main`)
3. `git log --oneline origin/main..HEAD`, commits that will land in the PR (adjust `origin/main` if the default branch differs)
4. `git diff origin/main...HEAD`, full diff the PR will contain

Then draft the PR and run `gh pr create --title "..." --body "$(cat <<'EOF' ... EOF)"`.

- Never include a "Generated with Claude Code" attribution footer in the body.

## Title

Format: `type(scope): description`

- `type` is one of `fix`, `feat`, `remove`, `change`, `test`, `refactor`, `cleanup`.
- `scope` is a short area name reflecting what part of the app changed, e.g. `auth`, `notifications`, `billing`, `search`.
- `description` is a concise summary of the change, not a copy of the branch name.
- Use `test` only when the PR's main content is new tests or eval scenarios themselves (not a product change that happens to include tests). For `test`, `scope` names the test layer instead of an app area, e.g. `e2e`, `backend`, `frontend`, `evals`.
- For `fix`, `scope` can also name the concern being fixed instead of an app area, e.g. `proxy`, `test`, `eval`, `response`, `cost`, `ui`, `guardrails`, `scripts`, `devcontainer`, `bug`, `hookify`.
- For `feat`, `scope` is whatever app area or feature the PR adds to, wide open, e.g. `auth`, `billing`, `search`, `notifications`, `skills`, `button`, `admin`, `settings`.
- Use `refactor` only when there's no behavior change, restructuring code, not what it does. `scope` names the thing restructured, e.g. `ui`, `api`, `skills`, `evals`, `tests`, `proxy`, `repo`.
- Use `cleanup` for removing dead code, unused files, stale comments, or tidying without restructuring anything (that's `refactor`) or changing behavior. `scope` names the thing cleaned up, e.g. `ui`, `tests`, `evals`, `frontend`, `backend`, `docs`, `database`, `analytics`, `skills`.

Examples:
- `add/slack-notification-data` → `feat(notifications): add Slack alerts for price drops`
- a branch fixing a stop-button race during checkout → `fix(agent): stop button race during in-flight checkout payment`
- a branch adding eval scenarios for a search fallback path → `test(search): add eval scenarios for results local-table miss fallback`

## Body

```
## Summary
- Bullet points of what changed and why.

## Testing
- Edge cases and important logic paths introduced or changed by this PR.
- Only add test cases that you or Claude can actually go run locally, right now.
- [ ] test case example.

## Screenshots
- Screenshots of what was added and what the new changes look like visually.
- The user adds these screenshots themselves.
- You don't need to add anything here.
```

Keep the whole description short and concise.

## Linking a GitHub Issue

Only add a "Fixes #N" line to the PR body when a GitHub issue was actually mentioned
somewhere in the conversation, never inferred from the branch name, commit messages, or
guessed on your own.

- If a specific issue number was mentioned, use it directly.
- If an issue was referenced without a number (described, not numbered), search open
  issues with `gh issue list` to find the match. If more than one issue could fit, confirm
  with the user before adding it.
- Add a single `Fixes #N` line as the first line inside `## Summary`, not as its own section.
- If no issue came up anywhere in the conversation, skip this section entirely, don't go
  searching for a related issue on your own initiative.

## Summary

The point of a summary is to be read and digested fast, not to fully document the change.

### Style

Each bullet is 1-2 sentences, plain and direct.
- State what changed and its practical consequence. Skip the narrative build-up (don't restate the bug's mechanism before the fix if the fix line already makes it obvious).
- Don't cite specs, RFCs, or standard numbers unless the number itself is load-bearing (e.g. needed to explain why a fix is unusual). "The provider has no revoke endpoint" is enough; "(it does not implement RFC 7009)" is not.
- Don't restate a fix's guarantee in a full explanatory clause when the mechanism already implies it. If the fallback is "still deletes on failure," that's the whole sentence, don't add "so a provider outage cannot trap a user in a connection they cannot remove."
- One idea per bullet. If a bullet needs "and" to stitch two separate behaviors together, split it into two bullets.
- Keep to implementation-free, user-visible language: state the result, not the trail that led to it. Leave out mechanics behind the fix (what data was already available vs. what was missing, which endpoint already returned what) and scope caveats about untouched paths ("X remains unchanged") unless a reader would otherwise wrongly assume the change applies there.

### Don't include

- Cosmetic or non-logic edits that ride along with the real change (wording simplification, punctuation cleanup, whitespace). E.g. trimming a redundant parenthetical from an explanation string, removing an em dash from helper copy.
- References to other PR numbers by default. Only cite another PR when it's actually necessary context (this PR directly builds on, reverts, or fixes a specific recent PR), not as a habit, and not for PRs from long ago in the repo's history.
- Variable, field, or parameter names (e.g. `grant_revoked`, `account_email`, `arrival_dt`), or asides about a field's type, optionality, or nullability: describe what changed in plain language, not by naming the code-level identifier that represents it.

### Exception, do include

- Changes to instructional text consumed as an instruction or prompt rather than display copy (agent/LLM system prompts, policy rule instructions fed to a model): even a small wording change there can meaningfully change behavior, so it belongs in the Summary.

## Testing

Only covers cases that you or Claude can actually go do locally, on your machine, right now. Every case is something you can click through the running app (or a real external system it talks to that you can reach from here) and watch happen, not a description of what changed in the diff. If a case needs staging, prod, or access neither of you has, it doesn't belong here. If a case can only be checked by reading source, running a script, or inspecting the database, it doesn't belong here either.

Before writing test cases, slow down and think through the diff's actual logic: what branches, conditions, or new paths did this change introduce. Each case should come from that thinking, then be written as something you'd do locally, not a summary of the logic itself.

### Style

Describe outcomes the way a person testing the app would actually notice them, not the way the code represents them internally.
- Good: on-screen labels and states, whether something errors/hangs/silently no-ops, checks on an external system the feature integrates with (e.g. the provider's own account page), whether a stale UI state comes back after a reload.
- Not good: internal field names, spec terminology, log line labels, record/event IDs, or counts of internal objects. If the check isn't something you'd see by using the app or the third-party system it talks to, it doesn't belong in a manual test case, that's what unit tests are for.
- Don't skip the *why* behind an expected outcome (e.g. "since the range end is exclusive"), just state what you'd see, drop the mechanism.
- Don't invent specific setup detail (a particular record, date, item) unless that specificity is actually what the case is testing. If the case works identically with any input, don't name one.

### Include

- Ground every case in this branch's diff: it must exercise a path that changed (new logic, a modified conditional, a new UI state), not pre-existing behavior the branch left untouched. If a scenario would pass identically on `main`, drop it.
- Each item is one complete, self-contained scenario, not a fragment that depends on a separate bullet's setup ("half" a test case).
- Each item tests exactly one feature, or one edge case that could plausibly fail.
- Diversify on logic, not surface dressing: each item targets a distinct logic path or edge case from every other item in the list. Reusing the same input across cases is fine, two cases covering the same underlying logic are wasted testing even when dressed up with different examples.
- Keep a case only when its variation reaches genuinely separate handling (e.g. an accept path vs. a decline path with its own logic). A case phrased as a bare variation of another ("same scenario, but then X happens" / "same setup, decline instead") with no new logic behind it is a near-duplicate, cut it.
- Narrate it as a full flow: starting state, each action taken, and the expected outcome after each action (including what the agent/UI does), driven through the actual app, not read off the code.
- Negative cases at each step matter as much as what should happen (agent asks nothing, no warning shown, etc).
- Cross-field/cross-surface consistency: a value shown in one place matches another, or a total matches its components.
- Exact UI copy, labels, or displayed values that need to be correct.

Example pair showing the target format (same logic path, each case fully self-contained with its own distinct setup):
```
- [ ] On the dashboard, select "Widget A" then "Widget B" as dependent items. Ask to "swap Widget B for a similar option" (naming no vendor): the alternatives render immediately, span at least 2 vendors, and the agent asks nothing about Widget A and does not warn about the dependency. Pick a different-vendor alternative: the agent sets it without asking, then tells you the dependency broke, states the updated total, and offers a matching partner item. Accept it and pick a matching partner: the original Widget A is removed and the selection becomes a single bundled pair.
- [ ] On the dashboard, select "Widget A" then "Widget B" as dependent items. Ask to "switch Widget A to a different vendor": the alternatives render, span at least 2 vendors, and the agent asks nothing about Widget B and does not warn about the dependency. Pick a different-vendor alternative: the agent sets it without asking, then tells you the dependency broke, states the updated total, and offers a matching partner item. Decline it: the selection ends up as the new Widget A plus the original Widget B, as a mixed pair.
```

### Don't include

- Unit tests, those run automatically in CI.
- A case that's really just a unit test restated as a manual step: if it's already asserted in code, walking through it by hand adds nothing.
- Test scripts or test files.
- A dev script's own CLI plumbing: argparse guards, flag rejection, "the script errors correctly", "the flag is accepted". Test what a flag lets you *see* in the product, never the flag validating itself. Rejected example:
  ```
  - [ ] Run `--all-items --send` and `--warn-unprocessed --send` with no `--to`. Both refuse with an
        error naming the flag, and no mail is sent. Pairing either with `--to EMAIL` is accepted.
  ```
  The flag belongs in a case only as the means of reaching a product state, e.g. "Run `--slot 1 --warn-unprocessed`: the email carries the 'Still Needs Action' section."
- Local DB tests / direct DB inspection.
- Checks that a UI element simply renders or appears on the page.
- Checks for any .env variables existing, or cases built around manipulating/naming specific env var keys (e.g. "with `CLIENT_ID` set on the frontend but not the backend"): that's a deployment/config scenario, not something reached by using the app, and env var names don't belong in a PR body. Unless naming the var is genuinely the only way to demonstrate the behavior being tested, in which case include it.
- Environment/setup prerequisites needed just to reach the point where a case can run (enabling a feature flag on your org, running a migration, registering a real third-party OAuth app, setting env vars in `.env`/`.env.local`, granting delegated permissions, configuring a redirect URI, supported account types). A case assumes local setup already works, it doesn't explain how to configure it.
- Any github checks. (pre-commit hooks, PR deploy, backend tests, frontend tests, etc)
- Any step that reads a frontend or backend log, even as one part of a larger case. Local testing exercises the app's behavior, so every expected outcome is something the app itself shows you.
- Tested locally.
- Migration Test.
- A case that requires waiting for elapsed time to pass (a scheduled job, a delayed email, a TTL expiring): if it can't be observed immediately, it doesn't belong here.

### Data-setup scripts

A script that manipulates data to reach a starting state is allowed only when that state comes from something that can't be produced locally on demand, a real-world condition like a market price drop, not something reachable by clicking through the app or waiting. It has to be an existing, purpose-built dev tool, not an ad hoc script or raw query written in the moment, and it should only touch the specific row it's testing.

That script sets up the precondition. It is never the test case itself, and its own output is never what the Testing bullet describes. The bullet is what you then see happen in the running app once the real logic runs on top of that setup.

A unit test is not this. A unit test is code that runs in CI and asserts on its own, nobody watches it happen. It never counts as a Testing section case, restated as a manual step or otherwise.
