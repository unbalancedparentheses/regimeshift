# regimeshift

A research project for detecting market regime shifts — transitions between risk-on and risk-off states — by combining signals from multiple traditions: variance risk premium, liquidity and funding stress, tail-risk and crash diagnostics, credit and macro indicators, and systemic risk measures. The goal is transparent, inspectable signals grounded in the academic literature, not black-box scores.

## Not financial advice

This project is for research and educational purposes only. Nothing here is investment advice or a recommendation to buy or sell any security or digital asset. Use at your own risk.

## Goals

- Provide transparent, inspectable signals (not black boxes).
- Combine multiple families of indicators:
  - Tail-risk / crash hazard measures (fat-tail diagnostics).
  - Variance risk premium (implied vs realized variance).
  - Liquidity / impact proxies (order-book or market microstructure).
- Emit a simple regime label (e.g., risk-on, neutral, risk-off) plus confidence notes.

## Planned structure

- `src/` Rust core for data ingestion, feature computation, and signal aggregation.
- `python/` optional ML/AI experiments when Rust is not ideal.
- `data/` optional cached datasets or example fixtures.

## Status

Scaffold only. Implementation TBD.

## Config and output schema

- Example config: `config.example.toml`
- Output schema: `signals.schema.json`
- Sample output: `signals.sample.json`

## Data sources

Signal availability varies by source. A practical breakdown:

- **Free / public**: FRED series (spreads, curve, stress indices), OFR FSI, NBER working papers.
- **Often paid or restricted**: Cboe indices (VIX, VVIX, SKEW), ICE MOVE, ETF flow data, detailed order-book depth.
- **Exchange-specific**: order-book depth (crypto/FX often easier than equities).

This affects which signals are usable in an MVP vs a production system.

### FRED API

Base URL: `https://api.stlouisfed.org/fred/series/observations`
Required params: `series_id`, `api_key`, `file_type=json`
Free API key: https://fred.stlouisfed.org/docs/api/api_key.html

Key series IDs:

| Signal | FRED ID | Frequency | Notes |
|--------|---------|-----------|-------|
| St. Louis FSI | `STLFSI4` | Weekly | Released Friday |
| VIX | `VIXCLS` | Daily | Cboe; redistribution restrictions apply |
| HY spread (OAS) | `BAMLH0A0HYM2` | Daily | ICE BofA US High Yield |
| IG spread (BBB OAS) | `BAMLC0A4CBBB` | Daily | ICE BofA BBB |
| IG spread (AAA OAS) | `BAMLC0A1CAAA` | Daily | ICE BofA AAA |
| Baa yield | `BAA` | Daily | Moody's; use BAA-AAA for spread |
| Aaa yield | `AAA` | Daily | Moody's |
| 10Y-2Y curve | `T10Y2Y` | Daily | |
| 10Y-3M curve | `T10Y3M` | Daily | Better recession predictor |
| SOFR | `SOFR` | Daily | Replaced LIBOR |
| Fed funds rate | `DFF` | Daily | |
| NBER recession | `USREC` | Monthly | 1=recession; useful for validation |
| EPU index | `USEPUINDXD` | Daily | Baker-Bloom-Davis uncertainty |

Note: TED spread (`TEDRATE`) was discontinued April 2022. Substitute: 3-month T-bill vs. SOFR.

### OFR Financial Stress Index

API: `https://dev.financialresearch.gov/`
CSV download: https://www.financialresearch.gov/financial-stress-index/
Daily series, ~1 business day lag.

### ACM term premium

Download from NY Fed: https://www.newyorkfed.org/research/data_indicators/term-premia-on-interest-rates
Monthly CSV. Series: `ACMTP10` (10-year term premium).

### Cboe indices

VIX, VVIX, SKEW historical data: https://www.cboe.com/tradable_products/vix/vix_historical_data/
VIX futures settlement prices (for term structure): https://www.cboe.com/products/vix-index-volatility/vix-options-and-futures/vix-futures/vix-futures-historical-data/
Note: Cboe data is free for personal/research use but has redistribution restrictions.

## Theoretical frameworks

Different schools offer distinct explanations of what a regime is and how transitions occur. Understanding the tension between them is important: this project draws on all of them, but they are not always compatible.

### The core tension: statistical vs. physical approaches

**Hamilton's regime-switching** treats regimes as latent discrete states inferred from data. The model is agnostic about what causes transitions — it estimates a transition matrix from history and outputs a probability distribution over states. It is statistically rigorous and well-understood, but it assumes a fixed number of regimes and stationary dynamics. It cannot predict when the next transition will occur; it can only tell you which regime you are probably in right now.

**Sornette's LPPL framework** treats regime endings as deterministic critical points driven by endogenous positive feedback (herding, leverage spirals). The approach is forward-looking — it aims to forecast crash timing — and is grounded in statistical physics rather than econometrics. But it is harder to validate out of sample and sensitive to fitting assumptions. It views markets as driven toward criticality, not randomly switching between stable states.

These are genuinely different ontologies. Hamilton says regimes are states you infer; Sornette says crashes are events you can anticipate. Most of the other frameworks below sit between these poles or address different aspects of the same phenomenon.

### Critical slowing down (Scheffer et al.)

Near a tipping point, systems recover more slowly from small perturbations. Observable signatures: rising variance, rising autocorrelation, and rising cross-correlation across variables as the system approaches a bifurcation. Practical use: monitor rolling variance and AR(1) coefficient of key signals — acceleration in both is a warning independent of any specific model. This is the empirical complement to Sornette's theoretical prediction.

### Hawkes processes and event clustering (Bacry, Muzy, Jaisson)

Extreme events cluster in time due to self-excitation: each event raises the probability of further events. Regime transitions appear as shifts in the branching ratio (ratio of triggered to background events). A branching ratio approaching 1 signals a system near criticality. Applies to order flow, volatility spikes, and default events. Provides a middle ground: more data-driven than LPPL, more mechanistic than Hamilton.

### Rough volatility (Gatheral, Rosenbaum)

Realized volatility has a Hurst exponent H ≈ 0.1, far below 0.5. Volatility is rougher than a random walk and has long-range anti-persistence at short scales. Standard GARCH-based models underestimate short-term vol clustering; rough vol models better capture the rapid spikes that precede stress regimes. Practically relevant for calibrating VRP and vol-of-vol signals.

### Intermediary asset pricing (Adrian, Shin, He, Krishnamurthy)

Leverage and balance-sheet constraints of financial intermediaries drive risk premia and liquidity. When intermediary capital is scarce, risk premia spike and liquidity dries up simultaneously — a defining signature of risk-off regimes. Provides a unified micro-foundation for why credit spreads, VRP, and funding stress co-move at regime transitions, something neither Hamilton nor Sornette explains directly.

## Signals (planned)

Signals are grouped into thematic families. Each entry in the spec below lists the data source, raw measurement, and normalization method. Not all signals are available for free; see Data availability above.

### Signals spec

This section defines each planned signal, its data source, and a suggested normalization method.

**Conventions**

- `value`: raw measurement
- `z`: z-score over a rolling window (default 3Y if daily, 5Y if weekly)
- `pct`: percentile rank over a rolling window
- `ema`: exponential moving average

**Macro stress**

- OFR Financial Stress Index (FSI)
  - Source: OFR FSI daily series
  - Raw: index level
  - Normalize: `z`
- St. Louis Fed Financial Stress Index (STLFSI)
  - Source: FRED `STLFSI4`
  - Raw: index level
  - Normalize: `z`

**Volatility regime**

- VIX term structure slope
  - Source: Cboe VIX futures curve
  - Raw: front-month vs 3M or 6M futures slope
  - Normalize: `z`
- VIX option skew (Cboe SKEW)
  - Source: Cboe SKEW index
  - Raw: index level
  - Normalize: `z`
- MOVE index
  - Source: ICE BofA MOVE
  - Raw: index level
  - Normalize: `z`
- VVIX (vol-of-vol)
  - Source: Cboe VVIX
  - Raw: index level
  - Normalize: `z`

**Liquidity and funding stress**

- Funding stress spreads
  - Source: FRED `SOFR`, `DFF` (fed funds); OIS from Bloomberg/ICE (paid)
  - Raw: SOFR-OIS spread in bps
  - Normalize: `z`
- TED spread
  - Source: FRED `TEDRATE` (discontinued April 2022); substitute `TB3MS` minus `SOFR`
  - Raw: spread in bps
  - Normalize: `z`
- Liquidity proxy (Amihud)
  - Source: price and volume data
  - Raw: |return| / volume
  - Normalize: `pct`
- Order-book depth
  - Source: exchange order book (when available)
  - Raw: top-of-book or X% depth
  - Normalize: `pct`
- Order flow imbalance
  - Source: exchange trade data or order book
  - Raw: signed order flow (buy-initiated minus sell-initiated volume), normalized by total volume
  - Normalize: `z`

**Credit and macro**

- Credit spreads
  - Source: FRED `BAMLH0A0HYM2` (HY OAS), `BAMLC0A4CBBB` (BBB OAS), `BAA`-`AAA` (Moody's)
  - Raw: spread in bps
  - Normalize: `z`
- Yield curve / term premium
  - Source: FRED `T10Y2Y` (2s10s), `T10Y3M` (3m10y); ACM term premium from NY Fed
  - Raw: spread or term premium
  - Normalize: `z`

**Returns risk premia**

- Variance risk premium (VRP)
  - Source: implied variance (options) minus realized variance
  - Raw: IV - RV
  - Normalize: `z` and `pct`
- Tail-risk premium (jump risk)
  - Source: option-based jump/tail risk decomposition
  - Raw: jump variance premium
  - Normalize: `z`

**Cross-asset structure**

- Cross-asset correlation spike
  - Source: rolling correlations (equity-credit, equity-rates, equity-commodities)
  - Raw: correlation level or delta
  - Normalize: `z`

**Positioning and flows**

- CFTC COT positioning
  - Source: CFTC COT (net speculative positions)
  - Raw: net position or z-scored net positioning
  - Normalize: `z`
- ETF flows
  - Source: ETF flow data
  - Raw: net flows as % AUM
  - Normalize: `z`

**Systemic risk and contagion**

- CoVaR (conditional VaR)
  - Source: equity returns (Adrian-Brunnermeier method)
  - Raw: ΔCoVaR — system VaR conditional on institution in distress minus unconditional
  - Normalize: `z`
- MES (marginal expected shortfall)
  - Source: equity returns
  - Raw: expected equity loss of a firm conditional on a market tail event
  - Normalize: `z`
- SRISK
  - Source: equity returns and balance sheet data (Engle-Brownlees)
  - Raw: expected capital shortfall in a crisis scenario
  - Normalize: `z`

**Text and sentiment**

- FOMC communication tone
  - Source: FOMC minutes and statements (NLP)
  - Raw: hawkish/dovish score or policy uncertainty index
  - Normalize: `z`
- News-based uncertainty
  - Source: Baker-Bloom-Davis Economic Policy Uncertainty index or similar NLP pipeline
  - Raw: aggregate uncertainty or sentiment score
  - Normalize: `z`

## Algorithm details

Computation details for each major signal type. All formulas use log returns `r_t = ln(P_t / P_{t-1})`.

### Realized variance

From daily returns (simplest):
```
RV_t = (252 / W) * Σ_{i=t-W+1}^{t} r_i²
```
Default W = 21 (monthly), 63 (quarterly). The factor 252 annualizes.

Parkinson estimator (from OHLC, no intraday data needed, more efficient):
```
RV_Park_t = (252 / (4·ln2·W)) * Σ_{i=t-W+1}^{t} (ln(H_i / L_i))²
```

Garman-Klass estimator (most efficient from OHLC):
```
GK_i = 0.5·(ln(H_i/L_i))² - (2·ln2 - 1)·(ln(Close_i/Open_i))²
RV_GK_t = 252 * (1/W) * Σ GK_i
```

From high-frequency (5-min) returns (preferred when available):
```
RV_HF_t = Σ_{i=1}^{M} r_{t,i}²   [M ≈ 78 per day for US equities]
```
Apply realized kernel or pre-averaging correction for microstructure noise (Barndorff-Nielsen, Hansen, Lunde, Shephard).

### Variance risk premium (VRP)

```
IV_t      = (VIX_t / 100)²           # annualized implied variance
IV_monthly = IV_t / 12               # monthly implied variance
RV_monthly = (252 / 21) * Σ_{i=t-20}^{t} r_i²
VRP_t     = IV_monthly - RV_monthly
```

VRP > 0: investors paying for variance protection (normal).
VRP < 0: realized vol exceeds implied vol (already stressed).
High positive VRP predicts positive equity returns at quarterly horizon (Bollerslev, Tauchen, Zhou 2009).

### Rolling z-score

```
μ_{t,W} = mean(x_{t-W+1} .. x_t)
σ_{t,W} = std(x_{t-W+1} .. x_t)   # sample std, ddof=1
z_t     = clip((x_t - μ_{t,W}) / σ_{t,W}, -4, 4)
```

Parameters: W = 756 days (3Y daily) or 260 weeks (5Y weekly). Require min 63 observations before emitting. Clip to ±4 to limit outlier influence. For highly skewed series (Amihud, depth), log-transform before z-scoring.

### Rolling percentile rank

```
pct_t = rank(x_t in {x_{t-W+1} .. x_t}) / W
```
Equivalent to empirical CDF at x_t over rolling window. Require min 126 observations.

### Amihud illiquidity

```
ILLIQ_t = (1/W) * Σ_{i=t-W+1}^{t} |r_i| / DollarVolume_i
```
Log-transform before normalizing: `log(ILLIQ_t)` is more Gaussian. Normalize with `pct`.

### Hamilton filter (2-state HMM)

**Model:**
```
s_t ∈ {0=normal, 1=stressed}
y_t | s_t=j ~ N(μ_j, σ_j²)
Transition matrix P:  P[i,j] = P(s_t=j | s_{t-1}=i)
  P = [[p00,   1-p00],
       [1-p11, p11  ]]
```

**Forward filter (Hamilton filter):**
```
Initialize: ξ_{0|0} = stationary distribution of P
For t = 1..T:
  Predict: ξ_{t|t-1}[j] = Σ_i P[i,j] * ξ_{t-1|t-1}[i]
  Densities: η_t[j] = N(y_t; μ_j, σ_j²)
  Update: ξ_{t|t} = (ξ_{t|t-1} ⊙ η_t) / Σ_j(ξ_{t|t-1}[j] * η_t[j])
Output: ξ_{t|t}[1] = P(s_t = stressed | data_1..t)
```

**Parameter estimation:** EM (Baum-Welch) or direct MLE via Nelder-Mead. Initial values: set μ_0/σ_0 from calm periods, μ_1/σ_1 from crisis periods (2008, 2020). Use log-space for numerical stability. Multiple random restarts to avoid local optima.

### LPPL fitting

**Model:**
```
ln P(t) = A + B·(tc-t)^m + C·(tc-t)^m·cos(ω·ln(tc-t) + φ)
```
Constraints: B < 0, 0.1 ≤ m ≤ 0.9, 6 ≤ ω ≤ 13, |C| ≤ 1, tc > t_end.

**Linearization (Filimonov & Sornette 2011):** For fixed tc, m, ω, define:
```
f1 = (tc-t)^m
f2 = (tc-t)^m · cos(ω·ln(tc-t))
f3 = (tc-t)^m · sin(ω·ln(tc-t))
```
Solve `ln P = A + B·f1 + C1·f2 + C2·f3` by OLS. Then `C = √(C1²+C2²)`, `φ = atan2(-C2, C1)`.

**Procedure:** Grid search over (tc, m, ω). For each candidate, solve OLS, check constraints, record RMSE. Retain fits meeting all constraints. Report distribution of tc across valid fits; tight clustering = stronger signal. Minimum data window: 250 observations.

### Critical slowing down

For signal x_t, rolling window W = 252 days:
```
AR(1) coefficient:  fit x_t = α + β·x_{t-1} + ε via OLS → β_t
Rolling variance:   var_t = variance(x_{t-W+1} .. x_t)
CSD warning:        β_t trending toward 1 AND var_t trending up simultaneously
```
Combined indicator: track 63-day rate of change of both β and var. Dual increase = elevated risk. Apply to credit spreads, VIX, and the largest eigenvalue of the cross-asset correlation matrix.

### Hawkes process branching ratio

**Model:**
```
λ(t) = μ + α · Σ_{t_i < t} exp(-β·(t - t_i))
Branching ratio: n = α/β   (n < 1 = stable; n → 1 = critical)
```

**MLE:**
```
log L = Σ_i ln λ(t_i) - ∫_0^T λ(t) dt
      = Σ_i ln λ(t_i) - μ·T - (α/β)·Σ_i (1 - exp(-β·(T - t_i)))
```
Analytic gradient available. Optimize with L-BFGS-B. Bounds: μ > 0, α ≥ 0, β > 0.

**Event definition:** VIX threshold crossings `VIX_t > VIX_{t-1} + 1.5·σ`, or large order flow imbalances. Estimate on 252-day rolling window. Rolling n approaching 1 is a warning signal.

## Scoring (proposed)

We combine normalized signals into a regime score in three steps:

1. Normalize each signal to a common scale (`z` or `pct`).
2. Aggregate into thematic buckets.
3. Combine buckets into a single regime score, then map to labels.

### Buckets

Each bucket score is the mean of its member signals (after normalization).

- **Risk premium**: VRP, tail-risk premium
- **Liquidity**: funding spreads, TED, Amihud, order-book depth, order flow imbalance
- **Volatility**: VIX slope, SKEW, MOVE, VVIX
- **Credit/macro**: credit spreads, yield curve/term premium, stress indices
- **Structure/flows**: cross-asset correlation, COT positioning, ETF flows
- **Contagion**: CoVaR, MES, SRISK
- **Sentiment**: FOMC tone, news-based uncertainty

### Regime score

Let each bucket score be in z-space. Define:

```
regime_score = -0.25*risk_premium
               -0.20*liquidity
               -0.15*volatility
               -0.15*credit_macro
               -0.10*structure_flows
               -0.10*contagion
               -0.05*sentiment
```

Negative weights mean higher stress → more risk-off. Weights are a starting point; re-weight after backtesting.

### Labels

- `risk_on` if `regime_score >= +0.50`
- `neutral` if `-0.50 < regime_score < +0.50`
- `risk_off` if `regime_score <= -0.50`

Also expose:

- `confidence = min(1, |regime_score| / 2)`
- `contributors`: top-3 buckets by absolute impact

## Evaluation (proposed)

A regime signal is only useful if it predicts something. Candidate target variables and evaluation approaches:

### Target variables

- **Forward drawdown**: does a risk-off reading predict above-average drawdown in the next 1–4 weeks?
- **Sharpe ratio by regime**: is realized Sharpe higher in risk-on periods and lower or negative in risk-off?
- **Crisis coincidence**: do known stress episodes (2008, 2011 EU crisis, 2020 COVID, 2022 rate shock) register as risk-off before or during the event?

### Evaluation approach

1. Label historical periods using the scoring formula.
2. Compute forward returns and drawdowns conditional on each regime label.
3. Check calibration: does `confidence` correlate with realized severity?
4. Backtest a simple rule: hold equities in risk-on, hold cash/bonds in risk-off. Compare Sharpe and max drawdown to buy-and-hold.
5. Re-weight bucket coefficients to maximize out-of-sample Sharpe using a held-out period.

### Signal lead/lag properties

Signals differ in how quickly they react to regime transitions. A rough taxonomy:

- **Fast / contemporaneous**: VIX, VVIX, order flow imbalance, SRISK — move with or slightly before price
- **Medium**: credit spreads, funding spreads, cross-asset correlation, FOMC tone — days to weeks
- **Slow / lagging**: COT positioning, ETF flows, CFTC data — weekly release cadence, structural rather than tactical
- **Potentially leading**: critical slowing down indicators (rising variance + autocorrelation), LPPL fit, branching ratio from Hawkes model

In the scoring formula, fast signals should dominate for short-horizon regime calls; slower signals provide structural context.

## Implementation notes

### Frequency alignment

Signals update at different cadences. Maintain all in daily resolution:

| Cadence | Examples | Treatment |
|---------|----------|-----------|
| Daily | VIX, credit spreads, OFR FSI, SOFR, yield curve | Use directly |
| Weekly | STLFSI (released Friday), COT (Tuesday data, Friday release) | Forward-fill to daily |
| Monthly | ETF flows, ACM term premium, macro series | Forward-fill to daily |

Mark each signal with a `last_updated` timestamp. If a signal is stale > 5 business days, treat as missing.

### Release delays and look-ahead bias

Many series are released with a lag. Using tomorrow's data today is look-ahead bias and will invalidate any backtest.

| Signal | Lag | Notes |
|--------|-----|-------|
| FRED daily series | ~1 business day | |
| OFR FSI | ~1 business day | |
| CFTC COT | 3 days | Tuesday data released Friday |
| FOMC minutes | ~3 weeks after meeting | |
| ETF flows | 1 business day | |
| STLFSI | Same day (Friday) | |

For backtesting, always index by the date the data became available, not the observation date.

### Missing data

Sources of missing values: weekends/holidays, API downtime, discontinued series, data before series start.

Strategy:
1. Forward-fill up to 5 business days.
2. Beyond 5 days: mark signal as `NaN`; exclude from bucket mean.
3. If > 50% of signals in a bucket are `NaN`: emit `NaN` for that bucket.
4. If > 50% of buckets are `NaN`: do not emit a regime label.
5. Never emit a regime label from insufficient data.

Discontinued series: TED spread (`TEDRATE`) ended April 2022. Substitute: SOFR vs. 3-month T-bill spread.

### Warmup periods

Rolling statistics require a minimum number of observations before they are meaningful:

| Statistic | Min observations |
|-----------|-----------------|
| Rolling z-score (daily) | 63 (3 months) |
| Rolling percentile | 126 (6 months) |
| Hamilton filter | 252 (1 year) |
| LPPL | 250 |
| Critical slowing down AR(1) | 126 |
| Hawkes MLE | 100 events |

Full signal availability for FRED-based signals starts around 2005 (limited by STLFSI and ICE BofA spread series). For backtesting from 1990, expect many signals to be unavailable.

### Numerical stability

**Hamilton filter:** work in log-space. Use `log_sum_exp` for normalization. Clip filtered probabilities to `[1e-10, 1 - 1e-10]`.

**LPPL:** `(tc - t)^m` blows up when `tc - t → 0`. Floor at `max(tc - t, 1e-10)`. Use double precision throughout.

**Hawkes MLE:** the integral term can underflow for long series. Use compensated summation (Kahan) for numerical accuracy.

**Rolling variance:** use Welford's online algorithm for numerically stable one-pass computation.

**Z-score:** if `σ = 0` (constant signal), emit `NaN` rather than dividing by zero.

### Rust crate suggestions

| Task | Crate |
|------|-------|
| HTTP / FRED API | `reqwest`, `ureq` |
| JSON parsing | `serde_json` |
| DataFrame / time series | `polars` |
| Linear algebra | `nalgebra`, `ndarray` |
| Optimization (L-BFGS-B for Hawkes) | `argmin` |
| Statistics | `statrs` |
| Date handling | `chrono` |
| Parallel computation | `rayon` |

## References

### Variance risk premium / tail-risk premia

- Bollerslev, Tauchen, Zhou (2009). Expected stock returns and variance risk premia. Review of Financial Studies. https://scholars.duke.edu/individual/pub732839
- Bollerslev, Todorov, Xu (2015). Tail risk premia and return predictability. Journal of Financial Economics. https://scholars.duke.edu/publication/1060835
- Bollerslev, Marrone, Xu, Zhou (2014). Stock return predictability and variance risk premia: international evidence. Journal of Financial and Quantitative Analysis. https://www.cambridge.org/core/journals/journal-of-financial-and-quantitative-analysis/article/abs/stock-return-predictability-and-variance-risk-premia-statistical-inference-and-international-evidence/0BE5DE1D942A0342DDBA24D7BFBEA5C8
- Bekaert, Hoerova (2014). The VIX, the variance premium and stock market volatility. Journal of Econometrics. https://www.nber.org/papers/w18995

### Liquidity, funding liquidity, and regime shifts

- Brunnermeier, Pedersen (2009). Market Liquidity and Funding Liquidity. Review of Financial Studies. https://academic.oup.com/rfs/article/22/6/2201/1592184
- Pastor, Stambaugh (2003). Liquidity Risk and Expected Stock Returns. Journal of Political Economy (see NBER working paper page). https://www.nber.org/papers/w8462
- Han, Leika (2019). Integrating Solvency and Liquidity Stress Tests: The Use of Markov Regime-Switching Models. IMF Working Paper 2019/250. https://www.imf.org/en/publications/wp/issues/2019/11/15/integrating-solvency-and-liquidity-stress-tests-the-use-of-markov-regime-switching-models-48752
- Flood, Liechty, Piontek (2015). Systemwide Commonalities in Market Liquidity. OFR Working Paper. https://www.financialresearch.gov/working-papers/2015/05/28/systemwide-commonalities-in-market-liquidity/

### Liquidity criticality / impact

- Toth, Lemperiere, Deremble, et al. (2011). Anomalous Price Impact and the Critical Nature of Liquidity in Financial Markets. Physical Review X. https://journals.aps.org/prx/abstract/10.1103/PhysRevX.1.021006
- Donier, Bouchaud (2015). Why do markets crash? Bitcoin data offers unprecedented insights. PLoS ONE. https://doi.org/10.1371/journal.pone.0139356

### LPPL / critical crash diagnostics

- Sornette (1997). Large financial crashes. Physica A. https://doi.org/10.1016/S0378-4371(97)00318-X
- Wheatley, Sornette, Huber, Reppen, Gantner (2019). Are Bitcoin bubbles predictable? Royal Society Open Science. https://doi.org/10.1098/rsos.180538

### Financial stress indices

- Monin (2019). The OFR Financial Stress Index. Risks. https://www.mdpi.com/2227-9091/7/1/25
- Office of Financial Research. Financial Stress Index (FSI) methodology and data. https://www.financialresearch.gov/financial-stress-index/

### Regime-switching

- Hamilton (1989). A new approach to the economic analysis of nonstationary time series and the business cycle. Econometrica. https://doi.org/10.2307/1912559

### Term structure and bond risk premia

- Cochrane, Piazzesi (2005). Bond risk premia. NBER Working Paper. https://www.nber.org/papers/w9178
- Adrian, Crump, Moench (2013). Pricing the term structure with linear regressions. NY Fed Staff Report 340. https://www.newyorkfed.org/research/staff_reports/sr340.html

### Credit spreads & risk premia

- Gilchrist, Zakrajsek (2012). Credit spreads and business cycle fluctuations. American Economic Review. https://www.nber.org/papers/w17021

### Volatility term structure

- VIX term structure and contango/backwardation. Journal of Risk and Financial Management (2019). https://www.mdpi.com/1911-8074/12/3/113

### Market structure & crash propagation

- Menkveld, Yueshen (2022). The flash crash: The role of market fragmentation and high-frequency trading. https://ink.library.smu.edu.sg/lkcsb_research/7285/

### Critical slowing down and tipping points

- Scheffer, Carpenter, Foley, Folke, Walker (2001). Catastrophic shifts in ecosystems. Nature. https://doi.org/10.1038/35098000
- Scheffer et al. (2009). Early-warning signals for critical transitions. Nature. https://doi.org/10.1038/nature08227

### Hawkes processes in finance

- Bacry, Mastromatteo, Muzy (2015). Hawkes processes in finance. Market Microstructure and Liquidity. https://doi.org/10.1142/S2382626615500057

### Rough volatility

- Gatheral, Jaisson, Rosenbaum (2018). Volatility is rough. Quantitative Finance. https://doi.org/10.1080/14697688.2017.1393551

### Intermediary asset pricing

- He, Krishnamurthy (2013). Intermediary asset pricing. American Economic Review. https://doi.org/10.1257/aer.103.2.732
- Adrian, Shin (2014). Procyclical leverage and endogenous risk. Journal of Political Economy. https://doi.org/10.1086/682903

### Systemic risk measures

- Adrian, Brunnermeier (2016). CoVaR. American Economic Review. https://doi.org/10.1257/aer.20120555
- Acharya, Pedersen, Philippon, Richardson (2017). Measuring systemic risk. Review of Financial Studies. https://doi.org/10.1093/rfs/hhw088
- Brownlees, Engle (2017). SRISK: A conditional capital shortfall measure of systemic risk. Review of Financial Studies. https://doi.org/10.1093/rfs/hhw060

## People to watch

Researchers and groups to monitor for new papers in this space.

### Variance risk premium and realized volatility

- Tim Bollerslev
- George Tauchen
- Hao Zhou
- Viktor Todorov
- Lai Xu
- Geert Bekaert
- Marie Hoerova
- Torben Andersen
- Ole Barndorff-Nielsen
- Neil Shephard
- Nour Meddahi
- Francis X. Diebold
- Caio Almeida
- Yacine Aït-Sahalia

### Fat tails, power laws, and crash hazard

- Didier Sornette
- Jean-Philippe Bouchaud
- Niklas Wheatley
- Tobias Huber
- Max Reppen
- Robert N. Gantner
- Xavier Gabaix
- Nassim Nicholas Taleb
- Paul Embrechts
- Rama Cont

### Volatility term structure and options

- Liuren Wu
- Gurdip Bakshi
- Peter Carr
- Jin-Chuan Duan

### Liquidity and funding stress

- Markus Brunnermeier
- Lasse Heje Pedersen
- Lubos Pastor
- Robert Stambaugh
- Hyun Song Shin
- Darrell Duffie
- Arvind Krishnamurthy
- Dimitri Vayanos
- Viral Acharya
- Nicolae Garleanu
- Zhiguo He

### Market microstructure and price impact

- Bence Toth
- Yvan Lemperiere
- Cecile Deremble
- Jonathan Donier
- Albert J. Menkveld
- Yueshen Bian
- Albert Kyle
- Thierry Foucault
- Maureen O'Hara
- Tarun Chordia
- Terrence Hendershott
- Marc Potters
- Fabrizio Lillo
- Jim Gatheral
- Mathieu Rosenbaum
- Charles-Albert Lehalle

### Financial stress indices and systemic risk

- Phillip Monin
- Michael Flood
- John Liechty
- Krista Piontek
- Jan Hatzius
- Stijn Claessens
- Robert Engle
- Stefano Giglio
- Bryan Kelly
- Monica Billio
- Andrew Lo
- Olivier Scaillet
- Markus Pelger
- Jon Danielsson

### Regime-switching

- James Hamilton
- Andrew Ang
- Allan Timmermann
- Roger Farmer
- Lars Hansen

### Credit spreads and macro

- Simon Gilchrist
- Egon Zakrajsek
- Mark Gertler
- Ben Bernanke
- Robert Merton
- Francis Longstaff
- John Geanakoplos
- Ricardo Reis

### Term structure and bond risk premia

- John Cochrane
- Monika Piazzesi
- Tobias Adrian
- Richard Crump
- Emanuel Moench
- John Campbell
