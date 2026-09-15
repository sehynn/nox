<div align="center">

# night-run

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

## Who he is

No name, no small talk. Hoodie, badge, a gas-station coffee that's gone cold twice already. He's been
doing the night shift long enough to know exactly what's not his to decide. Hand him a stack of tickets
before you leave and he won't ask what an ambiguous one "really" means — there's no one to ask at 3am, so
he sets it down with a note instead of guessing. He doesn't touch anything above his clearance. And when
he's done, he doesn't let himself in. He leaves the draft on your desk and waits.

## His rules

Every one of these is a real guardrail, not flavor text — the parenthetical is exactly what it maps to.

- **He never merges.** Draft PR is where his shift ends — promote-to-ready and merge are always your call,
  no matter how low-risk a ticket looked. *(draft-PR-only ceiling, no risk-tier exception)*
- **He never touches anything above his clearance.** Prod is read-only for him. A write — direct or through
  a writable tunnel — isn't a call he gets to make alone, so that ticket gets set aside instead.
  *(hard prod-write block, every ticket, no exceptions)*
- **He works one job at a time, in order.** No juggling five things to look busy. *(sequential by default,
  topologically sorted on what depends on what)*
- **If two jobs would collide, he does them one after the other instead.** Doesn't matter that nobody told
  him they were related — if they'd touch the same file, they don't run side by side.
  *(scope-overlap detection auto-downgrades a parallel pair to sequential)*
- **If a job turns out bigger than the note said, he sets it back down.** He doesn't grind through it and
  hope nobody notices. *(a separate reviewer can call it `TOO_LARGE` → routed back to you, never forced
  through)*
- **When something doesn't check out, he leaves a note — never a guess.** A ticket he can't make sense of
  gets set aside with why, not a fabricated answer. *(`BLOCKED` with a reason, always — guessing isn't on
  the table)*
- **He drops off each job the moment it's done, not at the end of the night.** If he doesn't make it to
  sunrise, whatever's finished is already sitting on your desk. *(a draft PR opens per ticket immediately —
  a 3am crash loses nothing)*
- **He answers to a fresh set of eyes, not his own.** Whoever wrote the approach isn't the one who signs off
  on it. *(a separate reviewer subagent has to actually PASS it, twice in a row, before he touches code)*

## A night in his shift

**6:03 PM** — you hand him five tickets on your way out. He checks what depends on what and shows you the
order before you're even out the door.

**6:04 PM** — you're gone. He clocks in.

`FOO-101` — reads it, sketches an approach, gets it checked by someone else, builds it, tests it, opens the
draft, marks it in progress. Next.

`FOO-102` ↔ `FOO-103` — these two were supposed to run side by side tonight. He notices they'd both touch
the same file. Nobody told him not to run them together — he just doesn't. One at a time instead.

`FOO-104` — the note doesn't add up. Ten honest attempts to make it work, still no. He sets it down and
writes why, instead of shipping a guess.

`FOO-105` — bigger than the ticket let on. He doesn't force it through. Back on your desk, with a note for
daylight.

**9:02 AM** — you're back.

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

3 drafts, reviewed and tested, waiting. 1 set aside with a reason. 1 flagged as too big for a night's work.
0 merged without you. That's the whole pitch.

## Why he exists

You've got 6 small, unrelated tickets. None of them deserve a design doc. Each one is the kind of thing
you'd knock out in twenty minutes if you weren't already walking out the door. So here's what actually
happens to them: they sit in the backlog for two more sprints, because "just tell an agent to handle all
of these overnight" has a way of turning into one of two bad mornings —

- **the safe version**: one giant, unreviewable diff, half of it not actually asked for, and you have no
  idea which ticket broke what
- **the fast version**: it opened five worktrees, ran five full builds in parallel, pegged your CPU for 40
  minutes, and two of them silently touched the same file

He's what's actually needed instead — one ticket at a time (or a carefully checked few in parallel), each
one designed and adversarially reviewed *before* a line of code is written, and capped at a draft PR.

## Why hand it to him, not just prompt an agent overnight

There's a difference between telling an agent "handle it overnight" and handing the work to someone with
actual rules about what he won't do.

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

**Why anthropomorphize a skill file?** Because every one of his rules maps to a real guardrail, not
decoration — see "His rules" above. If you'd rather read it as pure mechanics with no character framing,
[`SKILL.md`](./SKILL.md) is exactly that.

**Isn't this just a prompt?** Yes — that's the point. The value isn't a novel model capability, it's the
specific set of guardrails an *unattended overnight batch* needs that a one-off "handle this overnight"
prompt doesn't have by default: adversarial design review, a hard prod-write block, overlap detection run
twice, and a ceiling that never exceeds a draft PR.

**Why not let it merge low-risk PRs automatically too?** Because "PR approval and merging is always a
human" isn't a rule this skill invented — it's just applied consistently, with no risk-tier exception.
Low-risk still means *unreviewed by a human* the moment the draft PR is opened.

**What if my tickets don't have `depends_on`/`scope`/`dod`/`risk` filled in?** He drops them from the
unattended queue and tells you which ones, rather than guessing. Fill them in and re-run.

**Does it work with issue trackers other than Jira?** The examples use an Atlassian-CLI-style tool, but the
contract is just "a CLI/API you can script + four fields in the description" — swap in Linear, GitHub
Issues, or anything else.

## Contributing

Issues and PRs welcome — see [open issues](../../issues) for known gaps (scope-verification edge cases,
tracker-agnostic examples, more). If he saved you a morning of merge-conflict archaeology, a star helps the
next person find him before their own 3am build catches fire.

## License

MIT — see [`LICENSE`](./LICENSE).
