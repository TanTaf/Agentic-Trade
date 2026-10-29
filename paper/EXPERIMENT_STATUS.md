> **Superseded.** This file describes the early five-week TSLA/5-ticker prototype (June–July 2026).
> The final study (one year, ten stocks, 93 tests) is documented in `uiu_template/` (the FYDP report) and `RESEARCH_LOG.md`.

# Paper Experiment Status

## Completed

- **Offline Baseline**: TSLA single-trial run with No Memory, Plain RAG, FinMem, Ours, and Buy-and-Hold.
- **Leakage Protection**: Time-aware event ingestion and future-event leakage tests verified.
- **Evaluation Metrics**: Cumulative return, Sharpe ratio, annualized volatility, maximum drawdown, and hold rate.
- **Evidence Logging**: Full evidence logs tracking date, memory ID, layer, source, text, topic, reliability, active flag, rank, score, cited flag, decision, and realized return.
- **Pre-registered Case Study Cards**: Deterministic case-study card selection (`artifacts/research/matrix/case_study_cards.md`) with trial deduplication and evidence retrieval.
- **Risk Profiles (Table 4 & Figure 8)**: Conservative, Standard, and Aggressive runs with CSV export and cumulative return visualization.
- **Multi-ticker Matrix (Table 2)**: 5-ticker (TSLA, NFLX, AMZN, MSFT, COIN) single deterministic run with date-pooled paired Wilcoxon test (`artifacts/research/matrix/table2.md`, `wilcoxon.json`). The pooled difference vs. the strongest baseline is not significant (`p=0.50`).
- **Ablation Studies (Table 6)**: Component ablations (w/o Learned Provenance, w/o Contradiction Retirement, w/o Decision Recall, w/o Topic Matching) with consolidated reporting (`artifacts/research/matrix/table6_ablation.md`).
- **Probe Retrieval Benchmark (Table 7)**: 17-query pre-registered, ticker-scoped probe evaluation (`agentic_trade.probes`) evaluating entity/topic recall across all five tickers (`artifacts/research/matrix/table7_probes.md`). Cross-ticker keyword contamination is filtered out.
- **Paper Trading Execution Adapter**: Full paper trading module (`agentic-paper`) with capped Buy allocation, long position close on Sell, short support (`--allow-short`), portfolio reconciliation (`--reconcile`), and JSONL audit trail.
- **Groq + Alpaca Live Path**: `agentic-live` generates decisions with a Groq model, persists resolved decision history, supports differentiated risk profiles (`aggressive`, `standard`, `conservative`) to prevent all-`Hold` collapse, and optionally executes on Alpaca paper. Live-money orders are blocked unless `--live` and `ALPACA_LIVE_CONFIRM=YES` are both set.
- **Interactive Research & Trading Dashboard**: Zero-dependency dark-theme web dashboard (`agentic-dashboard` / `./run_dashboard.sh`) providing interactive multi-line SVG equity curves, memory retrieval auditor, research tables (Tables 2, 4, 6, 7 & Wilcoxon), Alpaca order logs, and runner console.
- **Configuration Management**: Declarative YAML configurations under `configs/` (`offline_tsla.yaml`, `multi_ticker.yaml`, `risk_conservative.yaml`, `risk_aggressive.yaml`).
- **Test Suite**: 36 automated unit and integration tests covering memory policies, leakage prevention, relationships, probe isolation, Groq decision flow, live opt-in guards, order execution, reconciliation, case study deduplication, dashboard API loaders, and LLM risk profile prompt differentiation.

## Known Limitations (Read Before Citing)

- The checked-in news snapshot covers `2026-06-01..2026-07-09`. The default split is train `2026-06-01..2026-06-15`, test `2026-06-16..2026-07-09`.
- All events are `source_type=news`; there are no 10-Q/10-K rows, so `fin_mem` is identical to `plain_rag` and intermediate/deep layers are untested.
- Provenance uses publisher attribution when present, otherwise a source-genre fallback. Only `media` and `analyst_note` have enough observations to learn reliability on this snapshot.
- Deterministic runs are single-trial. `run_matrix` collapses repeated deterministic trials to one and the Wilcoxon test averages by `(ticker, date)`.
- Table 6: provenance changes returns; contradiction retirement and decision recall are exercised (12 retired facts, 15 recalled decisions) but do not change deterministic PnL. Decision recall only affects LLM prompts.
- Trading performance is mixed and **not statistically significant** in aggregate (`p=0.50` vs. the strongest baseline). Do not claim a universal improvement.

## External API Dependencies (Pending Live Credentials)

- **Table 3 LLM Backbone Comparison**: Requires live API keys (`GROQ_API_KEY`, `OPEN_ROUTER_KEY`, or `OPENAI_API_KEY`) to run multi-trial evaluations against commercial LLMs (e.g., Llama 3, GPT models).
- **Extended News Window Refresh**: Script `scripts/fetch_real_data.py` includes pagination for Alpaca News API, pending live `ALPACA_KEY` / `ALPACA_SECRET`. It now also stores the publisher `source_name`, which enables publisher-level provenance.
- **Live Paper Execution**: Alpaca paper order submission (`--execute`) requires `ALPACA_PAPER_KEY` / `ALPACA_PAPER_SECRET`.
- **Groq Decisions**: `agentic-live` requires `GROQ_API_KEY` (or `GROQ_KEY`). Live-money execution is additionally guarded by `ALPACA_LIVE_CONFIRM=YES`.

## Honesty Rules

Do not fill missing tables with mock values. Report unavailable runs explicitly. Deterministic backbones are single-trial; do not present repeated identical runs as independent observations. Pool paired observations by `(ticker, date)` and report the sign of the effect, not just significance. Freeze risk mode and top-K before evaluating on the test window.
