# DeepSearchBenchmark — Dashboard

Live dashboard of **DeepSearchBenchmark**, a harness-agnostic arena benchmark for
live-web deep research: the 100 tasks are fixed, the **harness** varies
(Google Antigravity CLI / Gemini 3.5 Flash, MiniMax CLI / MiniMax-M3, Claude Code / Opus 4.8,
CTOX / MiniMax-M3). All harnesses work the same tasks in collect-only mode;
quality is ranked afterwards by a pure-prompt artifact judge (Claude Opus,
anonymized candidates, two swap passes, NO web access and NO tools — it ranks
the submitted evidence chains only; prompt-tournament winner across 4 variants).
Pairwise verdicts feed Bradley-Terry → Elo with bootstrap CIs.

**Current status: judge v2 preview (27/100 tasks judged).** The dashboard shows collection telemetry
(contract rates, timings, sources) — *not* a quality ranking. Elo ±CI,
head-to-head matrix and per-dimension scores appear after the judge run.

- Dashboard: open `index.html` (or the GitHub Pages deployment)
- Judge prompt, published verbatim: [scripts/judge_prompt_v2.md](scripts/judge_prompt_v2.md)
- Methodology: [docs/methodology.md](docs/methodology.md)

The task corpus, engine and judge reference materials live in a **private**
repository to protect benchmark integrity (the contestants research the live
web — a public corpus could be found by the very harnesses under test).
