# Methodology

A non-proprietary overview of the ideas behind the optimizer. It describes the
*approach*, not the implementation, calibration, or data.

## 1. Why mean-variance optimization falls short

Seven deficiencies of standard MVO motivate the design. None is new
individually; together they explain why the optimizer departs from MVO at every
stage rather than patching one.

1. **Backward-looking risk inputs.** Volatility and correlation are almost always
   estimated from history, but the distribution of future paths differs from the
   realized one. Calibrating a portfolio to 2003–2007 data in 2007 would have
   badly understated the probability and severity of what followed.

2. **No treatment of fat tails.** Standard deviation is the only risk measure,
   regardless of the actual shape of the distribution — historically or, more
   importantly, in the investor's view of the future. Financial returns are
   reliably fatter-tailed than a Gaussian with the same variance implies.

3. **No treatment of correlation breakdown.** Hedges that work in calm markets
   often fail under stress, and the failure is itself regime-dependent:
   Treasuries protected equity portfolios in 2008 and failed to in 2022. Sourcing
   a single historical correlation matrix assumes the next crisis resembles the
   average of past ones.

4. **Input sensitivity.** Small changes in expected returns or volatilities
   produce large changes in recommended weights. This is well documented
   (Michaud, 1989) and is a direct consequence of maximizing a quadratic
   objective against noisy estimates.

5. **The Sharpe ratio as objective.** Maximizing Sharpe selects the tangency
   portfolio, whose chosen risk rises as the risk-free rate rises. Theory resolves this
   by blending the tangency portfolio with cash or leverage. In practice,
   borrowing costs exceed the risk-free rate, producing a kink in the capital
   allocation line (Black, 1972); fat tails make leverage more dangerous than
   variance suggests; and levering a portfolio whose hedges may fail compounds
   both problems. The unintuitive results are that Sharpe-based optimization recommends a
   less risky portfolio when the equity premium over cash is widest and more risk when the premium is lowest.

6. **Standard deviation misprices asymmetric payoffs.** A long call option has
   high volatility and strictly bounded loss. A variance-based optimizer sees
   only the volatility and will never hold options, even though their risk is
   defined. The relevant question for an investor is not how much a position
   might move but how much it might lose.

7. **Arithmetic rather than geometric returns.** Wealth compounds, so returns are
   better described in log space, where a 50% loss and a 100% gain are symmetric.
   A portfolio's long-run growth rate is approximately its arithmetic mean less
   half its variance — exactly so for lognormal returns, and by more than that
   when returns are negatively skewed or fat-tailed, as equity returns are.
   Variance-based optimization understates the cost of volatility to compounded
   wealth, and the understatement grows with the square of volatility — and
   therefore sharply with leverage.

The optimizer replaces the single-covariance assumption with a **generative
scenario engine** that produces return paths including explicit crisis regimes —
addressing each deficiency where it arises rather than patching one.

## 2. The scenario engine

Rather than resampling history, the engine *generates* forward paths week by week.
Overlaid on a baseline are distinct crisis regimes, each with tunable:

- **frequency** — how often the regime is dialed to occur,
- **severity** — depth and duration of the drawdown,
- **correlation break** — how far assets (and hedges) move together once the
  regime hits.

This lets you ask "what does my portfolio do if a rate-shock crisis is twice as likely,
1.5x more severe than the baseline and equities are likely to be more strongly correlated than history suggests?" 
— a question historical covariance can't answer.

## 3. Objectives

Two interchangeable objectives, both grounded in utility theory:

- **CRRA log-utility** — maximize expected log growth (the Kelly criterion),
  then scale to a chosen fraction of full Kelly for a smoother ride. Relative
  risk aversion γ controls aggressiveness; γ = 1 recovers full Kelly.
- **Omega ratio** — the ratio of probability-weighted gains to losses relative
  to a threshold, with CRRA-based sizing on top. Omega captures the whole
  distribution, not just the first two moments.

A standalone, textbook implementation of the sizing math is open-sourced at
[kelly-sizing](https://github.com/harrisgreg07/kelly-sizing).

## 4. Two-stage solve

1. **Composition** — find an efficient blend of assets (at a reference leverage)
   under the chosen objective and the scenario distribution.
2. **Sizing** — given that composition, choose leverage subject to a max-leverage
   constraint, using the growth-optimal / fractional-Kelly fraction implied by
   the scenario tail.

Separating *what to hold* from *how much* keeps the problem well-conditioned and
makes the leverage decision explicit and tunable.

## 5. Out of sample valiations, Stress Tests and Backtesting

- **Out-of-sample validation** — Once optimized, the *fixed* portfolio is rerun on
  a fresh random seed to get stats on expected return, loss and drawdown. 
- **Per-crisis stress tests** — Isolate portfolio behavior conditional on each regime
  occurring: worst-1% outcome, per-asset weights, and each leg's contribution to
  crisis P&L.
- **Historical backtest** — Replay the sized portfolio over real history for
  a reality check (equity curve, CAGR, max drawdown, volatility).

## Further reading

Markowitz (1952) on mean–variance; Michaud (1989) on estimation-error
sensitivity; Black (1972) on the zero-beta CAPM and borrowing constraints;
Kelly (1956) and Thorp on the Kelly criterion; CRRA utility and the
growth-optimal portfolio; and the Omega ratio (Keating & Shadwick, 2002).
