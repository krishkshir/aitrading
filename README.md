# AITrading — Hermes v0

An automated options trading system built around a systematic volatility-premium harvesting strategy on US equities. The core thesis is that implied volatility on liquid US equities is structurally overpriced relative to realized volatility, and that disciplined premium-selling captures this edge without requiring directional forecasting.

## Strategy Overview

**Edge hypothesis:** Harvest the structural overpricing of implied volatility (IV) relative to realized volatility (RV) on liquid US equities by systematically selling cash-secured puts (CSPs) at delta ≈ −0.30 with 30–45 DTE, and selling covered calls (CCs) against any assigned shares (the "wheel").

This is a *harvesting* edge, not a *forecasting* edge. The strategy accepts a known small probability of large loss in exchange for a known high probability of small gain, where the premium collected structurally compensates for that asymmetry.

**Why the inefficiency persists:**
- Institutional hedgers structurally bid up put IV to hedge portfolios
- Retail option buyers sustain demand for OTM lottery tickets
- The strategy is capital-intensive (cash-secured), limiting arbitrage capacity

## Universe

US-listed optionable equities only (no futures options, no index options, no 0-DTE).

**Starting instruments:** BAC, VZ, F, KO, GILD, INTC

**Liquidity filters:**
- Option open interest > 500 at chosen strike
- Bid-ask spread < 5% of mid
- Underlying market cap > $10B
- At least 3 distinct bid levels within 2% of mid

**Exclusions:** earnings within DTE window, announced corporate actions, IV rank > 90, minimum premium < $0.29.

## Entry Rules

- Sell-to-open CSP when delta is between −0.25 and −0.35 (target −0.30)
- DTE: 30–45 days
- IV rank: 30–70
- 7-day cooldown per underlying after a closed position
- Annualized ROI must exceed 3-month T-bill yield + 5%

## Exit Rules

- **Take-profit (primary):** GTC LMT at 50% of credit received, attached at entry
- **Time stop:** close at market if DTE ≤ 7 and position not yet closed
- **Assignment:** immediately sell CC at delta ≈ 0.30 with 30–45 DTE against assigned shares
- No stop-loss on the option leg — assignment is part of the strategy
- STOP LMT only (never STOP LOSS market orders) if any stops are used

## Position Sizing

- Cash-secured; no margin for v0
- 1 contract per position (default)
- Max 5% of account equity per underlying
- Max 50% total deployed capital; raise to 75% only after 12+ weeks of clean operation

## Risk Limits

- Max 6 concurrent positions (one per universe name)
- Daily loss limit: −2% of account equity → halt new entries for the day
- Drawdown halt: −8% peak-to-trough → suspend all new entries pending review
- VIX > 35 sustained for 5+ days → suspend new entries

## Execution

- **Broker:** Interactive Brokers (TWS / IBKR API)
- **Entry orders:** LMT at midpoint, walk up to 5% of bid-ask spread, cancel if unfilled after 30 min
- **Profit-taker:** GTC LMT (not TRAIL LMT)
- **Order cutoff:** no new orders within 15 min of market close; auto-cancel unfilled orders at 15:45 ET
- **Bracketed orders:** One-Cancels-Other (OCO) tag

## Operational Stack

| Component | Choice |
|-----------|--------|
| Live data | IBKR TWS |
| Historical data | Polygon |
| Paper/live trading | IBKR (paper first) |
| Runtime | Local cron (v0) |
| Storage | SQLite |
| Kill switch | Toggle file at `~/.hermes/PAUSE` |

## Development Phases

| Phase | Description | Gate |
|-------|-------------|------|
| 0 | Spec review and sign-off | All checklist items complete |
| 1 | Backtest (Jan 2018 – Dec 2024) | Validated on Volmageddon, COVID crash, 2022 bear, 2024 regime |
| 2 | Paper trading | 8–12 weeks, 25+ closed combos |
| 3 | Live trading | All performance gates met |

**Phase 3 gate (must meet ALL):**
- Sharpe ratio (annualized) > 1.0
- CSP win rate > 70%
- Average closed-combo P/L > $200
- Max drawdown < 8%
- > 25 closed combos

## Falsification Criteria

The strategy is shelved for review if any of the following trigger:

- 8-week paper average combo P/L < $200
- CSP win rate < 70% over 20+ trades
- Peak-to-trough drawdown > 12% during paper phase
- VIX > 35 sustained for 5+ trading days
- > 70% of quarterly P/L is delta-driven (strategy has become effectively long-only)

## Known Failure Modes

- **Assignment in a falling regime:** wheel underperforms when an assigned underlying continues falling
- **Crowded vol-selling:** compressed premiums and amplified tail events when too much capital is short premium simultaneously
- **Thin contract gaming:** avoided by the liquidity floor in the universe filters
- **Gamma risk near expiration:** mitigated by the 7-DTE time stop
- **Strategy complexification under pressure:** v0 spec is frozen; modifications only at quarterly reviews

## Parked Strategies (Future Versions)

- **A.1** — Poisson-Cauchy randomized signal selection (unvalidated edge)
- **A.2** — Synthetic short straddle via STOP orders (unresolved whipsaw exposure)
- **A.3** — Futures + TRAIL LMT exit (leverage and monitoring overhead not appropriate for v0)
- **A.4** — Index options (SPX/XEO/NDX) — target v1 after 12 weeks of clean v0 operation
- **A.5** — Scoring formula for sub-ranking qualifying setups (not yet empirically validated)

## Repository Structure

```
.
├── README.md
├── strategy_v0.md      # Full strategy specification
└── ...                 # Source code, backtests, paper-trade logs
```
