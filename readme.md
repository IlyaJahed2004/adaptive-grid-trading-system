# 📉 Quantitative Martingale Grid Strategy (Long-Only)

![Python](https://img.shields.io/badge/Python-3.8%2B-blue?style=for-the-badge&logo=python)
![Pandas](https://img.shields.io/badge/Pandas-Data%20Analysis-150458?style=for-the-badge&logo=pandas)
![Status](https://img.shields.io/badge/Status-Phase%203%20Complete-success?style=for-the-badge)

## 📋 Executive Summary

This repository contains the implementation of a **Long-Only Martingale Grid Bot** designed for cryptocurrency markets. This project is part of a university assignment (Homework 5), aiming to simulate a high-frequency grid trading strategy that combines **Trend Following** with aggressive **Mean Reversion**.

Unlike static grid bots, this system employs a **Dynamic Volatility Model (ATR)** to adapt grid spacing to market conditions and uses **Geometric Martingale Sizing** to aggressively lower the average entry price during market dips.

### 🏗️ Core Architecture

The project is structured into three distinct modular phases:

1.  **Phase 1 (Data Engineering):** Universe selection, regime filtering, and volatility profiling.
2.  **Phase 2 (The Mathematics):** Grid level calculation, position sizing logic, and WAP (Weighted Average Price) math.
3.  **Phase 3 (The Engine):** A multi-timeframe backtesting engine (1D, 4H, 1H) with a state-machine architecture.

---

## 🔹 Phase 1: Universe Selection & Data Engineering

The foundation of any good strategy is clean data. We do not trade every asset; we strictly filter for "High Quality" trends to avoid the "Martingale Death Spiral."

### 1. Market Universe Selection
We utilize the **CoinMarketCap API** to fetch the top cryptocurrencies by market capitalization.
-   **Criteria:** Top 50-70 assets.
-   **Exclusions:** Stablecoins (USDT, USDC, DAI) and Wrapped assets (WETH, WBTC) are automatically removed to ensure we are trading volatile assets.

### 2. Regime Filtering (The "Trend" Filter)
Martingale strategies are vulnerable to prolonged bear markets. To mitigate this, we implement a **Regime Filter** based on the 200-day Exponential Moving Average (EMA200).

*   **Logic:** $Price > EMA_{200}$ indicates a structural bull market.
*   **The Filter:** An asset is selected for the backtest **ONLY IF** its daily close price was above the EMA200 for at least **60%** of the pre-load period (e.g., 2023-2024).
*   **Outcome:** This removes assets that are in a long-term downtrend, significantly reducing the probability of hitting the Stop Loss.

### 3. Volatility Profiling (ATR)
Instead of using fixed percentage grids (e.g., placing lines every 1%), we normalize grid spacing using the **Average True Range (ATR)**.
*   **Indicator:** ATR with a period of 14 days.
*   **Result:**
    *   *High Volatility Coin (e.g., SOL):* Wider grid spacing to absorb noise.
    *   *Low Volatility Coin (e.g., BTC):* Tighter grid spacing to capture smaller moves.

---


## 🔹 Phase 2: Mathematical Modeling (The Physics)

This phase defines the static rules of engagement. It calculates *where* to buy and *how much* to buy.

### 1. Grid Topology (Levels)
The grid consists of a **Base Order** (Level 0) and several **Safety Orders** (Level 1 to N).
Levels are calculated downwards from the initial entry price ($P_{entry}$).

$$ Level_i = P_{entry} \times (1 - (i \times Spacing_{dynamic})) $$

*   **$i$**: The index of the grid level ($0$ to $N$).
*   **$Spacing_{dynamic}$**: Calculated as $ATR(14) \times Multiplier$.
*   **Logic:** If the price drops to $Level_i$, a Limit Order is triggered.

### 2. Martingale Position Sizing
We use a geometric progression to scale position sizes. This is the "Martingale" component. As the price drops, the bot buys significantly more volume to pull the average price down closer to the current market price.

$$ Size_i (USD) = BaseOrder \times (Multiplier)^i $$

**Example Sequence** (Base \$100, Multiplier 1.5x):
*   **Level 0:** \$100
*   **Level 1:** \$150
*   **Level 2:** \$225
*   **Level 3:** \$337
*   ...

### 3. Weighted Average Price (WAP) Calculator
The "Break-Even" price is dynamic. It updates every time a safety order is filled. This is the most critical formula in the strategy.

$$ WAP_{new} = \frac{\text{Total USD Invested}}{\text{Total Tokens Held}} $$

$$ WAP_t = \frac{\sum_{i=0}^{k} (Price_i \times Volume_i)}{\sum_{i=0}^{k} Volume_i} $$

*   **Strategic Advantage:** By buying larger amounts at lower prices, the $WAP$ drops faster than the market price drops. This allows the strategy to exit with a profit even if the price only recovers slightly (it does not need to return to the initial entry price).

### 4. Grid Configuration Parameters
| Parameter | Description |
| :--- | :--- |
| **Max Levels** | Maximum number of safety orders (e.g., 8 or 10). |
| **Base Order** | USD value of the first trade. |
| **Volume Scale** | The multiplier for the Martingale sequence (e.g., 1.5). |
| **Step Scale** | (Optional) Multiplier to widen the grid spacing as levels go deeper. |

---



## 🔹 Phase 3: The Execution Engine

Phase 3 is the dynamic simulation. It runs a **State Machine** over historical data (`1D`, `4H`, `1H`) to execute trades, manage wallet balances, and track equity.

### 1. The Logic Loop (State Machine)
The engine processes market data candle-by-candle. For every timestamp, the bot exists in one of two primary states:

#### A. State: `GRID_INACTIVE` (Idle)
*   **Action:** Immediately place a **Market Buy** order (Base Order / Level 0).
*   **Transition:** State becomes `GRID_ACTIVE`.
*   **Calculation:** Calculate all future safety levels and buy limits based on current $ATR$ and $Price$.

#### B. State: `GRID_ACTIVE` (Monitoring)
The engine checks three conditions in specific order of priority:

1.  **Check Stop Loss (Hard Risk Control):**
    *   **Trigger:** If $Low_{candle} \le Level_{Max}$ (e.g., Level 10).
    *   **Action:** **Market Sell** ALL holdings immediately.
    *   **Result:** Realize loss, disable grid for the remainder of the month (Capital Preservation Mode).

2.  **Check Take Profit (Basket Exit):**
    *   **Trigger:** If $High_{candle} \ge WAP \times (1 + 0.025)$ (Target is WAP + 2.5%).
    *   **Action:** **Market Sell** ALL holdings.
    *   **Result:** Realize profit, add to Cash balance.
    *   **Transition:** State becomes `GRID_INACTIVE` (Ready to re-enter immediately).

3.  **Check Safety Orders (DCA):**
    *   **Trigger:** If $Low_{candle} \le Next\_Level\_Price$.
    *   **Mechanism:** Uses **Limit Order** logic. We assume the order filled at the specific grid level price, not the candle close.
    *   **Action:** Buy the calculated Martingale size. Update $WAP$. Update Holdings.

### 2. Multi-Timeframe Batching
To ensure high-fidelity testing without memory overflows:
*   **1D Data:** Pre-loaded for the entire year (used for ATR/EMA calculations).
*   **Intraday (1H/4H):** Fetched dynamically **Month-by-Month**.
    *   The engine downloads Jan 2025 (1H), runs the sim, saves results, clears memory, then downloads Feb 2025.

### 3. Capital Management
*   **Equity Tracking:** $Equity = Cash + (Holdings \times ClosePrice)$.
*   **ROI Calculation:** Returns are calculated based on the starting capital ($10,000) vs final equity.

---
