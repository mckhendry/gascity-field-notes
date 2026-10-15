# Two drivers, one spec: results

Results from the head-to-head described in [two-driver-comparison.md](two-driver-comparison.md). Gas City and a hand-rolled Claude Code orchestrator each built the same pinned spec, a Python package for a video-analysis tool, on their own branch from the same base commit. Every agent on both sides ran the same model (Claude Opus 5.5) at `medium` effort.

Gas City ran twice. The **first run** used the template's pack with only the setup fixes in the gotchas; it is the "Gas City" column in the milestone 1 and 2 sections. The **reconfigured run** started from scratch with every fix the docs research pointed to; it is [Run C](#run-c-gas-city-reconfigured). The orchestrator ran once.

- **Versions:** `gc` 1.5.0 with the `gascity` pack at `sha:3b3b89f`. Claude Code 2.1.293 at the start, 2.1.296 by milestone 2.
- **Reconfigured run versions:** `gc` 1.5.0 with pre-release pack main plus overrides in my own fork of the pack. Claude Code 2.1.296.
- **Dates:** first run milestone 1 on 2026-10-08 and milestone 2 on 2026-10-09. Reconfigured run 2026-10-10 to 2026-10-14. Reliability pass 2026-10-12 to 2026-10-14. Gas City dropped for this product on 2026-10-15.
- **Status:** final. Milestone 1 is graded. Milestone 2 was measured for delivery, time, cost and human input; the quality pass for it was stopped, and a smaller [reliability pass](#reliability-pass-all-builds) later measured every milestone-2 build. See [Outcome](#outcome-gas-city-dropped-for-this-product).

Numbers such as #18 refer to entries in [setup/gotchas.md](../setup/gotchas.md).

## Read this first

- **One run per side, per milestone.** No milestone was repeated on the same configuration. A one-video gap on the gate is within run-to-run noise.
- **No build met either gate.** All three builds missed the spec's bar at both milestones, so this compares incomplete builds.
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

This section covers the orchestrator and the first Gas City run. Both builds were later measured by the [reliability pass](#reliability-pass-all-builds), which is not the quality grading planned here.

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
| Blocked-question wait | None. No agent waited on an answer; all three questions closed `fail` unanswered | None |
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
| Agents that stopped to ask a human | 3, in plain text rather than through the question tool. All three closed `fail` unanswered (#20) | 0. Its subagents can't ask |
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

I stopped the milestone-2 quality and reliability analysis before it finished, because its answer wouldn't change what I do next. So there was no cross-grading (each side's graders run on the other side's outputs) and no comparison against the 6 regression baselines. **Milestone 2 has no quality verdict.** Partial output from the stopped grading was deleted, not used. The gate rerun, injected faults and blind code read it would have included ran later, in reduced form, in the [reliability pass](#reliability-pass-all-builds).

## Run C: Gas City reconfigured

After the first run, I checked every problem against Gas City's own docs ([what-the-docs-say.md](../setup/what-the-docs-say.md)) and asked one question: was the first run's result mostly a configuration failure? To answer it, I built a new city from scratch with every documented fix and ran milestones 1 and 2 again on the same spec, model and effort, under the same 6-hour cap.

Dollar figures in this section, like the rest of the page, are recorded list-price equivalents of model usage. Nothing was billed at these rates.

### What I configured differently, and why

Each change targets a named failure from the first run:

| Change | Why | Kind |
|---|---|---|
| Pre-release pack main instead of the release pin | The release pin has no check scripts (#21). Pack main fixes the paths, but its review check needs bash 4, so I installed Homebrew bash | Configuration, plus one host change |
| PyYAML in the check interpreter's main site-packages | Every artifact check failed on it (#21) | Host configuration |
| `interaction_mode=headless`, the ask tool blocked on the agents' command line and in settings, and never-ask text in the stage prompts | Agents stopped to ask a human, and nobody saw it (#20) | Configuration, plus prompt overrides in my fork |
| An attached branch per item, an integrate step, and a publish that pushes the integrated commit and opens a draft PR | `build-basic` never lands work on a branch (#9, #18) | Overrides in my fork |
| An advisor step in the review, empowered to rule on spending and the bar | Nobody could answer the review loop (#19) | Override in my fork |
| A workaround for the misrouted workflow roots: the prepare step removes the route from each root | Upstream bug [gastownhall/gascity#6461](https://github.com/gastownhall/gascity/issues/6461) (#17, #22) | Workaround for a Gas City defect, in my fork |
| An order on `execution.step_stalled` that mails the human mailbox | Stalls reached nobody (#20) | Configuration |
| Pool caps (3 implementation workers, 1 run operator) and effort pinned, checked on each agent's command line | Unbounded pools at effort `max` (#4, #22) | Configuration |
| `gc init --no-start` and a fresh city name | The always-on mayor kept its startup settings (#23); there's no documented city teardown | Configuration |
| A spend cap of $200 per milestone, enforced by my own transcript-priced collector that calls `gc suspend` | Gas City has no spend budget, and its own usage figure undercounts (#32) | My instrumentation, not Gas City |

Milestone 2 added two changes, decided after milestone 1's data and disclosed as such:

- a prompt override on all 12 live step prompts that tells agents never to end a turn while a background job runs, a workaround for the execution backstop killing active work (#28);
- a fence that keeps workers from editing the goldens, baselines and fixture manifest the gate reads (#36).

The milestone-1 goal bead also carried a preamble the first run didn't have. It replaced the brief's delivery and no-push lines with "Gas City's own publish step pushes the recorded commit to [the branch] and opens a draft pull request". So part of any difference at milestone 1 is a brief difference, not configuration.

**Setup, as one-time adoption cost:** setup ran from the evening of 2026-10-10 to midday on 2026-10-11, including an overnight stall. It took three delivery smoke tests, two of which failed, and ended with nine commits in my pack fork, plus one more before milestone 2. Setup spend and my attended minutes weren't recorded.

So this run answers "Gas City configured with overrides in my own pack fork", not "Gas City as shipped".

### The bar, set before data

Before the run, I wrote down what would count as a configured-Gas-City win, loss or tie against the orchestrator:

1. **Delivery is a condition, not a rank.** The final commit has to reach the pushed branch through Gas City's own steps, with no hand fold.
2. **Reliability first:** injected-fault handling, then the gate passing in 3 of 3 runs, then the number of videos that flip between runs.
3. **Then source quality,** as the median of three blind code reads. A gap of 2 points or less out of 25 is a tie.
4. **Then process cost:** dollars, my attended minutes, and wall clock without waits.

### Milestones 1 and 2 (frozen)

| | Milestone 1 (2026-10-11) | Milestone 2 (2026-10-12) |
|---|---|---|
| Delivery | **None.** Integrate hit a conflict and failed closed; publish pushed nothing (#33) | **Met.** Gas City's integrate and publish pushed the build and opened a draft PR, with no fold. The first Gas City delivery in the exercise |
| Gate | Not met, by Gas City's own report (not rerun). Tests 457 passed; eval 14/19, five failing on claim validity against the web-verified baselines | Not met. Layer 1: tests 667 passed, eval 13/19. Layer 2: one regression on a newly gated dimension. Layer 3: plugin eval 2/8 |
| Wall clock, start to stop | 3 h 18 m 57 s | 4 h 59 m 54 s |
| Sleep wait | 29 m 23 s | 69 m 33 s, in four spans |
| Usage-limit and blocked-question waits | None | None |
| Wall clock without waits | **2 h 49 m 34 s** | **3 h 50 m 21 s** |
| Cost, transcript-priced | **$94.14** (corrected from $90.13 at the stop, to include the driver session's turns after it) | **$109.19** |
| Gas City's own usage figure, same window | $24.63 | $37.45 |
| My typed turns during the run | 1: the start word | 1: the start word |
| Before the clock | 6 typed turns, between the end of setup and the driver's boot | 1: asking for the driver's prompt |
| Other touches during the run | 1: waking the Mac from sleep | None counted; closing and opening the lid weren't counted as touches |
| Agent questions to a human | 0 | 0 |
| Decisions ruled in-run by the advisor step | 5, in 2 min 25 s, none escalated | 5, none escalated |
| Routed to me by the final report, after the stop | 2: re-baseline, ungate or accept the claim-validity miss; move the branch and rerun integrate and publish | 3: authorize post-gate fixes; accept or ungate the regressed dimension; read the PR |
| Claim-race stalls (#27) | 4, plus 1 race without a stall: 5 of 8 items | 3 |
| Active work killed by Gas City | 2: a live batch and a gate rerun (#28) | 0 |
| Other stalls | 0 | 1, after the host slept mid-call; Gas City recovered it in about 3 min (#29) |

The transcript-priced figure includes the driver session that watched the run. Gas City's own figure can't see that session, so part of the gap is the driver, not undercounting (#32).

Two faults at milestone 1 were the agents' own judgment, not Gas City defects:

- A worker edited the product's scoring prompt so claim validity was scored without web verification, which changed what its own gate graded. That's why milestone 2 fenced the gate's inputs (#36).
- The advisor step ruled to rewrite history to drop a tracked directory, and the rewrite went below the target branch's tip. That rewrite is what broke integrate (#33).

The items routed to me reached me as information after each stop. Nothing done after a stop counts toward a frozen row.

### All three builds, frozen rows

| | Orchestrator | Gas City, first run | Gas City, reconfigured |
|---|---|---|---|
| Milestone 1 delivered | Yes | No | No |
| Milestone 2 delivered | Yes | No | Yes |
| Milestone 1 wall clock without waits | 115.2 min | 298.3 min (usage-limit wait out; the 52-min blocked question left in) | 169.6 min |
| Milestone 2 wall clock without waits | 136.6 min | 392.5 min | 230.4 min |
| Milestone 1 cost | $54.54 | $144.40 | $94.14 |
| Milestone 2 cost | $70.66 | $198.38 | $109.19 |
| Milestones 1 + 2 | $125.20 | $342.78 | $203.33 |

Configuration cut Gas City's cost by about 41% from the first run and got the build onto a branch at milestone 2. It still cost more and ran longer than the orchestrator.

### After the gate: recovery and a third attempt (outside the bar)

After milestone 2 stopped, I ordered a recovery run toward the milestone-2 gate, because the point of the exercise was also a usable tool. These rows sit **outside the bar**: the orchestrator and the first Gas City run never got a second attempt, so nothing below is compared with them. The frozen rows above stay as recorded.

| | Recovery: milestone-2 gate, second attempt (2026-10-12) | Third attempt (2026-10-14) |
|---|---|---|
| Work | 8 items: the final report's recorded defects, a plugin-eval diagnosis and fix, a diagnosis of the regressed dimension, a from-scratch index and batch, a network test, and the gate | 1 item: measure the gate once, with no code change. Its base included a fixture move made outside the run |
| Gate | Not met. Layer 1: tests 691 passed, eval 14/19. Layer 2: one regression (10/11 to 8/11). Layer 3: 2/8 | Not met. Layers 1 and 2 as in the recovery. **Layer 3: 8/8** |
| Delivery | **None as measured:** integrate collided with an untracked report (#30). Then **human-assisted:** an operator session removed the collision, and the failed steps were reopened so Gas City's own integrate and publish ran (#31) | **Met,** by Gas City's own steps |
| Wall clock, start to stop | 4 h 38 m 37 s | 50 m 31 s |
| Sleep wait | 29 m 35 s | None |
| Wall clock without waits | 4 h 09 m 02 s | 50 m 31 s |
| Cost, transcript-priced | $98.73 (corrected from $97.50 at the stop, to include the delivery retry) | $31.64 |
| Gas City's own usage figure | $27.53 | $6.42 |
| My typed turns during the run | 1: the start word | 1: the start word |
| Operator time | About 15 min of cleanup and step reopening | About 10 min |
| Claim-race stalls | 3 | 1 |
| Active work killed by Gas City | 0 | 0 |
| Other stalls | 1, after a host sleep; about 5 min redone | 0 |

Two notes on the recovery:

- The fixture move came from the recovery's own diagnosis. The plugin eval's sandbox can't read fixtures inside the suite directory, and the recovery's rules forbade moving them, so the move waited for my approval and was made outside the run.
- I also approved re-baselining the one regressed video's two context-dependent dimensions. That turned out infeasible: the spec's rule only copies existing values, and that video was the first ever ingested, so no populated-context reference exists. It stays a known issue.

**Reconfigured Gas City, total to my usable condition:** $94.14 + $109.19 + $98.73 + $31.64 = **$333.70**, plus about $13 for a resample of the eval and its share of the reliability work. My usable condition was 14/19 with the remaining misses listed as known issues, if layer 3 read 8/8. The third attempt met it. For scale, the first Gas City run spent $342.78 on milestones 1 and 2 and delivered neither.

## Reliability pass, all builds

After the reconfigured run's milestone 2, one executor measured all three milestone-2 builds by the method written before the run: gate runs on fresh directories, three injected faults, and a blind code read. A later addendum measured the recovered build the same way.

- **F1:** a model output that passes the schema but carries a fabricated quote and an out-of-range timestamp.
- **F2:** a kill partway through promoting a finished evaluation.
- **F3:** a rerun over an existing run directory.

| | Orchestrator | Gas City, first run | Gas City, reconfigured | Reconfigured, recovered (addendum) |
|---|---|---|---|---|
| Gate exit, one round | 1 | 1 | 1 | 1 |
| Eval count | 12/19 | 13/19 | 14/19 | 13/19 |
| Regressions against its own milestone 1 | No comparison shipped | 2 | 1 | 2 |
| Plugin eval (layer 3) | 8/8 | Not runnable in isolation: it hardcodes its own milestone-2 directories and indexes | 2/8 | 8/8 |
| F1 | Corrupted silently | Corrupted silently | Corrupted silently, and code rewrote the fake timestamp to a valid one, hiding it from the timestamp check | Same as reconfigured |
| F2 | Failed loudly | **Corrupted silently** | Failed loudly | Failed loudly |
| F3 | Handled correctly | Handled correctly | Handled correctly | Handled correctly |
| Blind read, median of three, out of 25 | 20 | 24 | 20 | 23 |

F1 is classed on the production `run` path. The package's own `eval` flagged F1 on every build, so "silent" describes the run layer, and eval is the mitigation.

The pass cost $108.79 against a $135 guard, and the addendum $30.39, for $139.18 in all. Both figures are closing totals that include the executors' report writing.

### What the pass can't tell you

- **One gate round per build, not three.** A round was projected at about $79 across three builds, and the guard covered one. So flakiness wasn't measured, the 3-of-3 gate line couldn't be computed, and the eval counts (12, 13 and 14 of 19) are one video apart, inside noise. I don't rank on them.
- **Same-model readers.** Every reader was Opus 5.5, the same model that wrote every build, so self-preference can't be checked.
- **Partial blinding.** Branch names that the builds' isolation guards require stayed in some code and test files.
- **The recovered build was read later, by fresh readers,** each reading only that one build. Its 23 isn't a within-reader comparison with the others.
- **F3's class, "handled correctly", was added after the data,** because none of the three planned classes fit a correct rerun. All four builds fall in it.
- **The eval resample overstated stability** ([lessons](lessons-from-the-exercise.md)). Four eval-only samples of the recovery's stored outputs all read 14/19, but only one grader calls the model, so most of that stability was by construction. A fresh batch on the recovered build read 13/19, with a different failing set and two regressions.

### Against the bar

- **Delivery:** the reconfigured run met it at milestone 2, not at milestone 1.
- **Faults:** the reconfigured build and the orchestrator landed in the same classes; the first Gas City build was worse on F2.
- **Gate:** every build failed its one round. The 3-of-3 line and flakiness weren't measured.
- **Source quality:** the orchestrator and the reconfigured build tied at 20. The first Gas City build scored 24.
- **Cost:** the orchestrator was lower on dollars and on wall clock.

The bar's reliability comparison is incomplete, so I don't compute a win, loss or tie from it. An outside reviewer (a different model) read the record and put it this way: the results support "configuration fixes important plumbing; it does not establish a configured-Gas-City win."

## Outcome: Gas City dropped for this product

On 2026-10-15 I dropped Gas City as the path to this product. The recovered build is being finished with the hand-rolled orchestrator instead, and I stopped both cities with Gas City's own commands (#34, #35). I may come back to Gas City on its own, but not for this tool.

I asked an outside reviewer (a different model) to read the whole record and recommend a way forward. Its verdict: "Gas City is not worth continuing as the main path to this [product]: configuration improved delivery and reduced waste, but it has not demonstrated better reliability than [the orchestrator], and reaching [my] accepted usable state required additional runs and [help from the sessions running the exercise]."

Its reasons, which I accepted:

- **No demonstrated reliability advantage.** The reconfigured build and the orchestrator had identical fault classes, both failed the gate, and both scored 20/25 on the blind read. The three-round comparison never happened.
- **Higher cost and longer runs.** The reconfigured run's two milestones cost $203.33 against the orchestrator's $125.20, and reaching the usable condition took another $130.37.
- **Continuing operational defects.** Claim races in every phase (#27), two kills of active jobs (#28), and an integration collision in the recovery (#30).
- **Low touch counts that depended on structure.** My few typed turns relied on an operator session, an observer and an advisor working around the run.

It also found that the remaining product gap is only partly a harness problem. Some gate misses come from single-judgment legacy baselines, one of them set on an empty knowledge base, and those misses were shared across builds. That's in [lessons-from-the-exercise.md](lessons-from-the-exercise.md).

## What I take from it

Configuration fixed Gas City's plumbing. With the documented fixes and a handful of overrides in my own pack fork, the reconfigured run delivered to a branch, ruled its own review decisions, never stopped to ask a human, and cost about 41% less than the first run. It did not make Gas City better than the hand-rolled orchestrator at this job: the builds were equally reliable where measured, and Gas City cost more, ran longer, and still hit stock defects in every phase (#27, #28, #30, #32). The exercise had process faults of its own, which I've kept separate in [lessons-from-the-exercise.md](lessons-from-the-exercise.md).
