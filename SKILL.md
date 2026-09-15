---
name: night-run
description: "An overnight batch skill that processes 4-8 small, mutually independent issues in a single local session/worktree, in topologically sorted order (limited parallelism, up to 4 at a time). For each issue it repeats: requirements analysis -> design -> design review (up to 10 rounds, needs 2 consecutive PASSes) -> implementation -> implementation verification -> commit -> push -> draft PR -> mark the issue in-progress in your tracker. The ceiling is draft PR creation -- it never marks a PR ready for review, merges, or deploys. Triggers: \"night run\", \"run this overnight\", \"batch process before I leave\", \"unattended overnight run\"."
---

# night-run -- overnight batch processing for small, independent issues

**What it is**: a lightweight batch variant of your normal development flow, meant for issues small enough that a full design-doc/spec process would be overkill.
**Precondition**: it does *not* require full upfront design artifacts (API schema, ERD, formal implementation plan) the way a "real" feature does. It's for the same kind of small, self-contained change you'd otherwise hand to a single implementation subagent directly.
**Next step**: a human reviews each draft PR the next day, in risk-tier order, and decides ready-for-review -> merge.

---

## What this skill is not

- **It is not a separate orchestrator, cron job, or CI workflow.** It runs *inside the session you invoked it from*, staying alive overnight. Closing the session (Ctrl+C) is the kill switch -- there is no separate "cancel" command to build.
- **It is not a replacement for a full implementation plan.** It handles independent issues across possibly-unrelated parts of the codebase; it does not assume a shared spec baseline or dependency graph rooted in one feature. If the issues turn out to be tightly coupled (really one feature split into tickets), stop and route them through your normal planning process instead.
- **It is not auto-deploy or auto-merge.** The ceiling is **draft PR creation**, full stop, regardless of how low-risk an issue looks. A human always converts draft -> ready and merges. This isn't a rule invented for this skill -- it's just "PR approval and merging is never unattended," applied consistently.

---

## Safety guardrail -- prod is read-only, never write

night-run runs unattended overnight, so nobody is there to catch a mistake in real time. The following applies to **every issue, with no exceptions**, regardless of its risk label:

- **Implementation and implementation-verification default to local/test environments.** If your standard test commands already default to a local/test env, just use them as-is. **Read-only** access to a prod read replica is fine when a step genuinely needs it (e.g. confirming a data condition actually occurs in prod as part of diagnosing the issue) -- but only through whatever read-only path your team already has (replica connection, read-only DB role, etc.), never a direct/writable connection. **Any prod *write* -- direct or via a credential/tunnel that permits writes -- is never allowed, under any circumstances.** If a step seems to require prod write access, **that issue is immediately marked BLOCKED** -- that decision is out of scope for an unattended run.
- **Backfilling or cleaning up data already sitting in prod is out of scope for night-run**, even if the issue also asks for it -- that's a write, however well-intentioned. If an issue conflates "fix the code so this doesn't happen going forward" with "clean up prod rows that already exist," split it: implement the former, and report the latter back to a human to handle by hand.
- This mirrors whatever read-only-replica / no-direct-prod-write discipline your team already has for AI-assisted database work -- night-run just enforces the write side of it more strictly, because nobody is watching.

---

## Preconditions for invoking this skill

- 4 to 8 independent issue keys/IDs from your issue tracker.
- Each issue's description must already be structured with the [input fields](#input-fields-issue-description-contract) below. If it isn't, ask the user to fill them in before entering unattended mode.

Don't invoke this skill if: there are only 1-2 issues, the issues are tightly coupled (really one feature), or most of the batch is *not* labeled low-risk ([see below](#when-not-to-use-this-skill)).

---

## Input fields (issue description contract)

night-run only needs a lightweight subset of whatever fields your team's full planning process defines. Each issue's description needs this block for topological sorting, parallelism decisions, and the risk gate to work automatically:

```
### night-run
- depends_on: [FOO-101, FOO-102] (or "none")
- scope: backend/src/profile/** (repo + rough path this touches)
- dod: pnpm lint && pnpm test -- profile.service.spec.ts (a command whose pass/fail is machine-checkable)
- risk: none | auth | payment | migration | security | infra
```

- If any of the four fields is missing, **that issue is dropped from the unattended queue**, and reported back: "{issue key} is missing night-run fields -- skipping. Fill them in and re-run." Never guess the missing value.
- `risk` is your team's standard risk tiers (typically: auth/authz, payment/billing, DB migration, security/privacy -- adapt to whatever your org already uses) plus one night-run-local addition, `infra` (CI/CD, deploy config, IaC changes). `infra` only exists inside this skill's scope; it doesn't redefine your org's actual risk-tier taxonomy. It exists because night-run's small-issue queue can include CI/deploy changes that don't fit any of the other categories but are still risky in an unattended context.

---

## Step 1: fetch issues + build the dependency graph (main session, not delegated)

```bash
# example using an Atlassian-CLI-style tool; swap for your tracker's CLI/API
yourtracker issue view {issue-key} --json
```

Parse each issue's description for the four fields. Build a DAG with `depends_on` as edges and run **Kahn's topological sort** (same principle as any dependency-DAG-based planning: no cycles allowed, same level = parallel candidates).

- **If a cycle is found**: report which issues are cycled and **stop**.
- **Issues missing fields**: dropped per the rule above; continue with the rest.

**`depends_on` and `scope` are self-reported -- cross-check them, don't just trust them.** The DAG above only encodes dependencies the issue author actually wrote down. Separately, compare every pair of queued issues' declared `scope` globs and flag any pair that overlaps (same file/directory reachable from both), *even if neither declares a `depends_on` on the other*. This catches the common real case: two issues look independent because nobody wrote `depends_on`, but they'd both touch the same shared util/type/migration.

This check is necessarily coarse: it can only compare what each issue's author *declared* in `scope`, not what the issue will actually touch. It can't catch an issue whose real diff reaches outside its declared `scope`. That gap is closed later, at the point where it actually matters -- see the design-time re-check in 3.9, which compares each issue's *actual* touched-file list (once a design exists) instead of the self-reported glob.

Any pair flagged here that was about to land in the same parallel group (3.9) is **automatically downgraded to sequential** -- this isn't left as a warning for a human to act on later, since silently running two issues that touch the same file concurrently, unattended, is exactly the failure mode worth preventing by default. A human can still put them back in the same parallel group at the Step 2 confirmation if they judge the overlap incidental (e.g. both just import the same read-only constant). Pairs with no shared parallel group are still reported, since even sequential issues touching the same file is useful for a human to know going in, but no automatic action is taken on them.

---

## Step 2: confirm the execution plan (human, at session start)

Since this is invoked before someone leaves for the night, a human is still present at this point. Show the processing order, parallel groupings, and risk labels as a table, and get an explicit **"start like this?"** confirmation before entering the unattended loop. ("Unattended" means *after* the start -- the start itself is attended.)

```markdown
| Order | Issue | Depends on | Parallel group | Risk | Scope overlap |
|-------|-------|------------|-----------------|------|----------------|
| 1 | FOO-101 | none | solo | none | -- |
| 2 | FOO-102, FOO-103 | FOO-101 | sequential (auto-downgraded from parallel) | none, infra | ⚠️ FOO-102 & FOO-103 both declare `src/shared/util.ts` in `scope` -- neither lists a `depends_on` on the other |
| 3 | FOO-104 | FOO-102 | solo | payment | -- |
```

FOO-102 and FOO-103 looked independent (no `depends_on` between them) and were headed for the same parallel group -- exactly the case #1 was about. Because their declared `scope` overlaps, they were automatically downgraded to sequential before this table was even shown. A human can still force them back into the same parallel group here if the overlap looks incidental (e.g. both just import the same read-only constant); the point is that it can't happen silently by default.

---

## Step 3: per-issue loop (unattended)

Process issues in topological order. Each issue goes **requirements analysis -> design -> design review (up to 10 rounds, 2 consecutive PASSes) -> implementation -> implementation verification -> commit -> push -> PR**, start to finish, before moving to the next. **Keep exactly one worktree by default** -- create a new branch per issue, but don't have multiple branches checked out simultaneously via `git worktree add` outside the explicit parallel case in 3.9.

> **Why**: a real failure mode in earlier iterations of this approach was opening a worktree per issue and running a *full build* in each, in parallel -- this pegged the CPU (40-minute builds) and produced merge conflicts. Sequential-by-default, single-tree is the fix.

> **Why finish each issue's PR immediately instead of batching**: pushing and opening the PR issue-by-issue (instead of all at once at the end) means that if the session dies partway through the night, everything completed so far already exists as a PR. Nothing is lost.

### 3.1 Branch setup

```bash
git checkout {base-branch}
git pull
git checkout -b {type}/{issue-key}/{short-slug}   # match your team's normal branch-naming convention
```

### 3.2 Requirements analysis

Summarize, in 1-2 paragraphs, exactly what this issue requires, using its title, description, and the `scope`/`dod`/`risk` fields.

- If ambiguity remains that `scope`/`dod` don't resolve -- there's nobody to ask, since this is unattended -- **mark it BLOCKED on the spot** (reason: "requirements ambiguous -- {specifically what's unclear}"). Never fill the gap with a guess.

### 3.3 Design

Have whichever domain expert owns the `scope`'s repo/stack (the same subagent that will implement it) sketch an approach: files/functions touched, data flow, edge cases. This is a short design note, not a formal design artifact (no schema/API-spec-level output, and it isn't saved as a separate document) -- that level of ceremony isn't worth it for an issue this small.

### 3.4 Design review loop (up to 10 rounds, locked in after 2 consecutive PASSes)

> **Separation of judge and author**: the subagent that wrote the design must not be the one that reviews it. Spin up a **separate, freshly-started reviewer subagent** to critique it adversarially. An agent being "skeptical" of its own output doesn't count as independent review.

1. Give the reviewer the design note plus `dod`/`scope`/`risk`, and get one of three verdicts:
   - **PASS** -- the approach satisfies `dod` and doesn't miss edge cases or side effects.
   - **FAIL** -- it doesn't, but the issue is still a reasonable fit for a one-shot lightweight design note; send feedback back to the design step, revise, and re-review.
   - **TOO_LARGE** -- once actually designing it, this issue turns out to be bigger or more entangled than its `scope`/`dod` suggested (touches more of the system than one lightweight design note can responsibly cover, or the "small independent change" precondition no longer holds). This is a distinct call from FAIL: FAIL means *this specific design* is wrong; TOO_LARGE means *no design at this weight class* is the right answer for this issue.
2. On **FAIL**, send the feedback back to the design step, revise, and re-review.
3. On **TOO_LARGE**, stop immediately -- don't wait for 10 rounds. Mark the issue **ROUTE_TO_PLANNING** (not BLOCKED) with the reviewer's reasoning, and report it as "needs your normal full planning process," not "retry differently."
4. **Lock in the design as soon as 2 consecutive PASSes occur** (no need to burn all 10 rounds).
5. **If 10 rounds pass without 2 consecutive PASSes** (and it was never called TOO_LARGE), mark the issue **BLOCKED** (reason: "design review inconclusive -- summary of round-N feedback"). Fail-safe instead of looping forever while unattended. As rounds accumulate, summarize the key open issues from prior rounds for the reviewer each time, so it doesn't repeat the same feedback.

### 3.5 Hand off implementation

Pass the finalized design straight to whichever implementation subagent/expert owns that stack -- same as you'd do for any small change that doesn't need a full plan.

```
Task(subagent_type=<backend|mobile|frontend>-expert,
     prompt="Issue={issue-key}+title+design note, scope={scope}, dod={dod}, risk={risk},
             Note: this is a night-run batch call -- skip the full test/e2e suite,
             only run lint + typecheck + the scoped test from dod, and return the result
             (the full suite runs separately at the Step 4 checkpoint)")
```

### 3.6 Implementation verification

Run whatever command is specified in the `dod` field (scoped lint + typecheck + targeted test) and confirm it passes.

> **Deliberate difference from your normal "done" checkpoint**: a normal feature-done checkpoint usually runs lint + the full test suite + e2e. night-run only runs the **scoped `dod` command** per issue, because running the full suite per issue is exactly what caused the 40-minute-build failure mode mentioned above. The full suite only runs at the Step 4 checkpoint.

- **On failure** (`dod` doesn't pass, or the implementing subagent reports a blocker): **mark only that issue BLOCKED**, record why, and clean up the incomplete change (`git stash` it or leave the branch and return to `base`) so it can't bleed into the next issue's work. Keep the queue going -- don't stop everything.

### 3.7 Commit -> push -> PR

```bash
git add -A && git commit -m "{type}: {issue-key} {summary}"   # match your normal commit/branch conventions
git push -u origin HEAD
```

Then invoke your team's normal PR-creation flow with **`gh pr create --draft`** to open it as a draft, chaining into your automated PR-review flow if you have one. **Never undraft (mark ready for review) or merge, under any circumstances** -- draft PR creation is this issue's final night-run output (undraft/merge/deploy is tomorrow's human's job). If `risk != none`, flag it via label/comment (see Step 5). Return to `base` afterward.

> **Why draft**: night-run output is something no human has looked at yet. A normal PR immediately triggers CI/reviewer notifications or reads as "ready for review." Draft makes it unambiguous: "pending morning review," until a human explicitly promotes it.

### 3.8 Mark the issue in-progress in your tracker

Right after the draft PR exists, reflect that real work happened:

```bash
yourtracker issue transition {issue-key} --status "In Progress"
```

- The exact status name is project-specific -- check your tracker's workflow. Try the obvious default first.
- If the transition fails (status name doesn't match, or it's already past "in progress" and the transition is a no-op) -- **don't guess-loop through alternate status names while unattended.** Treat the issue itself as successfully processed (not BLOCKED), just note "tracker status transition failed -- needs manual update" in the Step 6 report, and move on.

### 3.9 Limited parallelism (up to 4)

Only issues grouped in Step 1 as "no `depends_on` among each other + non-overlapping declared `scope` + all `risk: none`" are parallel candidates (up to 4 at once) -- and even then, only after clearing a second check below. Step 1's overlap check is coarse: it can only compare what each issue's author *declared*, not what the issue actually turns out to touch.

**Step A -- design and review first, sequentially, no worktree yet.** For every remaining candidate in the group, run 3.2 (requirements analysis), 3.3 (design), and 3.4 (design review) one at a time, in the main session. None of this touches code, so none of it needs a worktree. Each surviving design note already lists the real files/functions it touches (per 3.3) -- cross-check that actual list between every pair of candidates still in the group, not just their declared `scope`.

- **If a pair's real touched files overlap** (even though their declared `scope` didn't), drop the later one (by queue order) from the parallel group and fall back to sequential for it. This is precisely the "`scope` was under-declared" case Step 1 can't see, since Step 1 only had the author's claim to go on -- this second check uses what the design step actually found instead. Record the reason for the Step 6 report, e.g. "downgraded to sequential -- design revealed it also touches `src/shared/util.ts`, not listed in `scope`."
- Any candidate that gets a `TOO_LARGE` verdict during 3.4 exits here as `ROUTE_TO_PLANNING`, same as it would sequentially -- it never reaches Step B, so no worktree is wasted on it.
- Everything still standing after Step A proceeds to Step B, together.

**Step B -- implement in parallel, one worktree per surviving candidate.** Create a branch and worktree per issue and run 3.5 (implementation handoff) through 3.8 (tracker transition) concurrently:

```bash
git worktree add -b {type}/{issue-key-a}/{short-slug} ../night-run-{issue-key-a} {base-branch}
git worktree add -b {type}/{issue-key-b}/{short-slug} ../night-run-{issue-key-b} {base-branch}
git worktree add -b {type}/{issue-key-c}/{short-slug} ../night-run-{issue-key-c} {base-branch}
git worktree add -b {type}/{issue-key-d}/{short-slug} ../night-run-{issue-key-d} {base-branch}
```

**Each worktree still only runs the scoped `dod` command in 3.6** (full builds are still forbidden -- the actual root cause of the CPU-exhaustion failure mode was "running full builds in parallel," not "worktrees" per se, so this constraint stays no matter how many you run in parallel). `pnpm install` (or equivalent) may be needed per worktree, but a content-addressed package store makes repeat installs cheap. Clean up immediately after each finishes:

```bash
git worktree remove ../night-run-{issue-key-a}
git worktree remove ../night-run-{issue-key-b}
git worktree remove ../night-run-{issue-key-c}
git worktree remove ../night-run-{issue-key-d}
```

If any issue in the group turns out to have `risk != none` mid-flight, drop it from the parallel group and fall back to sequential.

---

## Step 4: checkpoint builds

Every 3 successfully processed issues, plus once more at the end of the queue, merge the branches of the most recently succeeded issues into a temporary integration branch and run the full verification suite once.

```bash
pnpm lint && pnpm test && pnpm test:e2e   # use whatever your repo's real full-verification command is
```

- **Pass**: continue to the next batch.
- **Fail**: since each issue is one commit, bisect backward from the most recent commit (no need for `git bisect` machinery) to isolate the cause, mark only that issue BLOCKED, and keep the rest. If a PR already exists for it, comment the checkpoint failure on the PR.

---

## Step 5: risk-tier issues

Issues with `risk != none` still go through PR creation and tracker status transition exactly as in 3.7-3.8, but get flagged via label/comment. They are never candidates for unattended auto-merge -- this isn't a night-run-specific rule, it's just "PR approval and merging is always a human," applied without exception.

---

## Step 6: morning report (final output of the session)

```markdown
## night-run summary -- {date}

| Issue | Status | Draft PR | Tracker transition | Design review rounds | Risk |
|-------|--------|----------|----------------------|------------------------|------|
| FOO-101 | done | {draft PR link} | in progress | 2 (consecutive PASS) | none |
| FOO-102 | done | {draft PR link} | in progress | 3 (PASS from round 2) | infra (needs a careful human read) |
| FOO-103 | done | {draft PR link} | transition failed -- manual needed | 1 | none |
| FOO-104 | BLOCKED | -- | -- | 10 rounds, never locked in | payment -- reason: {summary of the review feedback} |
| FOO-105 | ROUTE_TO_PLANNING | -- | -- | stopped at round 2 (TOO_LARGE) | infra -- reason: {reviewer's reasoning for why this exceeds the lightweight path} |

### Checkpoint results
- N/M issues succeeded; checkpoint build passed/failed X times

### Next steps
- A human reviews each draft PR, risk-tier first -> promotes to ready -> merges
- Manually transition any issue with a failed tracker-status update
- Decide whether to retry any BLOCKED issue after investigating why
- Send any ROUTE_TO_PLANNING issue through your normal full planning process instead of retrying it here
```

---

## When not to use this skill

- The issues are tightly coupled and really form one feature (route to your normal formal planning process instead).
- Only 1-2 small issues (calling the relevant domain skill directly is faster).
- Most of the batch is *not* labeled low-risk (this shape of batch isn't a good fit for unattended processing -- run it attended instead).

---

## Design history

| When | What changed |
|------|--------------|
| v1 | Initial version -- process a batch of independent small issues overnight in a single local session, sequential with limited parallelism (up to 2). PR creation only, merge/deploy always human. |
| v2 | Split the per-issue loop into requirements analysis -> design -> design review (up to 3 rounds, 2 consecutive PASSes, judge/author separation) -> implementation -> implementation verification -> commit -> push -> PR. Moved PR creation from "once at the end of the queue" to "immediately per issue" (prevents loss on session interruption). All PRs now created as drafts (undraft is human-only). |
| v3 | Added the prod-data safety guardrail: implementation and verification only ever run in local/test environments; prod backfills/cleanup are explicitly out of scope and reported to a human separately. (Prompted by a real incident where "fix the code for future cases" got conflated with "backfill data that already accumulated in prod" for the same issue.) |
| v4 | Added the post-draft-PR tracker status transition step. Transition failures are reported in the morning summary rather than treated as issue failure. |
| v5 | Raised the design-review cap from 3 to 10 rounds (the "2 consecutive PASSes to lock in" rule is unchanged -- the last two verdicts must still both be PASS). Reviewers are now given a summary of prior rounds' open issues each round. Raised the limited-parallelism cap from 2 to 4 (same `risk:none` / non-overlapping-scope conditions and "scoped command only, no full build per worktree" safeguard apply -- only the parallelism count increased, based on real-world usage). |
| v6 | Addressed [#1](https://github.com/sehynn/night-run/issues/1): `depends_on`/`scope` were purely self-reported with no cross-checking. Step 1 now flags scope-overlapping issue pairs even when no `depends_on` was declared, and automatically downgrades a flagged pair from parallel to sequential by default (a human can still override at Step 2) rather than just displaying a warning. Since that check can only compare *declared* `scope`, 3.9 now also runs a second, design-time cross-check for the parallel path: design + design review happen for every parallel candidate first, without a worktree, and candidates whose real touched-file lists overlap get bumped to sequential before any worktree is opened. Added a third design-review verdict, `TOO_LARGE` (Step 3.4), so an issue that turns out bigger than the lightweight path can handle exits immediately as `ROUTE_TO_PLANNING` instead of burning all 10 rounds and landing in an undifferentiated `BLOCKED`. |
| v7 | Addressed [#3](https://github.com/sehynn/night-run/issues/3): the safety guardrail banned all prod access, even reads, which was stricter than most teams' actual policy (read-only replica fine, writes forbidden) and made routine diagnostic reads get BLOCKED for no safety benefit. Reframed as "prod is read-only, never write" -- read-only replica access is allowed when a step genuinely needs it; any write, direct or via a writable credential/tunnel, stays absolutely forbidden with no exceptions. |

---

## Adapting this to your team

This skill assumes a few things you'll need to map onto your own setup:

- **An issue tracker with a CLI or API** you can script against (the examples use an Atlassian-CLI-style tool; swap in whatever fits -- Linear, GitHub Issues, etc.).
- **Domain implementation subagents/experts** per stack (backend/mobile/frontend, or whatever split makes sense for your codebase) -- night-run delegates design and implementation to these rather than doing it inline.
- **An existing PR-creation step/skill** that follows your team's branch/commit/PR-template conventions, ideally with `--draft` support.
- **Your own risk-tier taxonomy** (auth, payment, migration, security/privacy are common defaults) -- night-run just adds one local `infra` tag on top for its own gating purposes.
- **Whatever prod-safety discipline you already enforce** for AI-assisted work (read-only replicas, no direct prod writes, etc.) -- night-run's guardrail assumes that baseline exists and makes the no-write side of it non-negotiable, with no risk-label exceptions, for the unattended case.
