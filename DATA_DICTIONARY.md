# Coursework 1: Data Dictionary

**Maintainer:** Investment Specialist (IS) Team

**Context:** This data dictionary defines the schema for our PostgreSQL databases, tailored for the Flow-Based Multi-Factor Strategy.

---

### 1. `company_static` (Static Company Information)
Stores fundamental metadata for the investable universe.

| Field Name | Data Type | Constraints | Source | Description / Notes |
| :--- | :--- | :--- | :--- | :--- |
| `ticker` | VARCHAR(15) | PRIMARY KEY | yfinance | Stock ticker. **Note: Must strip trailing whitespace.** |
| `company_name` | VARCHAR(100)| NOT NULL | yfinance | Full name of the company. |
| `sector` | VARCHAR(50) | NULLABLE | yfinance | GICS Sector (e.g., Technology, Healthcare). |
| `industry` | VARCHAR(50) | NULLABLE | yfinance | GICS Industry classification. |
| `currency` | VARCHAR(5) | NOT NULL | Derived | Inferred from ticker suffix (USD, GBP, EUR, CAD, CHF). |

### 2. `daily_prices` (Daily Market Data)
Time-series data required for Momentum, MinVar optimization, and VaR computations.

| Field Name | Data Type | Constraints | Source | Description / Notes |
| :--- | :--- | :--- | :--- | :--- |
| `ticker` | VARCHAR(15) | FOREIGN KEY | yfinance | Stock ticker (maps to `company_static`). |
| `trade_date` | DATE | NOT NULL | yfinance | Trading date (YYYY-MM-DD). |
| `open` | FLOAT | NULLABLE | yfinance | Daily open price. |
| `high` | FLOAT | NULLABLE | yfinance | Daily high price. |
| `low` | FLOAT | NULLABLE | yfinance | Daily low price. |
| `close` | FLOAT | NULLABLE | yfinance | Daily raw close price. |
| `adj_close` | FLOAT | NOT NULL | yfinance | Adjusted close price (Critical for momentum). |
| `volume` | BIGINT | NULLABLE | yfinance | Daily trading volume. |

### 3. `fundamentals` (Quarterly Fundamentals & Valuation)
Accounting and valuation metrics for Value and Quality factors.

| Field Name | Data Type | Constraints | Source | Description / Notes |
| :--- | :--- | :--- | :--- | :--- |
| `ticker` | VARCHAR(15) | FOREIGN KEY | yfinance | Stock ticker. |
| `report_date`| DATE | NOT NULL | yfinance | Reporting or recording date of metrics. |
| `market_cap` | BIGINT | NULLABLE | yfinance | Market Capitalization. |
| `net_income` | FLOAT | NULLABLE | yfinance | Net Income (for Quality factor: ROE). |
| `shareholders_equity`| FLOAT | NULLABLE | yfinance | Shareholders' Equity (for ROE & leverage). |
| `total_debt` | FLOAT | NULLABLE | yfinance | Total Debt (for Quality factor: Leverage). |
| `eps` | FLOAT | NULLABLE | yfinance | Earnings Per Share. |
| `book_value_per_share`| FLOAT | NULLABLE | yfinance | Book Value Per Share. |
| `pe_ratio` | FLOAT | NULLABLE | yfinance | Pre-calculated Price-to-Earnings (P/E) ratio. |
| `pb_ratio` | FLOAT | NULLABLE | yfinance | Pre-calculated Price-to-Book (P/B) ratio. |
| `ev_ebitda` | FLOAT | NULLABLE | yfinance | Enterprise Multiple (EV/EBITDA). |

### 4. `macro_and_fx` (Macroeconomic, FX & Benchmarks)
Market environment data for risk management and performance benchmarking.

| Field Name | Data Type | Constraints | Source | Description / Notes |
| :--- | :--- | :--- | :--- | :--- |
| `trade_date` | DATE | NOT NULL | yfinance | Trading date (YYYY-MM-DD). |
| `vix_close` | FLOAT | NULLABLE | yfinance | CBOE Volatility Index (`^VIX`). |
| `risk_free_rate`| FLOAT | NULLABLE | yfinance | 3-Month Treasury Bill rate (`^IRX`). |
| `sp500_adj_close`| FLOAT | NULLABLE | yfinance | S&P 500 benchmark (`^GSPC`). |
| `gbp_usd` | FLOAT | NULLABLE | yfinance | GBP to USD daily exchange rate (`GBPUSD=X`). |
| `eur_usd` | FLOAT | NULLABLE | yfinance | EUR to USD daily exchange rate (`EURUSD=X`). |
| `cad_usd` | FLOAT | NULLABLE | yfinance | CAD to USD daily exchange rate (`CADUSD=X`). |
| `chf_usd` | FLOAT | NULLABLE | yfinance | CHF to USD daily exchange rate (`CHFUSD=X`). |
