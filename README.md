# Skills

Agent skills I use on my own projects. Each one is a single `SKILL.md` in the [Agent Skills](https://agentskills.io) format, so it works with Claude Code, Codex, and other agents that read skills.

## The skills

**[full-review](skills/full-review/SKILL.md)** reviews the change on your branch as a stranger would. Six subagents, each blind to the conversation that wrote the code, look for bugs, security holes, slow paths, convention violations, gaps, and untested behavior. Your agent then checks what they found and tells you which fixes it agrees with, which are tradeoffs, and which findings are wrong.

> Use the full-review skill on this branch.

**[submit](skills/submit/SKILL.md)** takes a finished branch to a pull request: it updates the branch from its base, has a fresh subagent trim the comments the branch adds, pushes, writes the pull request, and watches the checks until they pass. It never merges.

> Use the submit skill.

**[dependency-update](skills/dependency-update/SKILL.md)** updates every dependency to the newest version its `package.json` range allows and moves Node to its newest LTS release. It fixes what the updates break, researches the changelogs for anything worth acting on, and reports the updates your ranges block.

> Use the dependency-update skill and open a pull request.

**[security-update](skills/security-update/SKILL.md)** resolves the package audit and the CVEs in the Node version you pin. It takes every cheap fix, triages the rest for production exploitability, and reports what it fixed, dismissed, and left.

> Use the security-update skill and open a pull request.

The two update skills are written for pnpm and point out its traps. On npm or yarn, your agent should find the equivalent command for each step.

## Install

Copy a skill's directory where your agent finds skills:

- **Claude Code:** `.claude/skills/` in a project, or `~/.claude/skills/` for all your projects.
- **Codex:** `.agents/skills/` in a project, or `~/.codex/skills/`.

Commit the update skills into the project if you run them as scheduled tasks, such as [Claude Code routines](https://claude.com/resources/articles/introducing-routines-in-claude-code) or [Codex automations](https://learn.chatgpt.com/docs/automations). A cloud run sees only the repository.

## Make them yours

These are recipes, not packages. Run a skill once, review the result, and tell your agent what you'd change: a different package manager, a stricter review bar, a report in your team's format. Then ask it to edit your copy.

## License

[MIT](LICENSE). Copy and adapt the skills to your needs.
