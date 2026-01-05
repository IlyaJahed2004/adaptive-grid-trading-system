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

## Summary

Phase 1 ensures that:
- Only strong, liquid markets are traded
- Grid spacing adapts dynamically to market conditions
- The strategy starts from a statistically sound universe

This phase sets the **structural integrity** of the entire grid trading system.




# Phase 2 – Grid Construction, Position Sizing, and Average Price Logic

## Overview

Phase 2 transforms the outputs of Phase 1 into an **operational grid trading structure**.
While Phase 1 focused on asset selection and volatility estimation, Phase 2 defines **how trades are structured and managed**.

No capital deployment or backtest execution is performed yet.
This phase only defines **grid mechanics**.

---

## Inputs from Phase 1

Phase 2 relies directly on outputs generated in Phase 1:

| Input | Description |
|----|----|
| Selected assets | Assets filtered using EMA200 trend criteria |
| Grid spacing | ATR-based volatility-adjusted spacing |
| Last close price | Reference price for grid placement |

These inputs ensure the grid is:
- Trend-aware
- Volatility-adaptive
- Deterministic and reproducible

---

## Phase 2.1 – Grid Buy Level Construction

### Objective

Generate a **fixed number of buy levels** below the current market price.

### Design

- Buy levels are placed **below the current price**
- The number of levels is fixed (e.g., 10)
- Distance between levels is defined by ATR-based grid spacing

### Logic

BuyLevel_i = CurrentPrice − i × GridSpacing


Where:
- `i = 1 ... N`
- `N` = number of grid levels

### Output

- Ordered list of buy price levels
- Corresponding sell levels (one grid step above each buy)

---

## Phase 2.2 – Exponential Position Sizing (Martingale)

### Objective

Increase position size as price moves lower in the grid.

### Rationale

- Lower prices increase rebound probability
- Larger size reduces average entry price faster

### Position Sizing Rule

Position sizes follow a geometric progression:

Size_i = BaseSize × Multiplier^i

Example:

[100, 200, 400, 800, ...]


This approach is commonly known as **Martingale-style averaging**.

---

## Phase 2.3 – Average Price Calculation of the Active Grid

### Definition of Grid

The grid represents **all executed buy orders** up to the current moment.
Unfilled buy levels are not part of the grid.

### Objective

Compute the **true average entry price** of the accumulated position.

### Formula

Weighted average price:

\[
AveragePrice = \frac{\sum (BuyPrice_i × Size_i)}{\sum Size_i}
\]

### Importance

- Determines profitability thresholds
- Used for sell decision logic in later phases
- Reflects actual capital exposure

---

## Key Properties of Phase 2

- Stateless grid definition
- No look-ahead bias
- Fully deterministic
- Modular and testable
- Ready for integration with execution logic

---

## Outputs of Phase 2

For each asset:

| Output | Description |
|----|----|
| Buy levels | Fixed grid buy prices |
| Sell levels | Exit prices per grid step |
| Position sizes | Exponential sizing per level |
| Average price | Dynamic weighted entry price |

These outputs serve as **inputs to Phase 3 execution logic**.

---

## What Comes Next (Phase 3)

Phase 3 will:
- Simulate order execution
- Manage active positions
- Track capital and PnL
- Perform full backtesting over the selected period

---

## Summary

Phase 2 defines **how the grid behaves**, while Phase 3 will define **how the grid trades**.
This separation ensures clarity, correctness, and extensibility of the trading system.
