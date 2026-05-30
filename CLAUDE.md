# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Repository Status

This is a **specification-stage** repository — no application code exists yet. The tracked directories are empty placeholders:

- `code/` — trading system source code (not yet started)
- `data/` — datasets and backtest inputs (not yet started)
- `notes/` — research notes (not yet started)
- `spec/` — strategy specification documents (active)

## Source of Truth

`spec/strategy_v0.md` is the authoritative strategy document. `README.md` is a derived human-facing summary. **Whenever `spec/strategy_v0.md` changes, update `README.md` to match.**

## Strategy Overview

Hermes v0 is a **barbell options strategy**: Leg 1 sells cash-secured puts (delta ≈ 0.30, 30–45 DTE) on a fixed single-name universe (BAC, VZ, F, KO, GILD, INTC), turning assigned shares into a covered-call wheel. Leg 2 buys long OTM SPY puts and VIX calls, funded by 18–22% of Leg 1 premium, to inject convexity into the return distribution. Falsification is measured primarily on *distribution shape* (realized skew, worst-month ratio), not just P&L.

## Hard Constraints

- **v0 spec is frozen.** Do not propose spec changes outside quarterly reviews.
- **STOP LMT only — never STOP LOSS market orders** on either leg.
- **Do not commit trade records, databases, or secrets.** `.gitignore` excludes `*.xlsx`, `*.csv`, `*.sqlite`, `*.db`, `.env*`, and `~/.hermes/` by design.

## Auto-Update Rule

**After every `git commit`, assess whether the committed changes warrant updates to `README.md` or `CLAUDE.md`:**

- If `spec/strategy_v0.md` changed → update `README.md` to reflect the spec.
- If new code patterns, tooling conventions, or project structure emerged → update `CLAUDE.md`.

Make any such updates and commit them before finishing the task.

## Intended Stack (when code lands)

| Concern | Choice |
|---------|--------|
| Broker API | IBKR TWS / IBKR API |
| Historical options data | ORATS Delayed ($99/mo) |
| Trade log | SQLite (outside repo, gitignored) |
| Runtime | Local cron |
| Kill switch | `~/.hermes/PAUSE` — **distinct per-leg switches** (pausing Leg 1 must not pause Leg 2) |
