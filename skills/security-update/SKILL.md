---
name: security-update
description: Resolve the audit report and Node's own CVEs — take every cheap fix, triage the rest for production exploitability, and report what was fixed, dismissed, and left. Use when asked to resolve audit findings or vulnerabilities across the project.
---

Triage every vulnerability the package manager's audit reports, and every CVE in the Node runtime the project pins — no package audit sees those — and resolve each one: fix it, dismiss it with a reasoned comment, or surface it in the report.

Scope: every advisory with a **cheap fix** is fixed, whatever its severity or reach — an in-range refresh of the package, or for Node the pinned line's newest release. Working out whether production can reach the vulnerable code costs more than such a fix, and the answer goes stale as the code changes. Only the advisories left over — those whose fix needs an out-of-range bump, an override, or a disruptive cascade — are triaged for **production exploitability**: the exploitable ones are fixed where the cost is justified, and the rest are dismissed with a reasoned comment. Two absolutes: malware/compromised-package advisories are never dismissed, and high/critical advisories are never dismissed on uncertain reasoning.

Run the project's package manager and `git` the way the repo requires — a container prefix, a nonstandard lockfile location, a wrapper script. Read `CLAUDE.md`/`AGENTS.md` for it, and repeat the instruction in every subagent prompt; subagents don't inherit it.

**Package manager:** the flow below is written for pnpm, and the traps marked **pnpm:** are pnpm's own. On npm or yarn, find that package manager's equivalent rather than translating the command literally — the trap usually exists there too, under a different name.

**Dismissals are a file edit, not an API call.** That is the shape of this skill: the ignore list, the version bumps, and the overrides all live in the repo, so a run's entire output is a diff. There is no side channel and no state to carry between runs — everything the next run needs to know is either in the tree or recomputable from it. Node has no ignore list and needs none: a Node CVE is always fixed or reported, never dismissed.

## 1. Preflight

Read the audit **twice**, because one report cannot answer both questions the run asks.

- **The working set** — the audit as the repo is configured (**pnpm:** `pnpm audit --json`). These are the advisories still awaiting a decision; everything already dismissed is filtered out. Triage operates on this.
- **The full set** — the same audit with the ignore list emptied. Copy the manifest, the lockfile, and the workspace config into a temp directory, blank the ignore list in the copy, and audit there. This is the only way to see what is still in the tree behind the dismissals. Step 3 looks for cheap fixes across it, since a parent's range may have reached the fix since an advisory was dismissed, and step 4 prunes against it.

Know which numbers come from where, or the two reports quietly contradict each other: in the JSON, `advisories` is **filtered** by the ignore list, while `metadata.vulnerabilities` is **not** — so a fully dismissed tree reports zero advisories and a nonzero metadata count. That is not a bug and not a clean run. Only the human-readable table annotates the difference, as `(N ignored)`.

Each advisory carries what triage needs:

- `github_advisory_id`, `severity`, `title`, `url`
- `vulnerable_versions` / `patched_versions` — semver ranges
- `findings[].paths` — the full parent chain from the root, one entry per route into the tree
- `findings[].dev` — **whether that route is a dev dependency.** Trust this. It is resolved from the manifests, not guessed from the lockfile, and it is the single most expensive thing to work out by hand.

An advisory whose findings are *all* `dev: true` is dev-only. One with any `dev: false` finding has a production route, and `paths` names it.

Then read the **Node CVEs**. Find every place the repo pins Node — version files (`.node-version`, `.nvmrc`, `.tool-versions`, `mise.toml`), `devEngines.runtime`, `volta.node`, `FROM node:` in every Dockerfile, CI workflows that set `node-version:` — and compare each pin with Node's release index, `https://nodejs.org/dist/index.json`. A pin is missing a security release when the index lists a release on the pin's major line that is newer than the pin and marked `security: true`. Read that release's CVEs from its security-release post, linked from `https://nodejs.org/en/feed/vulnerability.xml`: each CVE gets a heading with its id and severity, a description naming the affected module, and a sentence listing the release lines it affects. Every CVE in the post that affects the pinned line applies, and a pin that skipped several security releases collects the CVEs of each. These join the working set; Node has no ignore list, so there is no full set to compare against.

A pin on a line past its `end` date in `https://raw.githubusercontent.com/nodejs/Release/main/schedule.json` gets no security releases, so the index check never finds one missing — yet Node's posts state that end-of-life lines are affected by every security release. Record such a pin as one high-severity Node finding, `hard-fix` (step 3).

Don't take Node CVEs from `nodejs/security-wg`'s `vuln/core/index.json`: its updates trail Node's security releases by months.

**Other runtimes:** on Bun or Deno, read the runtime's own security advisories (typically on its GitHub repository) and triage them as Node CVEs. A runtime whose advisories can't be read lands in the report as a failure, never as "no CVEs".

If the audit command fails — registry error, network, a malformed response — **stop and report the failure.** Never let an error read as "no vulnerabilities". Don't reach for `--ignore-registry-errors`; it turns exactly this case into a silent green.

The Node check fails the same way: a fetch failure, or an index, schedule, or post that doesn't parse, stops the run. Have the check print what it found — for each pin, the missing security releases and their CVE ids, or `none` — and stop on output that isn't one of those. A comparison script that errors out prints neither, and a run that reads its silence as "no CVEs" misses the release entirely. A missing security release whose post can't be found still applies: report it as triage incomplete.

An empty working set is a legitimate clean result — every advisory in the tree has already been decided — but it is not the end of the run: step 3 still looks for cheap fixes among the dismissed advisories, and step 4 still prunes against the full set. The run has nothing to change only when the working set is empty, no Node pin is missing a security release, *and* neither step finds anything to change — no fix, no ignore entry or override to remove.

## 2. Establish the release-age threshold

**pnpm:** `minimumReleaseAge` delays installs of freshly published versions. Read the effective value from `pnpm-workspace.yaml` / `.npmrc`; if unset, look up pnpm's current default rather than assuming one. A fixed version younger than this threshold cannot be installed yet — treat it as "too new" and defer to the next run. (`minimumReleaseAgeExclude` exists as an escape hatch; suggest it in the report only for a critical fix that shouldn't wait, don't apply it yourself.)

If the project's package manager has no such delay, skip this step and treat every published fix as installable.

Node has no such delay. A Node fix is `too-new` only while an artifact a pin needs is missing — the official Docker image can trail a Node release by a day (`https://hub.docker.com/v2/repositories/library/node/tags/<tag>` returns 200 once it's out).

## 3. Take the cheap fixes, then triage the rest

**Cheap fixes.** Take every package in the full set — dismissed advisories included — and re-resolve them all at once in a temp directory holding copies of the manifest, the lockfile, and any workspace config (**pnpm:** `pnpm update --depth 9999 --lockfile-only <pkg>...`). Then read each package's resolved versions in the copy's lockfile. Where every resolved version satisfies the advisory's `patched_versions` and was published before the release-age threshold, the verdict is `fixable` by lockfile refresh, with the observed versions as evidence. A dismissed advisory fixed this way also loses its ignore entry (step 4). Malware advisories are reported (step 4) whether or not the refresh moves past them.

Node needs no re-resolution: a pin missing a security release is `fixable`, targeting the line's newest release once its artifacts exist (step 2), whatever the CVEs' severity or reach. A pin on an end-of-life line is `hard-fix`: fixing it means changing lines, a runtime upgrade beyond this skill.

**Dismissals whose premise broke.** For each dismissed advisory still in the full set without a cheap fix, compare its ignore comment with its full-set findings. A comment that rests on the advisory being dev-only no longer holds once a finding is `dev: false`: remove the entry and triage the advisory with the rest.

**Triage the rest with subagents.** What remains of the working set, plus any dismissal sent back, has no cheap fix. Group it by package (one package often has several advisories), then spawn one `general-purpose` subagent per package — in a single message so they run concurrently, at most ~5 at a time. Keep raw advisory JSON out of the main context: pass each subagent the fields it needs and have it return only a compact verdict.

Subagents share one working tree and **must never mutate it** — no lockfile edits, no `git checkout --` reverts; concurrent subagents clobber each other. Any hands-on verification (testing that a version actually resolves) happens in a temp directory holding copies of the manifest, the lockfile, and any workspace config, re-resolved there with a lockfile-only install (**pnpm:** `pnpm install --lockfile-only`).

Each subagent prompt must be self-contained — package, GHSA ids, severity, vulnerable and patched ranges, the `paths` and `dev` flags from the findings, the release-age threshold, and the fact that an in-range refresh does not fix it — and instruct it to determine:

1. **Advisory type** — if the advisory is malware or a compromised/hijacked package, stop: verdict `malware`. Dev machines and CI run that code with credentials, so "dev-only" is no comfort here.
2. **Production exploitability** — could the vulnerable code run in production? Start from the findings: all-`dev` means dev-only, and you are done. Otherwise take the `dev: false` paths and check whether the vulnerable function or feature is ever invoked on a production path (Grep the codebase; for a transitive dep, consider how its direct parent uses it). If not exploitable in production, stop here: verdict `not-exploitable`, with the justification and an explicit confidence call — clear-cut (dev-only chain, wrong platform, feature never used) vs uncertain ("probably not called" is not "not reachable").
3. **Fix availability** (production-exploitable only) — does a version in `patched_versions` exist, and was it published before the release-age threshold?
4. **Fix cost** — can it land as a bump of a direct dependency within its major? An `overrides` pin is not cheap — it's debt someone must clean up once the parent's range catches up — so it qualifies as a fix mechanism only for **high/critical** advisories (which, being production-exploitable, are the only kind that reach this step). An override-only fix for medium/low, or anything requiring a disruptive cascade (major bumps of direct deps, breaking peer changes), is `hard-fix`.

Verdicts (one per advisory):

- `malware` — malware or compromised-package advisory; always reported, never dismissed.
- `not-exploitable` — vulnerable code cannot run in production; include the concrete justification and whether the evidence is clear-cut or uncertain.
- `pr-open` — an open PR already bumps the package to a fixed version (search open PRs). It resolves when that merges; nothing to do.
- `fixable` — fix available, old enough, cheap enough to land now; include the target version, the mechanism (direct bump / lockfile refresh / override where it qualifies), and the resolved version line actually observed in a temp-copy re-resolution. A `fixable` verdict without that observed evidence is unverified — subagents have confidently claimed fixes that were no-ops.
- `too-new` — fix exists but is younger than the threshold.
- `no-fix` — `patched_versions` is empty or unsatisfiable.
- `hard-fix` — fix available but only via a disruptive cascade; describe the cascade briefly.

If a subagent fails or returns a thin, unjustified verdict, retry once; after that, report the advisory as "triage incomplete" and take no action on it.

## 4. Act on the verdicts

Dismissals are entries in the audit ignore list (**pnpm:** `audit.ignore` in `pnpm-workspace.yaml`), keyed by GHSA id. Every entry carries a comment above it explaining the reasoning — that comment is the audit trail, and the only record of why. Write it for someone reading the file in six months with no memory of the run.

```yaml
audit:
  ignore:
    # GHSA-67mh-4wv8-2f99 — esbuild dev-server request forgery. Reached only
    # through drizzle-kit, a devDependency; the dev server never runs in
    # production. A later run takes the fix once drizzle-kit's range admits
    # it.
    - GHSA-67mh-4wv8-2f99
```

- `malware` → never dismiss, regardless of severity or reachability. Report prominently as needing attention.
- `not-exploitable` → add to the ignore list with the justification. For **high/critical** severity, only when the evidence is clear-cut; if the justification carries any uncertainty, report instead.
- `hard-fix` + severity **medium/low** → add to the ignore list, comment describing the cascade the fix would require and why the risk is acceptable.
- `hard-fix` + severity **high/critical** → never dismiss. Report prominently as needing attention.
- `fixable` → apply it (step 5). If the advisory was dismissed, remove its ignore entry too.
- Node `hard-fix` → never dismiss; there is no list to hold it. Report high/critical as needing attention, medium/low as deferred.
- `pr-open`, `too-new`, `no-fix` → no action; report only. Never ignore-list these — the list is for decisions, and an entry here would suppress the advisory permanently instead of letting the next run pick up the fix.

Prune as well as add, and prune against the **full set** from step 1 — never against the working set. An ignore entry is stale when its GHSA is absent from the full set: the advisory is gone from the tree, usually because a dependency update moved a parent forward. Remove those too.

Prune the overrides this skill added the same way — nothing else is guaranteed to, and an override left behind pins a package long after its parent has moved past the fix. An override is this skill's when its comment names an advisory; leave any other override alone. For each one: remove it, re-resolve with a lockfile-only install (**pnpm:** `pnpm install --lockfile-only`; a plain install does not re-resolve after an overrides change), and check the resolved version in the lockfile. If it still satisfies the advisory's patched range, the parent has caught up: keep the removal. Otherwise restore the override.

Checking the working set instead inverts the test. Every dismissed GHSA is filtered out of it by definition, so that check marks the entire list stale on the first run, empties it, and re-triages everything from scratch — the same advisories, re-dismissed, every run. (**pnpm:** `ignorePrune` is no protection here; it only acts during `pnpm audit --fix`, which this skill never runs, and `--fix` would rewrite `overrides` as a side effect.)

## 5. Apply the fixes

1. Apply each fix with the least invasive mechanism, minding two facts that are reliably gotten wrong:
   - **In-range fix** (the cheap fixes from step 3) → refresh the whole tree, not just the top level. **pnpm:** `pnpm update --depth 9999 <pkg>`; plain `pnpm update` defaults to `--depth 0` and is a _silent no-op_ for transitive packages.
   - **Direct dependency beyond its range** → bump it in `package.json`.
   - **Node** → move every pin from step 1 to the target release, keeping each image's variant suffix (`-bookworm`, `-alpine`). `@types/node` stays: the fix never leaves the line. Then rebuild whatever runs on the old runtime — a container image, with a dependency reinstall, since native modules are compiled for one Node ABI — following the repo's instructions. Where the repo's instructions reserve restarting that environment to the user, don't restart it: verify on the current runtime, let CI — which reads the pins — verify the new one, and note the pending rebuild in the report. If production takes Node from outside the repo (a platform setting, an image built elsewhere), the fix lands there only by hand: say so in a `## Deploy checklist`.
   - **Otherwise — only for high/critical affecting production** (per step 3) → an `overrides` entry, commented with the advisory it fixes and the condition for removing it — the parent's own range reaching the fix. Step 4 of a later run prunes it by that comment. **pnpm:** overrides live in `pnpm-workspace.yaml` on pnpm ≥11, and `pnpm install` — even with `--force` — does **not** re-resolve after an overrides change (it prints "Already up to date"); run `pnpm install --lockfile-only` to force re-resolution, then `pnpm install`.
2. Confirm each fix actually landed: the resolved version in the lockfile must satisfy `patched_versions`, and no Node pin may still be missing a security release on its line. Never count an advisory as resolved on the strength of having run a command.
3. Verify: the install completes cleanly, then run the project's check and test scripts (e.g. `pnpm run check`, `pnpm run test:unit`). A bump that breaks checks or tests gets dropped from the batch. A dropped cheap fix goes to triage (step 3), its subagent told that the refresh breaks the build: skipping a fix now needs a reason. Any other dropped fix, Node's included, is re-classified as `hard-fix` (re-enter step 4 with the new verdict).
4. Re-run both audits and reconcile them against each other, remembering which is filtered:
   - `advisories` in the working set should hold exactly the advisories you reported rather than acted on. Anything else there is an advisory you believed you dismissed — almost always a typo in a GHSA id, since a mistyped id silently suppresses nothing.
   - `metadata.vulnerabilities` should total the full set's advisory count, and should have dropped by the number you fixed. If it didn't move, a fix never landed, whatever the command printed.

## 6. Report

Report in the format below and add nothing else: no Test Plan, no Release Notes — a security update has no user-facing changes, and a section existing only to say so is noise. The one addition is the `## Deploy checklist` a Node fix needs when production takes Node from outside the repo.

```markdown
## Fixed

<N> advisories (a Node CVE lists its CVE id and `node` as the package):

| Advisory                 | Package | Severity   | Mechanism                                              |
| ------------------------ | ------- | ---------- | ------------------------------------------------------ |
| [GHSA-…](<advisory url>) | `<pkg>` | <severity> | <lockfile refresh / direct bump / override / Node pin> |

<Per package: where to look in the diff; call out incidental drift the
re-resolution pulled along. A cheap fix needs no production path; any other fix
names the path that justified it. For any override: why it qualified
(high/critical, production-exploitable) and its removal condition. Then the
overrides this run pruned, each with the parent that caught up.>

## Dismissed

<N> advisories added to `audit.ignore`, one line each: GHSA, package, severity,
and the reasoning in a clause. The full justification is the comment in
`pnpm-workspace.yaml` — don't duplicate it here, point at it. Then the entries
this run removed, one line each with the reason: fixed, gone from the tree, or
sent back to triage.>

## Needs attention

<Malware advisories; high/critical that are `no-fix`, `too-new`, `hard-fix`, or
triage-incomplete; and high/critical `not-exploitable` verdicts too uncertain to
dismiss. Package, severity, why it's blocked, and any escape hatch worth taking
(a release-age exclusion, a manual cascade). Omit if empty.>

## Deferred

<`too-new` with the date it becomes installable, `pr-open` with the PR,
medium/low `no-fix`, and medium/low Node `hard-fix`. A later run picks these
up. Omit if empty.>

## Verification

- Resolved in the lockfile: <pkg> <old> → <new>, …
- Node pins: <old> → <new>; no security release on the line is missing.
- Audit re-run: <N> remaining, <M> ignored.
- Install clean; check and test scripts pass.
```
