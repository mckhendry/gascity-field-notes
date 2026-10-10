# Two drivers, one spec: results

Headline results from the head-to-head described in [two-driver-comparison.md](two-driver-comparison.md). Gas City and a hand-rolled Claude Code orchestrator each built the same pinned spec, a Python package for a private video-analysis pipeline, on their own branch from the same base commit. Every agent on both sides ran the same model (Claude Opus 5.5) at `medium` effort.

- **Versions:** `gc` 1.5.0 with the `gascity` pack at `sha:3b3b89f`. Claude Code 2.1.293 at the start, 2.1.296 by milestone 2.
- **Dates:** milestone 1 on 2026-10-08, milestone 2 on 2026-10-09.
- **Status:** milestone 1 is graded. Milestone 2 has checked numbers for delivery, time, cost and human input, but **no quality grade and no reliability measurement**. That analysis wasn't run (see [Not run](#not-run)).

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

## Milestone 2 (measured, not graded)

Both sides started from their own milestone-1 code. For Gas City that took one extra step. Its publish couldn't deliver the detached milestone-1 commits (#18), so I fast-forwarded its branch to them by hand before the clock started. For milestone 2 only, Gas City ran with `push` and `open_pr` on.

**Neither side met its milestone-2 gate.** This section reports delivery, time, cost and human input. It gives no verdict on which build is better, because the analysis that would support one wasn't run (see [Not run](#not-run)).

### Gate and delivery

| | Gas City | Orchestrator |
|---|---|---|
| Gate met | No | No |
| Gate lines | Unit tests pass (887 tests). 17/19 inputs at or above their milestone-1 result. Plugin eval cases 0/8, because the eval sandbox stops the CLI (#25). Batch 16/19: three inputs' ingest hit Qdrant timeouts | 14/19 by its builder's own script, before a revert. After the revert, 3 inputs below their milestone-1 result |
| Delivery | **None.** The build sits on a detached HEAD in a local item worktree. Publish skipped with "Nothing approved to push or PR" | Delivered to the branch and its PR |

The orchestrator's drop from 14/19 comes from two rulings by its in-run advisor agent. First, it ruled the builder's post-call quote rewrite out of bounds, and the rewrite was reverted. The rewrite replaced any model quote that matched the transcript at 90% of words or better with the exact transcript text, so the quote check could no longer see a dropped or misheard word. Second, it ruled that rerunning failed inputs until they pass doesn't count toward the gate, under the milestone-2 spec's one-retry rule: "A second rejection is final, with no third request". That rule was set for both sides in the spec before the run.

Gas City's batch count and its per-input count are different measures. The batch is one run over all 19 transcripts; the per-input count scores the evaluations that run produced against milestone 1.

### Wall clock

Times are MST on 2026-10-09.

| | Gas City | Orchestrator |
|---|---|---|
| Start to stop | 11:26:56 AM to 5:58 PM | 11:26:54 AM to about 1:43 PM |
| Wall clock | **392.5 min** (6 h 32 m), of which 32.5 min falls past the 6 h cap, inside the one-hour extension I gave it (typed at 4:42 PM; cap moved to 6:27 PM) | **136.6 min** (2 h 17 m) |
| Usage-limit wait | None | None |
| Blocked-question wait | None. No agent waited on an answer; both questions closed `fail` unanswered | None |
| Milestone 1, for reference | 386.3 min | 115.2 min |

These replace the approximate figures this page gave earlier (about 6 h 31 m and 2 h 16 m), which came from the comparison record rather than the script-computed measures.

### Cost

As in milestone 1, these are list-price equivalents for Opus 5.5, not billed amounts.

- **Billable calls** are distinct billable API responses.
- **USD list** is list price, with 1-hour cache writes at $8/Mtok. **USD collector** is the cost-collection script's own figure, which prices every cache write at the 5-minute rate.
- **Coordination** is everything that isn't the built tool. For the orchestrator, that's its own session, its advisor and the subagents that wrote the code. For Gas City, it's all Gas City agents. So the orchestrator's coordination line includes its own session ($4.53), while Gas City's excludes the observer ($18.08), the driver session that watched Gas City for the exercise.
- **Package** is the built tool's own `claude -p` calls.

| | Gas City | Orchestrator | Gas City ÷ Orchestrator |
|---|---|---|---|
| Billable calls | 2,809 | 323 | 8.70× |
| USD list | **$198.38** | **$70.66** | 2.81× |
| USD collector | $167.88 | $61.50 | 2.73× |
| USD list, without the observer or orchestrator session | **$180.30** | **$66.13** | 2.73× |
| USD collector, without that line | $150.23 | $58.14 | 2.58× |
| Coordination, USD list | $154.60 | $32.66 | 4.73× |
| Package, USD list | $25.70 | $38.00 | 0.68× |

The dropped lines are Gas City's observer, $18.08, and the orchestrator's own session, $4.53. Spend inside the `claude plugin eval` sandboxes is outside these totals: Gas City $1.17 or more, the orchestrator $4.52.

### Human input

| | Gas City | Orchestrator |
|---|---|---|
| My typed turns during the run | 3: the sling command that started the run, the one-hour cap extension, and a command to stop a process | 1: the go message |
| My typed turns before the clock | 1: the go paste | 0 |
| Agents that stopped to ask a human | 2, in plain text rather than through the question tool. Both closed `fail` unanswered (#20) | 0. Its subagents can't ask |
| Items routed to a human for a ruling | Gas City's run report routed these to me (#19): approve a paid resume of 3 ingests that hit Qdrant timeouts (about $1); approve a paid eval rerun on one input; fix the plugin-eval harness or pick a different bar; decide whether one piece of evidence was enough for an acceptance criterion; decide what to do if the per-input regression repeats; and integrate the final commit, the human merge step built into finalize. None was answered, so finalize blocked | Ruled on during the run by its advisor agent |

That's a delegation gap. Gas City's escalations reached me only through the exercise's observer and the comparison's own orchestrator session (separate from the orchestrator side's), which recommended not answering them, for measurement purity. Gas City itself sent no notification. The orchestrator has an advisor agent empowered to rule on this kind of question during the run. Gas City's pack has no advisor and no spend budget.

### Misroutes: a Gas City bug, not setup

The misrouted workflow roots (#17) are a known Gas City bug, [gastownhall/gascity#6461](https://github.com/gastownhall/gascity/issues/6461). In `gc` 1.5.0 the root carries a route that makes it claimable, while the compiler treats it as a latch that's never routed. All 15 do-work roots carried the same route; the workers on the other 12 released them silently. A read-only trace ruled out my setup's hand repair of the rig import (#15). The repair put back an import entry byte-identical to the template default, with the same source and pin. The misroute comes from the pack's default variables plus `gc` core code, and the repair touched neither.

### Setup: a one-time adoption cost

Setup was part of using Gas City, so I count it rather than discount it, and report it apart from per-run cost.

- **Setup wall clock:** 15 h 21 m, from 5:42 PM MST on 2026-10-07 to 9:03 AM on 2026-10-08, including an overnight idle.
- **Attended setup time:** not recorded.
- **Setup spend:** not recorded. The milestone-1 and milestone-2 cost counts both exclude the setup sessions.
- **Milestone-2 prep:** time and spend not recorded.
- **What it took:** five Gas City-only setup differences: `gc`'s bd-init commit, an ignore-entries commit, a context-paths commit, `.beads/config.yaml` set skip-worktree, and the pack's check scripts installed untracked ([gastownhall/gascity#4239](https://github.com/gastownhall/gascity/issues/4239)). The orchestrator side carries one setup commit.

### Not run

I stopped the milestone-2 quality and reliability analysis before it finished, because its answer wouldn't change what I do next. So there was no cross-grading (each side's graders run on the other side's outputs), no comparison against the 6 regression baselines, no blind code read and no reliability pass. **Milestone 2 has no quality verdict, and neither build has a reliability measurement.** Partial output from the stopped grading was deleted, not used.

## What I take from it so far

At both milestones the orchestrator put its build on the branch and Gas City didn't. At milestone 1, code quality was within noise, and Gas City took longer, cost more and needed more of my time. Most of Gas City's gap has a named cause in the [gotchas](../setup/gotchas.md): delivery (#18), a review loop nobody was empowered to answer (#19), questions that went nowhere (#20), a missing dependency (#21) and an upstream routing bug (#17, #22). I'd rerun it with those fixed, plus a spend ceiling and a named decision owner in the goal bead, before saying much about the framework itself.
