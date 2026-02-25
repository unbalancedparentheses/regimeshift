# regimeshift

Regime and risk-on/risk-off signals for markets, combining fat-tail crash ideas, variance risk premium (VRP), and liquidity/impact proxies.

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


## Signals (planned)

- Financial stress indices (OFR FSI, STLFSI) as macro stress regime baselines.
- VIX term structure slope (contango vs backwardation) as a volatility regime indicator.
- Option skew (e.g., Cboe SKEW) as tail-risk pricing proxy.
- MOVE index (bond market volatility) as a rates/credit stress signal.
- Funding-liquidity stress proxies (e.g., TED spread / short-term funding spreads).
- Liquidity/impact proxies (e.g., Amihud illiquidity, order-book depth when available).
- Credit spreads (e.g., Baa-Aaa, HY-IG) as early stress signals.
- Yield curve / term premium (e.g., 2s10s, 3m10y) as macro regime proxies.
- Cross-asset correlation spikes as stress regime indicators.
- Vol-of-vol (e.g., VVIX) as a volatility-regime transition signal.
- Positioning/flows (e.g., CFTC COT, ETF flows) as crowding/risk signals.
- Funding stress spreads (e.g., SOFR-OIS, FRA-OIS, repo specials) as modern liquidity stress proxies.

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
  - Source: FRED (SOFR-OIS, FRA-OIS, repo specials)
  - Raw: spread in bps
  - Normalize: `z`
- TED spread
  - Source: FRED TED spread
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

**Credit and macro**

- Credit spreads
  - Source: FRED Baa-Aaa, HY-IG
  - Raw: spread in bps
  - Normalize: `z`
- Yield curve / term premium
  - Source: FRED (2s10s, 3m10y) or ACM term premium
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

### Regime-switching & macro regime signals

- Cochrane, Piazzesi (2005). Bond risk premia. NBER Working Paper. https://www.nber.org/papers/w9178
- Adrian, Crump, Moench (2013). Pricing the term structure with linear regressions. NY Fed Staff Report 340. https://www.newyorkfed.org/research/staff_reports/sr340.html

### Credit spreads & risk premia

- Gilchrist, Zakrajsek (2012). Credit spreads and business cycle fluctuations. American Economic Review. https://www.nber.org/papers/w17021

### Volatility term structure

- VIX term structure and contango/backwardation. Journal of Risk and Financial Management (2019). https://www.mdpi.com/1911-8074/12/3/113

### Market structure & crash propagation

- Menkveld, Yueshen (2022). The flash crash: The role of market fragmentation and high-frequency trading. https://ink.library.smu.edu.sg/lkcsb_research/7285/
