# Stigmergy Benchmark

Exploratory benchmark for measuring coordination cost across single-agent, cumulative message-passing, and trace-based multi-agent architectures.

## Research Status and Claim Boundary

This repository provides exploratory infrastructure, not proof of a general stigmergic advantage.

The released harness currently implements three conditions:

| Run | Architecture | Purpose |
| --- | --- | --- |
| A | Single Agent | Autonomous cost floor without inter-agent coordination |
| B | Cumulative Message Passing | Sequential control in which downstream agents receive predecessor state |
| C | Trace-Based Coordination | Experimental condition using [`stigmergy-mcp`](https://github.com/calabamatex/Stigmergy-mcp) |

The released measurements use heuristic token attribution. Fidelity was not evaluated, the published low- and high-agent observations use different task families, and no same-task agent-count sweep establishes a general scaling law or crossover. Bounded summarization, generic shared state, provenance-based attribution, and a fidelity gate remain specified future controls.

## Exploratory Phase 1 Observation

On one ten-agent sequential task using the documented model and cache configuration, the trace condition used fewer nominal inter-agent tokens than cumulative message passing. The canonical ten-trial run reported a median 46.3% inter-agent reduction and a median 71.9% content-transfer reduction.

The observation is task-specific and should not be interpreted as proof that trace coordination generally scales better, preserves fidelity, beats bounded summarization, or reduces billed cost under other cache and pricing regimes. The run used heuristic attribution and was flagged unreliable by the project cross-validation rule.

Data and artifacts:

- **[Phase 1 Release `phase-1-data-v1`](https://github.com/calabamatex/Stigmergy-mcp-benchmark/releases/tag/phase-1-data-v1)** — canonical SQLite export bundle, CSVs, schema, manifest, summary, and run logs
- **[Exploratory executive summary](https://github.com/calabamatex/Stigmergy-mcp-benchmark/blob/results/phase-1-data/artifacts/executive-summary.md)** — claim-bounded summary of the available observations
- **[`results/phase-1-data` branch](https://github.com/calabamatex/Stigmergy-mcp-benchmark/tree/results/phase-1-data)** — browsable result artifacts and exports
- **Canonical ten-agent result ID:** `c8e5cfb5-574b-4ffc-b0ad-7d0a3823cf00`

Result artifacts are stored on the results branch so binary snapshots do not enlarge the main code history. The release is the preferred immutable citation target.

A separate instrumentation gap is tracked at issue [#5](https://github.com/calabamatex/Stigmergy-mcp-benchmark/issues/5). The published snapshot predates that regression.

## Model Hypothesis

Under cumulative, non-summarizing, sequential handoff with approximately constant predecessor payloads, inter-agent content transfer can grow quadratically with agent count. Under bounded trace retrieval, bounded retrieved volume, and bounded per-agent operation frequency, trace-based coordination can grow approximately linearly.

Those are conditional model forms, not universal facts. Message passing can summarize, cache, prune, or route selectively. Trace size, retrieval breadth, call count, and mechanism overhead can also grow with agent count. The benchmark is intended to test where the assumptions hold and whether any cost reduction preserves task fidelity.

## What the Harness Measures

The harness runs matched task instances through the implemented conditions and separates usage into five functional categories:

- **Content Transfer (CT)** — work product passed between agents
- **Mechanism Overhead (MO)** — tool definitions, tool calls, serialization, and coordination substrate cost
- **Coordination Instructions (CI)** — prompts directing inter-agent behavior
- **Task Reasoning (TR)** — task-solving input and output
- **System Identity (SI)** — base system prompt

The current classifier is rule-based and suitable for exploratory diagnosis. A confirmatory implementation would require provenance-based prompt assembly and hard reconciliation with provider totals.

## Quick Start

```bash
pnpm install
pnpm build

# List available tasks
node packages/cli/dist/index.js tasks list

# Run a mock comparison
node packages/cli/dist/index.js compare --task research-report --trials 3 --provider mock

# Run with a real provider
ANTHROPIC_API_KEY=sk-... node packages/cli/dist/index.js compare \
  --task research-report --trials 10 --provider anthropic
```

## CLI Commands

```text
tasks list                              List benchmark tasks
compare --task <id> [options]           Run a comparison
results list                            List past results
results show <id>                       Show detailed results
```

### Compare Options

| Flag | Default | Description |
| --- | --- | --- |
| `--task <id>` | required | Task to benchmark |
| `--trials <n>` | 10 | Number of trials, minimum 3 |
| `--provider <p>` | mock | LLM provider: mock, anthropic, openai |
| `--model <m>` | provider-specific | Model identifier |
| `--temperature <t>` | 0 | Sampling temperature |
| `--skip-single-agent` | false | Skip Run A |
| `--db <path>` | `./stigmergy-benchmark.db` | SQLite database path |

## Dashboard

```bash
node packages/dashboard/dist/server.js
# Open http://localhost:3456
```

The dashboard provides task selection, live progress, category-level result charts, variance profiles, cross-validation status, and comparison history.

## Benchmark Tasks

| ID | Name | Agents | Category | Purpose |
| --- | --- | ---: | --- | --- |
| `research-report` | Research Report Pipeline | 3 | Sequential | Small sequential handoff observation |
| `multi-source-analysis` | Multi-Source Analysis | 4 | Parallel | Parallel fan-out and synthesis |
| `code-review` | Iterative Code Review | 3 | Iterative | Iterative refinement pattern |
| `single-agent-null` | Single Agent Null | 1 | Instrument check | Attribution and orchestration boundary check |
| `tiny-handoff` | Two-Agent Tiny Handoff | 2 | Sequential | Fixed-overhead boundary condition |
| `ten-agent-pipeline` | Ten-Agent Pipeline | 10 | Sequential | High-agent cumulative-handoff observation |

The tasks do not constitute a clean same-task scaling sweep. Agent-count comparisons across different tasks remain descriptive.

## Statistical Methodology

- **Paired trials:** each comparison runs A, B, and C on the same task instance
- **Bootstrap intervals:** 10,000 percentile resamples with a seedable PRNG
- **Wilcoxon signed-rank:** exact tables for `n <= 20`, normal approximation above 20
- **TOST equivalence:** available for project-defined crossover checks
- **Project reporting tiers:** RAW_ONLY (`n < 3`), PROVISIONAL (`3–4`), PRELIMINARY (`5–9`), FULL (`10–19`), PUBLICATION (`20+`)
- **Cross-validation:** drift threshold calibrated against Run A variance

The tier names are internal communication conventions. They do not replace formal power analysis, construct validation, fidelity controls, or representative task design.

## Future Confirmatory Controls

The research protocol identifies the following resource-contingent additions:

- bounded summarized message passing
- generic bounded shared state
- keyed retrieval without decay or reinforcement
- provenance-based token attribution
- blind fidelity non-inferiority evaluation
- declared strong- and weak-scaling regimes
- cache-adjusted dollar and latency accounting
- mechanism ablations for decay, reinforcement, and ranking

No completion date or funded execution is implied.

## Architecture

```text
core
  +-- stats
  +-- storage
  +-- llm-client
  +-- classifier
  +-- tasks
  +-- executors
  +-- engine
  +-- cli
  +-- dashboard
```

## Docker

```bash
docker compose run benchmark compare --task research-report --trials 3 --provider mock

ANTHROPIC_API_KEY=sk-... docker compose run benchmark compare \
  --task research-report --trials 10 --provider anthropic
```

Results persist in the `benchmark-data` Docker volume.

## Development

```bash
pnpm install
pnpm build
pnpm test
pnpm typecheck
pnpm lint
pnpm format:check
```

Husky and lint-staged run formatting and linting before commits. Conventional commit messages are enforced through commitlint.

Architecture decisions are documented in [`docs/adr/`](docs/adr/).

## License

MIT
