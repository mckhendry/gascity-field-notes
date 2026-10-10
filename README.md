# Gas City field notes

Notes from running [Gas City](https://docs.gascity.com) (the `gc` multi-agent orchestrator, built on beads and Dolt) on a real build instead of the tutorial sandbox. These are the breaks, fixes and numbers from that work, written down so others can use them.

This is for engineers about to try Gas City with Claude Code. It assumes you've read the official quickstart.

## What's here

| Path | What it covers |
|---|---|
| [setup/gotchas.md](setup/gotchas.md) | The traps I hit in setup and in running a build through two milestones, each with its symptom, cause, fix and how I know |
| [setup/what-the-docs-say.md](setup/what-the-docs-say.md) | Each problem checked against Gas City's own docs: what they say, what configuration fixes, and what needs code |
| [setup/checklist.md](setup/checklist.md) | A short pre-flight and during-the-run checklist built from those gotchas |
| [concepts/how-gas-city-works.md](concepts/how-gas-city-works.md) | A plain-language model of cities, rigs, beads, formulas, sling and packs, and what each agent is given to read |
| [experiments/two-driver-comparison.md](experiments/two-driver-comparison.md) | The design of a head-to-head build: Gas City against a hand-rolled Claude Code orchestrator |
| [experiments/two-driver-results.md](experiments/two-driver-results.md) | Its results: delivery, gate, time, cost and human inputs for milestones 1 and 2, with the caveats |

Every claim says whether I saw it happen on a live city or only read it in the source. Each page carries its versions and dates. If your `gc` version differs, assume something here is stale.

## Where it stands

Gas City and a hand-rolled Claude Code orchestrator each built one pinned spec on their own blind branch, using one model at one effort level, through two milestones. Milestone 1 is graded, and milestone 2's numbers are preliminary until its grading is done. The results, with their caveats, are in [experiments/two-driver-results.md](experiments/two-driver-results.md).

## License

The prose in this repo is licensed under [CC BY 4.0](LICENSE).
