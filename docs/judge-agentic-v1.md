# Agentic External Judge — V1 Design

Status: design locked for implementation. Judging runs ONLY after collection is complete
(collect-only runs of all harnesses). One LLM (Claude Opus) judges all results relatively.

## 1. Principle

The judge is a **verification agent, not a taste rater**. Per task it receives the original
task, the anonymized candidate results, and live web tools (WebSearch/WebFetch). It grades
what it can verify, not what sounds plausible. One agentic run per task; the runner turns
its per-dimension scores into reviews.v2 `pairwise_verdicts`, which feed the existing
Bradley-Terry/Elo pipeline (`scripts/compute_relative_scores.py`) unchanged.

## 2. Inputs per task (bundle, assembled by `external_judge.py collect`)

| Input | Source | Use |
|---|---|---|
| task statement | `tasks/<id>/task.md` (HTML comments stripped) | what was asked |
| category, capability | `tasks/<id>/meta.toml` | slice keys for BT rollup |
| collection window | `runs/<run>/<task>/window.json` (`window_open`/`window_close`) | recency is judged **relative to the window**, drift since is never penalized |
| orientation rubric | `tasks/<id>/reference/required_elements.json` (≤20 leaves) | leaf coverage; perishable leaves are live-resolved at judge time, evergreen leaves stay frozen (rubric_spec v2.1 `reference_free` principle) |
| canary | `tasks/<id>/reference/CANARY.txt` | integrity check (see §6) |
| candidates | `<harness>.result.json`: `answer`, `source_urls`, `hop_status`, `side_work`, `final_answer_supported`, `failure_mode`, `contract` | the material being judged |

Candidates are anonymized **K1..Kn, order shuffled deterministically per task** (seed =
task_id). Harness/model identifiers, wall times and token counts are **stripped** from the
judge bundle (timing is reported on the site as a separate efficiency axis, never mixed
into quality Elo).

## 3. Agentic protocol (what the judge must do)

1. Extract the required hops/sub-questions from the task.
2. **Spot-check the 2–3 most load-bearing claims per candidate** with WebFetch/WebSearch:
   does the cited URL exist, and does its content support the claim? Every spot-check is
   logged in the output (`spot_checks[]`) for audit.
3. Live-resolve perishable rubric leaves (e.g. "current index level") and interpret them
   against the collection window.
4. Detect hallucination signals: fabricated/unreachable load-bearing sources, claims
   contradicted by the candidate's own cited source, `final_answer_supported=true` without
   coverage.
5. Honesty rule: an **honestly documented blockade** (hop failed/partial + `failure_mode`
   set) is NOT hallucination — it earns honesty credit while costing final/completeness.
6. Canary scan (§6).

## 4. Dimensions and scoring rules

Six dimensions, 0.00–1.00 per candidate (same set as `bench.*.toml` and
`compute_relative_scores.py` `DIM_W = {hop .34, final .26, evidence .18, completeness .07,
recency .10, honesty .05}`):

- **hop** — research chain actually traversed correctly (right entity/document/value per hop, verified)
- **final** — end answer correct & precise (number, currency, timestamp) relative to the window
- **evidence** — cited sources exist and support the claims; primary sources > aggregators
- **completeness** — all sub-questions / rubric leaves covered
- **recency** — answer demonstrably based on live retrieval inside the window, not model memory; **scored conditional on correctness** (fresh but wrong earns nothing)
- **honesty** — self-report (`hop_status`, `final_answer_supported`, `failure_mode`) matches the judge's verification; fabrication ⇒ ≤0.2

Hard rules given to the judge:
- hallucination detected ⇒ honesty ≤ 0.2 AND the affected claim counts as wrong in hop/final/evidence
  (the legacy ×0.65 multiplier remains in the absolute-anchor path of `apply_review_scores.py`;
  in the relative path the penalty is expressed through the dimensions themselves),
- style/length neutrality: "you grade research quality, not writing style; length is not quality",
- no identity speculation about candidates,
- if a dimension cannot be judged from evidence: 0.5 + note (never silently guess).

## 5. Position-bias control (two passes)

Per task the runner makes **two judge calls**: pass A in shuffled order, pass B in reversed
order. From each pass's scores the runner derives per-dimension verdicts for every pair,
filling the reviews.v2 fields exactly:

```
{task_id, capability, dimension, pair: [hid_a, hid_b],
 order_a_first: {judge, verdict}, order_b_first: {judge, verdict},
 swap_consistent, length_controlled_p_a: null,
 selected_artifact_hash: sha256(answer_a + "|" + answer_b)}
```

`compute_relative_scores.py` already smooths swap-inconsistent verdicts toward 0.5 — no
changes needed there.

## 6. Integrity & disclosure

- **Canary**: if a candidate's own prose (outside quoted source text) contains the task's
  canary string, the run is flagged (`canary_hits`) → run invalidated (accessed judge materials).
- **Self-judging disclosure**: Claude Opus judges while Claude Code is a contestant. Mitigation:
  anonymization + shuffle + swap passes + verification-grounded scoring + full audit trail
  (every verdict incl. spot-checks is published on the site). Stated openly in methodology.
- **Abstention**: if a task's bundle is unusable (e.g. all candidates empty), the judge emits
  `abstain` and the task is excluded from BT for that slice (logged, never silently dropped).

## 7. Backend & mechanics

- New backend in `judge_backend()` (`scripts/external_judge.py:134`): `claude-agentic` →
  `claude -p --model claude-opus-4-8 --allowedTools "WebSearch,WebFetch" --dangerously-skip-permissions`,
  timeout 900 s per pass, 4 retries with backoff (rate-limit cascades happened before).
- `judge_task()` (line 149) gains the two-pass swap and the richer prompt (template below,
  stored at `scripts/judge_prompt_v1.md`); JSON extraction via existing `extract_json()`.
- **Resume**: per-task verdict files `runs/judge-v1/<task_id>/pass_{a,b}.json`; existing files skipped.
- **Oracle-less scale pinning**: V1 collection ran without the discovery oracle, so Elo is
  pinned to mean=1000 across contestants instead of the oracle anchor (deviation documented;
  oracle anchoring can be added in a later window without re-judging).
- Cost estimate: 100 tasks × 2 passes × ~2–4 min ≈ 7–13 h serial; 3–4 concurrent ≈ 2–4 h.

## 8. Judge prompt template (verbatim, German — published on the site)

See `scripts/judge_prompt_v1.md`. Placeholders: `{task_statement}`, `{category}`,
`{capability}`, `{window_open}`, `{window_close}`, `{required_elements_leaves}`,
`{candidates_block}`, `{canary}`, `{task_id}`.

## 9. Output contract (per pass)

```json
{
  "task_id": "...",
  "scores":        {"K1": {"hop": 0.0, "final": 0.0, "evidence": 0.0,
                            "completeness": 0.0, "recency": 0.0, "honesty": 0.0}},
  "leaf_coverage": {"K1": {"covered": ["leaf_id"], "missed": ["leaf_id"]}},
  "spot_checks":   [{"candidate": "K1", "claim": "...", "url": "...",
                     "verdict": "supported|contradicted|unreachable|not_found"}],
  "hallucination": {"K1": false},
  "canary_hits":   [],
  "ranking":       ["K3", "K1", "K2", "K4"],
  "abstain":       false,
  "notes":         "max 600 chars"
}
```

The runner maps K-labels back to harness_ids, builds pairwise rows (§5), writes
`reviews.v2.json`, calls `compute_relative_scores.py` → `leaderboard.json` → site build.
