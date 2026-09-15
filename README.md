<div align="center">

# night-run

<!-- mascot: drop mascot.png (square, transparent background) at the repo root and this will render -->
<img src="./mascot.png" alt="The night shift" width="220" />

### You go home. He clocks in.

He works through your tickets all night — designs each one, gets it reviewed, builds it, tests it.
By morning: draft PRs on your desk. He never merges. That's not a limitation bolted on afterward.
That's just where his shift ends.

*An unattended coding agent workflow. Works with Claude Code and Codex today — the pattern isn't tied to
either.*

[![License: MIT](https://img.shields.io/badge/license-MIT-blue.svg)](./LICENSE)
[![Claude Code](https://img.shields.io/badge/Claude%20Code-compatible-5A32FB)](#install)
[![Codex](https://img.shields.io/badge/Codex-compatible-10A37F)](#install)
[![PRs welcome](https://img.shields.io/badge/PRs-welcome-brightgreen.svg)](../../pulls)

</div>

---

## Meet the night shift

Deadpan. Dark circles. Hoodie, badge, gas-station coffee. He's been doing this since before you started
locking your screen before walking away. He doesn't ask what a ticket means when it's ambiguous — there's
no one to ask at 3am, so he sets it down with a note instead of guessing. He doesn't touch anything above
his clearance. And when he's done, he doesn't let himself in — he leaves the draft on your desk and waits.

Every part of that isn't flavor text. It's the actual guardrail:

| He | night-run |
|---|---|
| clocks in when you clock out | starts attended (you confirm the plan), then runs unattended overnight |
| works one ticket at a time, in order | sequential by default, topologically sorted |
| if two jobs would collide, does them one after the other instead | scope-overlap detection auto-downgrades a parallel pair to sequential |
| never touches anything above his clearance | prod is read-only — any write path blocks the ticket outright, no exceptions |
| sets an oversized job back down instead of guessing his way through it | a separate reviewer can call `TOO_LARGE` → routed back to you, not forced through |
| when something doesn't check out, leaves a note instead of a guess | `BLOCKED` with a reason — never filled in with an assumption |
| drops each finished job off the moment it's done, not at the end of the night | a draft PR opens per ticket immediately — a 3am crash loses nothing |
| never signs off on his own work | draft-PR-only ceiling — merge is always someone else's call |

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

**Why anthropomorphize a skill file?** Because every trait maps to a real guardrail, not decoration — see
the table above. If you'd rather read it as pure mechanics with no character framing, [`SKILL.md`](./SKILL.md)
is exactly that.

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
