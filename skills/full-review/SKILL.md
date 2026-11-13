---
name: full-review
description: Review the change as a stranger would — bugs, security holes, slow paths, convention violations, gaps, untested behavior — and collect the implementor's take on each finding
---

Review a change the way a maintainer would: fresh, with no memory of how it was built.

You built this change, and a reviewer who shares your context inherits your blind spots. So the review itself runs in subagents that see only the diff, the task, and the conventions — **never pass them the conversation that produced the code**. You then check what they found and report it to the user.

## 1. Scope

**The change** is everything on this branch that is not on the base branch: committed, staged, unstaged, and untracked. Untracked files don't show in a diff; include each one as wholly added.

The base is the branch this one would merge into, taken from the remote (`origin/main`), not the local branch of the same name — the local one is often behind, and then other people's merged work shows up as this change. Use the local branch only when there is no remote. If you can't tell the base, ask.

The skill reviews whatever is checked out and takes no arguments; anything the user adds is a steer, not a target to fetch.

**The task** — one or two sentences on what the change is for, from the PR title or commit messages. It helps tell a deliberate change from a stray one. It is context, not a spec: drift between it and the diff is not a problem to report.

## 2. Conventions

Read `CLAUDE.md` / `AGENTS.md` at the repo root and in every changed directory, and pass their full text to every subagent. Where they say nothing, the surrounding code is the rule.

## 3. The bar

Report only what concretely goes wrong — a bug, a leak, a misleading reader, a rule broken — and only when you're confident it's real. A preference is not a problem. Most changes have zero to two real problems in any one area; returning none is normal.

Weigh the harm against the fix. A real but rare problem with modest impact — a race that hits one user in ten thousand — is a tradeoff to present, not something to fix by default. Fixing it is the right call only when the fix is cheap and adds no new state, coupling, or failure modes. Rarity doesn't excuse security holes or data loss. Prefer the simplest adequate fix, including simplifying the design.

Out of scope: formatting and anything a linter owns, and problems the diff didn't introduce or touch.

## 4. Review

Launch these six as parallel `general-purpose` subagents and wait for all of them (`run_in_background: false`). If your harness can't run subagents to completion within this turn, apply each area yourself instead, one after another.

Each gets the diff, the task, the conventions, the bar, and its area below. Each may read beyond the diff where it helps. Each returns short notes for you, not prose for the user: `file:line`, what's wrong, the evidence, a suggested fix. Only problems: what it checked and found fine stays out, and an area with nothing to report says so in one line.

1. **Correctness** — what breaks at runtime by accident: edge cases, concurrency, retries. Look hardest at silent behavior change — what happens to existing data, callers, and users — and at checks that were silenced instead of answered: a `catch` that swallows, a `?? []` hiding a real null, a cast or `!` asserting away a case the type checker found, a default or optional parameter every caller already passes.
2. **Security** — what breaks with an adversary: missing or wrong-subject authorization, injection, leaked secrets, user input steering a fetch, redirect, or file path, user data reaching logs or third parties. No entry point or user input means nothing to report.
3. **Performance** — what gets slow or expensive as data grows: N+1 queries, unbounded reads, missing indexes, repeated or serial work, leaked handles. Name the size where it hurts. No data access or growing loops means nothing to report.
4. **Structure** — conformance with the conventions, and with the neighboring code where they're silent. Quote a rule word for word before calling it broken. Read callers and neighbors: a responsibility in the wrong place only shows from outside the diff. Also: boundaries bypassed, leftovers nothing uses, unneeded or misplaced dependencies, comments the code no longer matches, and code smells the change introduces.
5. **Gaps** — is the change whole, and only the change? Config that half-landed, a twin call site not updated, docs or copy describing the old behavior, hunks unrelated to the task.
6. **Tests** — would the suite catch a break in this change? Either nothing exercises the new behavior, or a test exists that would still pass with the behavior broken: assertions on mocks instead of outcomes, a name the body doesn't test, an assertion weakened until it went green. Say how the code could break with the suite still green.

## 5. Report

Check each note against the code before you use it — a plausible wrong problem left standing teaches the user to ignore the report. Then write the report yourself, in the words of the conversation that built the change: its names for things, its decisions. Don't quote the subagents.

Number entries continuously. **Agree** entries come first, most important first, then **Trade** entries:

```
### 1. <Name> — <short title> (path/to/file.ts:42)

<The problem, in one to three plain sentences.>

**Agree** — <the fix: what changes, and where.>
```

- **Agree** — real, and a proportionate fix is worth doing. Name the fix concretely enough to act on: what changes, and where — not just the kind of fix.
- **Trade** — real, but the fix costs enough to need a decision. Give the impact, how often it happens (or that it's unknown), and what the fix leaves behind, then recommend one way: leave it, a light mitigation (a clarifying comment, a guard, a log line), or the full fix. Usually it's one of the first two; name which.
- **Refute** — wrong, for a reason you can name.

When a problem has an established name — a refactoring smell (Shotgun Surgery, Feature Envy, Primitive Obsession), a bug class (N+1, TOCTOU race, lost update), a pattern — lead the title with it; the user wants to learn the vocabulary. Otherwise just title it.

Say each thing once: the problem, then your verdict and recommendation, no restating. Write about the code, not the review — no "finding", "reviewer", "lens", "victim", or who would be affected; the user can tell. An area with nothing to report gets no entry and no mention.

Refuted problems, and trades you'd leave alone, go after the entries in a list, still numbered. The user never saw the notes behind them, so each one opens with **Refute** or **Leave**, then states the concern so it stands on its own — what would go wrong, and where (`file:line`), in plain terms — then why it's wrong or not worth fixing. Two or three sentences is fine.

If something couldn't be checked — a service you couldn't run, a file you couldn't read — say so in one line; otherwise say nothing about coverage. Last, one line proposing what to do, by number — "I'd fix 1–5, add a comment for 6, and leave 7." If nothing survives, say so — a clean review is a real result.

Print the report in full as your final message. Apply no fixes; the user decides.

If the user then asks to fix what you agree with, that means **Agree** only; a **Trade** needs its own decision. Fixing per your recommendations means the **Agree** fixes plus each **Trade**'s recommended option. If an agreed fix turns out to need disproportionate machinery, stop on that one, present it as a **Trade**, and continue with the rest.
