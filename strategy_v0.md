# Hermes — Strategy v0 Spec (data-informed + notes-informed)

> Revised after reading your 2023–2025 trading notes (`notes_trading_strats.html`). The notes are 3,000 lines of mostly-sound mathematical reasoning that produced an elegant but unvalidated signal-generation apparatus. Your *actual* P&L came from the volatility-premium harvesting strategies (wheel + CC) you treated as sidelines. v0 follows the data, not the elegance. Parked ideas appendix at the bottom preserves the rest for future versions.

---

## 1. Edge hypothesis

**One-sentence claim:** We are *harvesting* (not predicting) the structural overpricing of implied volatility relative to realized volatility on liquid US equities, by systematically selling cash-secured puts at delta ≈ 0.30 with 30–45 DTE, and selling covered calls against any shares assigned.

**Why this framing matters:** This is a *harvesting* edge, not a *forecasting* edge. We are not predicting whether the underlying goes up or down. We are accepting a known small probability of large loss in exchange for a known high probability of small gain — and the premium we receive compensates for that asymmetry with a small structural overpay. This is the same edge an insurance company captures by underwriting policies it cannot predict will or won't claim.

**Why the inefficiency exists:**
- Institutional hedgers structurally bid up put implied vol to hedge portfolios
- Retail option buyers sustain demand for OTM lottery tickets
- The strategy is capital-intensive (cash-secured), which limits arbitrage capacity

**Why hasn't it been arbitraged away?**
- Capacity-limited: requires (strike × 100) capital per contract held
- Tail risk discourages full exploitation by large players
- Most retail participants chase forecasting strategies (your own notes are a case study) rather than structural harvesting

**Prior evidence in your own trading:** In 2025, 16 closed covered-call combos produced ~$8,721 at a 94% win rate; 27 wheel-pattern closed combos produced ~$11,877. These are your simplest strategies. By contrast, your randomized/filtered/scored signal-selection apparatus (developed across 2.5 years of notes) did not produce a comparable validated edge, and your stop-hedge inventions (Nov-Dec 2025) showed worse risk-adjusted returns and intractable failure modes documented in your own trade notes.

## 2. Universe

- **Asset class:** US-listed optionable equities only for v0 (no futures options, no index options, no 0-DTE)
- **Specific instruments (starting set, all validated in your 2025 data):** BAC, VZ, F, KO, GILD, INTC
- **Liquidity filters (canonized from your Nov 3, 2023 notes):**
  - Option open interest > 500 at chosen strike
  - Bid-ask spread < 5% of mid
  - Underlying market cap > $10B
  - Market depth check: at least 3 distinct bid levels within 2% of mid
- **Exclusions:**
  - Earnings within DTE window
  - Announced corporate actions (M&A, spinoff, special dividend) within DTE window
  - IV rank > 90 (likely event-driven, not volatility premium)
  - Minimum option premium < $0.29 (from your Nov 22, 2025 calculation: keeps transaction costs < 10% of total proceeds)
- **Watch list expansion:** add new tickers only after 8+ weeks of paper observation in the same volatility regime

## 3. Entry rules

- **Trigger condition:** sell-to-open short put when delta is between -0.25 and -0.35 (target -0.30) at chosen expiration
- **Timeframe:** evaluated on the daily close; orders entered the following morning during regular trading hours
- **DTE selection:** 30–45 days to expiration (your 2025 data shows this window dominated)
- **Confirmation:** IV rank between 30 and 70 (avoid both dead-low premium and event-driven extremes)
- **Cooldown:** no new put on same underlying within 7 days of a closed position
- **Soft filter (from your Oct 23, 2023 notes):** annualized ROI of the trade `r = (1 + P/L)^(365/n) − 1` must exceed current 3-month T-bill yield + 5%. This is the opportunity-cost filter from your own framework, slightly relaxed from your original "+11%" threshold to allow for the wheel's lower volatility profile.

## 4. Exit rules

**Strongly defaulted from your data — these are rules, not options.**

- **Take-profit (PRIMARY):** GTC LMT order attached on entry at 50% of credit received. Your 2025 trades show this is your single most disciplined automation pattern; v0 inherits it without exception.
- **Stop-loss on the option:** none. The position is cash-secured; assignment is part of the strategy, not a stop event. (Your Sept 2023 explorations of TRAIL LMT and bracket variants are *not* used for the option leg — see Parked Ideas appendix.)
- **Time stop:** if DTE ≤ 7 and position not yet closed at 50%, close at market on the next open regardless of P/L. Avoids gamma risk in the final week.
- **Assignment handling:** if assigned, immediately sell covered call at delta ≈ 0.30 with 30–45 DTE against the assigned shares (the second half of the wheel).
- **Never use STOP LOSS — only STOP LMT** if any stops are ever used. (Your June 28, 2023 lesson, paid for in real money.)

## 5. Position sizing

- **Sizing rule:** cash-secured. Each position requires (strike × 100 × number of contracts) in available cash. No margin for v0.
- **Default size for v0:** 1 contract per position
- **Maximum position size per underlying:** 5% of total account equity
- **Maximum total deployed capital:** 50% of account equity across all open positions for v0; raise to 75% only after 12+ weeks of clean operation
- **Position-level loss bound expressed as % of account, not absolute $** (from your Sept 24, 2023 lesson)

## 6. Risk limits

- **Maximum concurrent positions:** 6 (one per name in the validated universe)
- **Daily loss limit before halt:** -2% of account equity (closes new entries for the day; existing positions held per exit rules)
- **Drawdown threshold for full pause:** -8% drawdown from peak account equity → halt all new entries, review against falsification framework before resuming
- **Single-trade max loss bound:** position sizing physically prevents single-trade loss exceeding 5% of account. Your worst 2025 trade (-$4,742) was ~33% of your annual P/L on a single position; v0 cannot inherit that exposure profile.
- **Liquidity emergency rule:** if bid-ask spread on any open position widens beyond 3× normal during market hours, halt new entries until the regime clarifies. (From your Sept 20, 2023 Fed-meeting experience.)

## 7. Execution model

- **Broker:** Interactive Brokers (TWS / IBKR API)
- **Order type defaults:**
  - Entry: LMT at midpoint, willing to walk up to 5% of bid-ask spread; cancel if unfilled after 30 minutes
  - Profit-taker: GTC LMT attached at entry (NOT TRAIL LMT — see Parked Ideas)
- **Slippage assumption for backtests:** 5% of bid-ask spread on entry, 0% on profit-taker LMT
- **Commission model:** IBKR retail tiered — pull actual commissions from your IBKR statements rather than estimate
- **Order timing discipline (from your July 5, 2023 lesson):** no orders submitted within 15 minutes of market close; auto-cancel any unfilled orders at 15:45 ET
- **Order discipline (from your July 6, 2023 lesson):** any bracketed orders use One-Cancels-Other (OCO) tag

## 8. Falsification criteria

Calibrated against your simplest strategies (94% win rate covered calls, ~$440 avg combo P/L wheel), not your most complex.

- **Live performance threshold:** if 8-week paper-trade average combo P/L < $200 (vs. 2025 baseline of ~$440 wheel / $545 CC), shelve and review
- **Win rate threshold:** if CSP win rate (not counting assignments as losses) < 70% over 20+ trades, shelve. *(Note: this is set above your 2025 combo-win rate of 48%, because the 48% counts assignments as losses; pure CSP win rate is much higher.)*
- **Drawdown threshold:** if peak-to-trough equity drawdown > 12% during Phase 2 paper period, shelve regardless of P/L
- **Regime detector:** if VIX > 35 sustained for 5+ trading days, suspend new entries — wheel underperforms in true high-vol regimes, the failure mode least visible in your 2024-2025 data
- **Edge-source check (quarterly):** is realized P&L coming from delta exposure (i.e., we're directionally long via assignment) or from theta/IV-collapse (i.e., we're harvesting premium)? If > 70% of P&L is delta-driven over a quarter, the strategy is functionally long-only with extra commission drag — shelve and replace with simple buy-and-hold of the universe.

> **The trap to avoid:** when paper-trade performance is mediocre but "almost there," the temptation is to soften thresholds. Pre-commit now, in writing, that thresholds are evaluated mechanically. Your notes already document the related failure mode (complexifying a strategy to compensate for missing edge); the same psychology applies to softening falsification criteria.

## 9. Performance bar to go live (Phase 3 gate)

Paper-trade results must meet ALL of the following over 8+ weeks before real capital is deployed:

- Sharpe ratio (annualized): > 1.0
- Win rate on CSPs (not counting assignments): > 70%
- Average closed-combo P/L: > $200
- Maximum drawdown: < 8% of paper account
- Trade count: > 25 closed combos (small N = no decision)

## 10. Operational stack

- **Data source:** IBKR TWS for live data, Polygon historical for backtesting
- **Broker for paper / live:** IBKR (paper account first, live in same broker for consistent fills)
- **Runtime:** local cron on your Somerville machine for v0; migrate to EC2 only if uptime becomes a constraint
- **Storage:** SQLite for trade log, daily snapshot of portfolio state
- **Monitoring:** daily email to krish.kshir@gmail.com at market close with: positions open, today's P&L, orders triggered, errors. Weekly Saturday summary tied to the Hermes block.
- **Kill switch:** simple toggle file checked at the top of every order-generation cycle; if `~/.hermes/PAUSE` exists, no new orders generated. Phone access via SSH or a tiny mobile-friendly status page. Design before paper-trading, not before live.

## 11. Known failure modes

These are the lessons already encoded in your 2023–2025 notes and trade annotations. Re-learning them costs real money.

- **Assignment in a falling regime:** the wheel fails when you get assigned at strike and the underlying continues falling. Covered calls written against the underwater position generate trivial premium and lock you in to recovery. Bounded by universe filter (high-quality, dividend-paying names) but not eliminated.
- **Crowded volatility-selling regime:** when too much capital is short premium simultaneously, premiums compress and tail events amplify by short-covering. Feb 2018 "Volmageddon" is canonical. Detected by VIX regime check.
- **Market-maker gaming in thin contracts:** your Nov 3, 2023 lesson. Avoided entirely by liquidity floor in Section 2.
- **Cancel-before-close discipline:** your trade notes flag missed GTC cancellations causing unwanted fills near 4 PM ET. Encoded in Section 7.
- **Bid-ask gaming during open volatility:** your notes describe market-open volatility causing protective STOPs to trigger at terrible fills. v0 sidesteps entirely by not relying on STOPs; cash-secured + LMT-only fills.
- **Early assignment risk on deep ITM short options:** your June 28, 2023 GOOG put lesson. Mitigated by closing any position that goes > 5% ITM before expiration.
- **Strategy complexification under pressure:** the deepest failure mode in your notes. When a strategy isn't producing edge, the right response is to question the premise, not add layers. v0 commits to this discipline by *deferring* any modification to the spec until the next quarterly review point.

## 12. Validation plan

- **Backtest window:** Jan 1, 2018 — Dec 31, 2024 (spans Volmageddon, COVID crash, 2022 bear market, 2024 dollar-vol regime). **This backtest is now part of Phase 1, not optional.** Your notes don't show any historical validation pass on the strategies you deployed live — v0 corrects that.
- **Out-of-sample holdout:** Q4 2024 reserved for final validation — not touched during parameter tuning
- **Walk-forward design:** monthly re-evaluation of universe filters; fixed entry/exit rules
- **Paper-trade duration before live:** 8 weeks minimum; extend to 12 weeks if any single calendar month had < 4 closed combos
- **Backtest data check before Phase 1 starts:** confirm Polygon has option chain history for your universe back to 2018, or shrink window accordingly

---

## Sign-off checklist (end of Phase 0)

- [ ] Every section above reviewed and either accepted or modified — defaults are starting points, not gospel
- [ ] Falsification criteria personally committed to in writing (not just inherited from this template)
- [ ] You can explain the edge hypothesis (harvesting, not forecasting) to a competent friend in under 2 minutes
- [ ] You'd be willing to walk away from this strategy if the falsification criteria trigger — *honestly*
- [ ] Kill switch design specified concretely (file path, check frequency, manual override path)
- [ ] Polygon data availability confirmed for the backtest window

When all six are true, Phase 1 starts.

---

## Appendix A — Strategies parked for vN

Documented here so your 2.5 years of intellectual labor isn't lost, but kept out of v0 to prevent scope creep. Each parked strategy notes status and graduation criteria.

### A.1 — Randomized signal selection (Poisson-Cauchy framework, June 2023 onward)

**Status:** Parked indefinitely.
**Rationale:** Theoretically interesting but mathematically suspect. Malkiel's argument is "you can't pick winners better than the market, buy the index" — *not* "random entries with asymmetric payoffs produce positive expectancy." In a fair-value options market, random selection has expected return zero minus commissions. The 2.5 years of elaboration did not produce a validated edge.
**Graduation criteria:** demonstrate, with a formal backtest on 5+ years of historical data, that the random-selection framework with current filtering criteria produces Sharpe > 0.5 after realistic transaction costs. If achieved, revisit for vN.

### A.2 — Synthetic-short-straddle-via-STOP-orders ("new options trading strategy", Nov 21, 2025 onward)

**Status:** Parked. Has fundamental whipsaw exposure that complexification in Dec 22-25, 2025 notes does not resolve.
**Rationale:** The structure is mathematically a short straddle around K, achieved via order management rather than two short options. In stable markets it collects premium; in choppy markets each cross of K destroys premium through transaction costs. The Dec 22, 2025 "half the premium per hedge" modification adds complexity without addressing the underlying issue.
**Graduation criteria:** backtest demonstrates positive expectancy *after* transaction costs across at least three volatility regimes (low-vol stable, low-vol trending, high-vol). Until then, the wheel achieves the same exposure profile in a cleaner form.

### A.3 — Futures + TRAIL LMT exit strategy (Sept-Nov 2023)

**Status:** Parked. Promising framework but introduces leverage and 24/7 monitoring needs not appropriate for v0.
**Rationale:** Index futures have legitimate advantages (cash settlement, 60/40 tax, low commissions, leverage). But MTM volatility and the documented failures of TRAIL LMT (premature triggers in normal price oscillations, the Nov 1, 2023 0DTE lesson) make this higher-touch than v0 should be.
**Graduation criteria:** wheel is operating cleanly at scale; you have bandwidth for a higher-touch system; backtest shows TRAIL LMT parameters that don't trigger prematurely on at least 5 years of historical data.

### A.4 — Index options (SPX/XEO/NDX) instead of stock options

**Status:** Parked for v1, not eliminated.
**Rationale:** Your Aug 15, 2023 notes correctly identify real advantages — cash settlement, 60/40 tax treatment (significant for taxable accounts), European exercise removes assignment risk. The reason v0 uses stock options is your validated universe (BAC, VZ, F, KO, GILD, INTC) has empirical track record; SPX-equivalent strategy does not yet.
**Graduation criteria:** after v0 has 12 weeks of clean operation, run a parallel paper-trade test on XEO (S&P 100 European) wheel for 8 weeks. If P/L is competitive with stock-options wheel after tax adjustment, migrate.

### A.5 — Scoring formula (`100 × Σ measure / |breakeven − spot| + 10 × NetPremium − ...`, July 24, 2023)

**Status:** Useful framework, but not validated.
**Rationale:** Constructed heuristic, not empirically tested predictor. Cannot be used as a primary filter until it's validated.
**Graduation criteria:** if the wheel turns out to need a sub-ranking among multiple qualifying setups, backtest the scoring formula on historical wheel candidates and validate it correlates (r > 0.3) with realized P/L. Until then, "first qualifying trade per underlying per week" is sufficient v0 ranking.
