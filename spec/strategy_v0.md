# Hermes — Strategy v0 Spec (barbell revision)

> **Revision rationale.** The prior spec described pure short-premium harvesting (CSPs + covered calls). This is structurally concave — the canonical "pennies in front of a steamroller" payoff that Taleb argues against, regardless of positive expected value. This revision restructures Hermes as a **barbell**: short-premium harvesting on one side, long-vol tail hedging on the other, with the latter funded by a fraction of the premium harvested by the former. The combined exposure is intended to have positive or near-zero skew, not the deeply negative skew of unhedged short vol.

> **Economics correction (post-literature-review).** An earlier draft of this section claimed the edge was a volatility-premium differential — collecting a large premium on single-name short puts and paying a small one on index hedges. The empirical literature disconfirms this. The single-name variance risk premium is approximately zero (Carr & Wu 2009; Driessen, Maenhout & Vilkov 2009); the large negative VRP lives in the index, not in single names. The wheel's real expected-return engine is therefore the equity risk premium (a short put is, by put-call parity, a covered-call-equivalent net-long-equity position), not a vol harvest. The index hedge is honest negative carry — a cost paid for convexity — not a "funded" leg of a favorable spread. See Section 1 for the corrected framing and Appendix A.6 for the dispersion trade the literature actually supports.

> The expected return is lower than the unhedged version. That is the deliberate trade. The asymmetry being purchased is *survival of regime change*, which is the only thing that distinguishes a strategy that compounds for 20 years from a strategy that compounds for 8 years and then dies in a single month.

---

## 1. Edge hypothesis

**One-sentence claim:** Hermes is a structured net-long-equity income strategy with a tail hedge. It collects the equity risk premium (plus a near-zero single-name volatility premium) by selling cash-secured puts at delta ≈ 0.30, 30–45 DTE, on liquid large-caps, and it pays explicit negative carry for long index puts and VIX calls that supply convex protection in correlated crashes. The combined payoff is a barbell: bounded participation in benign regimes, convexity in crisis regimes.

**What the edge actually is (and isn't)**. The edge is not a volatility-risk-premium harvest. The empirical record is clear that the individual-equity VRP is approximately zero on average: Carr & Wu (2009) find the VRP "significantly negative for the S&P 100 whereas the variance risk premiums on individual stocks are often zero or even positive," and Driessen, Maenhout & Vilkov (2009) find "index options have a large negative variance risk premium whereas individual options on all index components do not." By put-call parity a cash-secured short put is a covered-call-equivalent — net long the underlying with capped upside — so the dominant source of expected return is the equity risk premium, the compensation for holding quality large-caps with an assignment buffer. The small-to-zero single-name VRP is a minor contributor; it is not a structural edge to be relied upon, and any name- or timing-specific vol edge (e.g., from the IV-rank entry filter) must be demonstrated in backtest, not assumed.

**Why the premium exists (it's a risk premium, not an inefficiency)**. The VRP that does exist — concentrated in the index — is compensation for bearing risk, not a mispricing. It is "compensation to traders that are willing to provide variance and price-jump protection to other investors" and is "largely a compensation for downside semivariance risk." This reframing matters for the falsification logic: we are not betting that a mispricing persists; we are accepting fair compensation for bearing equity and crash risk, and capping the tail. There is no expectation that the premium is "arbitraged away," because there is nothing to arbitrage — it is a price for risk.

**Why the hedge is worth its cost despite being "expensive."** The index hedge runs negative carry: index puts and VIX calls are expensive precisely because the index carries the large negative VRP and prices the correlation risk premium (Driessen-Maenhout-Vilkov: the gap between index and single-name VRP "is only reconcilable with priced correlation risk"). But that expensiveness is the same fact as their being good insurance — index puts pay off in exactly the correlated, market-wide crash that detonates a short single-name-put book. Paying the correlation premium for protection that is genuinely correlated with our tail is rational. Taleb's barbell never required the convex leg to be cheap, only that it pay off in the tail. We budget the hedge honestly as a cost, not as a "funded" leg of a favorable spread.

**The directional caveat we are deliberately on the wrong side of.** On a pure-volatility basis, selling single-name vol and buying index vol is the unfavorable side of the dispersion trade — the literature shows the profitable vol trade is the reverse (sell index, buy single-names) (Driessen-Maenhout-Vilkov 2009). We accept this because (a) our return engine is the equity risk premium, not vol, and (b) the "wrong-way" vol leg is exactly what gives us crash convexity. The favorable dispersion trade is noted as a future direction in Appendix A.6, not pursued in v0 (it is capital-intensive, correlation-exposed, and operationally heavier).

**Prior evidence**: Your 2025 single-leg trading produced ~$14k on the income side (94% win rate on covered calls, ~$440 avg combo P/L on wheel). Read correctly, that is equity-risk-premium income with a small vol cushion on quality names — consistent with the framing above, not evidence of a single-name vol edge. The hedge leg is new in v0; its purpose is to reshape the distribution of returns, at a known cost, not to add expected return.

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
- Costs ~$300/month at $100k scale as negative carry — roughly 30–60% of harvested income depending on the month, highest in low-premium/low-vol months (a deliberate expense for convexity, sized off tail exposure, not a "funded" spread leg; see Section 6)

**Coordination:** the two legs are *not* hedges of specific positions — they are exposures of opposite convexity in the same portfolio. The wheel is not delta-hedged. The hedge is not position-by-position. The combined book is structured for distributional shape, not position-level neutrality.

---

## 3. Universe

**Leg 1 (single-name short premium):**

- US-listed optionable equities, Penny Interval Program members
- **Core set**: KO, VZ, BAC. Selected for low-to-moderate idiosyncratic volatility, dividend durability, and (for BAC) tail risk that is largely systematic and therefore covered by the Leg 2 index hedge.
- **Optional reduced-size satellites**: F, GILD. Higher-premium but carry risk the hedge does not cover — F's dividend was suspended in the 2020 COVID drawdown (cyclical dividend cut in exactly the regime where you'd be assigned); GILD carries binary, idiosyncratic pipeline/FDA-decision risk. Include only if explicitly seeking cyclical/idiosyncratic exposure, and size at ≤ half a core position.
- **Removed**: INTC. Suspended its dividend starting Q4 2024, still suspended through 2025, deeply negative free cash flow, live turnaround/solvency tail, high idiosyncratic volatility. Fails every reframed selection criterion; its 2025 premium-capture record reflects a benign regime that did not test its tail.
- **Selection rule for adding names**: rank Penny-Program-liquid candidates by a composite of (a) low idiosyncratic volatility — IVOL is the risk Leg 2 cannot hedge, so minimize it; (b) dividend durability through the last two recessions — you will own these via assignment, so a dividend that survives drawdowns matters more than its current yield; (c) low pairwise correlation with names already in the set, to avoid assignment clustering in a single sector (e.g., do not pair BAC with another large-cap bank). Rank on these, not on premium richness — premium richness is compensation for exactly the idiosyncratic risk you are trying to exclude.
- **Target profile to backfill toward**: low-beta, high-quality (profitable, low-leverage, stable-margin), dividend-durable names spread across low-correlation sectors the current set underweights (utilities, staples beyond KO, stable large-cap pharma rather than binary biotech, low-vol industrials). Candidates to evaluate against the composite rule, not buy recommendations.
- **Liquidity filters**: option open interest > 500 at chosen strike; bid-ask spread < 5% of mid; underlying market cap > $10B
- **Exclusions**: earnings within DTE window; corporate actions within DTE window; IV rank > 90; option premium < $0.29

**Leg 2 (index/vol long premium):**

- SPY: front-month-plus for long puts (most liquid US equity option chain by orders of magnitude)
- VIX: standard VIX options (cash-settled European)
- No single-name long options for v0 (basis risk too noisy; index hedge is cleaner)

---

## 4. Entry rules

**Leg 1 — Short put entry:**

- Sell-to-open short put when delta is between -0.25 and -0.35 (target -0.30)
- DTE 30–45 days
- IV rank between 30 and 70 -- note this is a per-name timing gate (avoid entering a given name when its own vol is dead-low or event-spiked), NOT a cross-sectional premium-richness screen. Universe membership is decided by the Section 3 quality/IVOL/correlation rule first; IV rank only times entries within the already-selected names. Do not use high IV rank to justify adding a name to the universe.
- Annualized ROI > T-bill yield + 5%
- Cooldown: no new put on same underlying within 7 days of a closed position

**Leg 2 — Long-vol entry (initial sizing):**

- SPY puts: open a position whenever portfolio long-put notional protection drops below the sizing target (see Section 6). Buy 60–120 DTE puts at delta ≈ 0.08–0.10 (typically 8–12% OTM). One position per month if needed for replenishment.
- VIX calls: maintain a continuously rolled position. Buy 60–90 DTE calls at strike 25–30. Roll when DTE drops below 30 days. Size as a small fixed-dollar tail sleeve (~$30/month at $100k scale; scale proportionally to account size), not as a percentage of prior-month premium — see Section 6 on sizing off tail exposure rather than off income.

**Anti-cyclicality discipline:**

- If VIX < 15: *increase* hedge allocation by 20% (insurance is cheap; markets are complacent)
- If VIX 15–25: standard hedge allocation (this is the base case)
- If VIX > 25: maintain existing hedges; do NOT aggressively add (vol is already priced in; hedging at elevated VIX has poor entry economics)
- If VIX > 35: suspend new short-premium entries entirely (Section 6 risk limit)

---

## 5. Exit rules

**Leg 1 — Short premium exits:**

- Take-profit: GTC LMT order at 50% of credit received, attached on entry. Rationale is gamma/tail reduction, not profit-banking — the back half of a short put's life earns the remaining premium slowly while gamma (and thus tail sensitivity) rises, so closing at 50% exits the worst risk-adjusted portion of the trade and reduces the concavity Leg 2 has to insure. *The 50% level is a swept parameter, not a fixed constant — backtest 25%/50%/75%/hold-to-expiry per name (see Section 13); the optimal level may differ across names and may not be 50%.* Carry 50% as the v0 default because it is your single most disciplined, actually-executed automation pattern (2025 data), not because it is validated.
Known cost (new under the equity-risk-premium framing): closing early plus the 7-day cooldown (Section 4) creates recurring time out of the market. Since the dominant return engine is the equity risk premium, every day flat is forgone equity exposure — turnover is no longer "free" the way the old vol-harvest framing implied. Small on liquid names, but real and structural; Section 13 requires it to be measured.
- Time stop: if DTE ≤ 7 and not closed at 50%, close using a marketable limit a little after the open.
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

Cost as a fraction of premium — a diagnostic, not a control. The hedge dollar amount is set by the tail-coverage target above, NOT by a target percentage of premium. The percentage is therefore an output that swings month to month. With ~$300/month of hedge cost and harvested premium of ~$500–1,000/month, the cost runs *roughly 30–60% of premium — and is highest (60%) in exactly the low-premium months, which cluster in low-volatility regimes.* That is the dangerous combination: the hedge looks most expensive relative to income precisely when complacency is highest and protection is about to matter most. Do NOT treat a high ratio as a reason to cut the hedge — the ratio being ugly in calm markets is a feature of correct sizing, not a bug. (An earlier draft stated "18–22%"; that figure was inconsistent with these dollar amounts — $300 ÷ $500–1,000 cannot yield 18–22% — and is corrected here.)

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

The whole point of barbell is to change the shape of returns, not just the mean — so falsification is ultimately measured on shape and on the benchmark comparison, not on raw performance. But shape metrics need a year of data; the paper-trade gate has weeks. The criteria are therefore tiered by what data actually exists when they bind. Do not evaluate a Tier 2 metric on Tier 1 data — a skew estimate from 2–3 monthly observations is statistically meaningless and must not be computed as if it were valid.

**Tier 1 — Phase 2 paper-trade gate (8–12 weeks, limited data)**
Only criteria that 2–3 months of observations can actually support:

- Primary test — beats the benchmark: combined book (Leg 1 + Leg 2) beats the equity-plus-hedge benchmark (buy-and-hold the universe + identical Leg 2) on risk-adjusted terms, after all costs. This is the load-bearing falsification criterion (see Section 13). If the wheel machinery does not beat simply holding the names with the same hedge, the strategy is falsified — revert to equity-plus-hedge. Benchmarking against the unhedged wheel is deprecated; the unhedged book is not the reference point.
- Max drawdown: < 8% of paper account over the window.
- No catastrophic month: no single month worse than -10% of equity. (Note: barely observable in an 8-week window — its main force is as a Tier 2 ongoing check.)
- Mechanical execution verified: hedge coverage maintained continuously (no 30+ day gaps vs target), profit-taker and roll logic fired as designed, kill switch tested live.

*Diagnostics tracked but NOT falsification triggers at this stage* (they measure the near-zero-edge vol component, not the equity-risk-premium engine, and assignment is part of the strategy per Section 5, not failure): CSP win rate; per-closed-combo P/L. Record them to calibrate expectations; do not shelve on them.

**Tier 2 — Ongoing live falsification (once ≥ 12 monthly observations exist)**
The shape criteria — the direct expression of the Taleb-consistency objective — bind here, not at the paper gate:

- Realized skew (12-month rolling, monthly observations): must be > -0.5. If skew < -0.5 for two consecutive quarters, the hedge leg is undersized — recalibrate (increase the hedge) before any new short-premium entries.
- Worst-month / average-month ratio (12-month rolling): must be ≤ 4×. A ratio ≥ 6× indicates too much concave exposure has been retained.
- Continued benchmark outperformance: the combined book continues to beat equity-plus-hedge on risk-adjusted terms over rolling 12-month windows. Persistent failure → revert to the simpler book.

**Tier 3 — The trap, and pre-commitment**
*The trap to avoid*. The hedge will cost money every month it's not needed (30–60% of premium per Section 6, worst in calm months). The Tier 2 shape checks exist to prevent the most dangerous failure mode: gradually shrinking the hedge during benign periods because it "isn't earning its keep." If realized skew is acceptable, the hedge IS earning its keep — that's the point. Returns will look worse than unhedged short vol. The unhedged version is the one that blows up.

> Pre-commitment. The shape criteria are evaluated mechanically once the data exists to support them. The phrase "the hedge is fine, the market just doesn't have any tail right now" is the verbal signature of impending strategy death. If skew turns negative for two consecutive quarters, the response is to increase the hedge, not rationalize keeping it as-is.

---

## 10. Performance bar to go live (Phase 3 gate)

The gate can only test what 8–12 weeks of paper data supports — i.e., Tier 1 criteria (Section 9). It cannot test the 12-month shape criteria; those become a post-launch commitment, not a gate condition.

*Gate conditions (all required, over 8–12 weeks paper)*:

- Beats the equity-plus-hedge benchmark on risk-adjusted terms, after costs (the primary test)
- Combined Sharpe (annualized): > 0.8
- Max drawdown: < 8% of paper account
- Trade count: > 25 closed Leg 1 combos
- Hedge book maintained continuously (no 30+ day gaps in target coverage); kill switch tested live

*Post-launch commitment (not a gate — there isn't enough data yet)*: within the first 12 months live, the Tier 2 shape criteria (realized skew > -0.5; worst-month/average-month ≤ 4×; continued benchmark outperformance) must be established and thereafter evaluated mechanically per Section 9. Going live is conditional on pre-committing — in writing — to honor these once the data exists, including the commitment to increase the hedge if skew turns negative rather than rationalize shrinking it.

*Not gate conditions (diagnostics only)*: CSP win rate, per-closed-combo P/L — for the reasons in Section 9 Tier 1.

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
- **Idiosyncratic risk sits in the hedge's blind spot (the universe-selection failure mode)**: Leg 2 (SPY puts, VIX calls) protects against systematic, correlated crashes. It does essentially nothing for an idiosyncratic single-name blowup — accounting fraud, a failed drug trial, a foundry collapse, a sudden dividend suspension. Cao & Han (2013) show high-idiosyncratic-vol names pay more option premium, but that premium compensates exactly the risk Leg 2 cannot hedge. The failure mode is therefore selecting names for premium richness, which systematically loads exposure into the unhedged region. The defense is in Section 3: rank the universe on low idiosyncratic vol, dividend durability, and low cross-correlation — never on premium. This is why INTC was removed and F/GILD demoted. Residual idiosyncratic risk on the core set (KO, VZ, BAC) is small and accepted; do not let the universe drift back toward high-IVOL names because they "pay better."
- **Assignment in falling regime:** assigned at strike, underlying continues falling, covered calls produce trivial premium. Now partially offset by Leg 2 gains. Still painful, no longer existential.
- **Hedge-decay cost:** in sustained low-vol regimes, Leg 2 bleeds continuously. This is *expected and budgeted*. The discipline is not to interpret it as evidence the hedge is too expensive.
- **Strategy complexification under pressure:** the deepest failure mode from your notes. The barbell adds genuine complexity (two legs vs one). Resist the urge to add a third leg, modify hedge ratios mid-stream, or "improve" the strategy during a drawdown.
- **Market-maker gaming, stale orders, etc.:** all your existing lessons from 2023-2025 notes apply.

---

## 13. Validation plan

- Backtest window: Jan 1, 2018 — Dec 31, 2024 (must include Feb 2018, COVID March 2020, 2022 bear, Aug 2024)
- Take-profit parameter sweep: test close-at-25% / 50% / 75% / hold-to-expiry, per name, measured net of Muravyev-Pearson transaction costs AND net of the time-out-of-market drag (the equity risk premium forgone during the flat periods created by early close + 7-day cooldown). Report the optimal level per name; do not assume 50% is correct.
- Benchmark requirement: every configuration must be measured against the equity-plus-hedge benchmark (buy-and-hold the universe + Leg 2), not just against zero. The take-profit rule, the wheel structure, and the turnover they create only earn their keep if wheel-plus-hedge beats equity-plus-hedge on risk-adjusted terms after all costs. If it doesn't, the simpler book wins — a clean, honest result.
- Critical: backtest must include Leg 2 throughout. A backtest of Leg 1 alone is not valid — it tests the wrong strategy.
- Out-of-sample holdout: Q4 2024
- Walk-forward: monthly re-evaluation of universe filters; fixed entry/exit and sizing rules
- Paper-trade duration: 8 weeks minimum; extend to 12 weeks if any month had < 4 closed Leg 1 combos
- Data: confirm ORATS history covers both single-name option chains and SPY/VIX options back to 2018

### Benchmark construction — explicit Phase 1 deliverable

The benchmark must be built and validated before Phase 2 (paper trade) starts, not assembled ad hoc during it. The primary falsification criterion in Section 9 depends entirely on the benchmark's integrity — a sloppy benchmark makes the central test meaningless. The following construction decisions must be locked in at the start of Phase 1, in code, not left as interpretation:

#### Construction specification

- Universe: identical to the current Leg 1 set at the backtest start date. For the historical window (2018–2024) this includes INTC even though it has since been removed; if you exclude it retroactively you introduce lookahead. Run both (with and without INTC) and report both; the gap between them is your estimate of universe-selection impact.
- Equity weights: equal-dollar weight across universe names, rebalanced monthly. Alternatives (market-cap, equal-vol) are noted in Open Questions; equal-dollar is the v0 default. Commit to one before writing the first line of code.
- Rebalancing: monthly, at the same calendar timing as Leg 1 position reviews. Rebalancing itself incurs equity transaction costs — include them.
- Dividends: reinvested at ex-dividend date. Do not treat dividends as cash income taken out of the portfolio; that would inflate the benchmark's cash yield relative to the wheel and distort the comparison.
- Leg 2: identical to the strategy's Leg 2 — same instrument, same entry DTE, same strike-selection logic, same roll rules, same Muravyev-Pearson cost model for fills. The comparison must hold Leg 2 constant and vary only whether Leg 1 is wheel or buy-and-hold. Do NOT run a no-hedge benchmark; that would conflate the hedge's contribution with the wheel's, and conflating them is the error the whole comparison is designed to avoid.
- Performance metrics: total return, annualized Sharpe, max drawdown, realized skew (monthly observations), worst-month/average-month ratio — identical metrics to Sections 9 and 10. Produce side-by-side output for every take-profit configuration sweep.

#### Validity checks before Phase 2 begins

- No lookahead in the universe or weights (universe fixed at backtest start; weights from start-of-month prices, not end-of-month)
- Corporate actions handled consistently (splits, spin-offs, mergers) — verify your ORATS or equity data source covers these for 2018–2024
- Leg 2 costs are applied symmetrically in both the strategy and the benchmark; if you haircut fills in the strategy, apply the same haircut in the benchmark's Leg 2
- Sanity check: run the benchmark alone (no Leg 1 either side) against the S&P 500 for 2018–2024 and confirm it looks like a low-beta, dividend-tilted equity portfolio with a Leg-2 drag — if it doesn't, something in the construction is wrong before you've compared anything

#### Phase 1 sequencing

1. Build and validate the benchmark simulation first (weeks 1–2 of Phase 1)
1. Build the wheel simulation against the same data (weeks 2–4)
1. Run the take-profit parameter sweep comparing both (weeks 5–6)
1. Produce a single output: for each take-profit level, does wheel-plus-hedge beat equity-plus-hedge? By how much? Under which regimes does it lose?
1. Only after step 4 is complete does the strategy have an honest go/no-go basis.

---

## Sign-off checklist (end of Phase 0)

- [x] Every section reviewed and either accepted or modified
- [x] Hedge-sizing math reviewed and either accepted or refined with your own assumptions
- [x] Falsification criteria personally committed to in writing — including the shape criteria
- [x] You can explain why this is a barbell (not just "wheel with insurance") to a competent friend in under 2 minutes
- [x] You'd be willing to walk away from this strategy if shape criteria trigger — *honestly*
- [x] You have psychologically committed to the hedge bleeding money in benign months — *before* you start
- [ ] Kill switch design specified concretely, with distinct switches per leg
- [ ] Backtest data availability confirmed for both single-name and index options
- [ ] Benchmark construction decisions locked (equity weights, rebalancing, dividend treatment, cost model symmetry) — these must be agreed before Phase 1 code begins, not during it

When all eight are true, Phase 1 starts.

---

## Open questions to refine in Phase 0

These are decisions I made with reasonable defaults but that deserve your own analysis:

1. **SPY vs SPX for the hedge leg.** SPX has better tax treatment (60/40) and cash settlement; SPY has tighter spreads. For v0 I defaulted to SPY for liquidity; consider SPX if account size justifies and you're comfortable with 100× notional per contract.

2. **VIX call sizing.** I defaulted to a small fixed-dollar sleeve (~$30/month at $100k). This is rough. Aaron Brown and others argue VIX calls are the most efficient crisis-only hedge; a larger VIX allocation with a smaller SPY put allocation could plausibly deliver the same tail coverage at lower carry. Worth modeling — but size it off tail-coverage contribution, not off a percentage of premium.

3. **Hedge-sizing target.** I picked "hedge covers 50% of wheel pain in -20% scenario." More conservative would be 75% or 100% coverage (higher cost, lower expected return, higher Sharpe). Less conservative would be 25% (more like a "speed bump" hedge). The 50% level is a defensible middle; refine based on your own utility function.

4. **VRP-percentile entry filter.** Goyal-Saretto and successors show that IV-RV ratio at entry predicts short-premium returns. v0 doesn't use this — entry is purely delta-based. Adding a "skip month if IV-RV ratio below 30th percentile" filter could meaningfully improve risk-adjusted returns. Park for v1 unless you want to backtest now.

5. **Single-name long-vol allocation.** I excluded single-name long options as too noisy. There's a defensible argument for running a tiny long-OTM-puts book on the same wheel underlyings as a basis-risk-free hedge of the assignment exposure specifically. Could add as v1 enhancement.

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

### A.6 — Dispersion trade (sell index vol, buy single-name vol)

**Status**: Parked for vN (likely v3+). This is the vol trade the literature actually supports — the favorable side of the spread Hermes v0 deliberately sits on the wrong side of.
**Rationale**: Driessen, Maenhout & Vilkov (2009) show that because the index carries a large negative VRP while constituents carry approximately zero, "a strategy that sells index options and buys individual options will lead to a large Sharpe ratio." The edge is the correlation risk premium — payment for bearing the risk that correlations spike in a crash. Selling index vol while owning single-name vol is, structurally, short correlation.
**Why it is NOT in v0.** (a) It is the opposite convexity profile from the tail hedge — short correlation means you are short the crash, exactly the exposure Hermes is trying to reduce. Running it alongside the wheel would compound, not offset, crash risk. (b) DMV themselves find the premium "cannot be exploited with realistic trading frictions" — a limits-to-arbitrage result; the gross alpha is real but net-of-cost capture is hard. (c) It is capital- and operationally heavy (many single-name legs against an index leg). (d) It demands true regime awareness — the trade blows up in correlation spikes, which is precisely 2008/2020/Aug-2024.
**Graduation criteria**: Only consider after v0 is running cleanly and you have a validated correlation-regime detector; backtest must demonstrate net-of-cost positive returns across at least one correlation-spike episode, with explicit modeling of the friction wall DMV document. Until then this is a known, literature-supported edge that you are choosing not to harvest because its risk profile is opposite your primary objective (crash survival).
