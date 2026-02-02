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



## 🔹 Phase 3: The Execution Engine (Martingale)

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
    *   **Result:** Realize loss, reset grid state (Ready to re-enter new cycle).

2.  **Check Take Profit (Basket Exit):**
    *   **Trigger:** If $High_{candle} \ge WAP \times (1 + 0.025)$ (Target is WAP + 2.5%).
    *   **Action:** **Market Sell** ALL holdings.
    *   **Result:** Realize profit, add to Cash balance.
    *   **Transition:** State becomes `GRID_INACTIVE` (Ready to re-enter immediately).

3.  **Check Safety Orders (DCA):**
    *   **Trigger:** If $Low_{candle} \le Next\_Level\_Price$.
    *   **Mechanism:** Uses **Limit Order** logic. We assume the order filled at the specific grid level price.
    *   **Action:** Buy the calculated Martingale size ($Volume \times 1.5^n$). Update $WAP$. Update Holdings.

### 2. Multi-Timeframe Batching
To ensure high-fidelity testing without memory overflows:
*   **1D Data:** Pre-loaded for the entire year.
*   **Intraday (1H/4H):** Fetched dynamically **Month-by-Month**.

### 3. Data Persistence (Optimization for Phase 5)
Instead of storing only the final ROI, the engine now utilizes a **Nested Dictionary Structure** to preserve the entire path of the simulation:
*   **Structure:** `all_equity_curves[Ticker][Timeframe][Month_Label]`
*   **Purpose:** This allows Phase 5 to calculate advanced metrics (Sharpe Ratio, Sortino, MDD) and plot visualizations without re-running the heavy simulation loops.




## 🔹 Phase 4: The Anti-Martingale Engine (Pyramiding)

Phase 4 flips the logic of Phase 3. Instead of buying when the price drops (DCA), this strategy focuses on **Trend Following** and **Pyramiding**. It adds to the position only when the price moves *in favor* of the trade.

### 1. The Strategy Logic
The goal is to maximize gains during strong trends ("Ride the Wave") while keeping risk tight via a trailing stop effect.

#### A. Entry Mechanics
*   **Base Order:** Enters at Market Price when the grid is inactive.
*   **Trigger for Add-on:** A new buy order is placed ONLY if the price rises by a specific `spacing_pct` (Dynamic based on ATR).

#### B. Execution Logic (State Machine)
Once active, the engine monitors:

1.  **Take Profit (Momentum Exit):**
    *   **Trigger:** If $High_{candle} \ge WAP \times (1 + 0.025)$.
    *   **Action:** Sell entire position. Secure profit.

2.  **Stop Loss (Trailing Protection):**
    *   **Trigger:** If $Low_{candle} \le WAP \times (1 - 2 \times Spacing)$.
    *   **Dynamic Nature:** Since we buy at higher prices (Pyramiding), the Weighted Average Price ($WAP$) constantly increases. Therefore, the Stop Loss level naturally moves up, acting as a **Trailing Stop**.

3.  **Pyramiding (Adding to Winners):**
    *   **Condition:** If $High_{candle} \ge Next\_Buy\_Trigger$.
    *   **Volume Sizing:** Uses the Martingale Multiplier ($1.5^n$) but applies it on the way **UP**. This is aggressive position sizing during a breakout.
    *   **Constraint:** Max safety orders limited to 8 levels to prevent over-exposure at market tops.

### 2. Data Persistence Strategy
To prevent naming conflicts with Phase 3 and ensure data integrity for analysis:
*   **Storage Container:** `antimartingale_equity_curves`
*   **Structure:** `[Ticker] -> [Timeframe] -> [Month]`
*   **Usage:** This separated store ensures that when we visualize data in Phase 5, we can overlay or compare the "Dip Buying" (Martingale) vs. "Trend Following" (Anti-Martingale) performance curves directly.


