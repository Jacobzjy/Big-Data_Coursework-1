### 4.5 Data Source Evaluation & Technical Alternatives: Overcoming Fundamental Data Scarcity

**4.5.1 The Business Challenge (Feasibility Assessment)**
The Flow-Based Multi-Factor Strategy heavily relies on historical quarterly fundamental data (e.g., Net Income, Total Debt, Book Value) to compute the Value and Quality factors. Initial testing revealed a critical limitation with our primary data source, the `yfinance` library: while it excels at retrieving daily adjusted closing prices, its `quarterly_financials` method typically only returns the trailing 4 quarters (1 year) of fundamental data. This is insufficient for the PMs' requirement of a 5-year historical backtest.

**4.5.2 Technical Alternatives Research**
To resolve this data scarcity without incurring premium subscription costs, the Investment Specialist (IS) team conducted a comparative analysis of alternative Financial Data APIs.

| Evaluation Criteria | Yahoo Finance (`yfinance`) | Alpha Vantage | Financial Modeling Prep (FMP) |
| :--- | :--- | :--- | :--- |
| **Historical Fundamentals** | Limited (Trailing 4 quarters only) | **Excellent** (Up to 20 years quarterly) | **Excellent** (Up to 30 years quarterly) |
| **Price & Volume Data** | **Excellent** (Deep history, minute-level) | Good (Adjusted close available) | Good |
| **Free Tier Rate Limits** | **Unlimited** (IP-based blocking only) | 25 requests / day (Very Restrictive) | **250 requests / day** (Generous) |
| **JSON Response Quality** | Requires complex Pandas parsing | Clean and structured | Exceptionally clean and structured |
| **Cost** | Free | Requires Paid Tier (~$50/mo) | **Free** (Manageable over 3 days) |

**4.5.3 Architectural Recommendation: The Hybrid API Approach**
Based on the research above, relying solely on one API presents a bottleneck. Therefore, we propose a **Hybrid API Architecture** for the ETL pipeline:

1.  **For Prices, Macro & FX (High Volume):** We will continue to use `yfinance`. Its lack of strict rate limits allows the developers to rapidly download 5 years of daily `adj_close` and `volume` data for all 678 tickers, alongside `^VIX` and risk-free rates.
2.  **For Fundamentals (Deep History):** We will pivot to **Financial Modeling Prep (FMP)** as the primary source for the `fundamentals` table. FMP provides deep historical quarterly data. Although the free tier is capped at 250 requests per day, developers can implement a scheduled batch-downloading script to retrieve the 678 tickers over a 3-day window, storing the raw JSON responses directly into our MinIO Data Lake to prevent ever needing to re-download them.

This hybrid approach perfectly balances the PMs' deep historical data requirements with the project's zero-budget constraint.
