# Stigmergy Benchmark

Exploratory harness for measuring inter-agent coordination cost in multi-agent LLM systems. It compares full-history message passing with trace-based coordination via [stigmergy-mcp](https://github.com/calabamatex/stigmergy-mcp) and decomposes every API call into five functional token categories. Current results are descriptive: trace coordination used fewer coordination tokens than full-history handoff at every tested agent count from 5 to 10 in one pipeline task on one model, but output fidelity was not measured, bounded-summary and shared-state controls were not run, and per-agent retrieved content grew with agent count. See the technical report for the model, the protocol, and the limitations.

## Results

Phase 1 is exploratory: one sequential pipeline task, one model (`claude-sonnet-4-5`,
temperature 0), 5 to 10 agents. Trace-based coordination used fewer inter-agent tokens
than full-history message passing at every tested agent count.

| Agents | Valid trials | Median reduction, inter-agent tokens | Median reduction, content transfer |
| ------ | ------------ | ------------------------------------ | ---------------------------------- |
| 5      | 7            | 34.8%                                | 90.8%                              |
| 6      | 10           | 54.0%                                | 91.0%                              |
| 7      | 6            | 51.8%                                | 90.9%                              |
| 10     | 10           | 46.3% (95% bootstrap CI 38–54%)      | 71.9%                              |

A second, independent ten-agent run of 7 trials gave 53.4%. Every figure above is
recomputed from the trial-level rows committed in
`benchmark-exports/20260918-103059/` on the `results/phase-1-data` branch.

Content transfer — the work product itself, moved between agents — is where the
mechanism shows: traces replace replaying a predecessor's full output with a compact,
keyed summary. The aggregate reduction is smaller because trace coordination adds
fixed per-agent protocol overhead that full-history handoff does not pay.

What these numbers do not show:

- **Not efficiency.** Output fidelity was never measured. Fewer tokens for work of
  unknown quality is not a demonstrated saving.
- **Not linear scaling.** The reduction did not grow with agent count; it peaked at 6.
  Retrieved content per agent rose from roughly 0.6k to 3.7k tokens between 5 and 10
  agents, so the linear trace form is a hypothesis to measure, not a property of the
  architecture.
- **Not statistical strength.** p = 0.002 for the ten-agent cell is the smallest value a
  ten-trial paired test can produce. It records that all ten pairs shared a sign; it says
  nothing about effect size or about the probability that there is no effect.
- **Not a crossover, and not stigmergy-specific.** Every tested size favored traces, so
  no crossover was observed. Bounded-summary and shared-state controls were not run, so
  nothing here isolates the stigmergic mechanism from compact representation generally.

Where to find the data:

- **[`phase1-report-v1`](https://github.com/calabamatex/Stigmergy-mcp-benchmark/tree/phase1-report-v1)** — the tagged repository state cited by the technical report. Canonical.
- **[Executive summary (Markdown)](https://github.com/calabamatex/Stigmergy-mcp-benchmark/blob/results/phase-1-data/artifacts/executive-summary.md)** — findings, limitations, and what comes next.
- **[`results/phase-1-data` branch](https://github.com/calabamatex/Stigmergy-mcp-benchmark/tree/results/phase-1-data)** — export snapshots, audit runs, and per-agent pipeline logs.
- **[Release `phase-1-data-v1`](https://github.com/calabamatex/Stigmergy-mcp-benchmark/releases/tag/phase-1-data-v1)** — May 2026 bundle. Its executive summary predates this alignment and states claims withdrawn above; use the tag instead.

> **Note:** Result artifacts live on the `results/phase-1-data` branch, not `main`, so
> binary snapshots don't bloat the code history.

**Data caveat.** The `token_usage` table (one row per API call) is empty in every
committed export, including the April 2026 snapshot — see issue
[#5](https://github.com/calabamatex/Stigmergy-mcp-benchmark/issues/5). What is committed
is trial-level: per-trial, per-category subtotals in `trial_results`, which are the
numbers reported above. Per-call records for the reported cells do not exist in the
repository, so the classifier's category assignments cannot be re-derived after the fact.

## What It Does

Runs the **same task** through three architectures, measures token usage across five categories, and applies statistical rigor (bootstrap CIs, Wilcoxon signed-rank, TOST equivalence) over multiple trials.

| Run | Architecture          | Purpose                                                                                                  |
| --- | --------------------- | -------------------------------------------------------------------------------------------------------- |
| A   | Single Agent          | Single-agent reference (no coordination; not workload-matched to the pipeline sweep)                     |
| B   | Message-Passing Swarm | Control: agents share cumulative conversation history                                                    |
| C   | Stigmergy Swarm       | Experimental: agents coordinate via [stigmergy-mcp](https://github.com/calabamatex/stigmergy-mcp) traces |

### Five Token Categories

Every API call's tokens are classified into exactly one category:

- **Content Transfer (CT)** — work product passed between agents
- **Mechanism Overhead (MO)** — coordination protocol tokens (tool defs, tool calls)
- **Coordination Instructions (CI)** — prompts directing inter-agent behavior
- **Task Reasoning (TR)** — the agent's actual thinking and output
- **System Identity (SI)** — base system prompt

**Cost model (conditional):** Under full-history message passing, CT grows as O(N^2) because each agent ingests all predecessors. Under trace-based coordination, CT grows as O(N) _only if_ the content each agent retrieves stays bounded as N grows. Phase 1 measurements show it did not stay bounded from N=5 to N=10: retrieved content per agent rose from about 0.6k to about 3.7k tokens while fixed per-agent overhead stayed flat. Treat linearity as a hypothesis the harness measures, not a property of the architecture.

## Quick Start

```bash
# Install
pnpm install

# Build
pnpm build

# List available tasks
node packages/cli/dist/index.js tasks list

# Run a comparison (mock LLM, 3 trials)
node packages/cli/dist/index.js compare --task research-report --trials 3 --provider mock

# Run with real API
ANTHROPIC_API_KEY=sk-... node packages/cli/dist/index.js compare --task research-report --trials 10 --provider anthropic
```

## CLI Commands

```
tasks list                              List benchmark tasks
compare --task <id> [options]           Run a comparison
results list                            List past results
results show <id>                       Show detailed results
```

### Compare Options

| Flag                  | Default                  | Description                           |
| --------------------- | ------------------------ | ------------------------------------- |
| `--task <id>`         | (required)               | Task to benchmark                     |
| `--trials <n>`        | 10                       | Number of trials (min 3)              |
| `--provider <p>`      | mock                     | LLM provider: mock, anthropic, openai |
| `--model <m>`         | per provider             | Model name                            |
| `--temperature <t>`   | 0                        | Temperature                           |
| `--skip-single-agent` | false                    | Skip Run A                            |
| `--db <path>`         | ./stigmergy-benchmark.db | SQLite path                           |

## Dashboard

```bash
# Start the dashboard server
node packages/dashboard/dist/server.js

# Open http://localhost:3456
```

The dashboard provides:

- Task selection with configurable trials/provider
- Live comparison progress via WebSocket
- Results visualization with 5-category stacked bar charts
- Variance profile and cross-validation status
- History of past comparisons

## Benchmark Tasks

| ID                    | Name                     | Agents | Category   | Purpose                                           |
| --------------------- | ------------------------ | ------ | ---------- | ------------------------------------------------- |
| research-report       | Research Report Pipeline | 3      | Sequential | Tests CT growth in sequential handoffs            |
| multi-source-analysis | Multi-Source Analysis    | 4      | Parallel   | Tests parallel fan-out + synthesis                |
| code-review           | Iterative Code Review    | 3      | Iterative  | Tests iterative refinement patterns               |
| single-agent-null     | Single Agent (Null)      | 1      | —          | Validates instrumentation (~0% savings)           |
| tiny-handoff          | Two-Agent Tiny Handoff   | 2      | Sequential | Crossover detection (TOST)                        |
| ten-agent-pipeline    | Ten-Agent Pipeline       | 10     | Sequential | Quadratic-handoff form vs linear-trace hypothesis |

## Statistical Methodology

- **Paired trials:** Each trial produces a matched triplet (A, B, C on the same task)
- **Bootstrap CIs:** 10,000 resamples, percentile method, seedable PRNG
- **Wilcoxon signed-rank:** Exact tables for n <= 20, normal approximation for n > 20
- **TOST equivalence:** For crossover tasks, tests if savings are within +/-5%
- **Progressive reporting:** RAW_ONLY (n<3) -> PROVISIONAL (3-4) -> PRELIMINARY (5-9) -> FULL (10-19) -> PUBLICATION (20+)
- **Cross-validation:** Drift threshold calibrated against Run A variance (2x CV). At temperature 0 the Run A CV is near zero, so the flag fires on nearly every trial and is uninformative for the Phase 1 runs.
- **Reporting tiers are sample-count labels only.** `PUBLICATION` is a legacy name for 20+ valid trials and does not assert publication readiness. No Phase 1 cell reached it.

## Architecture

```
core        (types, enums, config)
  |
  +-- stats       (bootstrap, Wilcoxon, TOST, aggregator)
  +-- storage     (SQLite persistence)
  +-- llm-client  (Anthropic, OpenAI, Mock + retry + rate limiter)
  +-- classifier  (5-category rule-based classification)
  +-- tasks       (6 benchmark task definitions)
  |
  +-- executors   (Run A, B, C + MCP bridge to stigmergy-mcp)
  |
  +-- engine      (comparison orchestrator, progressive reporting)
  |
  +-- cli         (command-line interface)
  +-- dashboard   (Express + WebSocket + React frontend)
```

## Docker

```bash
# Build and run with mock provider
docker compose run benchmark compare --task research-report --trials 3 --provider mock

# Run with real API keys
ANTHROPIC_API_KEY=sk-... docker compose run benchmark compare --task research-report --trials 10 --provider anthropic
```

Results persist in a named Docker volume (`benchmark-data`).

## Development

```bash
pnpm install          # Install all dependencies
pnpm build            # Build all packages
pnpm test             # Run all tests (169 tests)
pnpm typecheck        # Type-check without emitting
pnpm lint             # ESLint 9 flat config
pnpm format:check     # Prettier check
```

### Pre-commit Hooks

[Husky](https://typicode.github.io/husky/) + [lint-staged](https://github.com/lint-staged/lint-staged) run automatically on commit:

- **TypeScript**: `eslint --fix` + `prettier --write`
- **JSON/MD/YAML**: `prettier --write`

Commit messages are enforced by [commitlint](https://commitlint.js.org/) (conventional commits format).

### Architecture Decision Records

Key design decisions are documented in [`docs/adr/`](docs/adr/):

| ADR                                                   | Decision                                |
| ----------------------------------------------------- | --------------------------------------- |
| [001](docs/adr/001-sqlite-persistence.md)             | SQLite for benchmark persistence        |
| [002](docs/adr/002-rule-based-classification.md)      | Rule-based token classification         |
| [003](docs/adr/003-wilcoxon-over-t-test.md)           | Wilcoxon signed-rank over paired t-test |
| [004](docs/adr/004-bootstrap-confidence-intervals.md) | Bootstrap CIs over analytical CIs       |
| [005](docs/adr/005-in-memory-mcp-bridge.md)           | In-memory MCP bridge for stigmergy      |

## License

MIT
