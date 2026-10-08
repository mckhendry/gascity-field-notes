# Gas City field notes

Notes from running [Gas City](https://docs.gascity.com) (the `gc` multi-agent orchestrator, built on beads and Dolt) on a real build instead of the tutorial sandbox. These are the breaks, fixes and numbers from that work, written down so others can use them.

This is for engineers about to try Gas City with Claude Code. It assumes you've read the official quickstart.

## What's here

| Path | What it covers |
|---|---|
| [setup/gotchas.md](setup/gotchas.md) | The setup traps I hit, each with its symptom, cause, fix and how I know |
| [setup/checklist.md](setup/checklist.md) | A short pre-flight checklist built from those gotchas |
| [concepts/how-gas-city-works.md](concepts/how-gas-city-works.md) | A plain-language model of cities, rigs, beads, formulas, sling and packs, and what each agent is given to read |
| [experiments/two-driver-comparison.md](experiments/two-driver-comparison.md) | The design of a head-to-head build, with results still to come |

Every claim says whether I saw it happen on a live city or only read it in the source. Each page carries its versions and dates. If your `gc` version differs, assume something here is stale.

## Coming next

A measured head-to-head is in progress. Gas City and a hand-rolled Claude Code orchestrator each build one pinned spec on their own blind branch, using one model at one effort level. Setup isn't timed, and both stop at the spec's first milestone. Results go in [experiments/two-driver-comparison.md](experiments/two-driver-comparison.md) once both sides finish.

## License

The prose in this repo is licensed under [CC BY 4.0](LICENSE).
