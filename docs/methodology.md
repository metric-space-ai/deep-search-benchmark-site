# Methodology

DeepSearchBenchmark measures web-search harnesses across live research tasks. The task set covers deep multi-hop chains, broad recall crawling, PDF and document extraction, official portals, technical live data and mixed provenance checks.

The benchmark is harness-agnostic. Every run contributes a `harnesses[]` entry and task-level records under `tasks[].results[harness_id]`. The UI renders comparison columns dynamically from those IDs, so additional harnesses do not require a redesign.

Scores and diagnostics are intentionally axis-separated:

- Run validity: did the harness run without runtime/config invalidation?
- Current dashboard score: reviewed research quality only
- Output contract: did it return parseable benchmark JSON?
- Latency: how long did the task take?
- Failure mode: model/tool/config/review issue?
- Hop correctness: reviewed externally after the run.

This prevents runtime/config regressions and formatting failures from being misread as research-quality failures.

## Current Score

The dashboard Score is reviewed research quality. It is not computed from JSON formatting, queue state, completion rate, elapsed time or whether the process returned zero.

Runs without task-level review records are shown as `unscored`. Runs invalidated by runtime/config failures are unranked and displayed as `invalid`.

## Reviewed Accuracy

| Component | Meaning |
|---|---|
| Hop depth | Number of required hops correctly completed, usually 0 to 5 |
| Final answer | Whether the end answer is correct, partial, or wrong |
| Evidence | Whether cited URLs or artifacts actually support the answer |
| Failure honesty | Whether blocked/partial work is reported instead of hallucinated |

The score is the aggregated reviewed accuracy once those records exist.

## Current Data

The current page still shows useful telemetry:

- Antigravity completed 50/50 commands in the existing baseline. 23/50 artifacts were strict JSON; 25 were recoverable; 2 were invalid. This is output-contract telemetry, not a research score.
- The prior CTOX run is unranked. It had 26 handled and 24 failed tasks, with the failed set dominated by a runtime/config issue (`service_tier=default`).

These values help debug harness behavior, but the CTOX run must be rerun before it becomes a benchmark result.

## Adding Harnesses

For each new harness:

1. Add one object to `harnesses[]` with stable `id`, display name, runtime, runner, completion data and available telemetry.
2. Add a result object under every task's `results[id]`.
3. Set `run_valid=false` when runtime/config failures invalidate the run.
4. Add failure-mode entries only for failures observed in that run.

The dashboard will automatically add plot points, leaderboard rows and task columns.
