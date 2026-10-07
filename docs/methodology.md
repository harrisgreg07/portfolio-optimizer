# Methodology

A non-proprietary overview of the ideas behind the optimizer. It describes the
*approach*, not the implementation, calibration, or data.

## 1. Why not mean–variance?

Classic mean–variance optimization assumes returns are well described by a single
covariance matrix estimated from history. Two problems in practice:

1. **Correlations break in crises.** Diversification and hedges that look great
   in calm data fail precisely when you need them — correlations converge and
   "uncorrelated" sleeves sell off together.
2. **Tails are not normal.** A covariance matrix says nothing about the shape of
   the loss tail, which is what actually determines ruin risk and sizing.

The optimizer replaces the single-covariance assumption with a **generative
scenario engine** that produces return paths including explicit crisis regimes.

## 2. The scenario engine

Rather than resampling history, the engine *generates* forward paths week by week.
Overlaid on a baseline are distinct crisis regimes, each with tunable:

- **frequency** — how often the regime is dialed to occur,
- **severity** — depth and duration of the drawdown,
- **correlation break** — how far assets (and hedges) move together once the
  regime hits.

This lets you ask "what does my book do if rate-shock crises are twice as common
and 1.5× as severe as the baseline?" — a question historical covariance can't
answer.

## 3. Objectives

Two interchangeable objectives, both grounded in utility theory:

- **CRRA log-utility** — maximize expected log growth (the Kelly criterion),
  then scale to a chosen fraction of full Kelly for a smoother ride. Relative
  risk aversion γ controls aggressiveness; γ = 1 recovers full Kelly.
- **Omega ratio** — the ratio of probability-weighted gains to losses relative
  to a threshold, with CRRA-based sizing on top. Omega captures the whole
  distribution, not just the first two moments.

A standalone, textbook implementation of the sizing math is open-sourced at
[kelly-sizing](https://github.com/gregharris73/kelly-sizing).

## 4. Two-stage solve

1. **Composition** — find an efficient blend of assets (at a reference leverage)
   under the chosen objective and the scenario distribution.
2. **Sizing** — given that composition, choose leverage subject to a max-leverage
   constraint, using the growth-optimal / fractional-Kelly fraction implied by
   the scenario tail.

Separating *what to hold* from *how much* keeps the problem well-conditioned and
makes the leverage decision explicit and tunable.

## 5. Keeping yourself honest

- **Out-of-sample validation** — once optimized, the *fixed* book is repriced on
  a fresh random seed. Reporting in-sample stats would flatter the result; OOS
  numbers are the ones that matter.
- **Per-crisis stress tests** — isolate book behavior conditional on each regime
  occurring: worst-1% outcome, per-asset weights, and each leg's contribution to
  crisis P&L.
- **Historical backtest** — finally, replay the sized book over real history for
  a reality check (equity curve, CAGR, max drawdown, volatility).

## Further reading

Kelly (1956), Thorp on the Kelly criterion, CRRA utility and the growth-optimal
portfolio, and the Omega ratio (Keating & Shadwick, 2002).
