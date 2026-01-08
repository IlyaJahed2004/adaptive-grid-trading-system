# Phase 1 — Market Universe Selection & Volatility-Aware Grid Construction

This phase builds a robust and lookahead-safe data pipeline for selecting tradable crypto assets and constructing adaptive grid parameters based on market volatility.

The goal of this phase is to prepare a clean and realistic trading universe for the Martingale Grid strategy used in later phases.

---

## Objectives

- Select a tradable universe from the top cryptocurrencies by market capitalization
- Filter assets using a long-term trend regime (EMA200)
- Remove all forms of lookahead bias
- Compute volatility-aware grid spacing using ATR
- Store clean historical data for multi-timeframe backtesting

---

## Data Source

- Market universe is obtained from **CoinMarketCap API** (Top 70 coins by market cap)
- Historical OHLC data is fetched from **Yahoo Finance (yfinance)**
- All timestamps are standardized to **UTC timezone**

---

## Backtest Window

- **Backtest period:**  
  `2025-01-01 → 2025-12-31`

- **Historical preload period:**  
  `2023-01-01 → 2024-12-31`  
  (used for indicator computation and regime filtering)

---

## Step 1 — Market Universe Construction

1. Fetch top 70 cryptocurrencies by market capitalization.
2. Download full daily OHLC history for each asset.
3. Store full historical data in a dictionary for later use in Phase 3.

Assets with missing or invalid price data are automatically discarded.

---

## Step 2 — Regime Filter using EMA200

A long-only grid strategy performs best in bullish or trending markets.  
To ensure we only trade in favorable regimes, an EMA200 filter is applied.

For each asset:

- EMA200 is computed **only on pre-backtest data**
- The fraction of daily closes above EMA200 is measured
- Assets are kept only if more than **60% of closes were above EMA200**

This ensures the strategy operates only on structurally strong assets.

---

## Step 3 — Volatility-Aware Grid Spacing (ATR)

Instead of using a fixed grid size, spacing is derived from real market volatility.

For each selected asset:

- ATR(14) is computed on pre-backtest data
- Grid spacing is defined as:

grid_spacing = ATR × GRID_MULTIPLIER


- Grid spacing is also stored as a percentage of price for normalization

This makes the grid adaptive to each asset’s volatility profile.

---

## Key Design Principles

- **No Lookahead Bias**  
  All indicators are computed strictly before the backtest window.

- **Volatility Adaptation**  
  Each asset has its own grid size based on real volatility.

- **Clean Market Regime Filter**  
  The strategy operates only in bullish environments.

- **Reusable Data Pipeline**  
  All historical data is stored for later multi-timeframe backtesting.

---

## Output Artifacts

At the end of Phase 1, the pipeline produces:

- A filtered trading universe (EMA200 regime filter)
- Historical OHLC data for each asset
- ATR-based grid spacing metadata for each asset

These outputs are consumed directly by Phase 2 (grid construction) and Phase 3 (strategy backtesting).

---

## Summary

Phase 1 establishes a professional-grade market data pipeline and asset selection process.  
It ensures that all subsequent strategy evaluation is performed on realistic, bias-free, and volatility-aware data.

This phase forms the backbone of the entire trading system.




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


## Summary

Phase 2 defines **how the grid behaves**, while Phase 3 will define **how the grid trades**.
This separation ensures clarity, correctness, and extensibility of the trading system.


# Phase 3 – Multi-Timeframe Grid Strategy Evaluation

## Overview

In this phase, a volatility-adjusted grid trading strategy is evaluated on **real cryptocurrency market data**.
The objective is to validate the **practical behavior** of the strategy across different market regimes,
time resolutions, and assets, rather than to optimize parameters.

This phase builds directly on the outputs of Phase 1 (asset selection) and Phase 2 (grid construction).

---

## Strategy Summary

The implemented strategy is a **long-only grid trading system** with the following characteristics:

- Grid levels are placed below a reference price
- Spacing between levels is derived from market volatility (ATR)
- Position sizes increase progressively for deeper grid levels
- Each grid position has its own predefined exit level
- Global risk is controlled via basket-level take-profit and hard stop-loss rules

The strategy operates in a fully rule-based and deterministic manner.

---

## Data and Assets

- Market data is sourced from Yahoo Finance
- Asset universe is derived from the top cryptocurrencies by market capitalization
- Assets are filtered using a long-term trend regime condition (EMA200)
- Approximately **35–40 symbols** pass the filter and are used in Phase 3

Only assets that satisfy the regime condition prior to the backtest period are included, ensuring no lookahead bias.

---

## Backtest Design

### Time Horizon
- Full calendar year: **January 2025 – December 2025**

### Evaluation Method
The backtest is structured as **independent rolling monthly simulations**:

- Each month is treated as a separate experiment
- Strategy state is reset at the start of every month
- No information is carried over between months

This design allows the analysis of consistency and robustness over time.

### Timeframes
The strategy is tested independently on three different time resolutions:

- 1-hour candles
- 4-hour candles
- Daily candles

This results in:

- **12 months × 3 timeframes × ~38 assets**
- Approximately **1,300 individual strategy runs**

---

## Execution Logic (High-Level)

For each monthly run:

- Price evolution is processed sequentially in time
- Grid buy entries are triggered when price reaches predefined levels
- Individual positions are closed when their respective sell levels are reached
- A basket-level take-profit closes all open positions once the weighted average entry price is exceeded by a fixed percentage
- A hard stop-loss closes all positions if price moves beyond the deepest grid level

The strategy does not assume perfect foresight and only reacts to information available at each candle close.

---

## Logging and Metrics

For each run, the following metrics are recorded:

- Realized profit and loss (PnL)
- Number of executed trades
- Whether the month ended via take-profit or stop-loss

Results are then aggregated by timeframe to compute:

- Mean monthly PnL
- Mean number of trades per month
- Take-profit frequency
- Stop-loss occurrence rate

This aggregation provides a concise overview of strategy behavior across resolutions.

---

## Key Observations

- Strategy performance varies meaningfully across timeframes
- Shorter timeframes generate higher trade frequency and higher average monthly PnL
- Stop-loss events remain rare across all resolutions, indicating effective volatility-adjusted spacing
- Results suggest stable behavior rather than overfitting to a single timeframe

These observations indicate that the strategy logic is structurally sound and suitable for further analysis.

---

## Design Notes

- The implementation prioritizes clarity and correctness over complexity
- No parameter optimization is performed in this phase
- Certain safety mechanisms (e.g. one-time fill constraints) are intentionally omitted to keep the core logic transparent
- The framework is designed to be easily extendable toward risk normalization, capital allocation, or live trading simulation

---

## Conclusion

Phase 3 confirms that the grid strategy, when combined with regime filtering and volatility-aware spacing,
behaves consistently across assets, timeframes, and market conditions.

This phase completes the end-to-end implementation of the strategy on real market data and establishes a
solid foundation for subsequent optimization or deployment phases.
