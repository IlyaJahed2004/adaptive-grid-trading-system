# Phase 1 – Market Universe Selection & Volatility-Based Grid Preparation

## Overview

This phase builds the **foundation of the grid trading system**.  
The goal is to construct a *robust and volatility-aware trading universe* before running any grid strategy or backtest.

Phase 1 consists of two main steps:

1. **Market universe selection using EMA200**
2. **Volatility measurement using ATR to prepare grid spacing**

No trades are executed in this phase.  
Only **asset selection** and **parameter preparation** are performed.

---

## 1. Data Sources

### CoinMarketCap
- Used to retrieve the **top cryptocurrencies by market capitalization**
- Only symbols are extracted
- Purpose: ensure liquidity and market relevance

### Yahoo Finance
- Used to download **historical daily OHLC data**
- Timeframe: Daily candles
- History starts early enough to avoid indicator warm-up bias

---

## 2. Time Configuration

| Parameter | Description |
|--------|------------|
| `HISTORY_START` | Start date for historical data (indicator warm-up) |
| `BACKTEST_START` | First day of the backtest period |
| `BACKTEST_END` | Last day of the backtest period |

All timestamps are converted to **UTC** to prevent timezone comparison errors.

---

## 3. Phase 1.1 – EMA200-Based Asset Selection

### Motivation

Grid strategies perform better in **structurally strong markets**.  
EMA200 is used as a long-term trend filter to exclude weak or sideways assets.

---

### EMA200 Rule

For each cryptocurrency:

1. Compute **EMA200** using *only pre-backtest data*
2. Count how many daily closes are **above EMA200**
3. Calculate the ratio:

ratio = (number of closes above EMA200) / (total pre-backtest closes)

---

### Selection Criterion

Asset is selected if:

ratio > 0.5


Meaning:
- The asset spent **more than 50% of the time above EMA200**
- Indicates long-term bullish or structurally strong behavior

---

### Why This Works

- Avoids single-day confirmation bias
- Uses *distribution over time*, not one snapshot
- Prevents look-ahead bias
- Filters out unstable or weak markets

---

### Output of EMA Phase

- A list of selected tickers (`selected`)
- A dictionary of full historical data (`main_dict`)
- Exactly **50 assets** (when available)

---

## 4. Phase 1.2 – Volatility Measurement Using ATR

### Why ATR?

Grid spacing must adapt to market volatility.

- Low volatility → tight grid
- High volatility → wide grid

ATR (Average True Range) measures **how much price typically moves**, independent of direction.

---

### ATR Calculation

ATR is computed using standard True Range (TR):

TR = max(
High − Low,
|High − Previous Close|,
|Low − Previous Close|
)

ATR is then defined as:

ATR = rolling mean of TR over N periods


Where:
- `N = 14` (standard choice)

Only **pre-backtest data** is used to avoid future leakage.

---

### Grid Spacing Formula


Parameters:
- `ATR_PERIOD = 14`
- `GRID_MULTIPLIER = 1.0` (tunable)

The spacing is also expressed as a **percentage of the last close** for normalization.

---

### Example

If:
- ATR(14) = 120 USD
- Multiplier = 1.0

Then:

Grid Spacing = 120 USD


Each grid level will be separated by 120 USD.

---

## 5. Outputs of Phase 1

For each selected asset, the following are produced:

| Field | Description |
|----|----|
| `last_atr` | Latest ATR value (pre-backtest) |
| `grid_spacing` | Absolute grid distance |
| `grid_spacing_pct` | Grid distance as % of price |
| `last_close` | Reference close price |

These values are stored in:

spacing_meta[ticker]


---

## 6. Design Principles

- No look-ahead bias
- Volatility-aware parameterization
- Clean separation between selection and execution
- Fully reproducible and deterministic

---

## 7. What Comes Next (Phase 2)

In the next phase, the system will:

- Construct grid price levels using ATR spacing
- Place buy/sell grid orders
- Introduce position sizing rules
- Run the full backtest over the defined period

---

## Summary

Phase 1 ensures that:
- Only strong, liquid markets are traded
- Grid spacing adapts dynamically to market conditions
- The strategy starts from a statistically sound universe

This phase sets the **structural integrity** of the entire grid trading system.
