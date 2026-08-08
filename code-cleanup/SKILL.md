---
name: code-cleanup
description: Clean up code that was just written, scoped to those changes only. Removes dead code, debug leftovers, and unneeded comments, condenses over-commented stretches, collapses duplicates, and checks naming.
---

## When to Use

Any request about cleaning up code that was just written:

- "clean up," "cleanup," "clean this up," "clean up the code," "code cleanup"
- "lets clean up our changes", "let's remove anything we don't need"
- "let's do some pr clean up", "lets clean up this branch"

# What to Look For

## Comments

Remove:

- Comments that restate the line below them (`# increment the counter` above `i += 1`)
- Any section that has like 5 or more comments which could be condensed down into 2-3 comments.
- Section-marker banners added to navigate the code while writing it
  (`# ---- helpers ----`) when the file has no existing convention for them
- Commented-out code, including a commented-out old version left next to the new one
- Comments the code already says on its own: a well-named function, a typed signature, or
  an obvious conditional does not need a sentence above it explaining what it is
- Comments narrating structure that is visible (`# loop through the results`,
  `# return the response`, `# imports`)

Condense:

Look for stretches of the new code where the comments outnumber what they earn, a comment
on nearly every line, a four-line block explaining a three-line function, the same point
made twice in different words. Collapse each stretch down to the one or two lines that
actually carry information and delete the rest. Prefer one comment above a block over a
comment on every line inside it.

Judge density against the file it lives in, not an absolute number. If the surrounding
code was already heavily commented before this work, match it. The target is the new code
reading like the rest of the file, not the sparsest possible version.

Keep:

- Comments explaining *why* something is the way it is: a workaround, a non-obvious
  constraint, a decision that looks wrong until you know the reason
- Docstrings and type/API documentation
- Comments that were already in the file before this work started
- TODOs pointing at real unfinished work, even ones written during this session

## Don't Repeat Yourself

Two shapes, and the difference decides which copy dies:

**The new code duplicates something the repo already had.** A helper written from scratch
that a util module already provides, a constant redefined, a query rebuilt when a service
method returns the same thing. Delete the new one and call the existing one. The
pre-existing version wins by default, it already has callers and test coverage.

**The new code duplicates itself.** Two near-identical functions added during the same
work, the same block pasted into three handlers, two variables holding the same value
under different names. Collapse to one and point everything at it.

### Finding it

Structural, so use tokensave before grepping:

- `tokensave_similar` on each function added, does something equivalent already exist
- `tokensave_redundancy` for repeated logic across the changed files
- `tokensave_search` on the concept, not just the name, before concluding nothing exists
  (a function called `fmt_money` will not turn up in a search for `formatCurrency`)

Then grep the changed files for repeated literals and duplicated blocks, tokensave sees
symbols, not copy-pasted bodies inside one function.

### When not to merge

Not all repetition is duplication. Leave it alone when:

- The two copies serve different domains and will drift apart. Merging them creates a shared function that grows a flag argument the first time one side changes.
- The similarity is incidental. Same shape, unrelated reasons.
- One is a test fixture or a fake. Tests are allowed to restate things on purpose.
- Merging them would couple two modules that are deliberately independent.

Three lines of honest repetition beat a premature abstraction. If the merge needs a boolean parameter to work, it is the wrong merge, say so and leave both.

### Doing it

Deleting a duplicate means editing its call sites, so this is the one part of the skill that changes working code. Keep it mechanical: delete the copy, repoint callers, change nothing about behavior. If the two versions are not actually identical (one handles a case the other misses), that is a decision for the user, report it instead of picking.

## Naming

`coding-standards` covers what a good name looks like in general. This section is the narrower question: does each name introduced by this work still fit what the thing became, and does it match the code around it.

### Names that drifted

The most common one. A name was accurate when it was written and stopped being accurate three edits later:

- The function grew past its name. `getUser` that now creates one when missing,
`validateInput` that also normalizes it, `filterActive` that now filters by role.
- The variable holds something else now. `userId` that holds a whole user object,
  `count` that holds a list.
- Iteration suffixes that survived. `sendEmailNew`, `UserServiceV2`, `handleSubmit2`, `processDataFinal`. If the old one is gone, the suffix is lying.

### Placeholder names

Scaffolding names from writing the code that were never replaced: `data`, `result`, `temp`, `obj`, `thing`, `helper`, `doStuff`, `x` outside a genuine one-line lambda. Say what it holds, not what shape it is.

### Names that ignore the file they live in

This is the "in context" half, and it matters more than generic descriptiveness:

- **Vocabulary.** Use the domain word this repo already uses. If everything else says `traveler`, new code saying `customer` reads as a different concept. Same for
`booking`/`reservation`, `org`/`tenant`, `trip`/`itinerary`.
- **Verb convention.** If the surrounding modules all use `fetchX` for network calls, a new `getX` doing a network call breaks the pattern that told you which was which.
- **Casing.** Match the language and file, not the last repo worked in.
- **Booleans read as booleans.** `is`, `has`, `should`, `can`. A bare noun holding a bool is a name that will be misread.
- **Abbreviations.** Only ones the repo already uses. Don't invent `usrPrefs` in a
  codebase that spells things out.

Grep a sibling module before deciding a name is fine in isolation.

### Renaming is not free

Rename blast radius is the reason this section is mostly a report, not an edit:

- Check call sites first with `tokensave_rename_preview` or `tokensave_impact`. tokensave sees committed code, so also grep the working tree for string references it cannot see: dynamic dispatch, `getattr`, template strings, test IDs, config keys.
- **Never rename across a serialization boundary on this pass.** API response fields, DB columns, ORM model attributes, event payload keys, anything in a JSON contract or a URL. A rename that looks internal but crosses one of these breaks a caller you cannot see from here. Flag it, don't do it.
- Only rename symbols introduced by this work. A pre-existing bad name is out of scope even when it is worse than the ones you are fixing.
- If a rename reaches beyond the files this work already touched, stop and ask. Cleanup should not widen the diff.

## Over-Engineering

Code that solves a problem nobody has. The test for every item below is the same: **name the concrete scenario this handles.** If you can name it, keep it. If the answer is "in case," it goes.

### Solving problems that don't exist

- **Handling states that cannot occur.** A null check on a value constructed non-null two lines up, a branch the type system already rules out, an `else` for a condition that is always true, a default case in a match over a closed enum where every variant is covered.
- **Catching errors that never fire.** A `try/except` around code that cannot raise, a bare `except Exception` that logs and continues so a real failure passes silently, a retry loop on an in-memory call.
- **Edge cases with no caller.** Empty-list handling where the only caller guarantees non-empty, pagination for a query capped at five rows, timezone conversion in a codebase that is UTC end to end, an `isinstance` branch for a type nothing passes.

Check it, don't assume it. Read the function being called before deciding it cannot raise, and use `tokensave_callers` to see what actually reaches the branch. "I don't think that happens" is not the same as having looked.

### Building more than was asked for

- **Abstraction with one user.** A base class with one subclass, an interface with one implementation, a factory that builds one thing, a wrapper that only forwards its arguments.
- **Parameters nobody varies.** A config flag with one value at every call site, an option threaded through four functions that is always `True`, a `**kwargs` nothing passes.
- **Output nobody reads.** Fields on a returned dict no caller touches, an enum with values nothing produces, a return value every caller discards.
- **Layers that pass through.** A service method that only calls the repository method with the same arguments, a hook that only re-exports.

If it exists because it might be useful later, delete it. Adding it back when a second caller appears is cheap, and by then you will know the actual shape it needs.

### Fitting the codebase

The other half of over-engineering is building your own version of something the repo already decided. New code should look like it belongs to the project, not like it was written next to it.

- **Go through the layers that exist.** If every other route calls a service that calls a repository, a new route querying the DB directly is not simpler, it is off-pattern.
- **Use the repo's own tooling.** Its HTTP client, error types, logger, config loader, date handling, validation. Rolling a fresh one because it was faster in the moment is the same mistake as a needless abstraction.
- **Put files where that kind of file lives.** A new helper in the module that already owns that concern, not a new directory next to it.
- **Match how similar things are already done.** Read a sibling that does the same job before deciding the new code is fine. If the repo has three examples of this pattern and the new code is the fourth shape, that is the finding.

If the new code genuinely needed a pattern the repo does not have, that is an architecture decision and not a cleanup one. Say so and let the user decide, do not quietly rewrite it into the old pattern or quietly leave a new one in place.

### Leave it alone

Do not strip these, they look defensive but the failure is real:

- Anything crossing a boundary: network, filesystem, database, subprocess, third-party API, user input. Those fail for real and the handling is not speculative.
- Validation at a trust boundary (API endpoints, form submissions) even when an inner layer validates too.
- Code with a test asserting it, or a comment naming the case it caught. Both are evidence somebody hit it.
- Structure a framework requires: an abstract method, an interface a library demands, a lifecycle hook that must exist.
- Anything guarding a case a comment says went wrong before. That is a bug report, not a hypothetical.

### Removing vs reporting

Deleting error handling is the riskiest thing in this skill, so split by blast radius:

- Just remove: unused parameters, unread return fields, single-implementation abstractions, and pass-through layers, all of them added by this work.
- Report to the user first: anything on an error path, and anything where "this cannot happen" rests on an assumption about data rather than something the code or types guarantee. Say what you would remove and why you think the case is unreachable, let them make the call.
