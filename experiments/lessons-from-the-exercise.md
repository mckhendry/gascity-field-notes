# Lessons from the exercise

The [two-driver comparison](two-driver-comparison.md) had process mistakes of its own, separate from anything Gas City did. This page keeps them apart from Gas City's [gotchas](../setup/gotchas.md), because several of them cost as much time and money as any Gas City defect, and they would have hurt any harness comparison.

- **Versions:** `gc` 1.5.0; every agent on Claude Opus 5.5 at `medium` effort; Claude Code 2.1.293 to 2.1.296.
- **Dates:** 2026-10-07 to 2026-10-15.
- **Sources:** my experiment record, an advisor session's filed rulings, the reliability reports, and a review of the whole record by an outside reviewer (a different model) on 2026-10-15. Numbers come from the [results page](two-driver-results.md).

Besides the build agents, the exercise ran a driver session per side, an operator session that set up Gas City, an advisor session that ruled on judgment calls, and an orchestrating session that kept the record. Several lessons below are about that structure.

## 1. Measurement purity crowded out the product goal

The exercise had two goals: improve a video-analysis tool, and learn whether Gas City was worth adopting. In practice the comparison's rules kept winning over the tool:

- After the reconfigured run's milestone 2, Gas City's final report asked for permission to fix its recorded defects. The advisor session's pick was no post-stop fixes, to keep the row clean. I had to overrule it: "you're moving the goal posts if I don't have a usable product".
- During the recovery, the rules that protected the gate's inputs also blocked moving a set of test fixtures out of a directory the plugin-eval sandbox can't read. The fix was diagnosed and proven on a copy, but it waited for a separate third run.

**Lesson:** freezing the measured rows and repairing the product can happen side by side. Authorize the repair at the same time as you freeze the row.

## 2. Delegation was asymmetric

The orchestrator had an advisor agent with standing authority to rule on spending and judgment calls during its run. Gas City's first run had none, and the sessions around it chose not to answer its six routed requests, for measurement purity. Its review loop iterated six times and failed. So part of the first run's milestone-2 result belongs to the exercise, not to Gas City.

The reconfigured run fixed this with an advisor step inside Gas City's review, which ruled five decisions in about two and a half minutes. But the first run's delivery gap can't all be charged to Gas City.

**Lesson:** give every side the same decision authority before the clock starts, or answer every side's questions the same way.

## 3. The reliability protocol couldn't be afforded as designed

The bar set before data put reliability first: three gate runs per build, three injected faults, and nine blind reads, under a $135 guard. The cost estimate behind that guard was a floor of $89.61 for three gate rounds, taken from an earlier, cheaper gate. The real milestone-2 gate came to about $79 per round across three builds, or about $236.80 for three rounds.

So only one round ran. Flakiness and the bar's 3-of-3 gate line were never measured, and the comparison couldn't compute the verdict it was designed around. Two more rounds would have cost about $160, and they weren't funded.

**Lesson:** price a protocol from the real thing it measures before committing to it, then fund it or cut it before any data arrives.

## 4. I re-measured failures that were already explained

- **The execution backstop.** One kill plus a source read explained it (gotcha #28). The fix waited for milestone 2 so milestone 1 would stay one configuration, and a second paid gate rerun was killed about 50 minutes later.
- **The fixture move.** The recovery diagnosed why the plugin eval failed and proved the fix on a copy. Confirming it still took a full third run, with planning, review and publish: $31.64 and 50 minutes, for what a targeted check could have settled.
- **The report collision.** The risk that an untracked report would block integrate was named at prep. The run launched anyway, failed at integrate, and needed a cleanup and two reopened steps (gotcha #30).

**Lesson:** once a cause is traced, fix it or record it. Don't pay for a run to watch it happen again.

## 5. I overstated stability from re-scoring stored outputs

After the recovery, an eval-only resample scored the same stored outputs four times and got 14/19 every time, so I called the misses stable. But only one grader calls the model. Every other grader is a pure function of the stored outputs, so most of that stability was by construction. A draft list of known issues repeated the claim.

A fresh batch on the recovered build then read 13/19, with a new failing video and two regressions instead of one. The draft was corrected before it was posted.

**Lesson:** say which layer you resampled. A model-scored product's stability needs fresh end-to-end runs, not re-scoring.

## 6. Attention measurement was too thin to support "0 minutes"

The logs recorded my touches as points in time. The reconfigured run's milestone-1 log had 8 touches, 6 of them without an end time, and every log summed to 0 captured minutes. Left out entirely:

- closing and opening the lid, and waking the Mac;
- reading the relays sent to me during the runs;
- the decisions I made with the advisor session;
- the operator session's work: 11 minutes of milestone-2 prep, about 15 in the recovery and about 10 in the third attempt.

The first run recorded no attended time at all. So "0 attended minutes" can't be claimed, and the low touch counts depended on the sessions working around the run.

**Lesson:** log attention with start and end times, and count operator and advisor work beside human touches.

## 7. The gate was built on single-judgment legacy baselines, one of them set on an empty knowledge base

The gate compared each build's scores with six baselines from the tool's older pipeline. Each baseline was one model judgment, and some misses were shared by every build, which points at the reference, not only at the code.

One baseline was scored on an empty knowledge base. Its novelty score was the empty-base default of 1.0. Milestone 2 deliberately adds knowledge-base context, which can correctly lower novelty, so the gate penalized the feature it was meant to test. Re-baselining it wasn't possible: the spec's rule only copies existing values, and that video was the first one ever ingested, so no populated-context reference exists.

That explanation has limits. Two other baselines did use a populated knowledge base, and fixing the one video's context-dependent scores would still leave its claims miss.

**Lesson:** before a gate decides anything, freeze the knowledge-base snapshot and retrieved context, test empty and populated context separately, and have a person adjudicate disputed reference examples. Use acceptable ranges, not one historical answer.

## What I'd do differently

- **Start from the product outcome.** The outside reviewer put it this way: "The next attempt needs one owner, one bounded product outcome, explicit decision authority, and a stop rule. It does not need another from-scratch harness contest."
- **Freeze the rows, fix the product.** Authorize repairs from the start, on their own labeled lines.
- **Give every side the same delegation,** written into each goal before the clock.
- **Price the protocol against its guard** before data, and cut it to what's funded.
- **Calibrate the gate first.** Adjudicate the legacy baselines, freeze the context they need, and decide what counts as an acceptable answer.
- **Trace once.** When a cause is known, fix it or record it instead of re-measuring it.
- **Measure variation end to end,** with fresh batches, and say which layer any resample covers.
- **Log attention as durations,** with operator and advisor work beside mine.
- **Clear known hazards before launch:** an awake host, a clear artifact root, and a spend cap priced from transcripts ([checklist](../setup/checklist.md)).
