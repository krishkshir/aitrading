# Hermes — Strategy v0 Spec (barbell revision)

> **Revision rationale.** The prior spec described pure short-premium harvesting (CSPs + covered calls). This is structurally concave — the canonical "pennies in front of a steamroller" payoff that Taleb argues against, regardless of positive expected value. This revision restructures Hermes as a **barbell**: short-premium harvesting on one side, long-vol tail hedging on the other, with the latter funded by a fraction of the premium harvested by the former. The combined exposure is intended to have positive or near-zero skew, not the deeply negative skew of unhedged short vol.
>
> The expected return is lower than the unhedged version. That is the deliberate trade. The asymmetry being purchased is *survival of regime change*, which is the only thing that distinguishes a strategy that compounds for 20 years from a strategy that compounds for 8 years and then dies in a single month.

---

## 1. Edge hypothesis

**One-sentence claim:** We harvest the volatility risk premium on liquid US equities through cash-secured short puts at delta ≈ 0.30 with 30–45 DTE, and we deliberately reinvest 18–22% of harvested premium into long out-of-the-money index puts and VIX calls, producing a barbell payoff: bounded participation in benign regimes, convex protection in crisis regimes.

**Why this framing matters.** We are not "selling insurance" in isolation. We are *running an insurance book that also buys reinsurance*. The premium gap between what we collect (single-name short puts, where retail and institutional hedging inflate IV) and what we pay (index/VIX long vol, where institutional supply is deeper and the premium is smaller) is our edge. This gap is small in absolute terms but structurally robust; it does not require us to be right about market direction, and it does not depend on the steamroller never arriving.

**Why the inefficiency exists:**
- Single-name IV on quality large-caps is structurally bid up by institutional hedgers and retail OTM lottery-ticket buyers
- Index/VIX long vol is supplied by a larger pool of vol sellers (systematic strategies, market makers, retail) which compresses its premium relative to single-name short vol
- The premium *differential* between paid and received vol is the harvestable spread; the absolute level of either is not the edge

**Why hasn't it been arbitraged away?**
- Operationally complex: requires running two coordinated books with opposite vol exposure
- Psychologically difficult: the long-vol leg looks like wasted money for years at a time
- Capital-intensive on the short leg (cash-secured)
- Most retail vol traders run pure short vol; most institutional tail-risk funds run pure long vol; few do both

**Prior evidence:** Your 2025 single-leg trading produced ~$14k on the harvest side (94% win rate on covered calls, ~$440 avg combo P/L on wheel). The harvest edge is empirically validated in your own data. The hedge leg is new in v0 — its purpose is to reshape the distribution of returns, not to add expected return.

---

## 2. Structure overview

Hermes runs as a *two-leg book* with explicit coordination rules:

**Leg 1 — Premium harvest (short vol, concave):**
- Cash-secured short puts at delta ≈ 0.30, 30–45 DTE, on validated single-name universe
- If assigned, sell covered calls at delta ≈ 0.30 against assigned shares (the wheel)
- Generates monthly cash inflow

**Leg 2 — Tail hedge (long vol, convex):**
- Long SPY puts: 60–120 DTE, delta ≈ 0.08–0.10, far OTM
- Long VIX calls: 60–90 DTE, strike 25–30, very small allocation
- Generates expected loss in benign months; expected large gain in crisis months
- Funded by ~18–22% of harvested premium from Leg 1

**Coordination:** the two legs are *not* hedges of specific positions — they are exposures of opposite convexity in the same portfolio. The wheel is not delta-hedged. The hedge is not position-by-position. The combined book is structured for distributional shape, not position-level neutrality.

---

## 3. Universe

**Leg 1 (single-name short premium):**
- US-listed optionable equities, Penny Interval Program members
- Starting set: BAC, VZ, F, KO, GILD, INTC
- Liquidity filters: option open interest > 500 at chosen strike; bid-ask spread < 5% of mid; underlying market cap > $10B
- Exclusions: earnings within DTE window; corporate actions within DTE window; IV rank > 90; option premium < $0.29

**Leg 2 (index/vol long premium):**
- SPY: front-month-plus for long puts (most liquid US equity option chain by orders of magnitude)
- VIX: standard VIX options (cash-settled European)
- No single-name long options for v0 (basis risk too noisy; index hedge is cleaner)

---

## 4. Entry rules

**Leg 1 — Short put entry:**
- Sell-to-open short put when delta is between -0.25 and -0.35 (target -0.30)
- DTE 30–45 days
- IV rank between 30 and 70
- Annualized ROI > T-bill yield + 5%
- Cooldown: no new put on same underlying within 7 days of a closed position

**Leg 2 — Long-vol entry (initial sizing):**
- SPY puts: open a position whenever portfolio long-put notional protection drops below the sizing target (see Section 6). Buy 60–120 DTE puts at delta ≈ 0.08–0.10 (typically 8–12% OTM). One position per month if needed for replenishment.
- VIX calls: maintain a continuously rolled position. Buy 60–90 DTE calls at strike 25–30. Roll when DTE drops below 30 days. One position per month, sized to ~3–5% of harvested premium for the prior month.

**Anti-cyclicality discipline:**
- If VIX < 15: *increase* hedge allocation by 20% (insurance is cheap; markets are complacent)
- If VIX 15–25: standard hedge allocation (this is the base case)
- If VIX > 25: maintain existing hedges; do NOT aggressively add (vol is already priced in; hedging at elevated VIX has poor entry economics)
- If VIX > 35: suspend new short-premium entries entirely (Section 6 risk limit)

---

## 5. Exit rules

**Leg 1 — Short premium exits:**
- Take-profit: GTC LMT order at 50% of credit received, attached on entry
- Time stop: if DTE ≤ 7 and not closed at 50%, close at market on next open
- Assignment: sell covered calls at delta ≈ 0.30, 30–45 DTE

**Leg 2 — Long-vol exits:**
- **Never close a profitable hedge to "lock in gains."** The whole point of convexity is to let it run when it works. If a hedge moves significantly into profit (>5× initial cost), consider rolling the strike up to maintain exposure, but do not close to take profit.
- DTE-based roll: close and reopen when DTE < 30 days, regardless of P/L
- Strike-based roll: if SPY has rallied significantly and your put is more than 20% OTM, roll up to bring it closer to delta target (cheap; restores convexity)
- The hedge book is intended to expire worthless most months. Plan for that emotionally and budgetarily.

**Never use STOP LOSS — only STOP LMT** (your June 28, 2023 lesson applies to both legs).

---

## 6. Position sizing

**Leg 1 — Short premium sizing:**
- Cash-secured: each position requires (strike × 100 × contracts) in available cash
- Default size: 1 contract per position
- Maximum per underlying: 5% of total account equity
- Maximum total deployed capital: 50% of account equity across all open short puts
- Cash-secured structure caps single-trade loss at 5% of account; this is the hard rule

**Leg 2 — Long-vol sizing (the load-bearing math):**

*Target.* Hedge book is sized so that a -20% one-month SPY move produces a hedge gain ≈ 50% of the expected wheel pain in that scenario. This is *partial* protection — sized to convert a catastrophic month into a survivable one, not to fully neutralize tail exposure. Full neutralization would cost essentially all the premium harvested.

*Working assumptions for sizing:*
- Wheel deployed at 50% of equity ($50k on a $100k account) → -20% market move with assignment cycle produces ~$8–12k of wheel pain (8–12% drawdown), accounting for assignment at strikes 3–5% below current spot, premium offset, and mark-to-market on open positions
- SPY put at 0.10 delta, 90 DTE, on SPY ≈ $550: costs ~$700–900 per contract. In a -20% SPY move (to ~$440), that put goes from ~$495 strike, ~$0 intrinsic to ~$55 intrinsic + remaining time value, total ~$5,500–6,000. Net hedge gain per contract: ~$5,000.
- To offset 50% of $10k wheel pain → need ~1 SPY put → ~$800 cost
- 90-DTE puts spread that cost over 3 months → ~$270/month
- VIX calls at strike 25–30, ~3% of premium → ~$30/month (small)
- Combined hedge cost: ~$300/month on a $100k account
- Premium harvested at ~$500–1,000/month → hedge cost is **18–22% of harvested premium**

*Sizing rule (encoded as a discipline):* recompute hedge requirements monthly. Buy SPY put protection to maintain the "50% of wheel pain in -20% scenario" target. Buy VIX calls in fixed proportion (3–5% of prior month's premium) regardless of price level. Do not skip a hedge purchase because "this month feels safe."

*Anti-shrinkage rule:* during sustained low-VIX periods, the hedge will appear "wasteful." Do not shrink. The whole point of barbell sizing is to maintain protection through complacency. The discipline is to size based on what you'd need in a true tail, not on what feels reasonable given recent history.

---

## 7. Risk limits

- Maximum concurrent short positions (Leg 1): 6
- Daily loss limit before halt on new entries: -2% of account equity
- Drawdown threshold for full pause on new entries: -8% (this is *defense in depth* behind the hedge, not the primary tail protection)
- Single-trade max loss bound: 5% of account (enforced by sizing)
- Liquidity emergency: if bid-ask spread on any open position widens beyond 3× normal during market hours, halt new entries
- VIX regime: if VIX > 35 sustained for 5+ trading days, suspend new Leg 1 entries; maintain Leg 2 positions
- Hedge maintenance rule: if combined long-vol notional falls below sizing target for 30 consecutive days, the system is in violation regardless of other P/L. Restore hedge before next short-premium entry.

---

## 8. Execution model

- Broker: Interactive Brokers (TWS / IBKR API)
- Order types:
  - Entry on both legs: LMT at midpoint, willing to walk 5% of spread; cancel if unfilled after 30 min
  - Leg 1 profit-taker: GTC LMT at 50% of credit, attached at entry
  - Leg 2: no profit-taker (convexity must be allowed to run)
- Slippage assumptions for backtests: 5% of bid-ask spread on entry, 30% of quoted half-spread on closing trades (Muravyev-Pearson algorithmic effective spread)
- Commissions: IBKR retail tiered; pull actual from statements
- Order timing: no orders within 15 min of close; auto-cancel unfilled at 15:45 ET
- Bracketed orders use One-Cancels-Other (OCO)

---

## 9. Falsification criteria

The whole point of barbell is to change the *shape* of returns, not just the mean. Falsification must be measured on shape, not just performance.

**Shape criteria (the new and most important section):**
- **Realized skew (12-month rolling, monthly observations):** must be > -0.5. If skew < -0.5 for two consecutive quarters, the hedge leg is undersized — recalibrate before any new short-premium entries.
- **Worst-month / average-month ratio (12-month rolling):** must be ≤ 4×. A ratio of 6× or more indicates the strategy has retained too much concave exposure.
- **Worst-month absolute floor:** any single month worse than -10% of equity triggers full strategy review.

**Performance criteria (calibrated to the *hedged* return profile, not the unhedged one):**
- 8-week paper-trade combined P/L (Leg 1 minus Leg 2 cost): > $150/closed-combo cycle (vs ~$440 unhedged baseline; reflects 18–22% hedge cost)
- CSP win rate (not counting assignments): > 70% over 20+ trades
- Combined book Sharpe (annualized): > 0.8 over 12+ weeks
- Max drawdown: < 8% of paper account

**The trap to avoid.** The hedge will cost money every month it's not needed. The shape-criteria checks are designed to prevent the most dangerous failure mode: gradually shrinking the hedge during benign periods because it "isn't earning its keep." If realized skew is acceptable, the hedge IS earning its keep — that's the point. Returns will look worse than unhedged short vol. The unhedged version is the one that blows up.

> **Pre-commitment.** The shape criteria are evaluated mechanically. The phrase "the hedge is fine, the market just doesn't have any tail right now" is the verbal signature of impending strategy death. If skew turns negative for two consecutive quarters, the response is to *increase* the hedge, not rationalize keeping it as-is.

---

## 10. Performance bar to go live (Phase 3 gate)

Paper-trade results must meet ALL over 8+ weeks before real capital:

- Combined Sharpe (annualized): > 0.8
- Realized skew (rolling): > -0.5
- Worst-month / average-month ratio: < 4×
- Max drawdown: < 8% of paper account
- Trade count: > 25 closed Leg 1 combos
- Hedge book maintained continuously (no 30+ day gaps in target coverage)

---

## 11. Operational stack

- Data source: IBKR TWS for live data, ORATS Delayed ($99/mo) for backtesting (NBBO bid/ask, history to 2007)
- Broker: IBKR (paper first, live in same broker)
- Runtime: local cron on Somerville machine for v0
- Storage: SQLite for trade log
- Monitoring: daily email at market close with: Leg 1 positions, Leg 2 positions, hedge coverage ratio (actual vs target), today's P&L by leg, errors. Weekly Saturday summary.
- Kill switch: `~/.hermes/PAUSE` file checked at top of every cycle. Phone access via SSH. Distinct kill switches per leg: pausing Leg 1 should NOT pause Leg 2 (continuing to roll hedges during crisis is critical).
- Triggers for manual review: VIX > 25, single-day SPX move > 3%, any Leg 1 assignment, any hedge expiring more than 5× its cost (large convex win — review whether to roll up strike)

---

## 12. Known failure modes

- **The concave-exposure trap (the new top item):** unhedged short vol blows up in crisis regimes (1987, 1998, 2008, Feb 2018, March 2020, Aug 2024). v0 explicitly addresses this with Leg 2. The most dangerous failure mode is not the trap itself — it's *shrinking the hedge during benign periods*. This is the single behavior most likely to convert v0 from a survivable strategy into LTCM-pattern.
- **Crowded volatility-selling regime:** when too much capital is short premium, premiums compress and tails amplify. Leg 2 specifically protects against the Feb 2018-pattern. Detected by VIX regime check.
- **Basis risk on hedges:** single-name idiosyncratic blowup (e.g., one of your names has accounting fraud) won't be fully hedged by SPY puts. Bounded by quality universe (S&P 100 names) but not eliminated. Acceptable residual risk for v0.
- **Assignment in falling regime:** assigned at strike, underlying continues falling, covered calls produce trivial premium. Now partially offset by Leg 2 gains. Still painful, no longer existential.
- **Hedge-decay cost:** in sustained low-vol regimes, Leg 2 bleeds continuously. This is *expected and budgeted*. The discipline is not to interpret it as evidence the hedge is too expensive.
- **Strategy complexification under pressure:** the deepest failure mode from your notes. The barbell adds genuine complexity (two legs vs one). Resist the urge to add a third leg, modify hedge ratios mid-stream, or "improve" the strategy during a drawdown.
- **Market-maker gaming, stale orders, etc.:** all your existing lessons from 2023-2025 notes apply.

---

## 13. Validation plan

- Backtest window: Jan 1, 2018 — Dec 31, 2024 (must include Feb 2018, COVID March 2020, 2022 bear, Aug 2024)
- Critical: backtest must include Leg 2 throughout. A backtest of Leg 1 alone is not valid — it tests the wrong strategy.
- Out-of-sample holdout: Q4 2024
- Walk-forward: monthly re-evaluation of universe filters; fixed entry/exit and sizing rules
- Paper-trade duration: 8 weeks minimum; extend to 12 weeks if any month had < 4 closed Leg 1 combos
- Data: confirm ORATS history covers both single-name option chains and SPY/VIX options back to 2018

---

## Sign-off checklist (end of Phase 0)

- [ ] Every section reviewed and either accepted or modified
- [ ] Hedge-sizing math reviewed and either accepted or refined with your own assumptions
- [ ] Falsification criteria personally committed to in writing — including the shape criteria
- [ ] You can explain why this is a barbell (not just "wheel with insurance") to a competent friend in under 2 minutes
- [ ] You'd be willing to walk away from this strategy if shape criteria trigger — *honestly*
- [ ] You have psychologically committed to the hedge bleeding money in benign months — *before* you start
- [ ] Kill switch design specified concretely, with distinct switches per leg
- [ ] Backtest data availability confirmed for both single-name and index options

When all eight are true, Phase 1 starts.

---

## Open questions to refine in Phase 0

These are decisions I made with reasonable defaults but that deserve your own analysis:

1. **SPY vs SPX for the hedge leg.** SPX has better tax treatment (60/40) and cash settlement; SPY has tighter spreads. For v0 I defaulted to SPY for liquidity; consider SPX if account size justifies and you're comfortable with 100× notional per contract.

2. **VIX call sizing.** I defaulted to 3–5% of premium. This is rough. Aaron Brown and others argue VIX calls are the most efficient crisis-only hedge; could plausibly go to 8–10% of premium with smaller SPY put allocation. Worth modeling.

3. **Hedge-sizing target.** I picked "hedge covers 50% of wheel pain in -20% scenario." More conservative would be 75% or 100% coverage (higher cost, lower expected return, higher Sharpe). Less conservative would be 25% (more like a "speed bump" hedge). The 50% level is a defensible middle; refine based on your own utility function.

4. **VRP-percentile entry filter.** Goyal-Saretto and successors show that IV-RV ratio at entry predicts short-premium returns. v0 doesn't use this — entry is purely delta-based. Adding a "skip month if IV-RV ratio below 30th percentile" filter could meaningfully improve risk-adjusted returns. Park for v1 unless you want to backtest now.

5. **Single-name long-vol allocation.** I excluded single-name long options as too noisy. There's a defensible argument for running a tiny long-OTM-puts book on the same wheel underlyings as a basis-risk-free hedge of the assignment exposure specifically. Could add as v1 enhancement.

---

## Appendix A — Strategies parked for vN

[Unchanged from prior version: randomized signal selection, synthetic-short-straddle-via-STOP-orders, futures + TRAIL LMT, index options migration, scoring formula. All deferred with explicit graduation criteria.]
