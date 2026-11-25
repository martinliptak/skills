---
name: retro
description: Look back over the past week of agent sessions in this repo and propose what would make the next ones go faster — a check, a pointer, committed tooling, a line in the agent's instructions — and how the user could ask for things more directly and write their messages faster. Use weekly, or after a session that was slower or harder than it should have been. Not for judging the code the sessions produced; use full-review for that.
---

Look back over recent sessions in this repo and find what made them slower or harder than they needed to be. Then propose changes to the agent's environment so the next session doesn't hit the same thing, and tips that help the user say what they want sooner.

## 1. Sessions

By default, read every session from the past 7 days that ran in this repo. The user can narrow that: "this session", "last 3 days", a transcript file.

Look in the session logs of every coding agent on the machine, not only your own; the user may switch between them. Claude Code keeps them in `~/.claude/projects/`, Codex in `~/.codex/sessions/`. A session ran in this repo if it ran in one of the repo's worktrees too, including ones since deleted.

If you find no logs, read the current conversation and say the report covers only that. Never quote a secret, token, or credential from a log.

## 2. Read

A week of logs doesn't fit one context. Fan the sessions out to subagents, giving each sections 3 and 4. Each returns short notes: every moment that cost time, tagged with session and timestamp, and phrases the user typed more than once. Merge them; a problem that recurs across sessions is one entry citing each.

## 3. The bar

Every entry cites the moments it came from. Count the cost in turns, tool calls, corrections, and words the user had to type. What recurs ranks first; a one-off counts only when it was costly. "Nothing to change" is a real result.

## 4. What to look for

On the agent's side:

1. **Rebuilt tooling** — scripts the agent wrote to do or check its work, such as taking a screenshot or seeding data, that every session builds again. Propose committing it, as a script or a project skill, with a pointer so the next session finds it.
2. **Navigation** — a file, fact, or command that took many searches to find.
3. **Automated checks** — mistakes a lint rule, type, test, or hook could have caught. Look at the existing checks and CI first; one that exists but isn't wired up is the finding.
4. **Missing information** — logs, state, or access the agent needed and couldn't reach.
5. **Expensive calls** — tool calls that returned far more than needed, or repeated what one call could do.
6. **Instructions** — a line in the agent's instruction files (such as `CLAUDE.md`, `AGENTS.md`) that was missing, ignored, misread, or contradicted, or one that changes nothing.

On the user's side:

7. **Phrasing** — requests the agent misread, corrections the user repeated, context that would have saved a round trip if it came first.
8. **Shorthand** — common chat abbreviations for phrases the user keeps typing: `wdyt`, `lgtm`, `nvm`, `iirc`. For a long instruction they keep repeating, propose a term of their own with the line that defines it in their instruction files; never one the agent would have to guess.
9. **Shorter messages** — when the user's messages habitually run long, quote one and show how a native speaker would say it in fewer words. Don't correct grammar or spelling the agent understood.

Prefer the fix that works without anyone remembering it: a check over a rule, a committed script over a tip, a line in the instructions over a habit.

## 5. Report

Number the entries, the costliest first:

```
### 1. <Category> — <short title>

<What happened, in one or two sentences, citing each session and moment.>

**Change** — <the exact text to add or remove and in which file, or the script, check, or skill to commit and where.>
```

End with one line proposing which entries to take — "I'd do 1–3 and skip 4." Apply nothing; the user decides.
