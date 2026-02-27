# Data Lineage & Transformation Log

**Maintainer:** Investment Specialist (IS) Team  
**Project:** Flow-Based Multi-Factor Strategy Data Pipeline  
**Context:** This document traces the lifecycle of our data from external extraction to final consumption in portfolio construction. It ensures transparency, auditability, and data quality.

---

## 1. High-Level Data Lineage Flow

| Pipeline Stage | Component | Output/Format | Purpose in Architecture |
| :--- | :--- | :--- | :--- |
| **1. Source** | Yahoo Finance API (`yfinance`) | Raw HTTP Responses | External data acquisition layer. |
| **2. Ingestion** | Apache Kafka | Streaming Topics | Buffers API responses to prevent rate-limit crashes and decouples extraction from processing. |
| **3. Raw Storage** | MinIO (Data Lake) | Immutable JSON/CSV | Stores an exact copy of the raw API response for backup, auditing, and potential reprocessing. |
| **4. ETL Processing** | Python Workers (Docker) | Validated DataFrames | Cleans, normalizes, and type-casts the raw data according to IS rules. |
| **5. Metadata** | MongoDB (NoSQL) | BSON Documents | Logs ETL execution metrics, dead-letter queues (failed tickers), and validation errors. |
| **6. Structured DB** | PostgreSQL | Relational Tables | The final structured data warehouse holding `company_static`, `daily_prices`, `fundamentals`, and `macro_and_fx`. |
| **7. Consumption** | Jupyter / CW2 Engine | Strategy Factors | Queries PostgreSQL to compute Momentum, Value, Quality factors, and Historical VaR. |

---

## 2. Detailed Lineage Paths & Transformation Rules

As part of our Data Governance framework, the ETL layer must strictly apply the following transformation rules before loading data into PostgreSQL.

### Path A: Price & Volume Lineage (For Momentum & MinVar)

| Stage | Details & Rules |
| :--- | :--- |
| **Source** | Yahoo Finance historical data (`yfinance.download`). |
| **Raw Format** | JSON array of daily OHLCV (Open, High, Low, Close, Volume) values. |
| **Transformations (ETL Layer)** | **1. Ticker Normalization:** Strip trailing whitespaces (`.strip()`) from `company_static`. <br>**2. Suffix Mapping:** Convert Swiss exchange suffixes (e.g., `NOVN.S` strictly mapped to `NOVN.SW`). <br>**3. Missing Data Treatment:** Forward-fill (`ffill`) missing `adj_close` prices for a maximum of 3 consecutive trading days. If missing > 3 days, flag the ticker in MongoDB error logs. |
| **Destination** | `daily_prices` table in PostgreSQL. |
| **Consumer (CW2)** | Momentum Alpha signal (using `adj_close`) and Minimum Variance Covariance Matrix. |

### Path B: Fundamentals Lineage (For Value & Quality)

| Stage | Details & Rules |
| :--- | :--- |
| **Source** | Yahoo Finance quarterly income statements and balance sheets. |
| **Raw Format** | Pandas DataFrames with dates as columns. |
| **Transformations (ETL Layer)** | **1. Transposition:** Melt DataFrame to ensure standard relational format (Row = Ticker + Date). <br>**2. Null Handling:** Null accounting fields (e.g., missing `total_debt`) are cast to SQL `NULL` (not `0`, to avoid skewed ratio calculations). <br>**3. Type Casting:** Cast all financial figures to `FLOAT` and market cap to `BIGINT`. |
| **Destination** | `fundamentals` table in PostgreSQL. |
| **Consumer (CW2)** | Value Factor (B/P, E/P ratios) and Quality Factor (ROE, Debt/Equity). |

### Path C: Macro & FX Lineage (For VaR & Benchmarking)

| Stage | Details & Rules |
| :--- | :--- |
| **Source** | Yahoo Finance tickers (`^VIX`, `^IRX`, `GBPUSD=X`). |
| **Raw Format** | Time-series data points. |
| **Transformations (ETL Layer)** | **1. Date Alignment:** Align all macro dates to standard NYSE trading days to ensure factor joining consistency in CW2. |
| **Destination** | `macro_and_fx` table in PostgreSQL. |
| **Consumer (CW2)** | Historical VaR position scaling and Volatility Regime Classification. |
