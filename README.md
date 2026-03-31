# Algorithmic Trading System — Fama-French 5-Factor Strategy

An end-to-end algorithmic trading system that constructs a long-short equity portfolio by combining the **Fama-French 5-Factor model** with **institutional sentiment signals** scraped from SEC 13F filings. Trades are executed automatically through Interactive Brokers via the `ib_insync` library.

---

## Overview

This project covers the full trading pipeline — from raw data collection all the way to live order execution — and was built to demonstrate the practical application of academic asset pricing theory in a real brokerage environment.

### The investment idea

The central idea is that stocks can be **systematically mispriced** relative to their fundamental risk profile. When a quantitative model flags a mispricing *and* major institutional investors are moving in the same direction, the signal is stronger than either source alone. The system acts only when both agree.

### Signal 1 — Fama-French 5-Factor Model (FF5)

The FF5 model, developed by Nobel Prize-winner Eugene Fama and Kenneth French, explains stock returns through five risk factors:

| Factor | What it captures |
|--------|-----------------|
| **Mkt-RF** | Exposure to the overall market |
| **SMB** | Small-cap vs. large-cap tilt |
| **HML** | Value vs. growth tilt |
| **RMW** | Profitability of the firm |
| **CMA** | Conservative vs. aggressive investment policy |

For every S&P 500 stock, the system estimates its sensitivity to these five factors using **20 years of monthly return data**, then uses the most recent factor values to forecast what the stock's price *should* be next month. If the predicted price is above today's price → **BUY**. If below → **SELL**.

### Signal 2 — Institutional Sentiment from SEC 13F Filings

Asset managers with over $100M in AUM must disclose their holdings quarterly to the SEC via 13F filings. The system scrapes these filings for **seven of the world's largest institutions** — BlackRock, Vanguard, Fidelity, State Street, Morgan Stanley, JP Morgan, and Goldman Sachs — and calculates the net change in portfolio weight (`WeightDiff`) for each S&P 500 stock. Stocks being collectively accumulated are treated as bullish; stocks being reduced are bearish.

### Portfolio construction

From the stocks where both signals agree, the system selects:

- **Top 5 by institutional accumulation** → long positions
- **Bottom 5 by institutional reduction** → short positions

Weights are proportional to the magnitude of the institutional shift. The result is a **market-neutral, 10-stock long-short portfolio** designed to profit from relative mispricing regardless of overall market direction.

### Execution

The portfolio is submitted to Interactive Brokers via `ib_insync`. Each position gets a **market order** (immediate entry) and a **limit order** at the FF5-predicted price as a take-profit target. A separate liquidation notebook handles end-of-period cleanup: it cancels open orders, closes all positions at market, and prints the realised annual return.

### Tech stack

The project is intentionally split across two languages — **Python** for data processing, statistical modelling, and IB API integration, and **C#/.NET** for the web scraping layer — using each where it performs best.

---

## Project Structure

```
Mega Project/
├── Formation of Portfolio.ipynb       # Main strategy notebook (data, model, execution)
├── Cancellation of the Portfolio.ipynb # Portfolio liquidation and performance calculation
├── final_dataframe.csv                 # Output portfolio: tickers, weights, limit prices
├── Documentation.pdf                   # Full project documentation
├── Outsourcing Paper.pdf               # Research paper / methodology reference
└── Scraping_Program/
    └── Scraping_Updated/
        ├── Scraping.exe                # .NET scraper (S&P 500 tickers + 13F filings)
        ├── Scraping.dll.config         # Configuration (URLs, institutional CIK codes)
        └── ExcelFiles/
            └── Results.xlsx            # Scraper output: tickers + institutional weight changes
```

---

## How It Works

### Step 1 — Data Collection (`Scraping.exe`)

A compiled C# .NET application that collects two things:

- **S&P 500 tickers** from [slickcharts.com](https://www.slickcharts.com/sp500)
- **Institutional holdings changes (WeightDiff)** by parsing 13F filings from SEC EDGAR for seven major asset managers: BlackRock, Vanguard, Fidelity, State Street, Morgan Stanley, JP Morgan, and Goldman Sachs

The output is `ExcelFiles/Results.xlsx`, which contains each ticker alongside the net change in institutional weight (`WeightDiff`) and an aggregated sentiment signal (`Decision`: BUY or SELL).

### Step 2 — Portfolio Formation (`Formation of Portfolio.ipynb`)

The main Python notebook carries out the full investment pipeline:

1. **Fama-French 5-Factor data** is downloaded directly from the Dartmouth website (monthly factors going back 20+ years).
2. **20 years of monthly price data** for all S&P 500 stocks is fetched from Yahoo Finance.
3. **FF5 regressions** are run for each stock to estimate its factor loadings (Mkt-RF, SMB, HML, RMW, CMA).
4. **Price prediction**: using the most recent FF5 factor values and the estimated loadings, a predicted return and hence a predicted price for next month is computed for each stock.
5. **Signal generation**: if `predicted_price > current_price` → BUY signal; otherwise → SELL signal.
6. **Signal confirmation**: only stocks where the FF5 signal matches the institutional sentiment (`Decision`) from the scraper are retained.
7. **Portfolio construction**: the top 5 stocks by `WeightDiff` among confirmed BUYs become the **long leg**; the bottom 5 by `WeightDiff` among confirmed SELLs become the **short leg**. Weights are normalised to sum to 1 within each leg.
8. **Output**: `final_dataframe.csv` is saved with tickers, normalised weights, decisions, and FF5-implied limit prices.
9. **Order execution**: the notebook connects to an Interactive Brokers account via `ib_insync` and places:
   - **Market orders** to enter positions immediately
   - **Limit orders** at the FF5-predicted price as the take-profit target

> **Note**: execution requires Interactive Brokers TWS or IB Gateway to be running. Use port `7497` for paper trading and `7496` for live trading.

### Step 3 — Portfolio Liquidation (`Cancellation of the Portfolio.ipynb`)

At the end of the holding period (typically one year), this notebook:

1. Connects to the same IB account
2. Cancels all outstanding open (limit) orders
3. Closes all positions with market orders (covers shorts, sells longs)
4. Calculates and prints the **annual return** by comparing ending account value to the starting portfolio value saved from Step 2

---

## Requirements

### Python

```
pandas
numpy
requests
statsmodels
matplotlib
seaborn
yfinance
ib_insync
relativedelta (python-dateutil)
openpyxl
```

Install with:

```bash
pip install pandas numpy requests statsmodels matplotlib seaborn yfinance ib_insync python-dateutil openpyxl
```

### C# Scraper

- .NET 6.0 runtime (required to run `Scraping.exe`)
- Libraries bundled in `Scraping_Updated/`: EPPlus, HtmlAgilityPack, CsvHelper, ServiceStack.Text, Microsoft.Extensions.*

---

## Usage

### 1. Run the scraper

```bash
cd "Mega Project/Scraping_Program/Scraping_Updated"
./Scraping.exe
```

This populates `ExcelFiles/Results.xlsx` with fresh ticker and institutional holdings data.

### 2. Run the formation notebook

Open `Formation of Portfolio.ipynb` in Jupyter and run all cells. Make sure:
- IB TWS / IB Gateway is open and connected on port `7497` (paper) or `7496` (live)
- The notebook is run during US market hours (after 09:30 ET)
- Note down the **Total Cash Value** printed in Cell 34 — you will need it for the cancellation step

### 3. At end of holding period: run the cancellation notebook

Open `Cancellation of the Portfolio.ipynb`, enter the saved starting portfolio value in the marked cell, then run all cells to liquidate the portfolio and compute the return.

---

## Example Portfolio Output (`final_dataframe.csv`)

| Ticker | WeightDiff | WeightDiff_normalized | Decision | Next_Month (Limit Price) |
|--------|------------|----------------------|----------|--------------------------|
| FTV    | +0.0045    | −0.0011              | BUY      | $78.02                   |
| IBM    | −2.5278    | +0.6298              | SELL     | $156.59                  |
| PNC    | −0.5059    | +0.1260              | SELL     | $129.22                  |
| JPM    | −0.4045    | +0.1008              | SELL     | $160.76                  |
| XOM    | −0.2965    | +0.0739              | SELL     | $96.03                   |

Positive `WeightDiff_normalized` → short position. Negative → long position.

---

## Important Notes

- This system is designed for **paper trading** by default. Switch to port `7496` only when ready to trade live capital.
- The strategy rebalances on a **monthly** cadence based on the Fama-French data update frequency.
- Past results (e.g., the −50.6% return visible in the cancellation notebook output) reflect paper trading performance and do not guarantee future results.
- This project is for educational purposes. It does not constitute financial advice.
