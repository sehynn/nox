# Nox

<p align="center">
  <img src="./mascot.png" alt="Nox mascot" width="420" />
</p>

### When you clock out, Nox clocks in.

Hand Nox a queue of tickets before you leave. By morning, the completed ones are waiting as reviewed and tested draft PRs.

Nox never merges. That's where the night shift ends.

*An unattended coding workflow for Claude Code and Codex.*

[![License: MIT](https://img.shields.io/badge/license-MIT-blue.svg)](LICENSE)
[![Claude Code](https://img.shields.io/badge/Claude%20Code-compatible-5A32FB)](#install)
[![Codex](https://img.shields.io/badge/Codex-compatible-10A37F)](#install)
[![PRs welcome](https://img.shields.io/badge/PRs-welcome-brightgreen.svg)](../../pulls)

---

## A night with Nox

**6:03 PM** — you hand him five tickets on your way out: `FOO-101` through `FOO-105`. He checks what
depends on what and shows you the order before you're even out the door.

**6:04 PM** — you're gone. Nox clocks in.

`FOO-101` gets designed, reviewed, built, tested, and opened as draft PR `#421`. Next.

`FOO-102` and `FOO-103` were supposed to run side by side tonight. Nox notices they'd both touch the same
file. Nobody told him not to run them together — he just doesn't. One at a time instead.

`FOO-104` doesn't add up, even after repeated review rounds. He sets it down and writes why, instead of
shipping a guess.

`FOO-105` turns out bigger than the ticket let on. He doesn't force it through — back on your desk, with a
note for daylight.

**9:02 AM** — 3 draft PRs waiting, reviewed and tested. 1 blocked with a reason. 1 sent back to planning.
0 merged without you.

---

## What Nox does

Nox processes structured coding tasks while you are offline.

It determines the execution order, checks for scope conflicts, validates each design, implements the
approved changes, runs the relevant tests, and opens a separate draft PR for every completed ticket.

Each ticket ends with one of three outcomes:

* a reviewed and tested draft PR,
* a clear explanation of why it was blocked,
* or a recommendation to return it to planning because the scope is too large or ambiguous.

Final review and merge always remain with you.

---

## Why Nox exists

Coding agents can already handle individual prompts. The problem begins when several tickets are left running unattended for hours.

A simple "handle these overnight" prompt can result in:

* one large branch containing unrelated changes,
* several worktrees competing for CPU and memory,
* parallel tasks modifying the same files,
* an agent continuing with an incorrect assumption,
* oversized changes that are difficult to review,
* or completed work being lost when a long-running session fails.

Nox adds an execution workflow around the coding agent.

It defines how tickets are ordered, reviewed, isolated, tested, checkpointed, and handed back to a human.

The goal is not maximum autonomy. The goal is useful progress that remains reviewable in the morning.

---

## Core guardrails

### Draft PRs only

Every completed ticket produces a separate draft PR.

Nox does not promote PRs to ready-for-review and never merges them. Final approval always belongs to a human.

### No production writes

Production access is treated as read-only.

If a ticket requires a direct production write—or access through any writable path—Nox blocks the task instead of attempting it.

### Sequential by default

Tickets run one at a time unless limited parallel execution has been explicitly enabled.

Before parallelizing tasks, Nox compares their declared and inferred scopes. If two tasks may touch the same files or components, they are automatically processed sequentially.

### Design review before implementation

Nox creates a design before changing code.

A separate reviewer evaluates the design. Implementation begins only after the design receives the required consecutive approvals.

The agent that creates the design does not approve its own work.

### No guessing

Ambiguous tickets are marked `BLOCKED` with a reason.

Nox does not invent missing requirements simply to produce a result.

### Oversized work returns to planning

A reviewer can classify a ticket as `TOO_LARGE` when its actual scope exceeds what can be completed and reviewed safely during an unattended run.

The ticket is returned as `ROUTE_TO_PLANNING` instead of being forced through implementation.

### Immediate checkpoints

A draft PR is opened as soon as each ticket is completed.

Results are not held until the entire queue finishes. If the session stops during the night, previously completed work remains available.

---

## Workflow

```text
Tickets
   ↓
Validate structure and dependencies
   ↓
Determine execution order
   ↓
Check for scope overlap
   ↓
Create design
   ↓
Independent design review
   ↓
Implement
   ↓
Run scoped tests
   ↓
Open draft PR
   ↓
Continue to next ticket
   ↓
Generate run report
```

For every ticket, Nox:

1. validates the required metadata,
2. checks dependencies and execution order,
3. identifies overlapping scopes,
4. creates a design,
5. sends the design to a separate reviewer,
6. implements the approved design,
7. runs the relevant tests,
8. opens a draft PR immediately,
9. updates the issue tracker,
10. records the result in the final report.

---

## Example run

At the start of a run, Nox receives five tickets:

```text
FOO-101
FOO-102
FOO-103
FOO-104
FOO-105
```

During execution:

* `FOO-101` completes successfully and produces draft PR `#421`.
* `FOO-102` and `FOO-103` appear independent, but both may modify the same file. Nox changes their execution mode from parallel to sequential.
* `FOO-104` remains ambiguous after repeated design reviews. It is marked `BLOCKED` with an explanation.
* `FOO-105` is substantially larger than described. It is returned as `ROUTE_TO_PLANNING`.

The completed run produces a report like this:

```markdown
## Nox's shift report — 2026-09-16

| Issue   | Result            | Draft PR | Tracker     | Design review | Risk    |
|---------|-------------------|----------|-------------|----------------|---------|
| FOO-101 | DONE              | #421     | In progress | PASS ×2        | None    |
| FOO-102 | DONE              | #422     | In progress | PASS ×2        | Infra   |
| FOO-103 | DONE              | #423     | In progress | PASS ×2        | None    |
| FOO-104 | BLOCKED           | —        | Unchanged   | Failed         | Payment |
| FOO-105 | ROUTE_TO_PLANNING | —        | Unchanged   | TOO_LARGE      | Infra   |

Completed: 3/5
Blocked: 1/5
Returned to planning: 1/5
Merged automatically: 0
```

The report shows what was completed, what needs attention, and which decisions remain with you.

---

## Nox vs. an ad hoc overnight prompt

|                        | Ad hoc prompt                               | Nox                                             |
| ---------------------- | ------------------------------------------- | ------------------------------------------------ |
| Task boundaries        | Often handled in one long branch or session | One ticket and one draft PR at a time           |
| Execution order        | Depends on the prompt                       | Derived from declared dependencies              |
| Scope conflicts        | Usually discovered after changes collide    | Checked before parallel execution               |
| Ambiguous requirements | The agent may continue with assumptions     | The task is marked `BLOCKED`                    |
| Oversized tickets      | The agent may continue until the diff grows | A reviewer returns the ticket to planning       |
| Design approval        | Usually self-evaluated                      | Reviewed by a separate agent                    |
| Production access      | Depends on prompt wording                   | Production writes are blocked                   |
| Failure recovery       | Results may remain only in the session      | Each completed task is checkpointed immediately |
| Final authority        | May be unclear                              | Human review and merge are always required      |

Nox is not a different coding model. It is a repeatable workflow for running coding agents unattended with explicit boundaries.

---

## Ticket format

Each ticket needs a small structured block so Nox can make execution decisions without waiting for human input.

```
### nox
- depends_on: [FOO-100] (or "none")
- scope: src/modules/payment/** (repo + rough path this touches)
- dod: pnpm test -- payment.refund.spec.ts (a command whose pass/fail is machine-checkable)
- risk: payment
```

### `depends_on`

Lists tickets that must be completed before the current ticket can begin, or `none`.

Nox uses this field to create a topologically sorted execution order.

### `scope`

Declares the rough path or component the ticket is expected to modify.

Nox compares scopes before running tasks in parallel. Inferred file overlap discovered during design can also force tasks back to sequential execution.

### `dod`

A single command whose exit code determines whether the ticket is complete — for example, a scoped test
run. It must be machine-checkable, not a prose checklist: implementation and verification run this command
directly and treat its pass/fail as the answer.

### `risk`

A single value identifying the area that requires additional scrutiny, such as:

* auth,
* payment,
* migration,
* security,
* privacy,
* or infra (a Nox-local addition for CI/CD and deploy-config changes).

Adapt the risk taxonomy to your project.

---

## Install

### Claude Code

Copy [`SKILL.md`](./SKILL.md) into your project:

```text
<your-project>/.claude/skills/nox/SKILL.md
```

Then invoke the skill from Claude Code.

### Codex

Copy [`SKILL.md`](./SKILL.md) into:

```text
<your-project>/.codex/skills/nox/SKILL.md
```

Add the following field to the skill frontmatter:

```yaml
codex_type: skill
```

No other workflow changes are required specifically for Codex.

---

## Requirements

Nox assumes that your project already has:

* an issue tracker accessible through a scriptable CLI or API,
* implementation agents or subagents appropriate for your stack,
* an existing branch, commit, and PR workflow,
* support for creating draft PRs,
* project-specific test commands,
* a risk classification policy,
* and a production-access policy suitable for AI-assisted development.

The included examples use an Atlassian-style CLI, but the workflow is not tied to Jira. You can adapt it to GitHub Issues, Linear, or another tracker.

Likewise, the default examples divide implementation work by backend, frontend, and mobile. Replace these roles with the agents or skills that match your repository.

See the **Adapting this to your team** section in [`SKILL.md`](./SKILL.md) for the required configuration points.

---

## Parallel execution

Nox runs sequentially by default.

Limited parallel execution can be enabled for tickets that:

* have no dependency relationship,
* have non-overlapping declared scopes,
* have no inferred file overlap,
* do not require full repository builds at the same time,
* and remain within the configured concurrency limit.

If any overlap is detected, the affected tickets are automatically returned to sequential execution.

Parallel tasks should run only their relevant scoped tests. Full builds remain checkpoint operations to avoid unnecessary CPU and memory contention.

---

## What Nox will not do

Nox will not:

* merge a pull request,
* promote a draft PR automatically,
* write to production,
* bypass repository permissions,
* guess missing product requirements,
* continue indefinitely on an unreviewable design,
* force an oversized ticket through implementation,
* or parallelize tasks with overlapping scopes.

These limits are part of the workflow, not recommendations left to individual prompts.

---

## Contributing

Issues and pull requests are welcome.

Useful contribution areas include:

* additional issue tracker adapters,
* improved scope-overlap detection,
* examples for different repository structures,
* safer concurrency strategies,
* and integrations for additional coding agents.

If you find a case where Nox makes an unsafe assumption, opens an unreviewable change, or fails to preserve completed work, please open an issue with a reproducible example.

---

## License

MIT. See [`LICENSE`](./LICENSE).
