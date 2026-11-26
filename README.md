# Skills

Agent skills in the [Agent Skills](https://agentskills.io) format, for Claude Code, Codex, and other agents that read skills.

## The skills

**[full-review](skills/full-review/SKILL.md)**: six subagents that haven't seen the conversation review the branch for bugs, security holes, slow paths, convention violations, gaps, and untested behavior. Your agent then says which findings it agrees with, which are tradeoffs, and which are wrong.

**[submit](skills/submit/SKILL.md)**: updates the branch from its base, trims the comments it adds, pushes, opens or updates the pull request, and watches the checks until they pass. It never merges.

**[retro](skills/retro/SKILL.md)**: reads the past week of agent sessions in the repo and proposes what would make the next ones faster: tooling the agent should have found, a missing check or pointer, a line for the agent's instructions, and shorter ways to ask. Each suggestion cites its sessions. Run it weekly or after a hard session.

**[dependency-update](skills/dependency-update/SKILL.md)**: updates every dependency to the newest version its `package.json` range allows, moves Node to the newest LTS, fixes what breaks, and reports what the ranges block.

**[security-update](skills/security-update/SKILL.md)**: resolves the package audit and the CVEs in the pinned Node version, takes every cheap fix, triages the rest for production exploitability, and reports what it fixed, dismissed, and left.

The update skills are written for pnpm. On npm or yarn, the agent finds the equivalent commands.

Invoke a skill by name, for example: "Use the submit skill."

## Install

Copy a skill's directory to:

- **Claude Code:** `.claude/skills/` in a project, or `~/.claude/skills/`.
- **Codex:** `.agents/skills/` in a project, or `~/.codex/skills/`.

To run the update skills as scheduled cloud tasks ([Claude Code routines](https://claude.com/resources/articles/introducing-routines-in-claude-code), [Codex automations](https://learn.chatgpt.com/docs/automations)), commit them into the project; a cloud run sees only the repository.

These are recipes, not packages: after a run, tell your agent what to change and have it edit your copy.

## License

[MIT](LICENSE).
