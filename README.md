# Pair Trading Backtest Project

This repository contains a full pipeline to **design, test, and evaluate** a statistical arbitrage (pairs trading) strategy using US equities.  
The workflow proceeds from **data collection → pair screening → signal generation → portfolio simulation → results reporting**.

---

## Screening

### 1. Data Collection
- Price data is pulled with `yfinance` from the top 50 stocks in the S&P 500
- We take adjusted close prices for the year 2021 up until the end of 2023

### 2. Log Transformation
- We work with **log-prices** (`log_data = np.log(data)`) to stabilize variance.
- This makes the correlation calculations more consistent across assets.

### 3. Pair Selection
- Correlation matrix between each possible combination of stock is calculated
- All pairs with correlation >= 0.75 are then passed on to test for Cointegration
- Using the ADF (Augmented Dickey Fuller) test, we take stock pairs with p-values <0.05

## Simulation

### 1. Historic Data
- Price data for each selected stock is pulled again with `yfinance`
- We take adjusted close prices 2 Financial Years (252 working days) before 2025-01-01
- We will train the data using the first 252 working days and the second to simulate

### 2. Hedge Ratios
- For each pair `(A, B)` we estimate the **hedge ratio**:
  - Regress `log(B)` on `log(A)` over the **training period** (2023).
  - Extract slope (`b1`) → static hedge ratio.
- Compute weights `wA, wB` so that positions are **dollar-neutral**.

### 3. Spread & Z-Score Calculation
- Spread is defined as:
  \[
  \text{spread}_t = \log(B_t) - (b_0 + b_1 \log(A_t))
  \]
- Compute rolling **mean** and **standard deviation** of the spread.
- Standardize into a **z-score**:
  \[
  z_t = \frac{\text{spread}_t - \mu}{\sigma}
  \]

### 5. Signal & State Logic
- Entry threshold: `|z| > 1.5`  
- Exit threshold: `|z| < 0.5`  
- Stop-loss: `|z| > 3.0`  
- Time-stop: calculated using 2x of half-life of each stock pair.
- State machine encodes:
  - `+1` → long A, short B  
  - `-1` → short A, long B  
  - `0` → flat  

### 6. Position Sizing
- Portfolio starts with **\$100,000 equity**.
- Each active pair receives an **equal gross allocation** (10% of equity).
- Convert to integer shares using hedge ratio weights (`wA, wB`).

### 7. Backtest & PnL
- Positions update **only when state changes**.
- Daily PnL =  
  - Holding PnL from overnight price moves,  
  - Minus trading costs (10 bps per leg per trade),  
  - Minus borrow costs for shorts (3% annualized).
- Portfolio equity is tracked through time.

### 8. Results Reporting
- **Per Pair**:
  - Daily and cumulative PnL  
  - Breakdown of expenses and costs  
  - Exported to `combined_summary.csv`
- **Portfolio**:
  - Daily PnL & cumulative equity  
  - Equity curve plotted over 2024  
  - Exported to `portfolio_summary.csv`

---

## Key Files
- `data.csv` → price data (Adj Close).
- `backtest_pair.py` → functions for single-pair backtesting.
- `portfolio_backtest.py` → portfolio-level backtest combining all pairs.
- `report.py` → builds results table and equity plots.
- `README.md` → this file.

## Next Steps
- Explore alternative allocations (equal-risk instead of equal-gross).
- Add more pairs and re-run out-of-sample tests.
