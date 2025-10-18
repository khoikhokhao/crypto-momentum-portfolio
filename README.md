# Crypto Momentum Futures – Backtest & Parameter Search (README)

This README is a quick, interview‑ready guide for running the **`main.ipynb`** notebook end‑to‑end on VS Code / Jupyter, reproducing the crawl → clean → feature engineering → rule-based backtest → parameter search → visualization pipeline.

> **Universe:** `BTCUSDT, ETHUSDT, BNBUSDT, XRPUSDT, SOLUSDT, TRXUSDT, DOGEUSDT` (USDT‑M Futures, 1D, ~5 years)  
> **Goal:** Portfolio with **Sharpe > 1**, **MDD > −30%**, real costs (0.05% in + 0.05% out + 0.02% slippage per fill).  
> **Method:** Rule‑based (no ML/DL), **no look‑ahead**, futures (long/short supported), **no leverage** in the baseline.  
> **Path conventions:** project root = `C:\Users\Admin\Desktop\phongvan` (but the code uses relative paths).

---

## 0) Environment & Dependencies

- Python ≥ 3.10
- Recommended packages (already installed in your venv):
  ```bash
  pip install pandas numpy plotly pyarrow fastparquet matplotlib requests ta
  ```
  > If `ta-lib` is unavailable on Windows, the notebook uses **custom RSI/ATR** implementations, so you’re covered.

- Launch VS Code → open the folder → select your `.crypto` kernel → open `main.ipynb`.

**Folder layout (created by the notebook):**
```
data/crypto_data/
  ├── um/daily/klines/<SYMBOL>/1d/*.csv    # raw REST klines files
  └── processed/
       ├── um_1d_OHLCV_5y.parquet
       ├── um_1d_OHLCV_5y.CLEAN.parquet
       ├── um_close_only_1d_5y.csv
       ├── um_close_only_1d_aligned.CLEAN.parquet
       ├── um_close_returns_1d_aligned.CLEAN.parquet
       ├── feat_momentum_trend.parquet
       ├── feat_risk_volatility.parquet
       ├── feat_volume_confirmation.parquet
       ├── feat_relative_strength.parquet
       └── panel_all_features_15.parquet
stageA_results_sharpe_gt1.csv
stageB_local_results_sharpe_gt1p5.csv
```

---

## 1) Notebook Cells (11 Cells) – What to Run & Why

**Cell 1 — Init & Paths**  
- Defines `ROOT`, `DATA_ROOT`, `PROCESSED`, symbol list, asset class (`um`), frequency (`1d`), START/END (UTC).  
- _Run once at the top of the session._

**Cell 2 — Imports & Versions**  
- Imports `pandas/numpy/plotly/requests` and utility helpers.  
- _Run once._

**Cell 3 — Crawl Data (REST + fallback)**  
- Pulls 1D futures klines from Binance REST (with multi‑endpoint) and saves per‑symbol CSVs.  
- Smoke‑tests rows per symbol.  
- **Flags:**  
  - `SYMBOLS`, `START`, `END`, `FREQ`, `ASSET_CLASS`  
  - Endpoints list (`FAPI_ENDPOINTS`) and `limit` (default 1500).

**Cell 4 — Build Panel & Close‑only**  
- Reads all raw CSVs, normalizes column names, parses timestamps (auto ms/us).  
- Drops duplicate rows, sorts by `open_time`, aligns index → builds **panel (OHLCV)** & **close‑only**.  
- Saves `um_1d_OHLCV_5y.parquet` and `um_close_only_1d_5y.csv`.

**Cell 5a — Cleaning & Alignment**  
- Validates price/volume, enforces **High/Low consistency**, fills safe gaps (ffill/bfill), aligns the common date range.  
- Outputs cleaned & aligned parquet files.

**Cell 5b — QC & Baseline Viz**  
- Prints missing/duplicated counts; quick plots (close/volume, volatility heatmap).  
- Confirms there’s no daily gap (crypto is 24/7).

**Cell 5c — Outlier & Gap Detector**  
- Flags return outliers (e.g., |return| > 50%) and lists their dates (DOGE 2021, SOL 2022 FTX, etc.).  
- Heatmap of 20D annualized vol for regime sensing.

**Cell 5d — Consistency & Range Stats**  
- Re-checks High/Low logic (should pass).  
- Shows distribution of `(High−Low)/Close` by symbol for sanity.

**Cell 6a — Momentum & Trend Features (5 “backbone”)**  
- Computes `ret_1`, `mom_20`, `mom_60`, `ret_mean_7`, `sma60_filter`; adds **RSI14** (custom).  
- Per‑symbol multi‑panel plots.

**Cell 6b — Vol & Risk Features**  
- `vol20` (ann.), `ATR14`, `vol_ratio_10_60`, `mdd60` (rolling).

**Cell 6c — Volume & Confirmation**  
- `vol_ratio_20`, `obv_change_20`, `price_vol_corr_30`.

**Cell 6d — Relative Strength & Cross‑Asset**  
- `rel_mom20`, `beta30` (corr proxy), `breadth_strength` (= share of coins with Close > SMA60).  
- Combines 6a–6d → `panel_all_features_15.parquet`.

**Cell 7 — Merge & Sanity**  
- Merges all feature blocks, de‑duplicates columns, persists the feature store.

**Cell 8 — Backtest Engine (Rule‑based, no leverage)**  
- Defines dataclass **`P8`** (all hyper‑params in one place) and `run_backtest_with_trades`.  
- **Absolutely no look‑ahead**: signals from day *t* → trade at **open t+1**; SL/TP intraday uses **t+1 high/low**.  
- Costs per fill: `FEE_IN`, `FEE_OUT`, plus `SLIPPAGE`.  
- **Positioning:** pick up to `MAX_COINS` leaders by `mom_60`, weight = `equal` / `inv_vol`, size capped by `FIXED_FRACTION_PER_TRADE`.  
- Regime filters: breadth / BTC RSI / BTC SMA60; optional **close all on regime off**.  
- Prints equity metrics (CAGR, Vol, Sharpe, MDD, Hit‑ratio) + returns **equity** and **trades**.

**Cell 9 — Stage A (Global Search)**  
- Random/LHS grid (2k–5k configs), coarse steps.  
- Saves configs with **Sharpe > 1** → `stageA_results_sharpe_gt1.csv`.  
- **Important knobs to widen search space:** `RSI_MIN_LONG`, `RSI_MAX_SHORT`, `MOM20/MOM60` thresholds (long/short), `VOLR20_MIN`, `PVCORR_MIN`, `RELMOM_MIN`, `BETA_MAX`, regime thresholds, `SL_ATR_MULT`, `TP_ATR_MULT`, `MAX_COINS`, `WEIGHTING`, `FIXED_FRACTION_PER_TRADE`.

**Cell 10 — Stage B (Local Search around Top‑10)**  
- For each of the Top‑10 configs from Stage A, spawn ~1,000 local neighbors (± small jitter) → ~10k total.  
- Filter **Sharpe > 1.5**, sort desc → save `stageB_local_results_sharpe_gt1p5.csv`.

**Cell 11 — Visualize Any Result by Rank**  
- Load Stage‑B CSV; set `N_SELECT` (1‑based) to pick rank‑#N by Sharpe.  
- Re‑run backtest with that config → print metrics + tables:  
  - Trades per month; Per‑symbol PnL & Win‑rate; Long vs Short stats; Histograms of trade return & holding days.  
- Plots: Equity + Drawdown; bar charts as above.

---

## 2) Key Flags & What They Do (Dataclass `P8`)

```python
@dataclass
class P8:
    START: str; END: str; SYMBOLS: list | None = None

    # Signal thresholds (>= 7 features must participate)
    RSI_MIN_LONG: float
    RSI_MAX_SHORT: float
    MOM20_MIN: float; MOM60_MIN: float
    MOM20_MAX_SHORT: float; MOM60_MAX_SHORT: float
    SMA60_FILTER_ON: bool
    VOLR20_MIN: float
    PVCORR_MIN: float
    RELMOM_MIN: float
    BETA_MAX: float
    ALLOW_SHORT: bool

    # Regime filters
    REGIME_ON_MIN_BREADTH: float      # e.g., 0.50 means at least half of coins > SMA60
    REGIME_OFF_MAX_BREADTH: float     # e.g., 0.30 to go fully risk-off
    REGIME_BTC_SMA60_ON: bool         # require BTC Close > SMA60?
    REGIME_BTC_RSI_MIN: float         # BTC RSI gate
    CLOSE_ALL_ON_REGIME_OFF: bool     # force flat when risk-off

    # Portfolio
    MAX_COINS: int                    # max concurrent positions
    WEIGHTING: str                    # "equal" | "inv_vol"
    MAX_GROSS_EXPOSURE: float         # cap gross (baseline = 1.0 when no leverage)
    FIXED_FRACTION_PER_TRADE: float   # per-trade size on equity

    # Costs (per fill, on notional)
    FEE_IN: float; FEE_OUT: float; SLIPPAGE: float

    # Stops / Take-Profit (ATR-based)
    SL_ATR_MULT: float; TP_ATR_MULT: float
    STOP_TP_PRIORITY: str             # "stop" | "tp" when both hit intraday
    COOLDOWN_AFTER_EXIT: bool         # avoid re-entering same symbol same day

    # Engine
    INITIAL_CAPITAL: float
```

**Execution logic highlights:**
- Signals computed with **t-1 features** (via `.shift(1)`), entries at **open(t)**.  
- SL/TP evaluated intraday against **high/low(t)** AFTER entry.  
- Position ranking by `mom_60`; weights equal or inverse vol20.  
- Costs: `notional * (fee + slippage)` per fill.

---

## 3) Reproducing the Final Chosen Configuration

From Stage‑B, you selected this config (seed_idx = 7):

```
Sharpe ≈ 1.7263 | CAGR ≈ 37.98% | Vol ≈ 19.77% | MDD ≈ -16.95% | Final ≈ 5,000,535
RSI_MIN_LONG=50, RSI_MAX_SHORT=45
MOM20_MIN=-0.2, MOM60_MIN=0.2
MOM20_MAX_SHORT=0.04, MOM60_MAX_SHORT=0
VOLR20_MIN=1.5, PVCORR_MIN=0.1, RELMOM_MIN=0, BETA_MAX=2
REGIME_ON_MIN_BREADTH=0.5, REGIME_OFF_MAX_BREADTH=0.3
REGIME_BTC_RSI_MIN=60, REGIME_BTC_SMA60_ON=False, CLOSE_ALL_ON_REGIME_OFF=True
SL_ATR_MULT=2, TP_ATR_MULT=3.5, STOP_TP_PRIORITY="tp"
MAX_COINS=3, FIXED_FRACTION_PER_TRADE=0.18, WEIGHTING="equal"
ALLOW_SHORT=False, MAX_GROSS_EXPOSURE=1, INITIAL_CAPITAL=1_000_000
FEE_IN=0.0005, FEE_OUT=0.0005, SLIPPAGE=0.0002
```

**How to run it in Cell 11:**
- Set `N_SELECT = 1` to pick the top row of `stageB_local_results_sharpe_gt1p5.csv` after sorting by Sharpe.  
- Or instantiate `P8(...)` directly with the above values and call `run_backtest_with_trades`.

---

## 4) Tips & Troubleshooting

- **KeyError(Timestamp)**: ensure all frames are reindexed to the same daily `idx`; use helper accessors `.get(d)` or safe `.loc[d]` after reindex.  
- **NaN after clean**: check Cell 5a ranges and the common aligned window (max first valid … min last valid).  
- **Too many trades / fees ballooning**: increase `VOLR20_MIN`, `RSI_MIN_LONG`, `MOM20_MIN`, or add cooldown.  
- **Chop in sideways markets**: tighten regime ON/OFF, require BTC RSI/SMA60 confirmation, raise `MOM60_MIN`.  
- **Outlier spikes**: ATR‑based SL/TP helps; consider `trailing ATR` as next step.

---

## 5) Interview‑Friendly Soundbites

- Strategy = **momentum (20/60d)** + **market confirmation (volume & corr)** + **regime filters (breadth/BTC RSI)** + **risk controls (ATR SL/TP, inv‑vol sizing)**.  
- **No look‑ahead** enforced: features shifted, entries next day open, intraday exits use high/low *after* entry.  
- **Costs included**: realistic `0.12%` round‑trip per fill.  
- **Optimization**: coarse global (Stage A) → local exploration around top seeds (Stage B) to reduce overfit risk.  
- Result: **Sharpe ~1.73**, **MDD −16.95%**, **CAGR ~38%** on 5y, baseline (no leverage).

Good luck in the interview!
