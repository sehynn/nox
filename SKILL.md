---
name: nox
description: "An overnight batch skill that processes 4-8 small, mutually independent issues in a single local session/worktree, in topologically sorted order (limited parallelism, up to 4 at a time). For each issue it repeats: requirements analysis -> design -> design review (up to 10 rounds, needs 2 consecutive PASSes) -> implementation -> implementation verification -> commit -> push -> draft PR -> mark the issue in-progress in your tracker. The ceiling is draft PR creation -- it never marks a PR ready for review, merges, or deploys. Triggers: \"nox\", \"run nox\", \"night run\", \"batch process before I leave\", \"unattended overnight run\"."
---

# Nox -- overnight batch processing for small, independent issues

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

Nox runs unattended overnight, so nobody is there to catch a mistake in real time. The following applies to **every issue, with no exceptions**, regardless of its risk label:

- **Implementation and implementation-verification default to local/test environments.** If your standard test commands already default to a local/test env, just use them as-is. **Read-only** access to a prod read replica is fine when a step genuinely needs it (e.g. confirming a data condition actually occurs in prod as part of diagnosing the issue) -- but only through whatever read-only path your team already has (replica connection, read-only DB role, etc.), never a direct/writable connection. **Any prod *write* -- direct or via a credential/tunnel that permits writes -- is never allowed, under any circumstances.** If a step seems to require prod write access, **that issue is immediately marked BLOCKED** -- that decision is out of scope for an unattended run.
- **Backfilling or cleaning up data already sitting in prod is out of scope for Nox**, even if the issue also asks for it -- that's a write, however well-intentioned. If an issue conflates "fix the code so this doesn't happen going forward" with "clean up prod rows that already exist," split it: implement the former, and report the latter back to a human to handle by hand.
- This mirrors whatever read-only-replica / no-direct-prod-write discipline your team already has for AI-assisted database work -- Nox just enforces the write side of it more strictly, because nobody is watching.

---

## Preconditions for invoking this skill

- 4 to 8 independent issue keys/IDs from your issue tracker.
- Each issue's description must already be structured with the [input fields](#input-fields-issue-description-contract) below. If it isn't, ask the user to fill them in before entering unattended mode.

Don't invoke this skill if: there are only 1-2 issues, the issues are tightly coupled (really one feature), or most of the batch is *not* labeled low-risk ([see below](#when-not-to-use-this-skill)).

---

## Input fields (issue description contract)

Nox only needs a lightweight subset of whatever fields your team's full planning process defines. Each issue's description needs this block for topological sorting, parallelism decisions, and the risk gate to work automatically:

```
### nox
- depends_on: [FOO-101, FOO-102] (or "none")
- scope: backend/src/profile/** (repo + rough path this touches)
- dod: pnpm lint && pnpm test -- profile.service.spec.ts (a command whose pass/fail is machine-checkable)
- risk: none | auth | payment | migration | security | infra
```

- If any of the four fields is missing, **that issue is dropped from the unattended queue**, and reported back: "{issue key} is missing nox fields -- skipping. Fill them in and re-run." Never guess the missing value.
- `risk` is your team's standard risk tiers (typically: auth/authz, payment/billing, DB migration, security/privacy -- adapt to whatever your org already uses) plus one nox-local addition, `infra` (CI/CD, deploy config, IaC changes). `infra` only exists inside this skill's scope; it doesn't redefine your org's actual risk-tier taxonomy. It exists because Nox's small-issue queue can include CI/deploy changes that don't fit any of the other categories but are still risky in an unattended context.

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
             Note: this is a Nox batch call -- skip the full test/e2e suite,
             only run lint + typecheck + the scoped test from dod, and return the result
             (the full suite runs separately at the Step 4 checkpoint)")
```

### 3.6 Implementation verification

Run whatever command is specified in the `dod` field (scoped lint + typecheck + targeted test) and confirm it passes.

> **Deliberate difference from your normal "done" checkpoint**: a normal feature-done checkpoint usually runs lint + the full test suite + e2e. Nox only runs the **scoped `dod` command** per issue, because running the full suite per issue is exactly what caused the 40-minute-build failure mode mentioned above. The full suite only runs at the Step 4 checkpoint.

- **On failure** (`dod` doesn't pass, or the implementing subagent reports a blocker): **mark only that issue BLOCKED**, record why, and clean up the incomplete change (`git stash` it or leave the branch and return to `base`) so it can't bleed into the next issue's work. Keep the queue going -- don't stop everything.

### 3.7 Commit -> push -> PR

```bash
git add -A && git commit -m "{type}: {issue-key} {summary}"   # match your normal commit/branch conventions
git push -u origin HEAD
```

Then invoke your team's normal PR-creation flow with **`gh pr create --draft`** to open it as a draft, chaining into your automated PR-review flow if you have one. **Never undraft (mark ready for review) or merge, under any circumstances** -- draft PR creation is this issue's final Nox output (undraft/merge/deploy is tomorrow's human's job). If `risk != none`, flag it via label/comment (see Step 5). Return to `base` afterward.

> **Why draft**: Nox output is something no human has looked at yet. A normal PR immediately triggers CI/reviewer notifications or reads as "ready for review." Draft makes it unambiguous: "pending morning review," until a human explicitly promotes it.

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
git worktree add -b {type}/{issue-key-a}/{short-slug} ../nox-{issue-key-a} {base-branch}
git worktree add -b {type}/{issue-key-b}/{short-slug} ../nox-{issue-key-b} {base-branch}
git worktree add -b {type}/{issue-key-c}/{short-slug} ../nox-{issue-key-c} {base-branch}
git worktree add -b {type}/{issue-key-d}/{short-slug} ../nox-{issue-key-d} {base-branch}
```

**Each worktree still only runs the scoped `dod` command in 3.6** (full builds are still forbidden -- the actual root cause of the CPU-exhaustion failure mode was "running full builds in parallel," not "worktrees" per se, so this constraint stays no matter how many you run in parallel). `pnpm install` (or equivalent) may be needed per worktree, but a content-addressed package store makes repeat installs cheap. Clean up immediately after each finishes:

```bash
git worktree remove ../nox-{issue-key-a}
git worktree remove ../nox-{issue-key-b}
git worktree remove ../nox-{issue-key-c}
git worktree remove ../nox-{issue-key-d}
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

Issues with `risk != none` still go through PR creation and tracker status transition exactly as in 3.7-3.8, but get flagged via label/comment. They are never candidates for unattended auto-merge -- this isn't a Nox-specific rule, it's just "PR approval and merging is always a human," applied without exception.

---

## Step 6: morning report (final output of the session)

```markdown
## Nox's shift report -- {date}

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
| v1 | Addressed [#1](https://github.com/sehynn/nox/issues/1): `depends_on`/`scope` were purely self-reported with no cross-checking. Step 1 now flags scope-overlapping issue pairs even when no `depends_on` was declared, and automatically downgrades a flagged pair from parallel to sequential by default (a human can still override at Step 2) rather than just displaying a warning. Since that check can only compare *declared* `scope`, 3.9 now also runs a second, design-time cross-check for the parallel path: design + design review happen for every parallel candidate first, without a worktree, and candidates whose real touched-file lists overlap get bumped to sequential before any worktree is opened. Added a third design-review verdict, `TOO_LARGE` (Step 3.4), so an issue that turns out bigger than the lightweight path can handle exits immediately as `ROUTE_TO_PLANNING` instead of burning all 10 rounds and landing in an undifferentiated `BLOCKED`. |
| v2 | Addressed [#3](https://github.com/sehynn/nox/issues/3): the safety guardrail banned all prod access, even reads, which was stricter than most teams' actual policy (read-only replica fine, writes forbidden) and made routine diagnostic reads get BLOCKED for no safety benefit. Reframed as "prod is read-only, never write" -- read-only replica access is allowed when a step genuinely needs it; any write, direct or via a writable credential/tunnel, stays absolutely forbidden with no exceptions. |
| v3 | Added the "Structural backstop" section below. The draft-PR-only rule previously existed only as an instruction inside this file -- nothing outside the agent's own compliance prevented a bug or misconfiguration from letting a Nox-opened PR get merged unattended. Documents GitHub's native branch protection as the actual fix for multi-reviewer repos (synchronous, server-side, no new code), and honestly states the case it can't fix (a solo maintainer running Nox under their own account, where GitHub's unconditional self-approval block makes "require approvals" brick merges rather than gate them). Considered and rejected a custom GitHub Action doing this reactively -- it reintroduced the same self-approval paradox and couldn't be race-free the way native branch protection already is. |
| v4 | Rewrote "Structural backstop"'s solo-maintainer case -- v3 concluded it was genuinely unfixable, conflating "solo project" with "the agent runs under my own personal account." Two more attempts were tried and rejected first: a time-based required-status-check gate (rejected -- proves wall-clock time elapsed, not that a human looked at anything, and Nox's own sessions already run for hours by design, so an unattended session going rogue has ample time to just wait out any reasonable cooldown and merge anyway), and narrowing Nox's own token to push-but-not-merge (confirmed impossible by checking GitHub's actual docs -- merging requires `Contents: write`, the identical permission needed to push branches, with no scope that separates them). The fix that actually works: run Nox under a separate machine-user identity (a second account or a GitHub App) instead of the human's own -- then "require approvals" branch protection works for a solo human too, since GitHub's self-approval block compares reviewer account to author account, not "how many humans are involved." No new mechanism, just not sharing an identity with the thing that needs to be overruled. |

---

## Structural backstop

Every rule in this document -- draft-PR-only above all -- is an instruction to the agent. Nothing about the prompt itself stops a bug, a misconfigured session, or a prompt injection from causing a Nox-opened PR to get merged unattended. This section documents the actual fix, including for the case (a solo maintainer) that earlier versions of this section incorrectly treated as unfixable.

**Run Nox under a GitHub identity separate from your own**, not your personal account. Two ways to get there:

- **Simple (the default recommendation):** a second GitHub account, registered as a machine user per GitHub's own documented allowance for bot/automation accounts -- not an undisclosed personal alt. Give it the `Write` repository role (not `Admin`/`Maintain`), and generate a **fine-grained personal access token** scoped to exactly what Nox needs: `Contents: write` (push branches/commits) and `Pull requests: write` (open PRs, comment, manage draft state) -- nothing else. A fine-grained PAT can be scoped to any subset of what the account's role already permits; `Write` is a superset of those two, so this is valid. Cost: the token expires (capped at up to a year) and needs manual rotation -- no silent renewal.
- **More setup, less recurring chore:** a GitHub App installation, with the same two permissions. Tokens auto-mint hourly, no manual rotation -- but building JWT-based App auth is materially more engineering than a PAT. Worth it for a team already investing in this pattern; probably not worth it for a single solo user just for this.

Either path, verify the token is actually what your session uses -- check `gh auth status` and your git credential helper before the first real run. If Nox's tooling silently falls back to your own cached personal credentials instead of the machine token, every PR gets authored by *you* again and this entire backstop is defeated without any error surfacing. This is the one step most likely to silently fail; confirm it explicitly, don't assume it.

Once Nox's PRs are authored by that separate identity, turn on native GitHub branch protection on the target branch(es):

- "Require a pull request before merging"
- "Require approvals" (>= 1)
- "Dismiss stale pull request approvals when new commits are pushed" -- so a fixup commit Nox pushes after you've approved doesn't silently ride on an approval that was for a different diff
- "Do not allow bypassing the above settings" (include administrators -- without this, your own account, usually an admin on your own repo, can ignore the rule entirely)

This is GitHub's own mechanism, enforced server-side as a precondition of the merge call itself -- the UI button, `gh pr merge`, the REST endpoint, and the GraphQL mutation all hit the same check, so there's no reactive window to race.

**Why this actually resolves the solo-maintainer case:** GitHub's block on an account approving its own pull request compares the *reviewing account* to the *PR-author account* -- there's no "how many humans are involved" heuristic. Your personal account approving a PR authored by a distinct machine-user identity is mechanically identical to approving a Dependabot or Renovate PR, which already works today. The earlier version of this section concluded a solo maintainer had no structural backstop at all; that conclusion conflated "solo project" with "the agent runs under my own personal account." Those aren't the same thing. The fix was never a new mechanism -- it was not sharing a GitHub identity with the thing you need to be able to overrule.

(We checked whether narrowing Nox's *own* token could solve this without a second identity at all -- give it enough to push branches and open PRs, but not enough to merge. It can't: GitHub's REST API requires `Contents: write` to merge a PR, the identical permission Nox needs to push its own branches. There's no fine-grained scope that separates "can push commits" from "can merge." Token-narrowing alone doesn't work; identity separation does.)

**Stated limitations, still real:**
- Doesn't defend against an owner who deliberately disables branch protection, merges, and re-enables it afterward, or who grants the machine identity admin rights (defeating the separation). This is a backstop against *accidental* unattended merging, not a defense against a deliberately cooperating owner.
- Depends on the machine identity's own token lacking permission to edit branch protection or repo settings itself -- scope it to `Write`/`Contents`+`Pull requests` only, nothing administrative, for exactly this reason.
- Branch protection on private repositories may be gated behind a paid GitHub plan depending on current pricing; unrestricted on public repos.
- If the repo is org-owned rather than under a personal account, the organization may require admin approval before a fine-grained PAT can access org resources at all, or may disable fine-grained PATs entirely (forcing a coarser classic PAT that can't be scoped this narrowly) -- worth checking your org's policy before assuming this setup is available.

---

## Adapting this to your team

This skill assumes a few things you'll need to map onto your own setup:

- **An issue tracker with a CLI or API** you can script against (the examples use an Atlassian-CLI-style tool; swap in whatever fits -- Linear, GitHub Issues, etc.).
- **Domain implementation subagents/experts** per stack (backend/mobile/frontend, or whatever split makes sense for your codebase) -- Nox delegates design and implementation to these rather than doing it inline.
- **An existing PR-creation step/skill** that follows your team's branch/commit/PR-template conventions, ideally with `--draft` support.
- **Your own risk-tier taxonomy** (auth, payment, migration, security/privacy are common defaults) -- Nox just adds one local `infra` tag on top for its own gating purposes.
- **Whatever prod-safety discipline you already enforce** for AI-assisted work (read-only replicas, no direct prod writes, etc.) -- Nox's guardrail assumes that baseline exists and makes the no-write side of it non-negotiable, with no risk-label exceptions, for the unattended case.
