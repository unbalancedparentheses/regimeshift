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


## References

See `REFERENCES.md` for the research papers and reading list that motivate the signals.
