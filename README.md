# night-run

An agent skill for **Claude Code** (and **Codex**) that batch-processes a handful of small, independent
issues overnight in a single local session: requirements analysis, design, an adversarial design-review
loop, implementation, verification, commit, push, and a **draft PR** for each one — then stops. It never
marks anything ready for review, merges, or deploys. A human reviews the draft PRs the next morning.

Built for the case where you have 4-8 small tickets that don't depend on each other, are boring enough
that a full spec/design-doc process would be overkill, but you'd still rather not babysit each one by hand.

## Why this shape

- **Draft-PR-only ceiling.** No matter how trivial an issue looks, the most this skill ever does is open
  a draft PR and mark the issue in-progress. Promote-to-ready, merge, and deploy always require a human.
- **Design review before code, with a separate judge.** The subagent that writes the design isn't the one
  that reviews it — a fresh reviewer subagent has to give 2 consecutive PASSes (up to 10 rounds) before
  implementation starts. An agent grading its own homework doesn't count as review.
- **Prod is off-limits, full stop.** Implementation and verification only ever touch local/test
  environments. If a step would require a prod credential or tunnel, that issue is blocked on the spot —
  that's not a call an unattended run gets to make.
- **Sequential by default, one worktree at a time.** Parallel worktrees running full builds is what
  actually causes CPU exhaustion and merge conflicts, not worktrees themselves — so the default is strictly
  sequential, and even the limited-parallel path (up to 4, `risk: none` + non-overlapping scope only) still
  runs only the scoped test command per worktree, never a full build.
- **PR per issue, not per queue.** Each issue is committed, pushed, and opened as a PR the moment it's
  done — so if the session dies at 3am, everything finished so far already exists as a PR. Nothing is lost.

## Install

**Claude Code**: copy [`SKILL.md`](./SKILL.md) into `<your-project>/.claude/skills/night-run/SKILL.md`.

**Codex**: copy [`SKILL.md`](./SKILL.md) into `<your-project>/.codex/skills/night-run/SKILL.md`, then add
one line to its frontmatter:

```yaml
codex_type: skill
```

That's the only difference Codex needs.

## Before you use it

night-run assumes a few things about your setup — see the **"Adapting this to your team"** section at the
bottom of [`SKILL.md`](./SKILL.md) for the full list, but in short:

- an issue tracker with a scriptable CLI/API (examples use an Atlassian-CLI-style tool; swap in Linear,
  GitHub Issues, whatever you use),
- domain implementation subagents/experts per stack (backend/mobile/frontend or your own split),
- an existing PR-creation step that follows your team's branch/commit/PR conventions and supports
  `--draft`,
- your own risk-tier taxonomy (auth, payment, migration, security/privacy are common defaults — night-run
  adds one local `infra` tag on top),
- whatever prod-safety discipline you already enforce for AI-assisted work (read-only replicas, no direct
  prod writes) — night-run's guardrail assumes that baseline and adds "never even attempt it" on top for
  the unattended case.

Each issue also needs a small structured block in its description (`depends_on` / `scope` / `dod` / `risk`)
so the topological sort, parallelism decision, and risk gate can run without a human in the loop. See
[`SKILL.md`](./SKILL.md) for the exact format.

## License

MIT — see [`LICENSE`](./LICENSE).
