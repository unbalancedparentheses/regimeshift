# regimeshift

A research project for detecting market regime shifts — transitions between risk-on and risk-off states — by combining signals from multiple traditions: variance risk premium, liquidity and funding stress, tail-risk and crash diagnostics, credit and macro indicators, and systemic risk measures. The goal is transparent, inspectable signals grounded in the academic literature, not black-box scores.

## Contents

- [Goals](#goals)
- [Theoretical frameworks](#theoretical-frameworks)
- [Financial regimes and the business cycle](#financial-regimes-and-the-business-cycle)
- [International regimes and contagion](#international-regimes-and-contagion)
- [Data sources](#data-sources)
- [Signals](#signals)
- [Algorithm details](#algorithm-details)
- [Scoring](#scoring)
- [Evaluation](#evaluation)
- [Implementation notes](#implementation-notes)
- [Project](#project)
- [References](#references)
- [People to watch](#people-to-watch)

## Goals

- Provide transparent, inspectable signals grounded in the academic literature — not black boxes.
- Combine multiple families of indicators:
  - Macro stress (financial stress indices: OFR FSI, STLFSI)
  - Variance risk premium (implied vs realized variance)
  - Volatility regime (VIX term structure, skew, MOVE, VVIX)
  - Liquidity and funding stress (spreads, Amihud, order flow imbalance)
  - Credit and macro (credit spreads, yield curve, term premium)
  - Systemic risk and contagion (CoVaR, MES, SRISK)
  - Cross-asset structure (correlations, positioning, flows)
  - Text and sentiment (FOMC tone, news uncertainty)
- Emit a regime label (risk-on / neutral / risk-off) with confidence score and per-bucket contributors.
- Support evaluation against known stress episodes and recession indicators.
- Ground all signals, algorithms, and design choices in the academic literature.

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

### Minsky's Financial Instability Hypothesis

Hyman Minsky argued that stability is destabilizing: long periods of calm encourage risk-taking, leverage, and the migration of borrowers from hedge finance (cash flows cover debt service) to speculative finance (cash flows cover only interest) to Ponzi finance (cash flows cover neither — survival depends on asset appreciation). This endogenous build-up of fragility eventually collapses under its own weight.

This is the conceptual precursor to Brunnermeier's funding liquidity spirals and Geanakoplos's leverage cycle. Minsky's framework suggests that regime-shift risk is highest after extended periods of low volatility and tight credit spreads — precisely when most models signal calm. Practically: low VRP, compressed credit spreads, and rising leverage ratios together are a Minsky-style warning even when no individual signal crosses a threshold.

### Financial network contagion (Acemoglu, Ozdaglar, Elliott, Golub, Jackson)

The topology of the financial network determines whether idiosyncratic shocks amplify into systemic crises. Key result: dense interconnection is "robust yet fragile" — the network absorbs small shocks but amplifies large ones past a critical threshold. When a major node fails, losses cascade through counterparty exposures.

DebtRank (Battiston et al. 2012) quantifies each node's systemic importance: how much economic value is lost if it defaults. Unlike CoVaR and SRISK, which measure statistical co-movement, DebtRank captures the transmission mechanism through the balance-sheet network.

This framework explains *why* funding stress, credit spreads, and equity volatility co-move at regime transitions: the network of counterparty exposures transmits and amplifies the initial shock. Practical signal: aggregate DebtRank or eigenvector centrality of major financial institutions; rising concentration of centrality = increasing fragility.

## Financial regimes and the business cycle

Financial regimes are not the same as business cycles, but they are closely related.

### Distinctions

- Not all recessions follow financial stress: the 2020 COVID shock was an external event with financial amplification, not a financial cycle peak.
- Not all financial stress causes recessions: the 2011 EU sovereign crisis was severe stress in Europe but only a soft patch in the US.
- Financial regime signals aim to be real-time; NBER recession dating is ex-post by 6–12 months.
- Financial regimes operate at shorter horizons (weeks to months); business cycles at longer horizons (years).

### Yield curve as recession predictor

The 10Y-3M spread (FRED: `T10Y3M`) has inverted before every US recession since 1960 with a lead of ~12 months. Estrella and Mishkin (1998) formalize this as a probit model. The 10Y-2Y spread is more widely watched but 10Y-3M has stronger empirical predictive power. A spread below zero for 3+ months implies elevated recession probability.

### Financial conditions indices

The Chicago Fed NFCI (FRED: `NFCI`, weekly) aggregates 105 indicators across money markets, debt markets, equity markets, and shadow banking. Positive = tighter than average. Correlation with OFR FSI is high but they capture different aspects; using both adds coverage.

### Typical sequence in a financially-driven recession

1. Credit spreads widen (6–18 months before recession onset)
2. Yield curve inverts (12–18 months before onset)
3. Equity market peaks (6–9 months before onset)
4. Financial stress indices spike (contemporaneous with onset)
5. NBER calls the recession start (6–12 months after onset)

A risk-off regime signal does not guarantee recession; it signals elevated probability. Use FRED `USREC` as the primary validation benchmark: does risk-off predict `USREC=1` in the subsequent 3–12 months at above-base-rate frequency?

## International regimes and contagion

Regime shifts propagate across borders through three channels:

1. **Trade**: recession in one country reduces demand for exports, spreading weakness.
2. **Financial**: common lenders, cross-border holdings, global funding markets. A US money market fund run (2008, 2020) instantly affects European bank funding.
3. **Information/confidence**: a shock in one market updates beliefs globally, triggering correlated repositioning even where fundamentals are unchanged.

### Twin crises and sudden stops

Kaminsky and Reinhart (1999) document the "twin crises" pattern: banking crises and currency crises tend to occur together and reinforce each other. Leading indicators: reserve depletion, M2/reserves ratio, credit growth, current account deficit, real exchange rate overvaluation.

Calvo (1998) defines sudden stops as large rapid reversals in capital inflows — the primary crisis trigger in emerging markets. Vulnerability indicators: current account deficit size, foreign currency debt share, reserve adequacy.

### BIS credit-to-GDP gap

The BIS credit-to-GDP gap (credit/GDP relative to its long-run trend) is the single best cross-country leading indicator of banking crises. The empirical basis for Basel III countercyclical capital buffer requirements. A gap above +10pp signals elevated banking crisis risk within 1–3 years. Available quarterly from https://www.bis.org/statistics/c_gaps.htm.

### Contagion vs. fundamentals

Forbes and Rigobon (2002) show that apparent cross-country correlation increases during crises largely disappear after correcting for heteroscedasticity — what looks like contagion is often common fundamentals being revealed simultaneously. True contagion is propagation beyond what fundamentals justify. This matters for signal interpretation: a cross-asset correlation spike may be information, not amplification.

### Cross-country data sources

- BIS credit-to-GDP gap: https://www.bis.org/statistics/c_gaps.htm
- IMF Global Financial Stability Report: https://www.imf.org/en/Publications/GFSR
- Chicago Fed NFCI: FRED `NFCI`

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
| Chicago Fed NFCI | `NFCI` | Weekly | Financial conditions index; positive = tighter than average |

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

### CFTC COT data

Weekly Commitments of Traders reports: https://www.cftc.gov/MarketReports/CommitmentsofTraders/index.htm
Published every Friday for positions as of the prior Tuesday (3-day lag). Download as CSV or use the CFTC API. Key fields: net speculative positions (large traders, non-commercial) for equity index, Treasury, and commodity futures.

## Signals

Signals are grouped into thematic families. Each entry lists the data source, raw measurement, and normalization method. Not all signals are available for free; see [Data sources](#data-sources).

**Conventions**

- `value`: raw measurement
- `z`: z-score over a rolling window (default 3Y if daily, 5Y if weekly)
- `pct`: percentile rank over a rolling window

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
  - Source: ICE BofA MOVE; not on FRED; typically via Bloomberg or paid data providers
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
  - Note: prefer SOFR-OIS over FRA-OIS for pure funding stress. FRA-OIS also embeds term premium and rate expectations, which can produce false signals during normal rate cycles. The FT documented this distortion during the 2018 Libor transition period.
  - Note: SOFR uses daily compounding in arrears (backward-looking), unlike LIBOR's forward-looking panel estimates. This design amplifies real shocks mechanically — the Sept 2019 repo crisis produced a 12.5 bps one-day SOFR spike (8+ sigma for a benchmark rate). Interpret extreme SOFR-OIS spikes with awareness that they may reflect plumbing dysfunction rather than broad funding stress.
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
- Cross-currency basis (xccy basis)
  - Source: Bloomberg or derived from FX forwards and OIS rates (no free equivalent)
  - Raw: deviation from covered interest rate parity, e.g., USD/EUR 3-month basis in bps
  - Normalize: `z`
  - Note: a negative USD basis means dollar funding via FX swaps is more expensive than via the interbank market — a sign of global dollar shortage. Captures international stress that domestic SOFR-OIS misses. Fired in 2008, 2011 EU crisis, and March 2020.
- Order flow imbalance
  - Source: exchange trade data or order book
  - Raw: signed order flow (buy-initiated minus sell-initiated volume), normalized by total volume
  - Normalize: `z`
- Treasury realized vol / equity realized vol ratio
  - Source: compute from daily returns of 2Y or 10Y Treasury vs. SPX
  - Raw: ratio of rolling 21-day realized vol (Treasury) to rolling 21-day realized vol (equity)
  - Normalize: `z`
  - Note: normally well below 1.0. When Treasury realized vol exceeds equity vol, something structural is breaking — the "safe haven" is no longer safe. This inversion occurred in Aug 2019 (2Y Treasury realized vol ran at 2x equity vol) and during parts of the 2022 rate shock. Captures regime-relevant information that MOVE alone does not, because the ratio accounts for the relative magnitude rather than the absolute level.

**Credit and macro**

- Credit spreads
  - Source: FRED `BAMLH0A0HYM2` (HY OAS), `BAMLC0A4CBBB` (BBB OAS), `BAA`-`AAA` (Moody's)
  - Raw: spread in bps
  - Normalize: `z`
- Excess bond premium (EBP)
  - Source: Gilchrist-Zakrajsek data file from Federal Reserve: https://www.federalreserve.gov/econres/feds/updating-the-recession-risk-and-the-excess-bond-premium.htm
  - Raw: residual corporate spread after removing the component explained by firm-level default risk (leverage, volatility, interest coverage)
  - Normalize: `z`
  - Note: EBP is the most predictive component of the GZ credit spread for recessions. When it rises, financial conditions are tightening purely from risk appetite compression — independent of actual default risk. Strictly more informative than the raw spread for regime detection.
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
- Cross-asset correlation eigenvalue
  - Source: rolling covariance matrix of equity, credit, rates, commodities
  - Raw: largest eigenvalue of rolling correlation matrix (concentration = systemic co-movement)
  - Normalize: `z`
- Price trend (time-series momentum)
  - Source: returns of equity, bond, commodity, and FX indices
  - Raw: 12-month return excluding the most recent month (r_{t-12, t-1}) for each asset class
  - Normalize: `z`
  - Note: persistent positive trend across risk assets = risk-on; simultaneous trend reversals or negative 12-month returns across multiple asset classes = risk-off warning. Moskowitz, Ooi, Pedersen (2012) show trend following is most effective during crisis periods — the signal earns its strongest returns exactly when diversification fails.

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
  - Note: pre-computed MES and SRISK data available from NYU Stern V-Lab: https://vlab.stern.nyu.edu/

**Text and sentiment**

- FOMC communication tone
  - Source: FOMC minutes and statements (NLP)
  - Raw: hawkish/dovish score or policy uncertainty index
  - Normalize: `z`
  - Note: use the Loughran-McDonald (2011) financial sentiment dictionary for domain-appropriate word scoring (general-purpose dictionaries misclassify financial language). Master dictionary: https://sraf.nd.edu/loughranmcdonald-master-dictionary/
- News-based uncertainty
  - Source: Baker-Bloom-Davis Economic Policy Uncertainty index or similar NLP pipeline
  - Raw: aggregate uncertainty or sentiment score
  - Normalize: `z`

**Momentum / trend (future consideration)**

Not currently a scored bucket, but cross-asset time-series momentum is a candidate for future inclusion. Jegadeesh-Titman (1993) established cross-sectional momentum; Moskowitz-Ooi-Pedersen (2012) showed time-series momentum earns its strongest returns during and after crisis periods. A simple signal: 12-1 month return on a diversified risk-asset basket turning negative is a regime warning. Trend-following strategies (Rattray et al. 2019) have positive convexity — they naturally profit during sustained regime transitions — making them both a signal source and an evaluation benchmark. Not prioritized for MVP because momentum is slow (monthly cadence) and the project already captures the faster signals that precede momentum turning negative.

## Algorithm details

Computation details for each major signal type. All formulas use log returns `r_t = ln(P_t / P_{t-1})`.

### VIX term structure slope

VIX futures settle monthly on the Wednesday 30 days before the standard monthly SPX expiration. Download settlement prices from Cboe (see Data sources).

```
# Label futures by days to expiration (DTE)
front_month_vix  = settlement price of nearest contract (DTE < 30)
second_month_vix = settlement price of next contract

# Roll-adjusted slope (interpolate to fixed 30-day tenor)
w = DTE_front / 30
constant_maturity_30d = w * front_month_vix + (1 - w) * second_month_vix

# Term structure slope
slope_t = VIX9D - VIX3M   # short end vs. 3-month (if available)
       OR = front_month_vix - second_month_vix   # front vs. second month
```

Interpretation: negative slope (backwardation) = near-term fear exceeds medium-term = stress. Positive slope (contango) = normal carry. Normalize with `z`.

### Cross-asset correlation eigenvalue

The largest eigenvalue of the rolling cross-asset correlation matrix measures systemic co-movement. When it rises sharply, diversification breaks down — a signature of stress regimes.

```
# Assets: equity (SPX), credit (HY spread), rates (10Y yield), commodities (e.g., oil)
# Build rolling correlation matrix C_t from W-day returns of each asset class

C_t = rolling_correlation_matrix([equity, credit, rates, commodities], window=W)

# Largest eigenvalue via power iteration or full eigen decomposition
λ_max_t = max eigenvalue of C_t

# Fraction of variance explained (normalized)
λ_frac_t = λ_max_t / sum(eigenvalues of C_t)   # = λ_max / n_assets at perfect correlation
```

Default W = 63 days. λ_frac approaching 1.0 means all assets moving together — crisis regime. Normalize λ_frac with `z`. Also apply critical slowing down (rising variance + AR(1)) to λ_frac itself as an early warning.

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

### Jump risk / tail-risk premium

The variance risk premium conflates two components: compensation for continuous vol fluctuations and compensation for jump risk. Bollerslev and Todorov (2011) show that the jump component dominates return predictability.

**Step 1: Decompose realized variance using BNS bipower variation (Barndorff-Nielsen, Shephard 2004).**

Bipower variation is robust to jumps; realized variance is not. Their difference estimates jump variance:
```
# Bipower variation (using adjacent absolute returns, scaled)
BV_t = (π/2) * (1/(W-1)) * Σ_{i=t-W+2}^{t} |r_i| * |r_{i-1}|

# Continuous component (jump-robust)
CV_t = BV_t

# Jump component (floored at 0 to avoid negative estimates)
JV_t = max(RV_t - BV_t, 0)
```
The BNS jump test statistic (for detecting whether JV_t is significant on a given day):
```
z_jump = (RV_t - BV_t) / sqrt(variance_of_RV - BV_t)
```
Use the realized tri-power quarticity for the denominator variance (see Barndorff-Nielsen, Shephard 2004 for the full formula).

**Step 2: Estimate implied jump variance from options.**

A model-free approach (Bollerslev-Todorov 2011) decomposes the VRP using deep out-of-the-money (OTM) option prices, which primarily reflect jump risk:
```
# Implied total variance (from VIX construction)
IV_t = (VIX_t / 100)² / 12   # monthly

# Implied jump variance ≈ price of deep OTM puts + calls above/below threshold k*
# Requires option prices at strikes below ~80% and above ~120% of spot
# Not available without access to full options chain
```

**Practical approximation** when full option chain is unavailable:
```
# Use high-frequency data to estimate BV and JV
# Jump risk premium ≈ JV_t (realized jump variance as proxy for jump fear)
JRP_t = JV_t   # jump risk premium proxy
```

Interpretation: JRP spikes around market dislocations (Flash Crash, Lehman, COVID). It rises faster than continuous vol during sudden regime transitions, making it a useful leading indicator of tail-event regimes. Normalize with `pct`.

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
Initialize: ξ_{0|0} = [0.5, 0.5] (or uniform prior); update to stationary distribution after parameter estimation converges
For t = 1..T:
  Predict: ξ_{t|t-1}[j] = Σ_i P[i,j] * ξ_{t-1|t-1}[i]
  Densities: η_t[j] = N(y_t; μ_j, σ_j²)
  Update: ξ_{t|t} = (ξ_{t|t-1} ⊙ η_t) / Σ_j(ξ_{t|t-1}[j] * η_t[j])
Output: ξ_{t|t}[1] = P(s_t = stressed | data_1..t)
```

**Parameter estimation:** EM (Baum-Welch) or direct MLE via Nelder-Mead. Initial values: set μ_0/σ_0 from calm periods, μ_1/σ_1 from crisis periods (2008, 2020). Use log-space for numerical stability. Multiple random restarts to avoid local optima.

**Number of states:** The canonical model uses K=2 states (normal vs. stressed). A K=3 model (expansion / neutral / contraction) adds resolution but requires substantially more data and has K*(K-1) = 6 free transition parameters vs. 2 for K=2. Use K=3 only if you have strong evidence that a third state is empirically distinct. More states increase overfitting risk. For a regime indicator, K=2 is the safe default; validate with AIC/BIC and out-of-sample persistence of inferred regimes.

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

**Structured product mechanics as self-excitation:** The February 2018 Volmageddon event is a clean example of an endogenous Hawkes cascade. Inverse-VIX ETPs (XIV, SVXY) had a structural rebalancing rule: at end of day they must sell VIX futures proportional to any intraday VIX rise to maintain their inverse exposure. A moderate afternoon VIX spike triggered their selling, which pushed VIX higher, which triggered more selling — a textbook self-exciting process. The branching ratio briefly exceeded 1 before the products terminated. This dynamic can appear whenever large structured-product AUM has mechanical hedging rules tied to the same underlying, and it is distinct from fundamental-driven stress. Watch for unusual concentration of delta-hedging flows in VIX futures open interest as an early indicator.

## Scoring

We combine normalized signals into a regime score in three steps:

1. Normalize each signal to a common scale (`z` or `pct`).
2. Aggregate into thematic buckets.
3. Combine buckets into a single regime score, then map to labels.

### Buckets

Each bucket score is the mean of its member signals (after normalization).

- **Risk premium**: VRP, tail-risk premium
- **Liquidity**: funding spreads, TED, cross-currency basis, Amihud, order-book depth, order flow imbalance
- **Volatility**: VIX slope, SKEW, MOVE, VVIX
- **Credit/macro**: credit spreads, EBP, yield curve/term premium, stress indices
- **Structure/flows**: cross-asset correlation, price trend, COT positioning, ETF flows
- **Contagion**: CoVaR, MES, SRISK
- **Sentiment**: FOMC tone, news-based uncertainty

**Signal correlation within buckets:** Signals in the same bucket are often highly correlated — VIX slope, SKEW, MOVE, and VVIX typically co-move; HY OAS and BBB OAS move together. Simple averaging treats them as independent and implicitly double-counts the same underlying factor. Three options: (a) pick the single most reliable signal per bucket and use it directly — simplest and works for an MVP; (b) apply PCA within each bucket and use the first principal component; (c) down-weight any pair with rolling pairwise correlation above 0.8. Start with option (a).

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

Negative weights mean higher stress → more risk-off. These weights are illustrative priors, not calibrated values. Equal weights (1/7 ≈ 0.14 each) are a defensible null hypothesis and the right baseline before you have data to justify departing from them.

**Bucket renormalization when data is missing:** If one or more buckets are NaN, renormalize the remaining weights to sum to 1 before computing the score — otherwise the score is systematically biased toward zero whenever buckets drop out:

```
available = [(w, b) for (w, b) in weights_and_buckets if bucket b is not NaN]
weight_sum = sum(w for w, _ in available)
regime_score = sum(-w / weight_sum * b for w, b in available)
```

**Note on VRP sign:** The risk_premium bucket includes VRP with a negative weight, so high VRP → more risk-off. This is defensible for real-time regime detection — high VRP reflects fear and variance demand — but the Bollerslev et al. (2009) literature frames high VRP as a *positive* forward return predictor (risk-on). These are not contradictory: high VRP can simultaneously signal current fear and predict future recovery. Be aware of this dual interpretation when evaluating the model.

### Labels

- `risk_on` if `regime_score >= +0.50`
- `neutral` if `-0.50 < regime_score < +0.50`
- `risk_off` if `regime_score <= -0.50`

Also expose:

- `confidence = min(1, |regime_score| / 2)`
- `contributors`: top-3 buckets by absolute impact

### Regime persistence

Raw thresholds applied to the regime score produce noisy label switching when the score hovers near ±0.50. Three practical approaches, in order of increasing complexity:

**1. Exponential smoothing** (recommended starting point):
```
score_smooth_t = α * regime_score_t + (1 - α) * score_smooth_{t-1}
```
Apply threshold rules to `score_smooth` instead of the raw score. α = 0.1 gives a ~10-day half-life (slow, suited to weekly-cadence signals); α = 0.3 gives a ~2-day half-life (faster response). One parameter, easy to reason about.

**2. Minimum duration (confirmation):**
Only flip the label when the candidate regime has held for N consecutive days. N = 5 (one week) prevents transient spikes from triggering transitions. Drawback: adds lag at genuine turning points.

**3. Hysteresis bands:**
Use asymmetric thresholds — a tighter band to stay in a regime than to enter it:
```
Enter risk-off:  score_smooth < -0.75
Exit risk-off:   score_smooth > -0.25
Enter risk-on:   score_smooth > +0.75
Exit risk-on:    score_smooth < +0.25
Neutral:         all else
```
Most robust against noise but requires four threshold parameters.

Start with option 1 (α = 0.2). This eliminates most flip-flopping with a single parameter and minimal lag cost.

## Evaluation

A regime signal is only useful if it predicts something. Candidate target variables and evaluation approaches:

### Target variables

- **Forward drawdown**: does a risk-off reading predict above-average drawdown in the next 1–4 weeks?
- **Sharpe ratio by regime**: is realized Sharpe higher in risk-on periods and lower or negative in risk-off?
- **Crisis coincidence**: do known stress episodes (2008, 2011 EU crisis, Feb 2018 Volmageddon, Sept 2019 repo crisis, 2020 COVID, 2022 rate shock) register as risk-off before or during the event? The single-bucket events (2018 vol-only, 2019 liquidity-only) are especially useful: do they produce brief, low-confidence risk-off from the correct bucket?

### Evaluation approach

1. Label historical periods using the scoring formula.
2. Compute forward returns and drawdowns conditional on each regime label.
3. Check calibration: does `confidence` correlate with realized severity?
4. Backtest a simple rule: hold equities in risk-on, hold cash/bonds in risk-off. Compare Sharpe and max drawdown to buy-and-hold and to a simple 12-1 time-series momentum strategy — the latter is a stronger benchmark since it also captures regime dynamics.
5. Backtest regime-aware rebalancing: use regime signal to delay rebalancing a 60/40 portfolio during risk-off (Rattray et al. 2019 show mechanical rebalancing has hidden negative convexity — it forces buying losers during trending drawdowns, amplifying max drawdown by ~5pp vs. buy-and-hold in 2008). Compare against: (a) monthly-rebalanced 60/40, (b) buy-and-hold 60/40, and (c) 60/40 with a 10% trend-following overlay (the positive convexity of trend offsets the negative convexity of rebalancing).
6. Re-weight bucket coefficients to maximize out-of-sample Sharpe using a held-out period.

**Overfitting warning:** with 7 buckets and ~20 underlying signals, re-weighting on historical data will overfit unless disciplined. Recommended approach: (a) use a long out-of-sample period (at least 10 years held out); (b) constrain weights to be positive and sum to 1; (c) prefer equal weighting as baseline and treat optimized weights with skepticism unless improvement is large and stable across sub-periods.

### Signal lead/lag properties

Signals differ in how quickly they react to regime transitions. A rough taxonomy:

- **Fast / contemporaneous**: VIX, VVIX, order flow imbalance, SRISK — move with or slightly before price
- **Medium**: credit spreads, funding spreads, cross-asset correlation, FOMC tone — days to weeks
- **Slow / lagging**: COT positioning, ETF flows, CFTC data — weekly release cadence, structural rather than tactical
- **Potentially leading**: critical slowing down indicators (rising variance + autocorrelation), LPPL fit, branching ratio from Hawkes model

In the scoring formula, fast signals should dominate for short-horizon regime calls; slower signals provide structural context.

### Historical calibration examples

These are sanity checks, not backtests: does each signal move in the expected direction during known episodes? Use this table to validate signal implementations before running a full evaluation.

| Signal | Sept 2008 (Lehman) | Mar 2020 (COVID) | Sept 2019 (repo crisis) | Feb 2018 (Volmageddon) | 2022 (rate shock) | 2013 (Taper Tantrum) |
|--------|---------------------|------------------|--------------------------|------------------------|-------------------|----------------------|
| VIX | ~80 (extreme) | ~85 (extreme) | ~18 (calm) | ~14 → ~37 (one day) | ~35 (elevated) | ~21 (moderate) |
| HY OAS | >1000 bps | ~1000 bps | ~400 bps (normal) | ~330 bps (tight) | ~600 bps | ~500 bps |
| 10Y-2Y | steepening (flight to safety) | steep | flat/inverted | normal | deeply inverted | flattening |
| SOFR-OIS / TED | ~450 bps | ~50 bps | overnight repo hit 600 bps | unchanged | ~20 bps | ~15 bps |
| VRP | negative (RV >> IV) | near-zero → negative | normal | brief inversion, then recovered | positive, elevated | positive |
| Cross-asset λ_frac | near 1.0 | near 1.0 | low | brief spike, equities only | moderate | low-moderate |
| STLFSI | extreme (+4 to +6) | extreme (+5) | mild | negligible | moderate (+1) | mild |
| **Expected regime** | **risk-off** | **risk-off** | **brief risk-off (liquidity only)** | **brief risk-off, rapid reversion** | **neutral to risk-off** | **neutral** |

Key observations:

- **2008 and 2020** are both unambiguous: all signal families fire simultaneously. The model should give high-confidence risk-off with no ambiguity across buckets.
- **Feb 2018 (Volmageddon)** is the purest structural vol event: the VIX spike was caused by forced buying from inverse-vol ETPs (XIV termination), not by credit or macro deterioration. HY spreads, SOFR-OIS, and STLFSI barely moved. The model should fire a brief risk-off signal from the volatility bucket alone and revert within days — a useful test of regime persistence logic.
- **2022** was a structural rate normalization, not a credit crisis. Credit spreads widened but not to crisis levels; funding stress (SOFR-OIS) stayed contained; VIX peaked around 35. The model should give moderate risk-off, not extreme — a useful check that the model doesn't over-react to rate moves alone.
- **2013 Taper Tantrum** was a brief false alarm: a rate spike reversed once the Fed signaled patience. Useful test case for signal decay and mean-reversion; the model should not stay risk-off for more than a few weeks.
- **VRP goes negative in acute crises** (2008, early 2020) because realized vol spikes faster than options reprice. Treat negative VRP as a stress signal in the regime context, not a contrarian buy signal.
- **Sept 2019 repo crisis** is the mirror image of Volmageddon: only the liquidity/funding bucket fires. Overnight repo rates hit 600 bps — the largest dislocation since 2008 — but VIX, credit spreads, and equity markets barely reacted. Caused by a structural reserve shortage (QT pushed bank reserves to the regulatory floor) colliding with corporate tax day and large Treasury settlement. The model should produce a brief, low-confidence risk-off driven entirely by the liquidity bucket. Also a key test of the SOFR-OIS signal: SOFR spiked ~12.5 bps in one day (an 8+ sigma event for a benchmark rate), demonstrating that SOFR's daily-compounding-in-arrears design can amplify mechanical shocks beyond what fundamentals justify.
- **Funding stress distinguishes 2008 from 2020**: the 2008 interbank market seized (TED ~450 bps); in 2020 the Fed intervened within days and funding spreads stayed contained relative to equity vol. This asymmetry is important for the liquidity bucket and confirms that no single bucket should dominate the regime call.

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

## Project

### Status

Scaffold only. Implementation TBD.

### Structure

- `src/` — Rust core for data ingestion, feature computation, and signal aggregation.
- `python/` — optional ML/AI experiments when Rust is not ideal.
- `data/` — optional cached datasets or example fixtures.

### Config

Example config: `config.example.toml` — covers API keys, rolling window lengths, thresholds, and smoothing parameters.

### Output

Each observation emits one record:

```
date          : ISO 8601 date
regime        : "risk_on" | "neutral" | "risk_off"
score         : float  # smoothed regime score
confidence    : float  # in [0, 1]
buckets:
  risk_premium  : float | null
  liquidity     : float | null
  volatility    : float | null
  credit_macro  : float | null
  structure_flows: float | null
  contagion     : float | null
  sentiment     : float | null
signals:
  vrp           : float | null
  hy_oas        : float | null
  ebp           : float | null
  vix_slope     : float | null
  # ... one entry per signal
```

`null` means the signal was unavailable or in warmup.

## References

### Variance risk premium / tail-risk premia

- Bollerslev, Tauchen, Zhou (2009). Expected stock returns and variance risk premia. Review of Financial Studies. https://scholars.duke.edu/individual/pub732839
- Barndorff-Nielsen, Shephard (2002). Econometric analysis of realized volatility and its use in estimating stochastic volatility models. Journal of the Royal Statistical Society B. https://doi.org/10.1111/1467-9868.00336
- Barndorff-Nielsen, Shephard (2004). Power and bipower variation with stochastic volatility and jumps. Journal of Financial Econometrics. https://doi.org/10.1093/jjfinec/nbh001
- Bhansali (2008). Offensive risk management: can tail risk hedging be profitable? Journal of Portfolio Management. https://papers.ssrn.com/sol3/papers.cfm?abstract_id=1573760
- Bhansali (2020). Monetization matters: active tail risk management and the Great Virus Crisis. Journal of Portfolio Management. https://papers.ssrn.com/sol3/papers.cfm?abstract_id=3668370
- Bhansali (2021). Tail risk hedging performance: measuring what counts. Journal of Portfolio Management. https://papers.ssrn.com/sol3/papers.cfm?abstract_id=3962552
- Bollerslev, Todorov (2011). Tails, fears, and risk premia. Journal of Finance. https://doi.org/10.1111/j.1540-6261.2011.01695.x
- Bollerslev, Todorov, Xu (2015). Tail risk premia and return predictability. Journal of Financial Economics. https://scholars.duke.edu/publication/1060835
- Bollerslev, Marrone, Xu, Zhou (2014). Stock return predictability and variance risk premia: international evidence. Journal of Financial and Quantitative Analysis. https://www.cambridge.org/core/journals/journal-of-financial-and-quantitative-analysis/article/abs/stock-return-predictability-and-variance-risk-premia-statistical-inference-and-international-evidence/0BE5DE1D942A0342DDBA24D7BFBEA5C8
- Bekaert, Hoerova (2014). The VIX, the variance premium and stock market volatility. Journal of Econometrics. https://www.nber.org/papers/w18995

### Liquidity, funding liquidity, and regime shifts

- Brunnermeier, Pedersen (2009). Market Liquidity and Funding Liquidity. Review of Financial Studies. https://academic.oup.com/rfs/article/22/6/2201/1592184
- Pastor, Stambaugh (2003). Liquidity Risk and Expected Stock Returns. Journal of Political Economy (see NBER working paper page). https://www.nber.org/papers/w8462
- Han, Leika (2019). Integrating Solvency and Liquidity Stress Tests: The Use of Markov Regime-Switching Models. IMF Working Paper 2019/250. https://www.imf.org/en/publications/wp/issues/2019/11/15/integrating-solvency-and-liquidity-stress-tests-the-use-of-markov-regime-switching-models-48752
- Flood, Liechty, Piontek (2015). Systemwide Commonalities in Market Liquidity. OFR Working Paper. https://www.financialresearch.gov/working-papers/2015/05/28/systemwide-commonalities-in-market-liquidity/
- Financial Times (2018). Do not trust the FRA-OIS spread! https://www.ft.com/content/5b0198fe-3035-3104-8262-c9723b96eb90 — argues FRA-OIS embeds term premium and rate expectations beyond pure funding stress; relevant caveat for SOFR-OIS vs. FRA-OIS signal choice.

### Liquidity criticality / impact

- Toth, Lemperiere, Deremble, et al. (2011). Anomalous Price Impact and the Critical Nature of Liquidity in Financial Markets. Physical Review X. https://journals.aps.org/prx/abstract/10.1103/PhysRevX.1.021006
- Donier, Bouchaud (2015). Why do markets crash? Bitcoin data offers unprecedented insights. PLoS ONE. https://doi.org/10.1371/journal.pone.0139356

### LPPL / critical crash diagnostics

- Sornette (1997). Large financial crashes. Physica A. https://doi.org/10.1016/S0378-4371(97)00318-X
- Filimonov, Sornette (2011). A stable and robust calibration scheme of the log-periodic power law model. Physica A. https://doi.org/10.1016/j.physa.2011.05.046
- Wheatley, Sornette, Huber, Reppen, Gantner (2019). Are Bitcoin bubbles predictable? Royal Society Open Science. https://doi.org/10.1098/rsos.180538

### Financial stress indices

- Monin (2019). The OFR Financial Stress Index. Risks. https://www.mdpi.com/2227-9091/7/1/25
- Office of Financial Research. Financial Stress Index (FSI) methodology and data. https://www.financialresearch.gov/financial-stress-index/

### Regime-switching

- Hamilton (1989). A new approach to the economic analysis of nonstationary time series and the business cycle. Econometrica. https://doi.org/10.2307/1912559
- Ang, Bekaert (2002). International asset allocation with regime shifts. Review of Financial Studies. https://doi.org/10.1093/rfs/15.4.1137
- Bai, Perron (1998). Estimating and testing linear models with multiple structural changes. Econometrica. https://doi.org/10.2307/2998540

### Term structure and bond risk premia

- Cochrane, Piazzesi (2005). Bond risk premia. NBER Working Paper. https://www.nber.org/papers/w9178
- Adrian, Crump, Moench (2013). Pricing the term structure with linear regressions. NY Fed Staff Report 340. https://www.newyorkfed.org/research/staff_reports/sr340.html

### Credit spreads & risk premia

- Gilchrist, Zakrajsek (2012). Credit spreads and business cycle fluctuations. American Economic Review. https://www.nber.org/papers/w17021

### Volatility term structure

- Whaley (2000). The investor fear gauge. Journal of Portfolio Management. https://doi.org/10.3905/jpm.2000.319728
- Carr, Wu (2006). A tale of two indices. Journal of Derivatives. https://doi.org/10.3905/jod.2006.616186

### Market structure & crash propagation

- Menkveld, Yueshen (2022). The flash crash: The role of market fragmentation and high-frequency trading. https://ink.library.smu.edu.sg/lkcsb_research/7285/
- Six Figure Investing (2019). What caused the February 5th, 2018 volatility spike / XIV termination. https://www.sixfigureinvesting.com/2019/02/what-caused-the-february-5th-2018-volatility-spike-xiv-termination/ — anatomy of Volmageddon: forced rebalancing of inverse-vol ETPs caused a +116% VIX spike on a -4.1% SPX day. Four contributing factors: flawed daily-rebalancing architecture, historical complacency, $5B asset accumulation in a $7B VIX futures market, and self-reinforcing liquidity exhaustion at settlement.

### Critical slowing down and tipping points

- Scheffer, Carpenter, Foley, Folke, Walker (2001). Catastrophic shifts in ecosystems. Nature. https://doi.org/10.1038/35098000
- Scheffer et al. (2009). Early-warning signals for critical transitions. Nature. https://doi.org/10.1038/nature08227
- Dakos et al. (2008). Slowing down as an early warning signal for abrupt climate change. PNAS. https://doi.org/10.1073/pnas.0802430105

### Hawkes processes in finance

- Hawkes (1971). Spectra of some self-exciting and mutually exciting point processes. Biometrika. https://doi.org/10.1093/biomet/58.1.83
- Bacry, Mastromatteo, Muzy (2015). Hawkes processes in finance. Market Microstructure and Liquidity. https://doi.org/10.1142/S2382626615500057
- Filimonov, Sornette (2012). Quantifying reflexivity in financial markets: toward a prediction of flash crashes. Physical Review E. https://doi.org/10.1103/PhysRevE.85.056108

### Rough volatility

- Gatheral, Jaisson, Rosenbaum (2018). Volatility is rough. Quantitative Finance. https://doi.org/10.1080/14697688.2017.1393551
- El Euch, Rosenbaum (2019). The characteristic function of rough Heston models. Mathematical Finance. https://doi.org/10.1111/mafi.12173
- Cont (2001). Empirical properties of asset returns: stylized facts and statistical issues. Quantitative Finance. https://doi.org/10.1080/713665670

### Intermediary asset pricing

- He, Krishnamurthy (2013). Intermediary asset pricing. American Economic Review. https://doi.org/10.1257/aer.103.2.732
- Adrian, Shin (2014). Procyclical leverage and endogenous risk. Journal of Political Economy. https://doi.org/10.1086/682903

### Systemic risk measures

- Adrian, Brunnermeier (2016). CoVaR. American Economic Review. https://doi.org/10.1257/aer.20120555
- Acharya, Pedersen, Philippon, Richardson (2017). Measuring systemic risk. Review of Financial Studies. https://doi.org/10.1093/rfs/hhw088
- Brownlees, Engle (2017). SRISK: A conditional capital shortfall measure of systemic risk. Review of Financial Studies. https://doi.org/10.1093/rfs/hhw060

### Financial networks and contagion

- Acemoglu, Ozdaglar, Tahbaz-Salehi (2015). Systemic risk and stability in financial networks. American Economic Review. https://doi.org/10.1257/aer.20130456
- Elliott, Golub, Jackson (2014). Financial networks and contagion. American Economic Review. https://doi.org/10.1257/aer.104.10.3115
- Battiston, Puliga, Kaushik, Tasca, Caldarelli (2012). DebtRank: Too central to fail? Scientific Reports. https://doi.org/10.1038/srep00541

### Business cycle and financial conditions

- Estrella, Mishkin (1998). Predicting U.S. recessions: financial variables as leading indicators. Review of Economics and Statistics. https://doi.org/10.1162/003465398557320
- Schularick, Taylor (2012). Credit booms gone bust: monetary policy, leverage cycles, and financial crises, 1870–2008. American Economic Review. https://doi.org/10.1257/aer.102.2.1029
- Jordà, Knoll, Kuvshinov, Schularick, Taylor (2019). The rate of return on everything, 1870–2015. Quarterly Journal of Economics. https://doi.org/10.1093/qje/qjy032 — long-run returns on housing, equities, bonds, and bills across 16 countries; useful baseline for regime-dependent return expectations.

### International regimes and currency crises

- Kaminsky, Reinhart (1999). The twin crises: the causes of banking and balance-of-payments problems. American Economic Review. https://doi.org/10.1257/aer.89.3.473
- Calvo (1998). Capital flows and capital-market crises: the simple economics of sudden stops. Journal of Applied Economics. https://doi.org/10.1080/15140326.1998.12040516
- Forbes, Rigobon (2002). No contagion, only interdependence: measuring stock market comovements. Journal of Finance. https://doi.org/10.1111/0022-1082.00494

### Momentum, trend, and rebalancing

- Jegadeesh, Titman (1993). Returns to buying winners and selling losers: implications for stock market efficiency. Journal of Finance. https://doi.org/10.1111/j.1540-6261.1993.tb04702.x — foundational cross-sectional momentum paper; positive momentum across risk assets is a risk-on indicator.
- Moskowitz, Ooi, Pedersen (2012). Time series momentum. Journal of Financial Economics. https://doi.org/10.1016/j.jfineco.2011.11.003 — time-series momentum earns its strongest returns during and after crisis periods; trend-following is a natural benchmark for regime-aware strategies.
- Rattray, Granger, Harvey, Van Hemert (2019). Strategic rebalancing. Journal of Portfolio Management. https://papers.ssrn.com/sol3/papers.cfm?abstract_id=3330134 — mechanical rebalancing is economically a short straddle (negative convexity); using trend signals to delay rebalancing during drawdowns improves crisis performance by 2–5pp. Motivates regime-aware rebalancing as an evaluation benchmark.

### Text and sentiment

- Loughran, McDonald (2011). When is a liability not a liability? Textual analysis, dictionaries, and 10-Ks. Journal of Finance. https://doi.org/10.1111/j.1540-6261.2010.01625.x — foundational financial NLP sentiment dictionary; general-purpose dictionaries (Harvard IV-4) misclassify common financial terms.
- Baker, Bloom, Davis (2016). Measuring economic policy uncertainty. Quarterly Journal of Economics. https://doi.org/10.1093/qje/qjw024

### Practitioner commentary and case studies

- Monday Morning Macro (2019). Impossible events are on the rise. https://monday-morning-macro.com/2019/08/19/impossible-events-are-on-the-rise/ — documents 4-sigma Treasury moves occurring far more frequently than VaR models predict; observation that HY credit lags other asset classes in repricing vol is a potential divergence signal.
- Monday Morning Macro (2019). Two year trouble. https://monday-morning-macro.com/2019/08/27/two-year-trouble/ — front-end Treasury realized vol exceeding equity vol, asset swap spreads at 20-year lows; motivates Treasury/equity realized vol ratio as a stress signal.
- Monday Morning Macro (2019). Far too little, far too late. https://monday-morning-macro.com/2019/09/17/far-too-little-far-too-late/ — anatomy of the Sept 2019 repo crisis: overnight repo hit 600 bps, structural reserve shortage from QT, Fed response inadequate. Key calibration episode for the liquidity/funding bucket.
- Monday Morning Macro (2019). The great LIBOR liquidation. https://monday-morning-macro.com/2019/09/24/the-great-libor-liquidation/ — LIBOR-to-SOFR transition risks; SOFR's daily-compounding-in-arrears design produced an 8+ sigma spike during Sept 2019 repo stress; design flaw implications for using SOFR-OIS as a benchmark stress signal.

## People to watch

Researchers and groups to monitor for new papers in this space.

### Variance risk premium and realized volatility

- **Tim Bollerslev** — ARCH; VRP and return predictability; tail risk premia
- **George Tauchen** — VRP, realized variance methodology (with Bollerslev)
- **Hao Zhou** — VRP and stock return predictability; jump risk premia
- **Viktor Todorov** — tail risk premia; jump processes; nonparametric high-frequency methods
- **Lai Xu** — tail risk premia; international VRP evidence
- **Geert Bekaert** — VRP and stock market volatility (with Hoerova); international finance
- **Marie Hoerova** — VRP and volatility (with Bekaert); financial stability
- **Torben Andersen** — realized volatility framework (with Bollerslev, Diebold); high-frequency econometrics
- **Ole Barndorff-Nielsen** — realized variance theory; BNS jump test; bipower variation
- **Neil Shephard** — realized variance; stochastic volatility (with Barndorff-Nielsen)
- **Nour Meddahi** — integrated volatility estimation; ANOVA for volatility
- **Francis X. Diebold** — realized volatility (with Andersen, Bollerslev); forecast evaluation; DY connectedness
- **Caio Almeida** — model-free tail risk measures from options; term structure
- **Yacine Aït-Sahalia** — jumps in equity prices; nonparametric high-frequency methods; VRP
- **Vineer Bhansali** — tail risk hedging as offensive risk management; monetization timing of tail hedges; performance measurement for asymmetric payoff strategies

### Fat tails, power laws, and crash hazard

- **Didier Sornette** — LPPL crash prediction; Why Stock Markets Crash (2003); dragon kings
- **Jean-Philippe Bouchaud** — market impact; econophysics; Theory of Financial Risk and Derivative Pricing
- **Niklas Wheatley** — Bitcoin bubble prediction with LPPL (2019)
- **Tobias Huber** — LPPL, Bitcoin bubbles (2019)
- **Max Reppen** — LPPL, Bitcoin bubbles (2019)
- **Robert N. Gantner** — LPPL, Bitcoin bubbles (2019)
- **Xavier Gabaix** — power laws in finance; granular origins of aggregate volatility
- **Nassim Nicholas Taleb** — Black Swan; fat tails; Incerto series; technical work on tail estimation
- **Paul Embrechts** — extreme value theory; Modelling Extremal Events (1997); copulas for tail dependence
- **Rama Cont** — heavy-tailed distributions; order book models; financial contagion networks

### Volatility term structure and options

- **Liuren Wu** — variance swaps; VIX dynamics; volatility term structure; jump risk
- **Gurdip Bakshi** — model-free option moments; higher-order risk measures; VVIX-type signals
- **Peter Carr** — variance swap replication underlying VIX construction; volatility derivatives
- **Jin-Chuan Duan** — GARCH option pricing; physical vs. risk-neutral volatility distributions

### Liquidity and funding stress

- **Markus Brunnermeier** — funding liquidity spirals; market/funding liquidity nexus (2009); liquidity black holes
- **Lasse Heje Pedersen** — funding liquidity; liquidity-adjusted CAPM; margin CAPM (with Garleanu)
- **Lubos Pastor** — liquidity risk and expected returns (2003, with Stambaugh)
- **Robert Stambaugh** — liquidity risk and expected returns (2003, with Pastor)
- **Hyun Song Shin** — leverage cycles; bank balance sheets as macro risk; risk-taking channel
- **Darrell Duffie** — OTC market liquidity; slow-moving capital; dynamic asset pricing theory
- **Arvind Krishnamurthy** — intermediary asset pricing (with He); safe asset scarcity; amplification
- **Dimitri Vayanos** — flight to quality; liquidity and asset pricing theory; search frictions
- **Viral Acharya** — systemic liquidity risk; SRISK; too-big-to-fail; financial regulation
- **Nicolae Garleanu** — margin-based asset pricing (with Pedersen); liquidity and portfolio choice
- **Zhiguo He** — intermediary asset pricing (with Krishnamurthy); leverage cycles; debt maturity

### Market microstructure and price impact

- **Bence Toth** — price impact; critical nature of liquidity; anomalous diffusion
- **Yvan Lemperiere** — price impact; market microstructure (CFM group)
- **Cecile Deremble** — market microstructure (CFM group)
- **Jonathan Donier** — critical liquidity; Bitcoin crash analysis (with Bouchaud)
- **Albert J. Menkveld** — flash crash; HFT; market microstructure; intermediation
- **Yueshen Bian** — flash crash; market fragmentation (with Menkveld)
- **Albert Kyle** — informed trading and price impact (Kyle 1985 model); lambda as illiquidity measure
- **Thierry Foucault** — limit order book theory; market microstructure design; HFT
- **Maureen O'Hara** — Market Microstructure Theory (textbook); liquidity and information
- **Tarun Chordia** — order imbalance and return predictability; empirical liquidity; arbitrage
- **Terrence Hendershott** — algorithmic trading; HFT and market quality; electronic markets
- **Marc Potters** — market impact; portfolio theory; risk management (CFM, with Bouchaud)
- **Fabrizio Lillo** — market impact scaling; order flow; price formation; order book dynamics
- **Jim Gatheral** — no-dynamic-arbitrage market impact model; rough volatility (with Rosenbaum)
- **Mathieu Rosenbaum** — rough volatility (H≈0.1); Hawkes processes; microstructure noise
- **Charles-Albert Lehalle** — optimal execution; market impact; market microstructure analytics

### Financial stress indices and systemic risk

- **Phillip Monin** — OFR Financial Stress Index (2019)
- **Michael Flood** — systemwide liquidity commonalities; OFR financial stability research
- **John Liechty** — systemwide liquidity (with Flood, Piontek)
- **Krista Piontek** — systemwide liquidity (with Flood, Liechty)
- **Jan Hatzius** — financial conditions indices; GS FCI methodology; macro forecasting
- **Stijn Claessens** — financial crises; systemic risk; cross-country financial cycle analysis
- **Robert Engle** — ARCH (Nobel 2003); DCC model for cross-asset correlations; SRISK
- **Stefano Giglio** — tail risk in the cross-section; long-run disaster risk; systemic risk measurement
- **Bryan Kelly** — TAIL risk measure; ML for factor models; return predictability
- **Monica Billio** — Granger-causality systemic risk across asset classes; connectedness
- **Andrew Lo** — adaptive markets hypothesis; systemic risk; hedge fund risk; financial innovation
- **Olivier Scaillet** — copulas; tail dependence; nonparametric econometrics; risk measures
- **Markus Pelger** — latent factor models with ML; generalized PCA; high-dimensional risk
- **Jon Danielsson** — procyclical VaR; model risk in financial regulation; endogenous risk

### Regime-switching

- **James Hamilton** — Markov regime-switching model (Hamilton 1989); the foundational reference
- **Andrew Ang** — regime-switching asset pricing; international equities under regime changes
- **Allan Timmermann** — regime shifts in return predictability; structural breaks; forecasting
- **Roger Farmer** — self-fulfilling crash equilibria; animal spirits; indeterminate equilibria
- **Lars Hansen** — GMM (Nobel 2013); model uncertainty and robust control; Hansen-Jagannathan bound

### Credit spreads and macro

- **Simon Gilchrist** — GZ credit spread as business cycle predictor (2012); financial accelerator
- **Egon Zakrajsek** — GZ spread; excess bond premium; credit conditions
- **Mark Gertler** — financial accelerator (with Bernanke, Gilchrist); DSGE models with financial frictions
- **Ben Bernanke** — financial accelerator; credit channel of monetary policy; financial crises
- **Robert Merton** — structural credit model (Merton 1974); distance-to-default
- **Francis Longstaff** — liquidity premium in credit spreads; Treasury microstructure; flight to liquidity
- **John Geanakoplos** — leverage cycle; collateral equilibrium; credit cycles
- **Ricardo Reis** — safe asset scarcity; monetary policy transmission; financial stability

### Term structure and bond risk premia

- **John Cochrane** — bond risk premia (with Piazzesi); discount rates; asset pricing theory
- **Monika Piazzesi** — bond risk premia (with Cochrane); macro-finance term structure
- **Tobias Adrian** — ACM term premium model; intermediary pricing; financial stability
- **Richard Crump** — ACM term premium model; term structure estimation
- **Emanuel Moench** — ACM term premium model; factor models for yield curve
- **John Campbell** — return predictability; yield curve; long-run risks; consumption-based pricing

---

*Not financial advice. This project is for research and educational purposes only. Nothing here is investment advice or a recommendation to buy or sell any security or digital asset. Use at your own risk.*
