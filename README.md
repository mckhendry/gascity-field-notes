# Gas City field notes

Notes from running [Gas City](https://docs.gascity.com) (the `gc` multi-agent orchestrator, built on beads and Dolt) on a real build instead of the tutorial sandbox. These are the breaks, fixes and numbers from that work, written down so others can use them.

This is for engineers about to try Gas City with Claude Code. It assumes you've read the official quickstart.

## What's here

| Path | What it covers |
|---|---|
| [setup/gotchas.md](setup/gotchas.md) | The traps I hit in setup and across two Gas City runs, each with its symptom, cause, fix and how I know, and for the later ones whether it's a Gas City defect, configuration or my mistake |
| [setup/what-the-docs-say.md](setup/what-the-docs-say.md) | Each problem checked against Gas City's own docs: what they say, what configuration fixes, what needs code, and what a live run with those fixes confirmed or disproved |
| [setup/checklist.md](setup/checklist.md) | A pre-flight, during-the-run and teardown checklist built from those gotchas |
| [concepts/how-gas-city-works.md](concepts/how-gas-city-works.md) | A plain-language model of cities, rigs, beads, formulas, sling and packs, and what each agent is given to read |
| [experiments/two-driver-comparison.md](experiments/two-driver-comparison.md) | The design of a head-to-head build: Gas City against a hand-rolled Claude Code orchestrator |
| [experiments/two-driver-results.md](experiments/two-driver-results.md) | Its results: delivery, gate, time, cost and human inputs for both Gas City runs and the orchestrator, the reliability pass, and why I dropped Gas City for this product |
| [experiments/lessons-from-the-exercise.md](experiments/lessons-from-the-exercise.md) | The exercise's own process mistakes, kept apart from Gas City's, and what I'd do differently |

Every claim says whether I saw it happen on a live city or only read it in the source. Each page carries its versions and dates. If your `gc` version differs, assume something here is stale.

## Where it stands

The exercise is finished. Gas City built one pinned spec twice, first close to stock and then reconfigured with every documented fix plus overrides in my own pack fork, against a hand-rolled Claude Code orchestrator on the same model and effort. Configuration fixed the plumbing: the reconfigured run delivered to its branch at milestone 2 and cost about 41% less than the first run, but it was no more reliable than the orchestrator where measured, and it cost more and ran longer. On 2026-10-15 I dropped Gas City for this product; the results and their limits are in [experiments/two-driver-results.md](experiments/two-driver-results.md), and the exercise's own mistakes are in [experiments/lessons-from-the-exercise.md](experiments/lessons-from-the-exercise.md).

## License

The prose in this repo is licensed under [CC BY 4.0](LICENSE).
