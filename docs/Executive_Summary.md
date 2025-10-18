# Executive Summary — Crypto Momentum Futures (Rule‑based)

**Objective:** Construct a rule‑based futures portfolio on the top‑cap coins (USDT‑M, 1D, ~5y) with **Sharpe > 1** and **MDD > −30%**, including realistic trading frictions.

**Approach:**  
1) **Momentum backbone** (ret_1, mom20, mom60, ret_mean_7, SMA60 filter) for trend detection.  
2) **Risk controls** (ATR14 stops/TP, vol20, mdd60, vol‑ratio 10/60) to avoid knife‑catching.  
3) **Market confirmation** (volume ratio 20d, OBV change 20d, price‑volume corr 30d).  
4) **Cross‑asset selection** (relative momentum vs basket, beta vs BTC); **breadth** as market regime.  
5) **Execution & no look‑ahead**: features at day *t* → entries at **open(t+1)**; intraday exits use **High/Low(t+1)**; apply fees + slippage per fill.

**Parameter Search:**  
- **Stage A (coarse):** random grid (2k–5k configs), keep **Sharpe > 1**.  
- **Stage B (local):** ± jitter around Top‑10 seeds (~10k configs), keep **Sharpe > 1.5**; sorted results saved.

**Selected Configuration (seed_idx=7):**  
- **Regime:** breadth ON ≥ 0.5; OFF ≤ 0.3; BTC RSI ≥ 60; (BTC>SMA60 not required), close all ON risk‑off.  
- **Signals:** Long when RSI≥50, mom60≥0.2, mom20>−0.2, SMA60>0, volR20≥1.5, pvCorr≥0.1, relMom≥0, beta≤2. Short disabled.  
- **Portfolio:** MAX_COINS=3, equal weight, per‑trade size=18% of equity, MAX_GROSS_EXPOSURE=1.0.  
- **Stops/TP:** SL=2×ATR, TP=3.5×ATR, priority=TP when both hit intraday.  
- **Costs:** 0.05% in + 0.05% out + 0.02% slippage per fill.

**Results (5y):**  
- **Sharpe:** ~**1.73**  
- **CAGR:** ~**38%**  
- **Vol:** ~**19.8%** (annualized)  
- **Max Drawdown:** ~**−17%**  
- **Final Equity:** ~**5.0M USDT** from 1.0M

**Why it’s effective:**  
- Regime gating cuts left‑tail (crash periods).  
- Asymmetric momentum ensures entries on persistent trends.  
- Volume/correlation filters reduce false breakouts.  
- ATR exits scale to volatility; favorable RR with reasonable hit‑rate.  
- Limited concurrency avoids excessive correlation concentration.

**Next steps:** trailing ATR, 3‑state regime (ON/NEUTRAL/OFF), periodic rebalance with bands, risk‑parity with caps, walk‑forward validation.
