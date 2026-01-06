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
