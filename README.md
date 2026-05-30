# AITrading — Hermes v0

An automated options trading system built as a **barbell**: a net-long-equity income engine on one leg, long-volatility tail hedging on the other. The hedge runs as honest negative carry — a deliberate cost paid for convexity, not a "funded" spread. The combined exposure is designed to have positive or near-zero skew — not the deeply negative skew of unhedged short vol.

The expected return is lower than an unhedged wheel. That is the deliberate trade. The asymmetry purchased is *survival of regime change*, which is what separates a strategy that compounds for 20 years from one that compounds for 8 years and dies in a single month.

## Strategy Overview

**One-sentence claim:** Collect the equity risk premium (plus a near-zero single-name volatility premium) by selling cash-secured puts at delta ≈ 0.30, 30–45 DTE on liquid large-caps, and pay explicit negative carry (~18–22% of income) for long index puts and VIX calls that supply convex protection in correlated crashes: bounded participation in benign regimes, convexity in crisis regimes.

**What the edge actually is (and isn't):** The dominant return engine is the equity risk premium — a short put is, by put-call parity, a covered-call-equivalent net-long-equity position. The single-name volatility risk premium is approximately zero (Carr & Wu 2009; Driessen, Maenhout & Vilkov 2009 — the large negative VRP lives in the index, not in individual stocks). This is not an inefficiency to arbitrage away; it is fair compensation for bearing equity and crash risk.

**Why the hedge costs what it does:** Index puts and VIX calls are expensive precisely because they carry the index VRP and price the correlation risk premium (Driessen-Maenhout-Vilkov: the gap between index and single-name VRP is "only reconcilable with priced correlation risk"). That expensiveness is why they pay off in exactly the correlated, market-wide crash that detonates a short single-name-put book. The hedge is budgeted honestly as a cost, not expected to be "cheap."

**Why the strategy is durable despite the hedge cost:**
- Operationally complex: two coordinated books with opposite convexity
- Psychologically difficult: the hedge looks like wasted money for years at a time
- The return engine (equity risk premium) is structural and persistent, not a mispricing

## Structure

**Leg 1 — Premium harvest (short vol, concave):**
- Cash-secured short puts at delta ≈ 0.30, 30–45 DTE, on validated single-name universe
- If assigned, sell covered calls at delta ≈ 0.30 against assigned shares (the wheel)
- Generates monthly cash inflow

**Leg 2 — Tail hedge (long vol, convex):**
- Long SPY puts: 60–120 DTE, delta ≈ 0.08–0.10, far OTM
- Long VIX calls: 60–90 DTE, strike 25–30
- Costs ~$300/month at $100k scale as deliberate negative carry — roughly 30–60% of income depending on the month, and highest in low-vol months (not a "funded" spread leg; sized off tail-coverage target, not off income)
- Expected to lose money most months; expected to produce large gains in crisis months

The two legs are exposures of opposite convexity in the same portfolio — not position-level hedges. The combined book is structured for distributional shape, not position-level neutrality.

## Universe

**Leg 1 (single-name short premium):**
- US-listed optionable equities, Penny Interval Program members
- **Core set**: KO, VZ, BAC — low-to-moderate idiosyncratic vol, dividend-durable, BAC tail risk is largely systematic and therefore covered by Leg 2
- **Optional satellites (≤ half a core position each)**: F, GILD — higher premium but carry idiosyncratic risk the hedge cannot cover (F's dividend was cut in COVID; GILD has binary FDA/pipeline tail)
- **Removed**: INTC — dividend suspended Q4 2024, negative free cash flow, live solvency tail, high idiosyncratic vol; fails every selection criterion
- **Selection rule for new names**: rank on (a) low idiosyncratic vol (IVOL is the unhedged risk), (b) dividend durability through the last two recessions, (c) low pairwise correlation with existing names. Do not add names for premium richness — high premium compensates exactly the idiosyncratic risk Leg 2 cannot hedge
- Liquidity filters: option OI > 500 at chosen strike; bid-ask spread < 5% of mid; underlying market cap > $10B
- Exclusions: earnings within DTE window; corporate actions within DTE window; IV rank > 90; option premium < $0.29

**Leg 2 (index/vol long premium):**
- SPY: 60–120 DTE puts (most liquid US equity option chain)
- VIX: standard VIX calls, cash-settled European
- No single-name long options in v0

## Entry Rules

**Leg 1 — Short put entry:**
- Sell-to-open when delta is between −0.25 and −0.35 (target −0.30)
- DTE 30–45 days; IV rank 30–70 (per-name timing gate only — not a cross-sectional premium-richness screen; universe membership is decided by the Section 3 quality/IVOL/correlation rule first)
- Annualized ROI > T-bill yield + 5%
- 7-day cooldown per underlying after a closed position

**Leg 2 — Long-vol entry:**
- SPY puts: buy 60–120 DTE at delta ≈ 0.08–0.10 whenever portfolio long-put notional drops below the sizing target; one position per month for replenishment
- VIX calls: maintain a continuously rolled position; buy 60–90 DTE at strike 25–30; roll when DTE < 30; sized as a fixed-dollar tail sleeve (~$30/month at $100k scale), not as a percentage of prior-month premium

**Anti-cyclicality discipline:**
- VIX < 15: increase hedge allocation by 20% (insurance is cheap; markets are complacent)
- VIX 15–25: standard allocation
- VIX > 25: maintain existing hedges; do not aggressively add (poor entry economics)
- VIX > 35: suspend all new Leg 1 entries; maintain Leg 2

## Exit Rules

**Leg 1 — Short premium exits:**
- Take-profit: GTC LMT at 50% of credit received, attached at entry. Rationale: closing at 50% exits the worst risk-adjusted portion of the trade (gamma rises in the back half while remaining premium shrinks). **50% is the v0 default pending backtest sweep** — 25%/50%/75%/hold-to-expiry must be tested per name, net of transaction costs and time-out-of-market drag (see Validation Plan). Known cost under the ERP framing: early close + 7-day cooldown creates recurring flat periods, and every day flat is forgone equity exposure — turnover is not free
- Time stop: close with a marketable limit order shortly after the open if DTE ≤ 7 and position not yet closed
- Assignment: sell covered calls at delta ≈ 0.30, 30–45 DTE

**Leg 2 — Long-vol exits:**
- Never close a profitable hedge to lock in gains; if a hedge moves >5× initial cost, consider rolling the strike up, not closing
- DTE roll: close and reopen when DTE < 30 days, regardless of P/L
- Strike roll: if SPY has rallied significantly and the put is >20% OTM, roll up to restore delta target

Never use STOP LOSS market orders — only STOP LMT on either leg.

## Position Sizing

**Leg 1:**
- Cash-secured: each position requires (strike × 100 × contracts) in available cash; no margin in v0
- Default size: 1 contract per position
- Max per underlying: 5% of account equity; max total deployed: 50% of account equity

**Leg 2 (the load-bearing math):**
- Sized so that a −20% one-month SPY move produces a hedge gain ≈ 50% of expected wheel pain — converts a catastrophic month into a survivable one, not full neutralization
- On a $100k account: ~1 SPY put contract (~$800 cost, spread over 3 months ≈ $270/month) + VIX calls (fixed-dollar sleeve ~$30/month) = ~$300/month combined hedge cost
- The dollar amount is set by the tail-coverage target, not by a target percentage of income. The cost-to-income ratio is an output: ~30–60% of monthly income, and highest (60%) in low-premium/low-vol months — precisely when complacency is highest and protection is about to matter most. Do not treat a high ratio as a reason to cut the hedge
- Recompute SPY put requirement monthly; do not skip a hedge purchase because "this month feels safe"

## Risk Limits

- Max 6 concurrent Leg 1 short positions (one per universe name)
- Daily loss limit: −2% of account equity → halt new Leg 1 entries for the day
- Drawdown halt: −8% peak-to-trough → suspend all new Leg 1 entries pending review
- Single-trade max loss: 5% of account (enforced by cash-secured sizing)
- Liquidity emergency: if bid-ask on any open position widens >3× normal, halt new entries
- VIX > 35 for 5+ trading days: suspend new Leg 1 entries; maintain Leg 2
- **Hedge maintenance rule:** if combined long-vol notional falls below sizing target for 30 consecutive days, the system is in violation — restore hedge before any new short-premium entry

## Execution

- Broker: Interactive Brokers (TWS / IBKR API)
- Entry both legs: LMT at midpoint; walk up to 5% of spread; cancel if unfilled after 30 min
- Leg 1 profit-taker: GTC LMT at 50% of credit, attached at entry
- Leg 2: no profit-taker (convexity must be allowed to run)
- Slippage for backtests: 5% of bid-ask on entry; 30% of quoted half-spread on closes (Muravyev-Pearson)
- Order cutoff: no new orders within 15 min of close; auto-cancel unfilled at 15:45 ET
- Bracketed orders: One-Cancels-Other (OCO)

## Operational Stack

| Component | Choice |
|-----------|--------|
| Live data | IBKR TWS |
| Historical / backtest data | ORATS Delayed ($99/mo, option NBBO back to 2007) |
| Paper / live trading | IBKR (paper first) |
| Runtime | Local cron on Somerville machine (v0) |
| Storage | SQLite trade log |
| Monitoring | Daily email at close (Leg 1 positions, Leg 2 positions, hedge coverage ratio, P&L by leg, errors); weekly Saturday summary |
| Kill switch | `~/.hermes/PAUSE` — **distinct switches per leg**; pausing Leg 1 must NOT pause Leg 2 (rolling hedges during crisis is critical) |
| Manual review triggers | VIX > 25; single-day SPX move > 3%; any Leg 1 assignment; any hedge expiring >5× cost |

## Development Phases

| Phase | Description | Gate |
|-------|-------------|------|
| 0 | Spec review and sign-off | All sign-off checklist items complete |
| 1 | Backtest (Jan 2018 – Dec 2024) | Validated across Feb 2018, COVID March 2020, 2022 bear, Aug 2024; **must include Leg 2 throughout**; take-profit parameter sweep (25%/50%/75%/hold) per name; wheel-plus-hedge must beat equity-plus-hedge on risk-adjusted terms after all costs |
| 2 | Paper trading | 8–12 weeks, 25+ closed Leg 1 combos |
| 3 | Live trading | All Phase 3 gates met |

**Phase 3 gate (must meet ALL):**
- Combined Sharpe (annualized): > 0.8
- Realized skew (rolling): > −0.5
- Worst-month / average-month ratio: < 4×
- Max drawdown: < 8% of paper account
- Closed Leg 1 combos: > 25
- Hedge book maintained continuously (no 30+ day gaps in target coverage)

## Falsification Criteria

**Shape criteria (primary):**
- Realized skew (12-month rolling, monthly observations): must be > −0.5. Two consecutive quarters below → recalibrate hedge before any new short-premium entries
- Worst-month / average-month ratio: must be ≤ 4×. A ratio ≥ 6× indicates too much concave exposure
- Any single month worse than −10% of equity: full strategy review

**Performance criteria (calibrated to the hedged return profile):**
- 8-week paper combined P/L (Leg 1 minus Leg 2 cost): > $150/closed-combo cycle
- CSP win rate: > 70% over 20+ trades
- Max drawdown: < 8% of paper account

**The trap to avoid:** the hedge will cost money every month it's not needed. The phrase "the hedge is fine, the market just doesn't have any tail right now" is the verbal signature of impending strategy death. If realized skew turns negative for two consecutive quarters, the response is to *increase* the hedge.

## Known Failure Modes

- **Concave-exposure trap (top item):** unhedged short vol blows up in crisis regimes (1987, 1998, 2008, Feb 2018, March 2020, Aug 2024). The most dangerous failure mode is *shrinking the hedge during benign periods* — this is the LTCM pattern
- **Crowded volatility-selling:** compressed premiums and amplified tails; Leg 2 specifically protects against Feb 2018-pattern; detected by VIX regime check
- **Idiosyncratic risk sits in the hedge's blind spot:** Leg 2 (SPY puts, VIX calls) covers systematic crashes; it does nothing for single-name blowups — fraud, a failed drug trial, a dividend suspension. High-IVOL names pay more premium (Cao & Han 2013), but that premium compensates exactly the unhedged risk. Defense is in the universe selection rule: rank on low idiosyncratic vol, dividend durability, and low cross-correlation — never on premium. This is why INTC was removed and F/GILD demoted
- **Assignment in a falling regime:** wheel underperforms when assigned underlying continues falling; now partially offset by Leg 2 gains, no longer existential
- **Hedge-decay cost:** in sustained low-vol regimes, Leg 2 bleeds continuously; expected and budgeted — do not interpret as evidence the hedge is too expensive
- **Strategy complexification under pressure:** the barbell already adds genuine complexity; resist adding a third leg, modifying ratios mid-stream, or improving the strategy during a drawdown

## Parked Strategies (Future Versions)

- **A.1** — Poisson-Cauchy randomized signal selection (no validated edge; backtest required before revisiting)
- **A.2** — Synthetic short straddle via STOP orders (fundamental whipsaw exposure unresolved)
- **A.3** — Futures + TRAIL LMT exit (leverage and 24/7 monitoring overhead not appropriate for v0)
- **A.4** — Index options (SPX/XEO/NDX) — target v1 after 12 weeks of clean v0 operation; XEO has better tax treatment
- **A.5** — Scoring formula for sub-ranking qualifying setups (not yet empirically validated)
- **A.6** — Dispersion trade (sell index vol, buy single-name vol) — the literature-supported favorable side of the vol spread (Driessen-Maenhout-Vilkov 2009); parked because its convexity profile is opposite to Hermes v0's crash-survival objective, and net-of-cost capture is hard (friction wall documented by DMV)

## Repository Structure

```
.
├── README.md
├── spec/
│   └── strategy_v0.md   # Full strategy specification
└── ...                  # Source code, backtests, paper-trade logs
```
