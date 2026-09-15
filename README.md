<div align="center">

# night-run

**Hand off a stack of small, boring tickets before you leave. Come back to draft PRs — not a merge-conflict crime scene.**

[![License: MIT](https://img.shields.io/badge/license-MIT-blue.svg)](./LICENSE)
[![Claude Code](https://img.shields.io/badge/Claude%20Code-compatible-5A32FB)](#install)
[![Codex](https://img.shields.io/badge/Codex-compatible-10A37F)](#install)
[![PRs welcome](https://img.shields.io/badge/PRs-welcome-brightgreen.svg)](../../pulls)

</div>

---

## The problem

You've got 6 small, unrelated tickets. None of them deserve a design doc. Each one is the kind of thing
you'd knock out in twenty minutes if you weren't already walking out the door. So here's what actually
happens to them: they sit in the backlog for two more sprints, because "just tell an agent to handle all
of these overnight" has a way of turning into one of two bad mornings —

- **the safe version**: one giant, unreviewable diff, half of it not actually asked for, and you have no
  idea which ticket broke what
- **the fast version**: it opened five worktrees, ran five full builds in parallel, pegged your CPU for 40
  minutes, and two of them silently touched the same file

night-run is what's actually needed instead: an unattended loop with the *specific* guardrails a
"handle it overnight" batch job needs to not blow up while nobody's watching — one ticket at a time (or a
carefully checked few in parallel), each one designed and adversarially reviewed *before* a line of code is
written, and capped at a **draft PR**. Never ready-for-review, never merged, never deployed. You wake up to
a stack of reviewable diffs, not a decision you have to reverse-engineer.

## What a morning actually looks like

```markdown
## night-run summary — 2026-09-16

| Issue | Status | Draft PR | Tracker | Design rounds | Risk |
|-------|--------|----------|---------|----------------|------|
| FOO-101 | ✅ done | #421 | in progress | 2 (PASS×2) | none |
| FOO-102 | ✅ done | #422 | in progress | 3 (PASS from rd. 2) | infra ⚠️ read carefully |
| FOO-103 | ⚠️ downgraded to sequential, then ✅ done | #423 | in progress | 1 | none |
| FOO-104 | ❌ BLOCKED | — | — | 10 rounds, never locked in | payment — {why} |
| FOO-105 | 🔀 ROUTE_TO_PLANNING | — | — | stopped rd. 2 (TOO_LARGE) | infra — {why} |

3/5 issues succeeded · checkpoint build passed 2/2 times
```

Five tickets in, three clean draft PRs, one honestly flagged as too big for this path, one blocked with a
real reason instead of a bad guess. That's the whole pitch.

## Why this shape (not just "prompt an agent to do it overnight")

| | Ad hoc overnight prompt | night-run |
|---|---|---|
| Two "independent" tickets touch the same file | found whenever someone notices | cross-checked before they're ever run in parallel — a real overlap **auto-downgrades them to sequential**, no human has to catch it |
| A ticket turns out way bigger than it looked | agent grinds through it anyway, diff balloons | a separate reviewer subagent can call it **`TOO_LARGE`** and route it back to you, instead of pretending it fit |
| Prod access | as careful as however you happened to phrase the prompt that night | hard guardrail: **read-only only**, any write path blocks the ticket outright, no exceptions |
| Parallel work | usually N full builds, N CPUs on fire | strictly sequential by default; even the limited-parallel path (≤4) runs only the ticket's own scoped test, never a full build |
| Where you land by morning | a branch, maybe, if you're lucky | **draft PRs only** — one per ticket, the moment it's done, so a 3am crash loses nothing |
| Who decides "ready to merge" | whoever's prompt it was | always a human, always the next morning, no exceptions |

## Design principles

- **Draft-PR-only ceiling.** No matter how trivial a ticket looks, the most this skill ever does is open a
  draft PR and mark the issue in-progress. Promote-to-ready, merge, and deploy are always a human's call.
- **A separate judge for every design.** The subagent that writes a design isn't the one that reviews it —
  a fresh reviewer subagent has to give 2 consecutive PASSes (up to 10 rounds), or call the ticket
  `TOO_LARGE` and kick it back to you, before implementation ever starts. An agent grading its own homework
  doesn't count as review.
- **Prod is read-only. Never write. No exceptions.** A read-only replica read is fine if a ticket genuinely
  needs it to diagnose something. Any write — direct or via a writable credential/tunnel — blocks the
  ticket on the spot. That's not a judgment call an unattended run gets to make.
- **Overlap gets caught twice, not assumed away.** Once from declared scope (before the loop even starts),
  once more from each design's *actual* touched-file list (before any parallel worktree opens) — because
  what a ticket's author wrote down and what the ticket actually touches aren't always the same thing.
  See [Issue #1](../../issues/1) for the failure mode this closes.
- **Sequential by default, one worktree at a time.** Parallel worktrees running full builds — not
  worktrees themselves — is what pegs the CPU and creates merge conflicts. So the default is strictly
  sequential, and even the limited-parallel path never runs more than the ticket's own scoped test per
  worktree.
- **A PR the moment a ticket is done, not at the end of the queue.** If the session dies at 3am, everything
  finished so far already exists as a PR. Nothing is lost to a crash.

## Install

**Claude Code**: copy [`SKILL.md`](./SKILL.md) into `<your-project>/.claude/skills/night-run/SKILL.md`.

**Codex**: copy [`SKILL.md`](./SKILL.md) into `<your-project>/.codex/skills/night-run/SKILL.md`, then add
one line to its frontmatter:

```yaml
codex_type: skill
```

That's the only difference Codex needs.

## Before you use it

night-run assumes a few things about your setup — see **"Adapting this to your team"** at the bottom of
[`SKILL.md`](./SKILL.md) for the full list, but in short:

- an issue tracker with a scriptable CLI/API (examples use an Atlassian-CLI-style tool; swap in Linear,
  GitHub Issues, whatever you use),
- domain implementation subagents/experts per stack (backend/mobile/frontend, or your own split),
- an existing PR-creation step that follows your team's branch/commit/PR conventions and supports
  `--draft`,
- your own risk-tier taxonomy (auth, payment, migration, security/privacy are common defaults — night-run
  adds one local `infra` tag on top),
- whatever prod-safety discipline you already enforce for AI-assisted work (read-only replicas, no direct
  prod writes) — night-run's guardrail assumes that baseline and makes the no-write side of it
  non-negotiable for the unattended case.

Each ticket also needs a small structured block in its description (`depends_on` / `scope` / `dod` /
`risk`) so the topological sort, parallelism decision, and risk gate can run without a human in the loop.
See [`SKILL.md`](./SKILL.md) for the exact format.

## FAQ

**Isn't this just a prompt?** Yes — that's the point. The value isn't a novel model capability, it's the
specific set of guardrails an *unattended overnight batch* needs that a one-off "handle this overnight"
prompt doesn't have by default: adversarial design review, a hard prod-write block, overlap detection run
twice, and a ceiling that never exceeds a draft PR.

**Why not let it merge low-risk PRs automatically too?** Because "PR approval and merging is always a
human" isn't a rule this skill invented — it's just applied consistently, with no risk-tier exception.
Low-risk still means *unreviewed by a human* the moment the draft PR is opened.

**What if my tickets don't have `depends_on`/`scope`/`dod`/`risk` filled in?** night-run drops them from the
unattended queue and tells you which ones, rather than guessing. Fill them in and re-run.

**Does it work with issue trackers other than Jira?** The examples use an Atlassian-CLI-style tool, but the
contract is just "a CLI/API you can script + four fields in the description" — swap in Linear, GitHub
Issues, or anything else.

## Contributing

Issues and PRs welcome — see [open issues](../../issues) for known gaps (scope-verification edge cases,
tracker-agnostic examples, more). If night-run saves you a morning of merge-conflict archaeology, a star
helps the next person find it before their own 3am build catches fire.

## License

MIT — see [`LICENSE`](./LICENSE).
