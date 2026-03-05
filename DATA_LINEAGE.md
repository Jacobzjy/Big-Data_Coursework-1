## 2. Data Lineage and Transformation Mapping

### 2.1 Lineage Overview
Data lineage tracks the lifecycle of financial data from its raw extraction point to its final state in the structured warehouse. In this pipeline, the lineage follows a strict Polyglot Persistence paradigm: **External Source -> MinIO (Raw Data Lake) -> Python ETL (Transformation) -> PostgreSQL (Structured Warehouse) -> Jupyter (CW2 Analytics).** Following the IS team's strategic decision to deprecate the ESG news factor (due to <34% universe coverage), the pipeline strictly processes structured numeric data. The explicit lineage for each core alpha driver is documented below.



### 2.2 Lineage 1: Time-Series Market Data (Prices & Volume)
This data stream powers the **Momentum Factor** (12M-1M returns), **Minimum Variance Optimization**, and the **Historical VaR** risk model.

* **Source System:** Yahoo Finance API (`yfinance`).
* **Ingestion Trigger:** Polled based on the 678 tickers defined in `company_static`.
* **Raw Storage:** `s3://datalake/raw/prices/YYYY-MM-DD/` (JSON/CSV files in MinIO).
* **IS Transformation & Cleaning Rules:**
  1. **Ticker Sanitization:** `.strip()` trailing whitespaces from database inputs.
  2. **Exchange Mapping:** Swiss equities ending in `.S` are programmatically mapped to the `.SW` suffix required by Yahoo Finance.
  3. **Missing Value Imputation:** Forward-fill (`ffill`) missing `adj_close` prices for a maximum of 3 consecutive trading days to handle micro-halts without fabricating long-term trends.
  4. **Data Type Casting:** Force `adj_close` to `NUMERIC(15,4)` and `volume` to `BIGINT`.
* **Target Destination:** PostgreSQL -> `systematic_equity.daily_prices` table.
* **CW2 Consumption:** Cross-sectional ranking for Momentum; Rolling covariance matrix estimation.

### 2.3 Lineage 2: Deep Historical Fundamentals
This data stream powers the **Value Factor** (B/P, E/P, CF/P) and **Quality Factor** (ROE, Earnings Stability, Leverage Safety).

* **Source System:** Financial Modeling Prep (FMP) API (Primary for deep quarterly history) & `yfinance` (Fallback).
* **Ingestion Trigger:** Quarterly batch requests triggered by ticker list.
* **Raw Storage:** `s3://datalake/raw/fundamentals/YYYY-MM-DD/` (MinIO).
* **IS Transformation & Cleaning Rules:**
  1. **Feature Extraction:** Extract `book_value`, `net_income`, `shareholders_equity`, `total_debt`, `eps`, and the newly added `operating_cash_flow` (required for v7.0 CF/P calculation).
  2. **Null Handling:** If a company is newly listed and lacks 3-year historical EPS (preventing Earnings Stability calculation), the ETL pipeline inserts `NULL`. CW2 logic will handle cross-sectional median imputation or penalty ranking.
  3. **Temporal Alignment:** Align disparate fiscal reporting dates (`report_date`) to the nearest standard calendar quarter end.
* **Target Destination:** PostgreSQL -> `systematic_equity.fundamentals` table.
* **CW2 Consumption:** Z-score standardization for Value and Quality composites.

### 2.4 Lineage 3: Macroeconomic & FX Reference Data
This data stream powers the **Volatility Regime Classification** and ensures currency normalization across the global investable universe.

* **Source System:** Yahoo Finance API.
* **Ingestion Trigger:** Daily extraction of specific tickers (`^VIX`, `GBPUSD=X`, `EURUSD=X`, `CADUSD=X`, `CHFUSD=X`).
* **Raw Storage:** `s3://datalake/raw/macro/YYYY-MM-DD/` (MinIO).
* **IS Transformation & Cleaning Rules:**
  1. **Date Alignment:** Outer join FX rates and VIX closes to the standard NYSE trading calendar. Fill missing FX rates (due to bank holidays) using the previous day's rate (Forward-fill).
* **Target Destination:** PostgreSQL -> `systematic_equity.macro_and_fx` table.
* **CW2 Consumption:** VIX percentiles determine factor weight adjustments; FX rates normalize all local equity returns into USD base returns for the portfolio optimizer.
