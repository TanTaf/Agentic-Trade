# Research Log (notes for writing the paper later)

Running record of decisions, problems found, and results. Every number here must come
from a file under `artifacts/study/`; copy the path next to the number.

## 2026-10-29 — Audit of the previous version

Problems found in the earlier (5-week) study that a reviewer would reject:

1. `fin_mem` was *not* FinMem: it sorted by date only, identical to `plain_rag`. No
   recency/relevancy/importance score, no layer decay, no access counter, no promotion.
2. Data: 312 headlines over 5 weeks, ~17 test days, `p = 0.50`.
3. No 10-Q / 10-K, so intermediate/deep layers never fired.
4. Decisions came from a keyword lexicon; no transaction costs; Hold = flat (FinMem: Hold = keep).
5. LLM client silently failed over between models, so a "backbone" result could mix models.

## New study design (src/agentic_trade/study/)

- **Data** (`scripts/build_dataset.py`): Alpaca News (Benzinga, same source as FinMem)
  2025-09-01..2026-10-27, 23,247 unique articles; SEC EDGAR XBRL 10-Q/10-K for all 10
  tickers since 2024-07 (93 filings, rendered as numeric "insight" text with YoY growth);
  Alpaca SIP daily bars (split/dividend adjusted), 10 stocks + SPY + QQQ.
- **Point-in-time rule**: item visible at the close of the first trading day whose 16:00 ET
  close is >= its timestamp (after-hours news -> next day). Filings: available after the
  filing day's close. Decision at close t earns close(t)->close(t+1).
- **Shared annotation**: FinBERT (ProsusAI/finbert) polarity p+ − p− on headlines;
  all-MiniLM-L6-v2 embeddings on headline+summary. Same for every system, so differences
  come from the memory architecture only. (FinMem used GPT summaries + ada-002 embeddings;
  we substitute open models for reproducibility at zero cost.)
- **Splits**: warm-up Sep 2025; train Oct 2025–Feb 2026; validation Mar–Apr 2026
  (only decision threshold / ML margin selected here); test May 1 – Oct 26 2026.
- **Faithful FinMem**: Q = 14/90/365; importance v∈{40,60,80} with FinMem's layer
  probabilities, decayed by α = 0.9/0.967/0.988; purge at recency<0.05 or importance<5;
  access counter +5 for pivotal memories and promotion (60 -> intermediate, 80 -> deep)
  with recency reset; top-K = 5 per layer; self-adaptive character in LLM prompts.
- **Ours** = FinMem + N1..N7 (see `study/memory.py` docstring):
  N1 hierarchical Beta-Bernoulli source credibility (author->category->global), updated
  online from market-adjusted outcomes; N2 near-duplicate consolidation (corroboration);
  N3 semantic contradiction retirement; N4 situational query; N5 data-driven importance
  (attention surprise × classifier confidence) replacing FinMem's random draw;
  N6 point-in-time co-mention peer graph; N7 outcome-aware episodic recall / abstention.
- **Policies**: (a) shared rule policy on the retrieved evidence (threshold tuned per system
  on validation); (b) LightGBM / LogReg on price features ± memory features;
  (c) LLM backbones (free: gpt-oss-120b, gpt-oss-20b, qwen3.8-27b on Groq; nemotron-3-ultra-550b on
  OpenRouter; gemma-4 was rate-limited upstream) on the 5 FinMem tickers, last 50 test days,
  temperature 0, single trial per backbone (free-tier quota).
- **Costs**: 5 bps per unit position change. Long/short, FinMem action semantics.
- **Stats**: Wilcoxon signed-rank on pooled (ticker, date) daily returns (Holm-corrected),
  paired t, Diebold–Mariano, moving-block bootstrap CI for ΔSharpe, Spearman IC.

## Results (run 2026-10-29, `scripts/run_study.py --fresh`)

Source files: `artifacts/study/{rolling,fixed}/summary.json`, `table_main.md`,
`table_ablation.md`; figures in `paper/figures/fig01..fig19` (render: `render_all`).

### Primary protocol: rolling walk-forward, 2025-12-01..2026-10-26 (227 days × 10 stocks)

| System | Sharpe | CR | MDD | FinMem-5 Sharpe / CR |
|---|---:|---:|---:|---:|
| Buy & Hold | 0.61 | +11.2% | −16.5% | – / −11.6% |
| LGBM (price) | 1.15 | +19.4% | −17.2% | |
| Plain RAG | −0.65 | −9.5% | −21.1% | −1.59 / −28.8% |
| FinMem (faithful) | −0.72 | −11.0% | −19.7% | −1.59 / −28.6% |
| **Ours** | **−0.20** | **−3.6%** | **−15.2%** | **+0.28 / +3.5%** |
| Ours (signed N1, post-hoc) | −1.24 | −16.8% | −23.9% | −0.88 / −16.7% |

- Ours vs FinMem: +3.45 bps/day, ΔSharpe +0.52, block-bootstrap 95% CI [−2.73, +3.12],
  Wilcoxon p = 0.39 (Holm 1.0) → **directionally better, not significant**.
- Ours vs Daily-News-Sentiment (no memory): Wilcoxon Holm p = 0.001 (memory clearly beats
  reacting to the day's headlines).
- IC of memory signal vs next-day market-adjusted return: Plain RAG −0.037 (p 0.08),
  FinMem −0.000, Ours +0.003. → News-memory carries ~zero next-day directional info in
  2025-26; this is the main honest finding about the FinMem premise.
- Ablation (rolling, ΔSharpe when removed): N4 situational −0.70, N6 peers −0.32,
  N5 importance −0.28, N1 credibility −0.03; removing N3 (+0.33), N2 (+0.22), N7 (+0.10)
  *improved* Sharpe — i.e. those three do not help the rule policy (still useful for LLM
  context: redundancy/age diagnostics, fig13).
- Top-K is unstable (FinMem K=1: Sharpe 2.27; K=3: 0.11; K=10: −0.38) → sensitivity
  to K exceeds architecture differences; report as caveat.
- Signed N1 (post-hoc): better in fixed test (0.07 vs −2.21), worse in rolling →
  does not generalise; report as negative result.

### Secondary: fixed split, test 2026-05-01..2026-10-26 (123 days)
Ours −2.21, FinMem 0.67 (θ=0.4 ⇒ FinMem ≈ long-only from positive 10-Q scores),
Plain RAG −1.34, B&H 0.68, Mean-Reversion 1.80. Single-regime protocol is fragile:
validation IC ranking (Ours 0.087 > FinMem 0.065 > Plain −0.006) flipped in test.

### Retrieval diagnostics (fig13)
Mean memory age: Plain ≈ 1 d, FinMem ≈ 57 d, Ours ≈ 22 d. Ours merged 2,260 duplicate
stories and retired 667 contradicted memories; FinMem promoted 1,221 memories.

### Story for the paper
1. Faithful re-implementation shows FinMem's layered memory does not transfer to a
   2025-26, ten-stock, cost-aware, point-in-time evaluation (Sharpe −0.72).
2. Our memory is the best memory architecture under the primary protocol and on
   FinMem's own tickers, with smaller drawdown, but the gain is not significant.
3. Price-only LightGBM and B&H beat all news-memory rule policies → contribution should
   be framed as evaluation rigor + memory design + LLM-context quality (LLM experiment
   pending), not as "alpha".

## Application (dashboard) built on the study — 2026-10-29

- `src/agentic_trade/agent_service.py`: live service running Plain RAG / FinMem / Ours on the
  newest data (`artifacts/app/`, falls back to `artifacts/study/`). Per-ticker brief, ten-stock
  scan, LLM decision with the backbone-experiment prompt, incremental refresh
  (`study.data.refresh_dataset`: bars + filings re-download, news since last article − 3 d,
  FinBERT/MiniLM on new rows only), Research Lab payload.
- Consensus rule (motivated by the finding that no single signal is reliable): memory action
  and price-LightGBM vote (price votes only if |P(up) − 0.5| ≥ 0.03); disagreement → Hold;
  episodic abstention → Hold; confidence capped at 0.6.
- Dashboard tabs: Memory Agent, Scanner, Research Lab replace the obsolete 5-week
  Performance/Memory/Benchmarks tabs. `/api/ai_trade` and the auto-agent now use this agent.

## Additional findings while writing the report — 2026-10-29

- Ours abstains (N7) on 49% of ticker-days; Hold keeps the position, so memory policies are
  never flat. Short share: Ours 58%, Plain RAG 43%, FinMem 29%. Portfolio beta: Ours −0.33,
  FinMem +0.64 → in a rising market this explains much of the gap to Buy & Hold.
- Removing N7 raises MDD from −15.2% to −25.1%; removing N1 to −18.6% → N1/N7 act as risk
  controls (future work: use memory as a risk overlay on a price model).
- Adding memory features to LightGBM lowers Sharpe (1.15 → 0.42/0.44).
- Costs: Ours stays the best memory at 0–30 bps; FinMem overtakes Plain RAG above ~8 bps.
- Report: `paper/uiu_template/` (all tables generated by `scripts/export_latex_tables.py`).

## LLM backbone run — status 2026-10-29

- Groq free tier: 200k tokens/day per model per account (~115 decisions/day/model).
  Backbone client now pools several accounts (GROOQ_KEY, GROOQ_KEY2, ...), paces each key
  and parks a key on 429 until its retry-after; rate limits never become a recorded Hold.
- OpenRouter dropped: free account => 50 free-model requests/day shared by all free models
  (nemotron-3-ultra stopped at 51/750); xing4.0-29b was uncapped but throttled upstream to
  ~30 req/h (~24 h for 750) and was stopped by decision. Final LLM table = 3 Groq backbones.

## LLM backbone results — final (2026-10-29)

Source: `artifacts/study/llm/llm_decisions.parquet` (2,250 decisions, 0 failures), table
`paper/uiu_template/tables/llm.tex`, figure `fig20_llm_backbones`. Window 2026-08-17..10-26,
FinMem's 5 stocks, temperature 0, single trial, 4 Groq accounts in a key pool.

| Backbone | No-Memory CR | FinMem CR / MDD | Ours CR / MDD |
|---|---:|---:|---:|
| gpt-oss-120b | −3.61% | −12.00% / −14.7% | +2.55% / −6.3% |
| gpt-oss-20b | −6.59% | −10.70% / −10.7% | +1.33% / −4.8% |
| qwen3.8-27b | +3.48% | −4.79% / −10.7% | −1.34% / −5.8% |

- Ours > FinMem for every backbone; pooled +20.9 bps/day, Wilcoxon p = 0.123 (n.s.).
- Ours vs No-Memory: +6.1 bps/day, p = 0.508 (n.s., qwen best without memory).
- Buy & Hold over the window +3.89%. Backbones agree on the action ~1/3 of the time.
- Ours context → more Hold for gpt-oss-20b (73% vs 61%) and qwen (92% vs 70%); gpt-oss-120b
  holds equally (45% vs 46%) yet still gains → not only an effect of trading less.
- Bug found & fixed during the run: concurrent pandas MultiIndex `.loc` from threads is not
  thread-safe (one chain crashed); lookups now use a prebuilt dict.

## Extension experiments and report completion — 2026-10-29

Source: `src/agentic_trade/study/extensions.py` (`scripts/run_extensions.py`, 531 s on CPU),
results `artifacts/study/rolling/extensions_{daily.parquet,summary.json,logs.json}`, figures
fig21–fig32 (`scripts/make_extension_figures.py`), tables `tables/ext_*.tex`
(`scripts/export_extension_tables.py`). Same rolling protocol, test window, monthly refit,
margin 0.02 and 5 bps costs as LightGBM; settings fixed before the first run.

| Model (10 stocks, 227 days) | CR | Sharpe | MDD | beta | alpha/yr |
|---|---:|---:|---:|---:|---:|
| Buy & Hold | +11.2% | 0.61 | −16.5% | 1.62 | −10.8% |
| LGBM (price) | +19.4% | 1.15 | −17.2% | −0.11 | +23.2% |
| GRU (price) | +3.0% | 0.28 | −11.8% | 0.21 | +1.3% |
| Transformer (price) | +11.6% | 0.95 | −7.9% | −0.12 | +14.9% |
| Transformer (price+memory) | +0.2% | 0.09 | −18.0% | 0.07 | +0.5% |
| RL Q-learning (long/flat) | +4.4% | 0.39 | −14.7% | 0.84 | −7.2% |
| Pattern analogs (kNN 20/10/8) | +17.7% | 1.45 | −6.4% | 0.49 | +11.2% |
| Ours | −3.6% | −0.20 | −15.2% | −0.33 | +2.3% |
| FinMem | −11.0% | −0.72 | −19.7% | 0.64 | −21.7% |
| *LGBM + memory risk overlay (post-hoc)* | +10.4% | 1.03 | −11.6% | 0.06 | +10.6% |
| *Model vote (post-hoc)* | +14.9% | 1.09 | −14.9% | −0.29 | +21.2% |

- No difference significant (all bootstrap CIs contain 0; smallest Wilcoxon p = 0.085, post-hoc vote vs Ours).
- Memory features hurt the Transformer (0.95 → 0.09) as they hurt LightGBM (1.15 → 0.44).
- Deep nets: best validation BCE 0.683/0.689/0.693 ≈ ln 2 → almost no learnable next-day signal;
  attention nearly uniform (last 5 days 25.6% vs first 5 days 24.8%).
- RL: long on 54% of stock-days, beta 0.84 → mostly follows the market; beats B&H on AAPL, AMD, MSFT, COIN.
- Pattern analogs are fragile: on a common test period only the 20-day pattern with 5/10-day
  look-ahead is positive (mean Sharpe +0.25/+0.50 vs B&H +0.35); 6 of 9 cells negative; the app
  default happens to be the best cell. Report as a multiple-testing caveat, not a finding.
- Allocation (walk-forward, monthly): equal weight 0.77, risk parity 0.75, min variance 0.51;
  80% SPY 1.26 (the ten stocks are highly correlated).
- Robustness: without TSLA Ours −0.20 → +0.23, FinMem −0.72 → −0.43, ranking unchanged;
  Ours beats FinMem in 9/10 leave-one-stock-out cases (exception: leaving out COIN).
- Regimes: B&H +100/−95 bps on SPY up/down days; Ours −17/+16; Transformer −8/+19.
- Power: Ours − FinMem = +3.4 bps/day, sd 133.8 bps → MDE at n=227 is 24.9 bps/day;
  ≈11,800 trading days (≈47 years) needed for 80% power. Waiting for a stock to recover
  cannot make the comparison significant and would bias the test window — decided against.

App changes in the same session: algorithm board now lists the extension models; Research Lab
has a model-zoo card; fixed duplicate element ids (Insights RL vs Research Lab) that blanked the
Research Lab equity chart; screenshots regenerated with `scripts/capture_app_screens.py`.
