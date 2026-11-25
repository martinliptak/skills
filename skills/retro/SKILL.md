---
name: retro
description: Look back over the current session and propose what would make the next one go faster — a check, a pointer, committed tooling, a line in the agent's instructions — and how the user could ask for things more directly and write their messages faster. Use at the end of a session that was slower or harder than it should have been. Not for judging the code the session produced; use full-review for that.
---

Look back over this session and find what made it slower or harder than it needed to be. Then propose changes to the agent's environment so the next session doesn't hit the same thing, and tips that help the user say what they want sooner.

Read the session itself: the current conversation, or a transcript file the user points you at. If the context was compacted and only a summary is left, say that the early part can't be checked, and work from what's left.

## 1. The bar

Every candidate has to come from a specific moment in this session: a misread request, a wrong turn, a search that took many tries, something built from scratch. If you can't point to the moment, drop the candidate. Generic advice is noise.

Count the cost in turns, tool calls, corrections, and the words the user had to type. Something that cost one extra call isn't worth a permanent change. A smooth session has little to teach, and "nothing to change" is a real result.

## 2. What to look for

On the agent's side:

1. **Rebuilt tooling.** Scripts or commands the agent wrote just to do or check its work: launching the app, taking a screenshot, seeding data, reading logs, driving a simulator or browser. If the repo doesn't already have them, the next session builds them again. Propose committing the working version, as a script or a project skill, and a pointer from the agent's instructions so the next session finds it.
2. **Navigation.** Places where the agent took many searches to find a file, a fact, or a command: the check command, the base branch, where config lives. Propose a pointer in a file the agent already reads.
3. **Automated checks.** Mistakes a lint rule, type, test, or hook could have caught. Read the repo's existing check command and CI first: a check that exists but isn't wired up, or is broken, is the finding, not a new one. If the rule is mechanical (a banned API, an import shape, a file location), propose a check that fails instead of a sentence that asks.
4. **Missing information.** Something the agent needed but couldn't reach: server logs, a service's state, credentials for a read-only API. Propose how to give it access, such as writing the dev server's log to a file.
5. **Expensive calls.** Tool calls that returned far more than they needed, or ran many times when once would do. Propose the cheaper call or a small tool.
6. **Instructions.** A line in `CLAUDE.md` / `AGENTS.md` that was missing, ignored, misread, or contradicted by another. Also lines that changed nothing in this session and look like they never would. Propose the edit, or the deletion.

On the user's side:

7. **Phrasing.** Requests the agent misread, corrections the user had to repeat, context that would have saved a round trip if it had come first. Say what to say next time, in the user's own words where you can.
8. **Shorthand.** Phrases the user typed more than once that a common chat abbreviation replaces: "what do you think" → `wdyt`, "looks good to me" → `lgtm`, "never mind" → `nvm`, "if I remember correctly" → `iirc`. Agents know these already. For a long instruction the user keeps repeating, propose a term of their own, paired with the line that defines it in `CLAUDE.md` / `AGENTS.md`; never suggest one that relies on the agent guessing.
9. **Shorter messages.** When the user's messages tend to run long, quote one and show how a native speaker would say the same thing in fewer words. Only for a habit across several messages, never for one long message. Don't correct grammar, spelling, or word choice the agent understood; wording that caused a misread belongs under phrasing.

Where a fix can live in the repo or in the user's instructions, prefer the one that works without anyone remembering it: a check over a rule, a committed script over a tip, a line in the instructions over a habit.

## 3. Report

Number the entries, the costliest first:

```
### 1. <Category> — <short title>

<What happened, in one or two sentences, pointing to the moment in the session.>

**Change** — <the exact text to add or remove and in which file, or the script, check, or skill to commit and where.>
```

Give the change ready to apply: quote the line to paste, name the file and section. Say each thing once.

Last, one line proposing which entries to take — "I'd do 1–3 and skip 4." If nothing survives the bar, say so in one line.

Print the report as your final message and apply nothing; the user decides.
