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

## References

### Variance risk premium / tail-risk premia

- Bollerslev, Tauchen, Zhou (2009). Expected stock returns and variance risk premia. Review of Financial Studies. https://scholars.duke.edu/individual/pub732839
- Bollerslev, Todorov, Xu (2015). Tail risk premia and return predictability. Journal of Financial Economics. https://scholars.duke.edu/publication/1060835
- Bollerslev, Marrone, Xu, Zhou (2014). Stock return predictability and variance risk premia: international evidence. Journal of Financial and Quantitative Analysis. https://www.cambridge.org/core/journals/journal-of-financial-and-quantitative-analysis/article/abs/stock-return-predictability-and-variance-risk-premia-statistical-inference-and-international-evidence/0BE5DE1D942A0342DDBA24D7BFBEA5C8

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

### Regime-switching & macro regime signals

- Cochrane, Piazzesi (2005). Bond risk premia. NBER Working Paper. https://www.nber.org/papers/w9178
- Ang, Timmermann (2012). Regime changes and financial markets. NBER Working Paper. https://www.nber.org/papers/w17182
- Ang, Bekaert (2004). How do regimes affect asset allocation? NBER Working Paper. https://www.nber.org/papers/w10080

### Market structure & crash propagation

- Menkveld, Yueshen (2022). The flash crash: The role of market fragmentation and high-frequency trading. https://ink.library.smu.edu.sg/lkcsb_research/7285/
