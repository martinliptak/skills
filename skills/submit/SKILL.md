---
name: submit
description: Bring the branch up to date with its base, trim the comments it adds, push, open or update the pull request, and watch the checks until they pass. Use when the branch is ready for review. It never merges.
---

Take a finished branch to a pull request. Never merge it, and never push to the base branch.

The base is the remote branch this one would merge into (`origin/main`), not the local copy, which is often behind. If you can't tell, ask. Where the project's instructions say how to do a step, follow them. Commit what belongs to the change first.

1. **Update.** Merge the remote base into the branch, or rebase if the project does, and then push with `--force-with-lease`, never `--force`. A clean merge can still be wrong: look where the base's changes meet the branch's code and fix what they now mean differently, at each conflicting commit when rebasing. Run any local step the incoming changes need, such as migrations.

2. **Trim comments.** You can't read your own comments as a stranger; a fresh subagent can. Give it only the diff, the standard below, and any comment rules the project sets, never the conversation. It checks every comment and docstring the diff adds or touches, except Markdown, LLM prompts, and comments a tool reads. For each, it writes the shortest comment a future reader needs, or nothing, and returns only the changes: `file:line` and the new text. Apply them, but skip a change that removes a real constraint. If you can't run a subagent, do this yourself from the diff alone.

   A comment says what the code cannot: why, a hidden constraint, a surprising edge case. It never mentions the session: what was tried, what changed, what was asked for.

3. **Check.** Run the project's check command once and fix what fails. Leave the tests to CI.

4. **Push**, then create the pull request, or rewrite the title and body of the open one.

   The pull request describes the net diff against the base branch, not the commit history. Use the repo's pull request template if it has one. If not, use these sections, and omit a section that does not apply:

   - `## Problem` — why the change exists. End with the ticket link if there is one.
   - `## Approach` — lead the reviewer through the change in reading order. Wherever you weighed another approach or implementation, including ones you tried and dropped, name it and say why this one won, so the reviewer doesn't propose it again.
   - `## Test Plan` — a `- [ ]` list of manual checks. Nothing CI runs.
   - `## Release Notes` — two to four plain sentences for users.
   - `## Deploy Steps` — a `- [ ]` list of what someone must do by hand around the deploy: env vars to set, commands to run. Mark each step before or after deploy, and give the exact variable name or command. Nothing the deploy pipeline runs itself.

5. **Watch the checks** until they pass. Fix the cause of a failure; never loosen a check. Report a failure that isn't the branch's fault.
