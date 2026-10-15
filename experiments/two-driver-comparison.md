# Two drivers, one spec

This page describes the design. **Results are in [two-driver-results.md](two-driver-results.md):** milestone 1 graded, milestone 2 measured, a reconfigured Gas City run, and a reliability pass over every milestone-2 build.

## The question

Given the same written spec, does Gas City build working software better, faster or more cheaply than a hand-rolled Claude Code orchestrator? And what does each one cost the person running it?

## The two drivers

| | Side A: hand-rolled orchestrator | Side B: Gas City |
|---|---|---|
| Driver | One Claude Code session running a home-grown orchestration setup that plans, then dispatches Claude Code subagents to build | One Claude Code session that drives `gc` and records what it sees |
| Who writes code | Subagents the session dispatches | Gas City's `gc.*` role agents, started by the supervisor |
| How work is given | The spec and brief, read by the session | A target bead slung with `gc sling gc.run-operator <bead> --on build-basic`, plus a committed context bundle |
| Workflow settings | n/a | `interaction_mode=autonomous`, `drain_policy=separate` |
| Model and effort | The same model at `medium` effort | The same model at `medium` effort, pinned in `city.toml` (Gas City's built-in default is `max`) |
| Claude profile | The operator's normal profile | A blank Claude Code profile, so the operator's personal config doesn't reach the agents |

Effort is pinned to the same level on both sides on purpose. Gas City's default of `max` is a config setting, and comparing `max` against `medium` would measure the setting, not the tool.

## The target

A Python package for a private video-analysis pipeline. Both sides get the same read-only inputs: 19 transcripts, 6 baseline evaluations with their briefings, and 7 test files, all fixed at one commit of the source pipeline.

## Design rules

**Same pinned spec.** Both sides build from the same spec, step inventory and brief, pinned to one commit before either side starts. A panel of three models reviewed the spec over three passes until no reviewer still had a blocking objection. The brief records the spec's commit, and both tracking records carry the pin.

**Setup is off the clock.** All environment setup happens before either driver starts: Gas City's `init`, provider config, guard hook, rig and imports, plus both sides' worktrees, data directories and store namespaces. Setup ran in Claude Code's default permission mode, approved step by step, after the first attempt failed at launch (see [setup gotchas](../setup/gotchas.md) #1 and #2). Each side's build clock starts at its first edit under the package path. Setup time is reported separately.

**Blind branches.** Each side builds on its own branch, cut from the same base commit, in its own worktree. Both branches carry an identical exercise commit, except that side B also gets the context bundle Gas City needs. Each side has its own data directory, vector-store collection and graph label prefix. Neither side is supposed to look at the other's branch. On side B that depends on the agents following their brief, backed by a guard hook with known gaps (gotcha #10). Gas City has no confinement of its own.

**Observe only on side B.** The Gas City driver doesn't fold, rebase or move its agents' commits. It records where each commit lands. Given the pinned pack's detached worktrees (gotcha #9), where commits land is a finding in itself.

**Measured outcomes.** Planned measurements for each side:

- **Correctness:** the evaluation gate over all 19 transcripts. A pass is 19 of 19.
- **Quality against a baseline:** output compared with the 6 baseline evaluations.
- **Cross-grading:** each side's package grades the other side's output, in separate stores.
- **Time:** build wall-clock time from the first package edit, with setup reported separately.
- **Model usage:** call counts against fixed ceilings (8 model calls per video per batch run, 3 per video per evaluation pass).
- **Operator effort:** how much the person running it had to step in, and where.

**Stopping rules.**

- **Milestone 1:** both sides stop at the spec's first milestone for review.
- **Time cap:** 6 hours of wall-clock time per side, measured from boot.
- **Stall:** a side counts as stalled after 30 minutes with no new transcript activity while work is open.

## Status

| Phase | State |
|---|---|
| Spec pinned and reviewed | Done |
| First launch attempt | Failed during setup; reset and rerun |
| Setup, second attempt | Done |
| Milestone 1, both sides | Done; neither met the gate |
| Milestone 2, both sides | Done; neither met the gate |
| Reconfigured Gas City run, milestones 1 and 2, plus a recovery and a third gate attempt | Done; no build met the gate |
| Reliability pass | Done, one gate round per build |
| Results | [Final; Gas City dropped for this product on 2026-10-15](two-driver-results.md) |
