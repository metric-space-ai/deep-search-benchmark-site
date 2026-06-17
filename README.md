# DeepSearchBenchmark

**A harness-agnostic arena for live-web deep research.** Fix the task — vary the harness.

## → Live dashboard

### **https://metric-space-ai.github.io/deep-search-benchmark-site/**

Quality (Elo) vs. time and cost, the head-to-head matrix, and a per-task explorer
with the judge's reasoning. The dashboard always shows the latest published run.

---

## What it measures

100 fixed German deep-research tasks (six categories) are run **simultaneously**
by every harness in collect-only mode. There is no frozen ground truth and no
required-keyword scoring — quality is ranked *afterwards*, relatively, by a single
external LLM judge. The **same task** runs on each harness, so the differences
measured are the *harness* (its web search, fetching, multi-hop orchestration, and
honesty about failed sources), not a single fixed model.

## Contestants — harness / model

| Harness | Model |
|---|---|
| CTOX | MiniMax-M3 |
| OpenAI Codex CLI | GPT-5.5 |
| opencode | MiniMax-M3 |
| MiniMax Agent (Mini-Agent) | MiniMax-M3 |
| Claude Code | Opus 4.8 |
| Google Antigravity CLI | Gemini 3.5 Flash (High) |

All six are **real autonomous agent CLIs** driven headlessly with the same task and
the same I/O contract. Three of them run the same model (MiniMax-M3) inside different
agent harnesses — CTOX, opencode, and the official [MiniMax Mini-Agent](https://github.com/MiniMax-AI/Mini-Agent)
loop — which isolates the *harness* as the variable.

## How quality is judged

A **pure-prompt artifact judge** (Claude Opus). Candidates are anonymized
(K1…Kn); the judge sees **only their submitted artifacts** — the final answer, the
evidence/source chain, and the per-hop status — with **no web access and no tools**.
It runs every pairwise duel, twice with the positions swapped (swap-consistency is
reported), and the verdicts feed **Bradley–Terry → Elo** with bootstrap confidence
intervals.

Why pure-prompt and not an agentic judge: a judge with its own web access would
reward or penalize harnesses for the quality of *its* sources rather than theirs,
and a same-family judge carries bias. This judge ranks the evidence each harness
actually produced — nothing else. The exact prompt is published verbatim below.

## In this repo

- [`index.html`](index.html) — the dashboard (served live at the link above)
- [`docs/methodology.md`](docs/methodology.md) — full methodology
- [`scripts/judge_prompt_v2.md`](scripts/judge_prompt_v2.md) — the judge prompt, verbatim

The task corpus, benchmark engine and reference materials live in a **private**
repository. The contestants research the live web, so a public corpus could be
discovered by the very harnesses under test — only the dashboard, the methodology,
and the judge prompt are public, to protect benchmark integrity.
