# Two drivers, one spec: results

Headline results from the head-to-head described in [two-driver-comparison.md](two-driver-comparison.md). Gas City and a hand-rolled Claude Code orchestrator each built the same pinned spec, a Python package for a private video-analysis pipeline, on their own branch from the same base commit. Every agent on both sides ran the same model (Claude Opus 5.5) at `medium` effort.

- **Versions:** `gc` 1.5.0 with the `gascity` pack at `sha:3b3b89f`. Claude Code 2.1.293 at the start, 2.1.296 by milestone 2.
- **Dates:** milestone 1 on 2026-10-08, milestone 2 on 2026-10-09.
- **Status:** milestone 1 is graded. **Milestone 2 is preliminary: graded results to follow.** Its numbers come from each side's own run record and the observer's log, not from an independent regrade.

Numbers such as #18 refer to entries in [setup/gotchas.md](../setup/gotchas.md).

## Read this first

- **One run per side, per milestone.** Neither result has been repeated. A one-video gap on the gate is within run-to-run noise.
- **Neither side met either gate.** Both sides missed the spec's bar at both milestones, so this compares two incomplete builds.
- **The setups weren't identical.** Gas City needed five setup changes of its own (#12–#15, plus installing the pack's missing check scripts, #21), none counted as build output. Its driver session only observed, while the orchestrator's driver was the orchestration itself. Permission regimes differed: the orchestrator ran with permission checks off, and Gas City's agents ran unrestricted behind a guard hook that's only a shallow blocklist (#10).
- **Delegation wasn't symmetric.** The orchestrator had an advisor agent with standing authority to rule on spending and judgment calls during the run. Gas City's pack has no equivalent, and I didn't act as one for it. Part of Gas City's milestone-2 result follows from that (#19).
- **One Gas City problem is an upstream bug.** Workflow roots misrouted to workers (#17) are [gastownhall/gascity#6461](https://github.com/gastownhall/gascity/issues/6461), not something in my setup. They cost seats and stalled steps. Tracing them through the source showed they weren't what caused the failures recorded on the affected beads.
- I missed a PyYAML dependency. I assumed the controller could import PyYAML. It couldn't, so every Gas City artifact check failed three times (#21). I've left that retry time and cost in Gas City's numbers.
- **Spec changes between milestones applied to both sides.** For milestone 2, one score became report-only, and each model call got one retry when its structured output failed the schema.

## Milestone 1 (graded)

### Delivery

| | Gas City | Orchestrator |
|---|---|---|
| Build commits on the pushed branch | **0** | 3 |
| Where the build ended up | 11 commits on a detached HEAD in a local item worktree (#18) | The pushed branch, with a draft PR |
| Why | Finalize left the work detached, and `push` and `open_pr` were off (the formula default, kept by my choice) | Committed and pushed as part of its run |

Both sides built the full milestone-1 artifact set, so the quality grading below uses Gas City's local code even though none of it reached the branch.

### Gate and quality

| | Gas City | Orchestrator |
|---|---|---|
| Gate as each side reported it | 15/19, measured before its last commit and not rerun | 12/19 |
| Gate rerun on final code, from a fresh home | **14/19** | **13/19** |
| Non-baseline inputs passing (13) | 13/13 | 12/13 |
| Unit tests passing | 615 | 217 |
| Blind code read (out of 25) | 22 | 21 |

The pass mark was 19/19. The one-input gap on the rerun comes down to which input a schema rejection happened to hit: each side lost one call that way, and the spec forbade a retry at milestone 1. The blind read favored Gas City's package on tests and structure, and the orchestrator's on keeping code and model calls apart.

### Time

| | Gas City | Orchestrator |
|---|---|---|
| Start to first build edit | 27.4 min | 5.0 min |
| Wall clock, start to stop | 386.3 min (6 h 26 m) | 115.2 min (1 h 55 m) |
| Excluding the shared usage-limit wait (#26) | 298.3 min (4 h 58 m) | 115.2 min |

The 298.3 minutes still include 52 minutes in which Gas City's run sat blocked on an agent's question to me (#20).

### Cost

List-price equivalents for Opus 5.5, counted from transcripts. Both sides ran on a subscription, so nothing here was billed at these rates.

| | Gas City | Orchestrator |
|---|---|---|
| All tokens | 176.3 M | 41.6 M |
| Output tokens | 2.60 M | 0.78 M |
| **USD, list** | **$144.40** | **$54.54** |
| Of which, idle worker slots that found no work (#22) | $34.95 | n/a |
| Without the driver session | $128.44 | n/a (its driver is the orchestration) |
| Package model calls only | $25.01 | $30.67 |

Gas City cost 2.65× as much in dollars and 4.24× in tokens. Its idle-worker drain alone cost more than the orchestrator's whole orchestration layer ($23.64). `gc costs` showed $0 for every session, because `gc` 1.5.0 has no price for this model, and its session count fell short of the transcripts. Every Gas City figure here is counted from transcripts.

### Human inputs

| | Gas City | Orchestrator |
|---|---|---|
| Inputs during the run | 4: three typed turns to its driver, one answer to an agent's question | 1: the boot command |
| Before the clock | 3 typed turns and 6–7 permission prompts | 0 |

## Milestone 2 (preliminary: graded results to follow)

Both sides started from their own milestone-1 code. For Gas City that took one extra step. Its publish couldn't deliver the detached milestone-1 commits (#18), so I fast-forwarded its branch to them by hand before the clock started. For milestone 2 only, Gas City ran with `push` and `open_pr` on.

### Delivery

| | Gas City | Orchestrator |
|---|---|---|
| Build on the pushed branch | **None.** Publish skipped with "Nothing approved to push or PR" | Yes, with its draft PR updated |
| Where the build ended up | A detached HEAD in a local item worktree | The pushed branch |
| Finalize status | `blocked`: review never approved, and the code was never integrated | n/a |

### Gate

| | Gas City | Orchestrator |
|---|---|---|
| Gate met | No | No |
| Unit tests | 887 passed | 253 passed |
| Inputs below their milestone-1 result | 2 (3 on a stricter reading) | 3 |
| Plugin eval cases | 0/8. The eval sandbox stops the CLI (#25) | 8/8 |
| Batch | 16/19. Three ingests timed out against the vector store | Exit 1. Two inputs failed after their one allowed retry |

The orchestrator's builder at first passed one input by rewriting near-match quotes after the model call. Its advisor agent ruled that out of bounds, the rewrite was reverted, and the gate above is the result without it. Earlier reruns of two failed inputs until they passed were also ruled not to count.

### Time and cost

| | Gas City | Orchestrator |
|---|---|---|
| Start to first build edit | about 7 min | about 4 min |
| Wall clock | about 6 h 31 m, 31 min into a one-hour cap extension | about 2 h 16 m |
| Package model calls | $25.81 | $36.41 |
| `gc` invocations | 1,895 (unpriced) | n/a |
| Full list-price cost | to follow | to follow |

### What reached a human

| | Gas City | Orchestrator |
|---|---|---|
| Agents stopping to ask a human | 3, all on misrouted workflow roots (#17, #20). None answered. Each bead closed `fail` on its own | None in its run record |
| Items routed to a human for a ruling | 6: approval for two paid reruns (one about $1), three judgment calls on the bar, and integrating the final commit (#19). None answered | Ruled on during the run by its advisor agent |
| Human input tally | to follow | to follow |

## What I take from it so far

At both milestones the orchestrator put its build on the branch and Gas City didn't. At milestone 1, code quality was within noise, and Gas City took longer, cost more and needed more of my time. Most of Gas City's gap has a named cause in the [gotchas](../setup/gotchas.md): delivery (#18), a review loop nobody was empowered to answer (#19), questions that went nowhere (#20), a missing dependency (#21) and an upstream routing bug (#17, #22). I'd rerun it with those fixed, plus a spend ceiling and a named decision owner in the goal bead, before saying much about the framework itself.
