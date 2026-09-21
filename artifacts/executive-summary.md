# Stigmergy-MCP Benchmark: Executive Summary

**Phase 1 exploratory results. Runs April 27 to 30, 2026. Summary revised September 2026 to match the technical report.**

---

## The problem

Multi-agent AI systems spend tokens on coordination: transmitting, re-reading, and reconstructing what other agents produced. Under the simplest pattern, full-history handoff, each agent reads everything every predecessor wrote, so coordination tokens grow with the square of the agent count. Aggregate token totals do not show where the spend goes, so the same total can hide useful reasoning, duplicated work, or protocol overhead.

## The hypothesis

Trace-based coordination replaces full-history handoff with a shared store of compact, keyed traces. If each agent reads a bounded amount of state, coordination cost grows linearly with agent count instead of quadratically, in exchange for a fixed per-agent setup cost. Whether the amount each agent reads stays bounded is an empirical question, not a property of the architecture.

## What was measured

One model (`claude-sonnet-4-5`, temperature 0), one parameterized sequential-pipeline task run at 5, 6, 7, and 10 agents, plus three small instrument-check tasks at 1, 2, and 3 agents. Every API call was split into five token categories by a heuristic classifier. Output quality was not measured.

## The findings

1. **Trace coordination used fewer coordination tokens at every tested agent count from 5 to 10.** Median paired reductions were 34.8 percent (5 agents, 7 trials), 54.0 percent (6 agents, 10 trials), 51.8 percent (7 agents, 6 trials), and 46.3 percent (10 agents, 10 trials; 95 percent bootstrap interval 38 to 54 percent). A second independent 10-agent run of 7 trials gave 53.4 percent.

2. **The reduction did not grow with agent count.** The advantage peaked at 6 agents and was smaller at 10.

3. **The linear-cost assumption did not hold across the tested range.** Fixed per-agent overhead was stable (about 2.5k to 2.8k tokens of tool and protocol overhead plus about 0.8k of coordination instructions per agent). Retrieved content per agent was not: it rose from roughly 0.6k tokens at 5 to 7 agents to roughly 3.7k at 10 agents, and that rise replicated across the two independent 10-agent runs. A cost model calibrated at 10 agents would have predicted trace coordination to be _more_ expensive at 5 agents; it was 34.8 percent cheaper.

4. **Trace-condition agents used far fewer "task reasoning" tokens at 5 to 7 agents** (about 40 percent of the message-passing condition) **and about the same at 10.** This is unexplained. It may mean less redundant re-reading, less useful work because agents saw less, or misattribution by the classifier. Without a quality measure the three cannot be separated.

## What these results do not establish

- **Not that the outputs were as good.** No fidelity evaluation exists. A file of per-agent surface metrics in the repository measures only the final agent's output and cannot assess the assembled deliverable. Token savings without a fidelity measure are not efficiency.
- **Not a crossover point.** Every pipeline measurement favored traces, so no transition from "message passing cheaper" to "traces cheaper" was observed. If one exists it lies below 5 agents. The earlier statement placing a crossover somewhere from three to ten agents compared different tasks at those two sizes and is withdrawn. A later reversal above 10 agents is not excluded, because the per-agent trace cost was rising at 10.
- **Not that stigmergy specifically matters.** Decay and reinforcement were effectively inactive over these short runs. Bounded summaries and plain shared state were not tested; either might perform as well. The comparison is between full-history replay and one compact representation.
- **Not statistical strength from p = 0.002.** That value is the smallest a ten-trial paired test can produce. It records that all ten trials pointed the same way and says nothing about the size or generality of the effect. It is not the probability that there is no effect.
- **Not a real-deployment advantage.** No claim is made that production systems with longer-running agents would show a larger effect. That is a hypothesis for a future phase, not an expectation.
- **Not viability for coding or other high-dependency domains.** Nothing here shows that.

## Data and limitations

Trial-level rows are committed for the April 28 export snapshot. The 10-agent ten-trial cell and the 5 to 8 agent cells are documented by committed command-line transcripts; their trial-level rows are committed at `benchmark-exports/20260918-103059/`. Three mock-provider rows in the export are harness self-tests and are excluded. The 5, 7, and 8 agent cells lost trials to API credit exhaustion mid-run; the 8-agent cell has no valid trials. Conditions ran in a fixed order. The classifier contains a rule that treats tool definitions differently by condition; it has no effect on these results but must be fixed before shared-state controls are run. The harness's "UNRELIABLE" cross-validation flag fires on every run at temperature 0 and is uninformative.

## What's next

The decisive comparison is among bounded-compression schemes (bounded summaries, plain shared state, keyed retrieval with and without decay) at matched output quality on the assembled deliverable, with condition-independent attribution and randomized condition order. Phases varying turns per agent and handoff size remain planned and unfunded.

## The bottom line

In one sequential pipeline on one model, trace-based coordination cut coordination tokens by roughly a third to a half at 5 to 10 agents, but the cut did not grow with scale because the amount each agent retrieved grew instead. That is a useful negative result for the simple "linear" story and a useful positive result for measuring coordination cost by category rather than by total. It is not evidence that the outputs were as good, and it is not a general advantage for stigmergy. The full model, protocol, and limitations are in the technical report.

The methodology, source code, transcripts, and exports are at github.com/calabamatex/stigmergy-mcp-benchmark on the `results/phase-1-data` branch. The ten-trial 10-agent result is Result ID `c8e5cfb5-574b-4ffc-b0ad-7d0a3823cf00`.
