---
name: submit
description: Bring the branch up to date with its base, trim the comments it adds, push, open or update the pull request, and watch the checks until they pass. Use when the branch is ready for review. It never merges the pull request.
---

Take a finished branch to a pull request a reviewer can read: current with its base, free of comments only its author needed, described by what it changes, and green.

**Never merge the pull request, and never push to the base branch.** Both stay with the user. If the branch you're on is the base, stop and ask.

## 1. Scope

The base is the branch this one would merge into, taken from the remote (`origin/main`), not the local branch of the same name — the local one is often behind. Use the local branch only when there is no remote. If you can't tell the base, ask.

Commit any uncommitted work that belongs to the change first, so the update and the pull request include it. If a file doesn't look like part of the change, ask before committing it.

Read `CLAUDE.md`, `AGENTS.md`, and `CONTRIBUTING` at the repo root and in every changed directory. They may set how branches are updated, how comments are written, what the check command is, and how pull requests are titled. Where they say something, they win over this skill.

## 2. Update from the base

Fetch, then bring the base's changes in the way the project does. If it says nothing, merge the remote base branch into this one.

A clean merge can still be wrong. Look where the incoming changes meet the branch's code — a function the branch calls that the base renamed or changed, a table both touch, a config key both read — and fix what they now mean differently. If the base brings new migrations, apply them to the local database.

If the project rebases, fix where the base's changes meet the branch at each conflicting commit, so every commit still makes sense on its own. Don't run the check at each one; step 4 runs it once.

## 3. Trim comments

You wrote these comments, and you can't read them as a stranger would. A fresh subagent can. Launch one `general-purpose` subagent and wait for it (`run_in_background: false`). **Give it only the diff against the base, the standard below, and any comment rules the project sets — never the conversation that produced the code.** If your harness can't run a subagent to completion within this turn, do the trimming yourself, reading only the diff.

It checks every comment and docstring the diff adds or touches, but not Markdown files, LLM prompts, or directives a tool reads (lint and type-checker suppressions, pragmas, license headers). For each one, it writes the shortest comment a future reader needs, or nothing. It returns only the changes: `file:line` and the new text, or that the comment goes.

The standard:

- A comment says what the code cannot: why, a hidden constraint, a surprising edge case.
- A comment that repeats what the names already say goes.
- A comment never mentions the session: what was tried, what changed, what someone asked for. That belongs in the commit message or the pull request.

Apply the changes. Skip one that removes a real constraint — a reason the code must stay as it is that nothing else records. Commit.

## 4. Check

Run the project's check command once — lint, type check, format, whatever the conventions or the package scripts name. CI runs the tests, so don't run the suite unless the check includes it. If a failure comes from the update or the trim, fix it and commit. If the project has no check command, say so in the report.

## 5. Push and describe

Push. If you rebased, push with `--force-with-lease`, never plain `--force`.

If the branch has an open pull request, rewrite its title and body; otherwise create one against the base. Never open a second pull request for the same branch.

The title says what the change does, in the repo's style if it has one. The body describes the net diff against the base, not the commit history. Use the repo's pull request template if it has one. If not, use these sections, and omit a section that does not apply:

- `## Problem` — why the change exists. End with the ticket link if there is one.
- `## Approach` — lead the reviewer through the change in reading order. Name an alternative only where there was a real decision.
- `## Test Plan` — a `- [ ]` list of manual checks. Nothing CI runs.
- `## Release Notes` — two to four plain sentences for users.
- `## Deploy Steps` — a `- [ ]` list of what someone must do by hand around the deploy: env vars to set, commands to run. Mark each step before or after deploy, and give the exact variable name or command. Nothing the deploy pipeline runs itself.

## 6. Watch the checks

Watch the pull request's checks until they finish (`gh pr checks --watch` on GitHub). When one fails, read its log and fix the cause, then commit, push, and watch again.

**Never loosen a check to make it pass**: no skipped or deleted tests, weakened assertions, raised timeouts, disabled rules, or retries until a flaky test goes green.

A failure that isn't the branch's fault — the base is already red, a service is down, a test fails the same way without this change — gets reported, not fixed here. Say what failed and why you think it isn't the branch.

## 7. Report

End with the pull request link, then one line each for anything the user should know: conflicts or semantic fixes from the update, migrations applied, comments trimmed and any skipped, check failures you fixed, and checks still failing or not run. Nothing else.
