---
name: dependency-update
description: Update every dependency to the newest version its package.json range allows and Node to its newest LTS release, clean up stale overrides, and report out-of-range updates, pinned specifiers, and resolved advisories. Use for routine maintenance of all dependencies, not for bumping a single package.
---

Update all dependencies to the newest versions their `package.json` specifiers allow, and move Node to its newest LTS release. It is routine maintenance, not incident response: it absorbs bumps, low/medium or dev-only vulnerabilities, Node release lines, and stale `overrides` entries, and leaves fixing production-exploitable vulnerabilities to whoever triages the audit.

The governing rule: **update what the specifiers allow, report everything else.** Most dependencies use caret ranges, so an in-range update usually means the latest minor/patch of the same major — but the boundary is the range, not the major: a tilde admits only patches, an exact pin admits nothing, and anything beyond the range is out of scope regardless of semver level. Never change a specifier's operator, alias, or pin; the exceptions are the `latest` auto-convert in step 2, moving `@types/node` to a new Node major in step 4, and unpinning a specifier whose reason has expired in step 6. Node follows the same rule with its newest LTS line as its range (step 4).

Run the project's package manager and `git` the way the repo requires — a container prefix, a nonstandard lockfile location, a wrapper script. Read `CLAUDE.md`/`AGENTS.md` for it, and repeat the instruction in every subagent prompt; subagents don't inherit it.

**Package manager:** the flow below is written for pnpm, and the traps marked **pnpm:** are pnpm's own. On npm or yarn, find that package manager's equivalent rather than translating the command literally — the trap usually exists there too, under a different name.

## 1. Preflight

**pnpm:** `minimumReleaseAge` delays freshly published versions — during resolution, a version younger than the threshold simply won't be picked. Don't fight it: `pnpm outdated` may show a newer version than what actually installs; that's expected, and the next run picks it up.

## 2. Apply in-range updates

Have one **inventory subagent** build a compact classification — no raw JSON in the main context. The package manager's `outdated` command with a JSON flag is only a starting list, with two known holes (**pnpm** 11): `wanted` echoes the lockfile version, so comparing it to `current` finds nothing — compute the real in-range target as the highest registry version satisfying the declared range (`semver.maxSatisfying`); and npm-aliased packages are omitted entirely — read them from `package.json` and check each alias target against the registry directly.

- **in-range** — a newer version satisfies the declared range; this run updates them.
- **out of range** — `latest` is outside the declared range: the next major behind a caret, the next minor behind a tilde, anything behind an exact pin; report-only (step 6).
- **unusual specifier** — npm alias, prerelease, exact pin, git URL, or `latest` tag; collected for the pinned-dependencies report (step 6). If the specifier still declares a range (e.g. a caret inside an npm alias), its in-range updates apply like any other — only the specifier itself is off-limits.

Then apply:

1. Update direct dependencies **and** re-resolve transitive ones. **pnpm:** `pnpm update --depth 9999`; plain `pnpm update` defaults to `--depth 0` and silently skips transitives, and the transitive refresh is exactly what absorbs the low/medium vulnerabilities an audit triage dismissed.
2. The diff is the source of truth: confirm `package.json` ranges moved with operators preserved and the lockfile re-resolved. **pnpm:** `pnpm update` is known to drop the caret from prerelease specifiers (`^1.0.0-beta.4` → `1.0.0-beta.4`) — revert any operator it changed. Never report a bump the diff doesn't show.
3. If any dependency is specified as `"latest"`, convert it to `^<currently installed version>` — the one unambiguous hygiene fix (lockfile-less installs silently float on `latest`). Every other unusual specifier is report-only.

## 3. Clean up overrides

For each `overrides` entry (typically left by a security fix with a removal comment; **pnpm:** they live in `pnpm-workspace.yaml` on pnpm ≥11): remove it, run a lockfile-only install — **pnpm:** `pnpm install --lockfile-only`, because pnpm does **not** re-resolve after an overrides change on a plain `pnpm install`, even with `--force` — and check the resolved version in the lockfile. If the parents' ranges have caught up (resolved version still satisfies the override's fixed range), keep the removal. If not, restore the override and report it as still needed.

## 4. Update Node

**Other runtimes:** this step is written for Node. On Bun or Deno, find the runtime's equivalent of each part rather than skipping it: its release policy for which line to target (neither has Node's LTS schedule, so expect the newest stable release), every place that pins it (version files, `packageManager`, `oven/bun` / `denoland/deno` images, `setup-bun` / `setup-deno` in CI), and its types package (`@types/bun`).

Node's range is its **newest LTS line**: the newest line in Node's release schedule (`https://raw.githubusercontent.com/nodejs/Release/main/schedule.json`) whose `lts` date has passed. Don't also require its `maintenance` date to be ahead — the schedule can put one line into Maintenance days before the next reaches LTS, and that test finds no line in between. Read it from the schedule; don't infer it from even/odd majors — from Node 27 on, every major becomes LTS. The target is that line's newest release in `https://nodejs.org/dist/index.json`. Two kinds of update follow, and both are applied:

- a newer release of the pinned line — the in-range case;
- a newer LTS line than the pinned one. Moving lines is Node's routine upgrade path; skipping it only means arriving at Maintenance LTS and then end-of-life.

A line whose `lts` date is still ahead goes under **Available**, with that date as its revisit condition. A line with no `lts` date is never mentioned.

Find every place the repo pins Node and move them together — CI typically reads only one of them, so a pin left behind drifts unnoticed:

- version files: `.node-version`, `.nvmrc`, `.tool-versions`, `mise.toml`;
- runtimes the package manager or a version manager installs: `devEngines.runtime`, `volta.node`;
- container images: `FROM node:<version>…` in every Dockerfile, keeping the variant suffix (`-bookworm`, `-alpine`);
- CI workflows that set `node-version:` instead of reading a version file;
- `@types/node`, whose major tracks the runtime's. On a line change, move it to the newest published release of the new major, operator kept — its minors lag the runtime's, so don't expect matching numbers. Within a line, step 2 already updated it.

Leave `engines.node` alone unless it excludes the target: it declares what the project supports, not what it runs on. Confirm each pin's artifact exists — the official Docker image can trail a Node release by a day (`https://hub.docker.com/v2/repositories/library/node/tags/<tag>` returns 200 once it's out); if it hasn't been published, target the line's newest release that has.

Step 5 runs on the new runtime where it can. When the runtime comes from a container image, rebuild the image and reinstall dependencies — native modules are compiled for one Node ABI — the way the repo's instructions say. Where the repo's instructions reserve restarting that environment to the user, don't restart it: verify on the current runtime, let CI — which reads the pins — verify the new one, and note the pending rebuild in the report.

A line change that breaks is handled like any bump in step 5. If the fix balloons, hold the line change — stay on the newest release of the current line — and report it under **Held**. Production usually reads one of the pins; if it takes Node from somewhere outside the repo (a platform setting, an image built elsewhere), a line change needs a `## Deploy checklist` item to move it there too.

## 5. Verify — and fix what breaks

The install completes cleanly, then run the project's check and test scripts (e.g. `pnpm run check`, `pnpm run test`). A breaking bump is not dropped from the batch: **fix the code in this run**. In-range updates should almost never break, and when they do the fix is usually small — a renamed option, a stricter type. One failure class needs no code fix at all: a deep re-resolution routinely leaves two copies of the same transitive package, and a type error naming two declarations of one symbol (`… separate declarations of a private property …`) is a duplicate-resolution problem — dedupe the tree (**pnpm:** `pnpm dedupe`) before touching code, and delete `tsconfig.tsbuildinfo` before re-checking (incremental state survives `node_modules` changes and keeps reporting the fixed error); a cast would only bury it. A code fix _might_ mean the report needs a test plan or a deploy checklist (step 7): add a test plan when the fix changes something verifiable in the browser, a deploy checklist when it needs a manual step, and neither for a trivial internal fix.

If a fix balloons into a real migration, hold that one package: redo the update excluding it (**pnpm:** `pnpm update --depth 9999 '!<pkg>'`) from a clean base. Restoring the manifest range alone holds nothing when the breaking version is in-range; the next install re-resolves forward. The exclusion binds only this run — nothing stops an ordinary install or update elsewhere from re-resolving forward — so a **Held** entry is a request for a decision, not a guardrail. Making the hold real means narrowing the specifier or adding an override (safe: step 3 restores an override that's still needed rather than cleaning it up); leave that to a human: finish the run and report the package under **Held** with the pin-vs-migrate trade-off.

## 6. Research with subagents

Group the updated packages before dispatching — one subagent per package is far too slow. Group by: shared release cadence (same family shipping lockstep versions, e.g. `@tanstack/*`, `@sentry/*`, `prisma` + `@prisma/*`), peer coupling (`react` + `react-dom` + their `@types`), or a coordinated stack (the Tailwind ecosystem). Everything left over goes into grab-bag groups of 5–8 packages. Node is its own group when its line changed (its changelog is the major's breaking-change list, read against our usage); a release within the line joins a grab-bag. Launch all agents in a single message.

Steps 5 and 6 overlap by design — research runs while verification is live, so "act now" findings can land as fixes in the same pass. That concurrency is why research agents must not touch shared mutable state: the test database, the project's `node_modules`, the lockfile, and the manifests all belong to the live verify step. Concretely: no test runs, no installs, no lockfile edits, no `node_modules` changes (A/B symlink swaps included). Read-only commands against the real tree are fine and encouraged — Grep, `git`, or running a candidate tool in check mode (`pnpm dlx oxfmt@<new> --check`) — and a subagent that needs mutating evidence (does a version resolve) works in a temp directory on copies of the manifest, the lockfile, and any workspace config. Say this in every subagent prompt — each mutation looks like reasonable verification from inside one agent, and a concurrent test run truncating the shared test database under the real verify step is how a false **Held** gets shipped.

Each research agent receives its packages with old → new versions and returns, per package or group:

- What changed between the two versions (npm registry → repo → GitHub Releases / CHANGELOG).
- **Benefits relevant to this codebase**, each written as a standalone paragraph an agent could later pick up and implement in a follow-up PR — a new API option we could use, a deprecation we should move off. This run only bumps versions; adoption is deliberately separate.
- Anything that requires action _now_ (behavior change within a minor, removed default) — feed these back into step 5 fixes.
- Patch-only, changelog-boring bumps get one line each, no paragraph.

In the same batch, also launch:

- **Out-of-range agent(s)** — for each update the declared ranges block, and the next Node line before it reaches LTS, grouped the same way: what taking it would bring this codebase (benefits, as above) and what it would risk (breaking changes cross-referenced with our actual usage — delegate the usage pass to an `Explore` subagent when needed).
- **Pinned-specifiers agent** — for each unusual specifier: `git blame` the `package.json` line to recover the reason, then check whether upstream has shipped something that changes the calculus (a stable release for a prerelease pin, the referenced fix released for a git URL). When the reason has verifiably expired and the conversion is easy, unpin in this run — convert to a caret on the shipped version, verify like any other bump, and report it under Updated. When the reason still holds or the blame is inconclusive, report only.
- **Advisories agent** — run the audit (**pnpm:** `pnpm audit --json`) on the updated tree and diff it against the audit from before the update. Report the advisories this run's re-resolution cleared. Most will be entries in the audit ignore list (**pnpm:** `audit.ignore` in `pnpm-workspace.yaml`) whose reasoning says the fix will arrive with a routine update — this run is that arrival, and the report should say so. Leave the entries in place: they are keyed by GHSA, and pruning them belongs to whatever maintains the list, which re-runs the audit and can tell a stale entry from one still doing work. A failed audit lands in the report as a failure, never as "no advisories".

## 7. Report

Report in the format below. Append a test plan when the run introduces something worth a manual click-through — a code fix, or a bump whose behavior change reaches the UI — and a `## Deploy checklist` when a step needs manual action.

Write the report for a skimming human; the research is exhaustive, but the prose must not be. Two audiences read it: a reviewer deciding what to do, and — only inside the `> ` follow-up blockquotes — a future agent executing a hand-off. Keep them separate:

- Open every package/group subsection with one bolded verdict line — what happened or what the reader must do (**Bumped, nothing to note**, **Take it: ~10 min**, **Decide: pin or migrate**) — then at most three plain sentences. One idea per sentence; no stacked parentheticals, no citation chains.
- Only three things earn length: Held entries, Available updates recommended for action, and the follow-up blockquotes (dense is fine there — they're written for an agent, and the blockquote makes them skippable).
- State conclusions, not methodology. "No exported icon was removed" plus one number; the byte-diff procedure that proved it goes in a `<details>` block or gets cut.
- File-and-line citations only where the reader acts on them (a test-plan step, a hand-off); not as evidence trails.

```markdown
## Updated

<One subsection per group: a bolded verdict line, at most three sentences with
the packages (`old` → `new`), then any agent hand-off as a `> ` blockquote —
that is where the self-contained benefit paragraphs from research go, and the
only place detail belongs. Boring patch bumps go in one compact table at the
end. Then, where applicable: overrides removed and why the
parent caught up; specifiers unpinned, with the expired reason; code fixes made
and which bump forced them. Node is a row in the table within its line, and a
subsection of its own when the line changed.>

## Available

<Updates the declared ranges block — the next major behind a caret, the next
minor behind a tilde, anything behind an exact pin — and the next Node line
until it reaches LTS, flipping on its `lts` date. Subsection per
group/dependency: current → available, benefits for this codebase, risks with
the breaking changes that touch our usage. End every leave-for-now entry with
the condition that would flip it. Report-only.>

## Pinned dependencies

<Each unusual specifier still in place: what it is, why it's pinned (from
blame), what's newer, and a recommendation. Omit if none.>

## Advisories resolved

<Advisories the re-resolution cleared, including ignore-list entries that
waited for a routine update. Omit if none.>

## Held

<Bumps excluded because the fix ballooned: package, what broke, pin vs
migrate trade-off. Omit if none — which should be the normal case.>
```
