# Trading Agent Idea — Research & Discussion Log

**Date:** 2026-09-05
**Status:** Exploration only — no code written yet. To be continued in a future session.

---

## 1. Original idea (session start)

An AI agent that watches Binance (crypto) and Charles Schwab (US equities/options) 24/7, performs Fundamental + Technical + Sentimental analysis, and **autonomously decides when to enter and exit trades** for large, consistent profit — swing-trading timeframe (days–weeks).

## 2. Research findings (targeted web search, not full deep-research pass — that was judged overkill mid-session and stopped)

### Prior art — this has already been built as research
- **[TradingAgents](https://arxiv.org/abs/2412.20138)** (Dec 2024, code on [GitHub](https://github.com/tauricresearch/tradingagents)) — near-exact match: multi-agent LLM framework with dedicated fundamental/sentiment/technical analyst agents, bull/bear researchers, risk management layer. Reports improved backtested returns/Sharpe/drawdown vs. baselines. **Read this before designing anything from scratch.**
- **FinRL / FinRL-Meta** — standard academic RL-trading library. Its own authors flag low signal-to-noise ratio, survivorship bias, and rampant backtest overfitting as core known problems.
- **FinGPT / FinLlama / sentiment-trading LLM papers** — LLM sentiment extraction does correlate with next-day returns (~72–74% classification accuracy, FinBERT/FinGPT-class), but single-paper backtest wins are notoriously non-reproducible out of sample.
- **Freqtrade / Hummingbot / QuantConnect-LEAN** — real open-source infra retail users run live. Profitability evidence is thin and self-reported.

### Technical feasibility per venue
- **Binance:** mature, bot-friendly API, generous rate limits (300k req/5min as of Apr 2026). Easy half.
- **Charles Schwab:** TD Ameritrade API shut off May 2024; Schwab's own Trader API is live (~120 req/min data, 2-4 trade req/sec) but **"automated/AI-driven order placement" triggers a commercial-approval review** — a fully autonomous agent is exactly the case that review exists to scrutinize. Ecosystem has largely migrated to **[Alpaca](https://alpaca.markets/algotrading)** instead for US equities/options automation (API-first, commission-free, official SDKs, paper trading built in).
- **Regulatory:** SEC eliminated the Pattern Day Trader $25k rule in April 2026, replaced with a proportional-equity model + mandatory broker-side algorithmic circuit breakers. Helps swing-trading plans; confirms this space is actively regulated and moving.
- Swing-trading timeframe (days–weeks) means rate limits are a non-issue for either venue — this isn't HFT.

### Three analysis pillars — realistic state of the art for a solo builder
- **Technical:** fully commoditized (indicators, backtesting engines all solved).
- **Fundamental:** solid data exists for equities; crypto has no true fundamentals equivalent.
- **Sentiment:** LLM-based extraction beats pre-LLM tools but is a weak, noisy signal, not an oracle.
- Known pitfalls across all three: look-ahead bias, overfitting to backtest history, non-stationarity (regime shifts silently break a tuned strategy).

### Does it actually work / is it worth it
- **UC Berkeley research:** retail trading bots lose **77x more per user than human traders** on average; an estimated **95% of retail "AI" bots are rule-based scripts with AI marketing labels**, not real learned models.
- Roughly **70–80%+ of retail bot users lose money**; consistent with 74–89% of retail traders overall losing money, bot or not.
- On Polymarket, winning bots succeeded via **execution-speed advantage**, not superior prediction — a specific, checkable finding.

### Risks flagged
- **Technical:** runaway-bug risk (Knight Capital, 2012, $440M lost in 45 minutes from a deployment bug — institutional example, same mechanism risk at any scale).
- **Operational:** 24/7 uptime, API/exchange outages.
- **Regulatory:** market manipulation rules apply regardless of account size; Schwab's automated-order-placement approval gate is real.
- **Security:** Binance API keys are bearer tokens to funds — scope to trade-only (no withdrawal), IP allowlist.

### Original recommendation (staged path, before the idea changed)
1. Read TradingAgents paper/code first.
2. Start with Binance + Alpaca (not Schwab) for build/paper-trading.
3. Build technical-analysis-only first, walk-forward validate (not single in-sample backtest).
4. Add sentiment, then fundamentals, each measured for actual incremental value.
5. Paper trade for months across a real regime change before real capital.
6. Start real capital small, with hard circuit breakers enforced outside the agent's own decision loop.

---

## 3. Idea pivot #1 — alert-only, no autonomous execution

User's reasoning: 70%+ of bots lose money; manual trading causes constant phone-checking (focus/attention difficulty). New idea: agent continuously runs F+T+S analysis; only **alerts** when all three pillars strongly align on a buy/sell opportunity; user executes manually. No autonomous order placement.

**Assessment:**
- **Real risk reduction:** removes autonomous-execution risk (no Knight-Capital-style runaway bug), removes trade-permission API key exposure, mostly sidesteps Schwab's automated-order-placement compliance review since no orders are placed by the agent.
- **Does NOT fix the core problem:** signal quality. A wrong composite signal is just as wrong whether a bot executes on it or a human does after a push notification. Switching to alerts moves the trust decision from "the bot trades" to "I trust the alert enough to act" — doesn't make the underlying F+T+S composite any more predictive.
- **Pseudo-independence flaw:** "three pillars agree" can be one underlying market event observed three times (price breaks resistance because good news hit, sentiment score also reflects that same news, fundamentals confirm the same catalyst) rather than three independent confirmations. Documented research: correlated indicators weaken the value of "multiple confirmation" systems — need to explicitly check the correlation structure between pillars, not assume agreement = strength.
- **Alert-threshold dilemma:** loose thresholds → frequent alerts → back to constant phone-checking, just via push notification instead of choice. Tight thresholds → rare alerts, good for "signal not noise," but then still need to backtest whether that rare-alignment event actually predicts a profitable move.
- **New risk introduced — automation bias:** framing something as "reliable, not noise" makes the user trust it *more* than manual judgment (that's the intended effect). If it's only moderately reliable, danger is taking bigger/more confident positions on its alerts than gut instinct would justify.
- **Existing products already do this** — worth trying before building: Tickeron, OVTLYR, Stock-Alert.AI (equities, combine technical+fundamental+sentiment into confidence-scored alerts); LunarCrush (crypto social sentiment, "AltRank"), Santiment (crypto on-chain + social sentiment, publishes ~64% directional accuracy backtest on BTC over 30-day windows — a concrete, checkable number).
- **The one real precondition before building:** define a concrete, backtestable rule for "all three pillars agree" and show it beats a naive baseline (buy-and-hold, or single-pillar strategy), net of the fact that execution happens some time after the alert fires (not instantly) — if this can't be stated precisely enough to backtest, "alert me when it's worth it" isn't a system yet, it's a wish.

---

## 4. Idea pivot #2 — market-wide scanner + Telegram alerts (two sub-systems)

User's refinement: "my eyes constantly on the whole market" — scan broadly (not just a watchlist) for F+T+S alignment, and separately, alert on Telegram once an existing position hits a profit target (e.g. $200/$500/$1000).

**Assessment — these are two different systems, differing sharply in difficulty:**

### A. Exit/profit-target alert — easy, solved problem
- Position tracking + price feed + threshold check + Telegram bot message (Telegram Bot API is free, trivial to integrate).
- Binance and most brokerages already ship native price/P&L alerts — check what's already available before building custom.
- **Gap identified:** user only specified profit-target (upside) alerts, not symmetric loss-side/stop-loss alerts. This is backwards from a risk-management standpoint — loss aversion means people avoid checking losers and hope they recover, exactly the failure mode an alert system should guard against. Build both thresholds symmetrically from day one.

### B. Market-wide entry scanner — inherits the unresolved signal-quality question AND adds a new one
- **Data snooping / multiple hypothesis testing problem:** scanning thousands of tickers daily for a rare "all three pillars align" event guarantees false-positive matches purely by chance, even with zero real underlying signal. Documented finance literature ([Sullivan, Timmermann, White — "A Reality Check for Data Snooping"](https://www.researchgate.net/publication/4896389_A_Reality_Check_for_Data_Snooping)): **at least 50% of reported positive trading-rule backtest results may be false discoveries** when multiple hypotheses/assets are tested against the same data without correction (Bonferroni, FDR control, White's Reality Check, Hansen's SPA test are the standard corrections).
- Scanning the *whole* market (vs. a small watchlist) doesn't add edge — it raises the noise floor, since more tickers scanned = more chances for random alignment.
- **Architecture recommendation:** staged funnel, not uniform full-analysis coverage. Cheap technical screen first (trivial at scale) narrows to a shortlist; only then apply expensive sentiment/fundamental analysis to that shortlist. Cheaper to run AND reduces data-snooping exposure (testing the rare "all three agree" hypothesis against a pre-filtered set, not the entire market).
- **This is literally the existing "stock screener" product category** (Finviz, Trade Ideas, Tickeron, OVTLYR) — "my own eyes on the whole market, automated" is not a new invention, it's a screener + Telegram-forwarding wrapper. Worth using one of these for a few weeks before building, to learn exactly where they fall short.

### Net assessment of pivot #2
- Exit-alert half: easy, real, low-risk — build this first, and make it symmetric (profit + loss).
- Entry-scanner half: same unresolved core question as pivot #1 (does F+T+S alignment have genuine edge?), plus a new statistical risk (data snooping from widening scope to the whole market) that a smaller, hand-picked watchlist would avoid.

---

## 5. Open question for next session

Before building anything: can a concrete, backtestable rule for "F+T+S pillars align" be defined and shown to beat a naive baseline, net of realistic execution delay and corrected for multiple-comparisons risk if scanning many tickers? This is the load-bearing question the whole idea rests on — everything else (venue choice, alert delivery, exit thresholds) is comparatively easy engineering once this is answered.
