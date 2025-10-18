# Notebook Guide — Full Walkthrough (11 Cells, Deep Dive)

This is a **self‑contained, interview‑ready** narrative of the entire project as implemented in `main.ipynb`. It explains **what each cell does**, **why we designed it that way**, **how signals and portfolio logic are engineered without look‑ahead**, and **how we searched for hyper‑parameters** (coarse→local) to reach the final configuration. It closes with a **balanced analysis** of the chosen solution (strengths/weaknesses, failure modes, and next steps).

> **Universe:** `BTCUSDT, ETHUSDT, BNBUSDT, XRPUSDT, SOLUSDT, TRXUSDT, DOGEUSDT` (USDT‑M Futures)  
> **Frequency & Horizon:** Daily bars (`1d`), ~5 years (`2020‑10‑18` → `2025‑10‑16`)  
> **Objective:** Rule‑based portfolio with **Sharpe > 1** and **MDD > −30%** under realistic trading costs  
> **Costs:** per fill: **0.05% in + 0.05% out + 0.02% slippage** (total ~0.12% round‑trip)  
> **Constraints:** **No ML/DL**, **no look‑ahead**, **long/short allowed** (baseline reports are without leverage by default)

---

## Design Philosophy — From Idea to Strategy

**Why momentum + regime?** Crypto is structurally volatile and regime‑driven. Pure momentum works in trending regimes but bleeds in chop. Our approach:  
1) **Momentum backbone** (20d/60d) and **RSI14** to time entries.  
2) **Market confirmation** (volume ratios, OBV, price–volume correlation) to avoid “weak” breakouts.  
3) **Regime gating** using **breadth** (share of coins >SMA60) and **BTC health** (RSI and optionally BTC>SMA60). This reduces left‑tail risk and improves risk‑adjusted returns.  
4) **Risk controls** with **ATR stops/TP**, inverse‑vol or equal weighting, and caps on concurrent positions.

**No look‑ahead enforcement** is a first‑class citizen: every indicator used for a decision at date *t+1* is computed **only with data up to date *t*** (via `.shift(1)`), entries are executed at **open(t+1)**, and intraday exits (SL/TP) are evaluated against **high/low(t+1)**.

**Parameter search** is deliberately **two‑stage**:  
- **Stage A (coarse)** explores a wide space with large steps (2k–5k configs) and keeps **Sharpe > 1**.  
- **Stage B (local)** starts from the **Top‑10 seeds** and explores **± small jitter** neighborhoods (~10k configs total), keeping **Sharpe > 1.5**. This combination gives coverage (lower miss risk) and local exploitation (less overfit).

---

# Cell‑by‑Cell Breakdown

## **Cell 1 — Init & Paths**
- Define `ROOT`, `DATA_ROOT = data/crypto_data`, and `PROCESSED` subfolder.  
- Set global config: `SYMBOLS`, `ASSET_CLASS="um"`, `FREQ="1d"`, `START`, `END`.  
- **Why:** Ensure all later code uses relative paths under the project folder (portable), and centralize temporal scope.

**Key choices:**  
- We align **all symbols** to a **common daily index** to guarantee synchronized features and returns across the panel.  
- UTC dates are used for reproducibility.

---

## **Cell 2 — Imports & Utilities**
- Imports: `pandas`, `numpy`, `plotly`, `requests`, `pathlib`, etc.  
- Helper functions for: timestamp unit detection (`ms`/`us`), safe CSV reader, and consistent column naming for klines:  
  `open_time, Open, High, Low, Close, volume, close_time, ...`

**Why:** reduces boilerplate and isolates brittle parsing logic; if any exchange field changes, update here once.

---

## **Cell 3 — Crawl Data (REST multi‑endpoint)**
- **REST download** of Binance USDT‑M futures klines with **multiple base endpoints**:  
  `https://fapi.binance.com`, `https://fapi1.binance.com`, `https://fapi.binancefuture.com`.  
- Iterates over time with `limit=1500` to fetch the entire 5‑year span quickly.  
- Per‑symbol CSVs saved to `data/crypto_data/um/daily/klines/<SYMBOL>/1d/REST_YYYY-MM-DD_YYYY-MM-DD.csv`.

**Why REST (fast) vs monthly ZIP (slow):** REST returns large blocks in one call; error handling rotates endpoints to be robust to transient SSL/451 issues. We **don’t** rely on Colab/Drive—fully local & reproducible.

**Sanity outputs:** row counts per symbol are printed for a quick smoke test.

---

## **Cell 4 — Panel Build & Close‑Only**
- Read all CSVs per symbol → **concat → sort by `open_time` → drop duplicate bars**.  
- Convert timestamps (`ms`/`us` auto) to `datetime64[ns]`, set index = `open_time` → rename to `date`.  
- Produce **panel** with MultiIndex columns `(symbol, field)` for OHLCV and a **wide close‑only** frame.  
- Save:
  - `um_1d_OHLCV_5y.parquet` (full panel)  
  - `um_close_only_1d_5y.csv` (for quick inspection)

**Why:** the panel becomes the single source of truth for all later feature engineering.

---

## **Cell 5a — Cleaning & Alignment**
- **Price sanity:** price > 0; volume ≥ 0.  
- **OHLC logic:** enforce `High ≥ max(Open, Close)` and `Low ≤ min(Open, Close)`; rows violating get OHLC set to NaN, then filled safely.  
- **Filling:** `ffill` then `bfill` for isolated gaps; **no synthetic bars** are invented.  
- Align all symbols to a **continuous daily index** from the max first valid date to the min last valid date (common window).  
- Save cleaned artifacts:  
  - `um_1d_OHLCV_5y.CLEAN.parquet`  
  - `um_close_only_1d_aligned.CLEAN.parquet`

**Why:** robust downstream indicators require clean, consistent OHLCV. We avoid aggressive interpolation to not distort volatility.

---

## **Cell 5b — QC & Baseline Visualization**
- Report, for each symbol:  
  - First/last dates, non‑NaN counts, NaN ratios.  
- Plot:  
  - Close & volume (spot holes),  
  - 20D annualized vol heatmap across the panel (detect regime changes).

**Observation:** No daily gaps (crypto 24/7), volumes look plausible; known stress dates (e.g., FTX) show up in vol heatmap.

---

## **Cell 5c — Outlier & Gap Detector**
- Detect **gaps in index** (>1 day): none expected.  
- Detect **return outliers**: `|Close_t/Close_{t-1} − 1| > 50%` and list them.  
  - Typical: DOGE spikes (early‑2021), SOL crash/rebound (Nov‑2022), etc.  
- Visualize distribution and mark outliers on return series.

**Why:** If returns show impossible discontinuities, revisit data source; in our case, spikes are real market moves.

---

## **Cell 5d — Consistency & Range Stats**
- Re‑check OHLC consistency after clean (should pass for all symbols).  
- Compute `(High−Low)/Close` distribution (mean/median/95/99/max) to confirm microstructure plausibility by coin.

**Why:** sanity guardrails before feature engineering; if ranges looked absurd, ATR/SL would misbehave downstream.

---

## **Cell 6a — Momentum & Trend Features (Backbone + RSI14)**
We compute per symbol:
- **Daily return** `ret_1 = Close_t / Close_{t-1} − 1`.  
- **mom_20 = Close_t / Close_{t‑20} − 1`** and **mom_60** similarly — core momentum signals.  
- **ret_mean_7** = rolling mean of `ret_1` (recent drift).  
- **sma60_filter** = sign(Close − SMA60): >0 implies up‑trend regime.  
- **rsi14** = custom implementation equivalent to Wilder’s RSI(14) using rolling average gain/loss.

We also plot multi‑panel charts per coin: price with SMA60; mom20/mom60; RSI14.

**Rationale:**  
- Using both short and medium momentum balances re‑activity and persistence.  
- RSI gives an oscillator check to avoid buying when too stretched or selling when too weak.  
- SMA60_filter avoids long entries against a dominant downtrend (optional gate).

---

## **Cell 6b — Volatility & Risk Features**
- **vol20**: rolling 20D std of daily returns, annualized with √365 (daily data).  
- **ATR14**: absolute volatility proxy for stop/take‑profit sizing.  
- **vol_ratio_10_60**: phase detector (calm vs turbulent regimes).  
- **mdd60**: rolling local max drawdown over 60 days (reject crashing coins).

**Rationale:** Control risk; scale stops to regime; de‑emphasize coins in active drawdowns.

---

## **Cell 6c — Volume & Confirmation**
- **vol_ratio_20**: today’s volume vs 20D mean (breakout confirmation).  
- **obv_change_20**: On‑Balance‑Volume delta over 20 days (accumulation/distribution).  
- **price_vol_corr_30**: corr of ΔPrice vs ΔVolume over 30 days (is volume “supporting” price?).

**Rationale:** Price moves with rising participation are higher quality than “dry” moves.

---

## **Cell 6d — Relative Strength & Cross‑Asset**
- **rel_mom20**: a coin’s mom20 minus the basket mean — “leader vs pack” selector.  
- **beta30**: correlation proxy vs BTC over 30 days (we prefer not too high when seeking diversification).  
- **breadth_strength**: share of **coins with Close>SMA60** (panel‑level health metric).

**Rationale:** We want leaders that are not entirely BTC‑beta. Breadth acts as a market regime switch.

At the end of 6a–6d we **merge all 15 features** into one store: `panel_all_features_15.parquet` (MultiIndex columns).

---

## **Cell 7 — Merge & Sanity**
- Consolidate feature blocks, **de‑duplicate column names** across groups (Close/volume appear in several blocks).  
- Persist a single feature panel for the backtester to consume (fast reloads).

**Why:** Keeps backtest code simple (single loader), and ensures consistent index across features.

---

## **Cell 8 — Backtest Engine (Rule‑based, Futures, No Look‑Ahead)**
This is the **heart** of the notebook. It consists of:

### Dataclass `P8` — All Hyper‑params in One Place
```python
@dataclass
class P8:
    START, END, SYMBOLS
    # Signals (≥ 7 features must be present in the rules)
    RSI_MIN_LONG, RSI_MAX_SHORT
    MOM20_MIN, MOM60_MIN, MOM20_MAX_SHORT, MOM60_MAX_SHORT
    SMA60_FILTER_ON, VOLR20_MIN, PVCORR_MIN, RELMOM_MIN, BETA_MAX, ALLOW_SHORT
    # Regime gating
    REGIME_ON_MIN_BREADTH, REGIME_OFF_MAX_BREADTH
    REGIME_BTC_SMA60_ON, REGIME_BTC_RSI_MIN, CLOSE_ALL_ON_REGIME_OFF
    # Portfolio & sizing
    MAX_COINS, WEIGHTING, MAX_GROSS_EXPOSURE, FIXED_FRACTION_PER_TRADE
    # Costs
    FEE_IN, FEE_OUT, SLIPPAGE
    # Exits
    SL_ATR_MULT, TP_ATR_MULT, STOP_TP_PRIORITY, COOLDOWN_AFTER_EXIT
    # Engine
    INITIAL_CAPITAL
```
**Why:** one‑stop, auditable knob set; easy for grid search and interview demos.

### Data Loader
- Builds aligned data dict: `open/high/low/close`, `atr14`, `vol20`, `mom60`, `features_by_symbol`, `breadth`, `btc_rsi`, `btc_sma`.  
- Creates a **daily index** `idx = date_range(START, END, "D")` and **reindexes** all series to it.

### **No Look‑Ahead Mechanics**
- All features used for a decision on day **t** are **shifted by 1** (computed from data up to **t−1**).  
- **Entries** are placed at **open(t)** of the decision day.  
- **Stops/TP** are checked **intraday** using high/low of **t**, after entry is created at open(t).

### Signals
- **Long** when **all** are true (example rule skeleton):
  - `rsi14 ≥ RSI_MIN_LONG`
  - `mom_20 > MOM20_MIN` and `mom_60 > MOM60_MIN`
  - `(SMA60_FILTER_ON ⇒ sma60_filter > 0)`
  - `vol_ratio_20 ≥ VOLR20_MIN`
  - `price_vol_corr_30 ≥ PVCORR_MIN`
  - `rel_mom20 ≥ RELMOM_MIN`
  - `beta30 ≤ BETA_MAX`
  - **Regime ON:** `breadth ≥ REGIME_ON_MIN_BREADTH`, optional `BTC>SMA60`, `BTC_RSI ≥ REGIME_BTC_RSI_MIN`
- **Short** is symmetrical when enabled (may also require regime OFF).  
- Ties across many candidates are broken by **ranking with mom60** (leaders first).

### Portfolio Construction
- **Select up to `MAX_COINS`** per day based on signals and **rank by mom60** (top leaders).  
- **Weights:** `"equal"` or `"inv_vol"` (using `vol20`).  
- **Sizing:** each new position uses up to `FIXED_FRACTION_PER_TRADE` of current equity (and capped by `MAX_GROSS_EXPOSURE`).  
- **Costs:** fee + slippage applied on **notional** at both entry and exit.

### Exits (ATR‑based)
- For a long: `stop = entry − SL_ATR_MULT*ATR`, `tp = entry + TP_ATR_MULT*ATR`.  
- For a short: `stop = entry + SL_ATR_MULT*ATR`, `tp = entry − TP_ATR_MULT*ATR`.  
- If both SL & TP hit intraday, we follow `STOP_TP_PRIORITY` (e.g., `"tp"`).  
- Optional **cooldown** prevents re‑entry in the same day after a SL/TP.

### Metrics
- **Equity** = cash + mark‑to‑market of open positions (EOD).  
- **CAGR**: \((\frac{E_{T}}{E_{0}})^{\frac{365}{\text{days}}} - 1\)  
- **Vol (annualized)**: daily std × √365  
- **Sharpe (rf=0)**: daily mean / daily std × √365  
- **MDD**: min of `equity / cummax − 1`  
- **Hit ratio (daily>0)**: share of positive daily returns.

**Why it works:** strict no look‑ahead, robust exits, regime gating, and cost accounting keep results realistic.

---

## **Cell 9 — Stage A Parameter Search (Coarse)**
- Sample **2k–5k** configurations using coarse steps / Latin Hypercube.  
- **Vectorized** sampling & `apply` to avoid deep nested loops (performance & readability).  
- Filter **Sharpe > 1**; save hits to `stageA_results_sharpe_gt1.csv` (sorted by Sharpe).

**Why:** broad exploration reduces the risk of missing viable regions and helps understand sensitivity.

---

## **Cell 10 — Stage B Local Search (Fine around Top‑10)**
- For each of the **Top‑10** seeds from Stage A, create **~1,000 nearby variants** by adding **small random jitter** to each hyper‑param (clipped to valid ranges).  
- Combine to ~**10k** configs; run & keep **Sharpe > 1.5**; save as `stageB_local_results_sharpe_gt1p5.csv`.  
- Still avoids nested Python loops by constructing candidate DataFrames and mapping in batches.

**Why:** exploitation around promising seeds increases Sharpe while containing overfit risk (local, not micro‑tuning).

---

## **Cell 11 — Visualization of Any Saved Configuration**
- Load Stage‑B CSV, choose rank‑#N (by Sharpe) → map to `P8` → run backtest again.  
- **Printed analytics:**  
  - Equity metrics (Sharpe, CAGR, Vol, MDD, final equity)  
  - **Trades executed**, **win rate**, **profit factor**, **average holding**  
- **Tables:**  
  - **Per‑symbol performance** (trades, win‑rate, total/avg PnL, avg holding days)  
  - **Long vs Short** performance (if shorts enabled)  
  - **Trades per month** (seasonality/overtrading checks)  
- **Plots:**  
  - Equity curve + drawdown  
  - Bar charts of PnL by symbol, win‑rate by symbol  
  - Bar charts of trade counts & PnL (Long vs Short)  
  - Histogram of trade returns & holding duration

**Why:** Visibility into *how* the system makes money (or not), not just headline Sharpe.

---

# The Final Selected Configuration (from Stage B)

```
Sharpe ≈ 1.7263 | CAGR ≈ 37.98% | Vol ≈ 19.77% | MDD ≈ −16.95% | Final ≈ 5,000,535 (from 1,000,000)
RSI_MIN_LONG=50, RSI_MAX_SHORT=45
MOM20_MIN=−0.2, MOM60_MIN=0.2, MOM20_MAX_SHORT=0.04, MOM60_MAX_SHORT=0
VOLR20_MIN=1.5, PVCORR_MIN=0.1, RELMOM_MIN=0, BETA_MAX=2
REGIME_ON_MIN_BREADTH=0.5, REGIME_OFF_MAX_BREADTH=0.3
REGIME_BTC_RSI_MIN=60, REGIME_BTC_SMA60_ON=False, CLOSE_ALL_ON_REGIME_OFF=True
SL_ATR_MULT=2, TP_ATR_MULT=3.5, STOP_TP_PRIORITY="tp"
MAX_COINS=3, FIXED_FRACTION_PER_TRADE=0.18, WEIGHTING="equal"
ALLOW_SHORT=False, MAX_GROSS_EXPOSURE=1, INITIAL_CAPITAL=1,000,000
FEE_IN=0.0005, FEE_OUT=0.0005, SLIPPAGE=0.0002
```

## Why this configuration is effective

1. **Regime gating is strict but not overbearing.**  
   - Market **ON** if breadth ≥ 0.5 (at least half of coins >SMA60) and BTC RSI ≥ 60.  
   - Market **OFF** if breadth ≤ 0.3, with **close all** behavior.  
   → Cuts exposure during bear/sideways regimes, keeping MDD ~ −17%.

2. **Asymmetric momentum with confirmation.**  
   - Require **mom60 ≥ 0.2** (persistent trend), **mom20 > −0.2** (avoid catching immediate dips), **RSI ≥ 50** (not weak).  
   - **vol_ratio_20 ≥ 1.5** and **price_vol_corr ≥ 0.1**: trend supported by participation.  
   → Avoids “weak rallies”; focuses on solid breakouts.

3. **Simplified book (MAX_COINS=3, equal weights, no shorts).**  
   - Reduces correlation crowding while keeping focus on leaders.  
   - Shorts are disabled here (ALLOW_SHORT=False) since they under‑performed in coarse search on this 5y window.

4. **ATR exits with favorable RR.**  
   - `SL=2×ATR`, `TP=3.5×ATR`, prioritizing **TP** when both touched same day.  
   - Combined with regime gating, this achieves **Sharpe ~1.73** without leverage and modest turnover.

5. **Costs included.**  
   - 0.12% round‑trip per fill makes results realistic (some candidates died on costs during search).

## What the metrics tell us
- **Sharpe ~1.73** with **Vol ~19.8%** signals *risk‑efficient compounding*.  
- **CAGR ~38%** with **MDD ~−17%** ⇒ excellent control of left‑tail for a crypto futures strategy **without** leverage.  
- Typically **win‑rate modest** but **profit factor > 1** thanks to RR>1 and regime filters cutting long drawdowns.

## Strengths
- **No look‑ahead rigor** (shifted features, open‑next‑day entries, intraday exits).  
- **Robust to chop** via regime OFF and breadth filters.  
- **Transparent & explainable** rules—great for interviews and risk review.  
- **Search process** balances exploration/exploitation, improving Sharpe while reducing overfit.

## Limitations / Risks
- **Static thresholds** might drift as market microstructure changes (liquidity cycles, alt rotations).  
- **Daily bars only** → slippage model is simple; intraday fills are approximated by OHLC.  
- **Universe size** fixed at 7 coins; breadth sensitivity depends on this set.  
- **Parameter search still in‑sample** (use walk‑forward / rolling OOS for production).

## Sensitivity & Robustness Checks (suggested)
- Walk‑forward validation (e.g., 2‑year train / 1‑year test, rolling).  
- Vary **`MAX_COINS`** and weighting (equal vs inv‑vol).  
- Try **3‑state regime** (ON / NEUTRAL / OFF) to reduce whipsaw transitions.  
- Trailing ATR stops instead of fixed ATR from entry day.  
- Add turnover constraints / minimum holding days to throttle costs.

---

# Appendix — Key Formulae (for review)

- **Momentum:** \(\text{mom}_k(t) = \frac{Close(t)}{Close(t-k)} - 1\)  
- **RSI(14):** RSI = \(100 - \frac{100}{1 + RS}\), \(RS = \frac{\text{avg gain}_{14}}{\text{avg loss}_{14}}\)  
- **ATR(14):** rolling mean of true range \(TR = \max(High-Low, |High-Close_{t-1}|, |Low-Close_{t-1}|)\)  
- **Vol (ann.):** \(\sigma_{20} \times \sqrt{365}\)  
- **Breadth:** share of symbols with \(Close > SMA_{60}\)  
- **Sharpe:** \(\frac{\mu_{daily}}{\sigma_{daily}} \sqrt{365}\) (rf=0)  
- **CAGR:** \((\frac{E_{T}}{E_{0}})^{365/\text{days}} - 1\)  
- **Max DD:** \(\min\left(\frac{Equity}{\text{cummax}(Equity)} - 1\right)\)

---

## How to Demo in an Interview (2 minutes)
1) **What we built:** rule‑based futures momentum strategy with **15 features**, regime filters, ATR exits, costs; **no look‑ahead**.  
2) **How we searched:** coarse (2k–5k) → local (10k); kept Sharpe thresholds at each stage to avoid noise.  
3) **Result:** Sharpe ~1.73, CAGR ~38%, MDD ~−17% (no leverage) over 5y, realistic costs.  
4) **Why it works:** trends + participation + regime gating; exits scaled by ATR; controlled concurrency.  
5) **Limits & next steps:** walk‑forward, 3‑state regime, trailing ATR, turnover controls.

---

> See also: **`README.md`** (how to run), **`Executive_Summary.md`** (short narrative).

