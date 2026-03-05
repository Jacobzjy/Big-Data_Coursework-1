## 1. Data Dictionary and Relational Schema Design

### 1.1 Schema Overview
The Coursework 1 (CW1) data pipeline populates a highly normalized relational database in PostgreSQL (Schema: `systematic_equity`). The architecture strictly adheres to the principle of storing cleaned, tabular data required for the Coursework 2 (CW2) backtesting engine. Unstructured databases were explicitly excluded following the decision to drop the ESG news sentiment factor due to insufficient data coverage (<34%). 

The following data dictionary defines the exact data types, constraints, and business logic mappings for the core tables.

### 1.2 Table: `company_static`
**Description:** Defines the investable universe of 678 equities. Acts as the primary dimension table.
**Data Source:** Provided CSV (Cleaned by IS pipeline: stripped whitespace, mapped `.S` to `.SW`).

| Column Name | Data Type | Constraints | Description | Strategy Mapping (CW2) |
| :--- | :--- | :--- | :--- | :--- |
| `ticker` | VARCHAR(15) | PRIMARY KEY | Cleaned ticker symbol used for API calls. | Universal identifier for factor scoring. |
| `company_name` | VARCHAR(100) | NOT NULL | Full legal name of the entity. | Reporting and tearsheet generation. |
| `gics_sector` | VARCHAR(50) | NOT NULL | Global Industry Classification Standard sector. | Sector-neutral portfolio constraints. |
| `country` | VARCHAR(50) | NOT NULL | Country of domicile/exchange. | Used for currency mapping. |
| `currency` | VARCHAR(3) | NOT NULL | Inferred base currency (e.g., USD, GBP, CHF). | FX normalization trigger. |

### 1.3 Table: `daily_prices`
**Description:** Stores 5 years of daily time-series market data for all tickers.
**Data Source:** Yahoo Finance API (`yfinance`).

| Column Name | Data Type | Constraints | Description | Strategy Mapping (CW2) |
| :--- | :--- | :--- | :--- | :--- |
| `ticker` | VARCHAR(15) | PK, FK | Foreign key referencing `company_static`. | Time-series join key. |
| `date` | DATE | PK | Trading date (Daily frequency). | Time-series alignment. |
| `close` | NUMERIC(15,4) | NULL | Unadjusted closing price. | Raw price reference. |
| `adj_close` | NUMERIC(15,4) | NOT NULL | Adjusted close (accounts for splits/dividends). | **Momentum Factor** (12M-1M returns), Covariance Matrix, Historical VaR. |
| `volume` | BIGINT | NOT NULL | Daily trading volume. | Liquidity filtering for long/short legs. |

### 1.4 Table: `fundamentals`
**Description:** Stores deep historical quarterly accounting data (minimum 3 years / 12 quarters depth).
**Data Source:** Hybrid extraction (FMP API primarily for deep history, `yfinance` as fallback).

| Column Name | Data Type | Constraints | Description | Strategy Mapping (CW2) |
| :--- | :--- | :--- | :--- | :--- |
| `ticker` | VARCHAR(15) | PK, FK | Foreign key referencing `company_static`. | Cross-sectional join key. |
| `report_date` | DATE | PK | End date of the fiscal quarter. | Temporal alignment with prices. |
| `book_value` | BIGINT | NULL | Total Book Value of Equity. | **Value Factor**: Book-to-Price (B/P). |
| `net_income` | BIGINT | NULL | Quarterly Net Income. | **Quality Factor**: Return on Equity (ROE). |
| `shareholders_equity`| BIGINT | NULL | Total Shareholders' Equity. | **Quality Factor**: ROE and Leverage Safety. |
| `total_debt` | BIGINT | NULL | Short-term + Long-term debt. | **Quality Factor**: Leverage Safety (Debt/Equity). |
| `eps` | NUMERIC(10,4)| NULL | Earnings Per Share. | **Value Factor** (E/P) & **Quality Factor** (Earnings Stability std dev). |
| `operating_cash_flow`| BIGINT | NULL | Cash generated from core operations. | **Value Factor**: Cash-Flow-to-Price (CF/P). |

### 1.5 Table: `macro_and_fx`
**Description:** Centralized storage for macroeconomic indicators and daily currency conversion rates.
**Data Source:** Yahoo Finance API.

| Column Name | Data Type | Constraints | Description | Strategy Mapping (CW2) |
| :--- | :--- | :--- | :--- | :--- |
| `date` | DATE | PRIMARY KEY | Trading date (Daily frequency). | Global alignment. |
| `vix_close` | NUMERIC(10,4)| NOT NULL | Daily close of CBOE Volatility Index (`^VIX`). | **Volatility Regime Classification** (Dynamic factor weighting). |
| `gbp_usd` | NUMERIC(10,6)| NOT NULL | Daily exchange rate (GBP to USD). | Normalizes UK equity returns for MinVar. |
| `eur_usd` | NUMERIC(10,6)| NOT NULL | Daily exchange rate (EUR to USD). | Normalizes EU equity returns for MinVar. |
| `cad_usd` | NUMERIC(10,6)| NOT NULL | Daily exchange rate (CAD to USD). | Normalizes Canadian equity returns for MinVar.|
| `chf_usd` | NUMERIC(10,6)| NOT NULL | Daily exchange rate (CHF to USD). | Normalizes Swiss equity returns for MinVar. |

### 1.6 Table: `ingestion_log` (Pipeline Metadata)
**Description:** Dead-letter queue and operational logging for the ETL pipeline.
**Data Source:** Generated internally by Python ETL Workers.

| Column Name | Data Type | Constraints | Description | Purpose |
| :--- | :--- | :--- | :--- | :--- |
| `log_id` | SERIAL | PRIMARY KEY | Auto-incrementing identifier. | System tracking. |
| `timestamp` | TIMESTAMP | NOT NULL | Execution time of the pipeline batch. | Run duration analysis. |
| `ticker` | VARCHAR(15) | NULL | Ticker that caused the event (if applicable). | Identifying problematic stocks. |
| `status` | VARCHAR(20) | NOT NULL | e.g., 'SUCCESS', 'RATE_LIMITED', 'DELISTED'. | Data quality monitoring. |
| `error_message` | TEXT | NULL | Full stack trace or error description. | Debugging for Developers. |
