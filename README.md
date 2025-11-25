# 📊 Lab 2 – End-to-End Stock Analytics Pipeline

**Course:** DATA 226  
**Author:** Akash Kumar, Centhur velan
**Tools:** Airflow • Snowflake • dbt • Preset

---

## 🎯 Objective

Build an end-to-end data analytics pipeline using **Airflow**, **Snowflake**, **dbt**, and **Preset** to process, transform, and visualize stock-price data.

This lab extends Lab 1 (ETL + Stock Load) by adding:
- ✔ ELT transformations using **dbt**
- ✔ Automated orchestration in **Airflow**
- ✔ BI dashboards using **Preset/Superset**

---

## 🏗️ System Architecture

The project follows a modern **ELT pattern**:

### **Airflow (ETL)**
- Extract stock data using `yfinance`
- Light transformation + cleanup
- Load into Snowflake → `RAW.TWO_STOCK_V2`

### **Snowflake (Storage)**
- **RAW schema** → original API-loaded data
- **ANALYTICS schema** → dbt-created staging + fact tables

### **dbt (ELT)**
- `stg_stock_data.sql` → cleans & standardizes
- `fct_stock_metrics.sql` → moving averages, percent change
- **Tests**: not_null, unique, custom generic test
- **Snapshots**: track historical data changes

### **Airflow Orchestration**
Two DAGs:
1. **ETL DAG** → loads raw data
2. **dbt DAG** → run → test → snapshot

### **Preset (BI Visualization)**
Visualizes metrics from ANALYTICS schema.

---

## 📂 Project Structure
```
project-root/
├── airflow/
│   └── dags/
│       ├── lab_etl.py              # ETL pipeline
│       └── dbt_stock_analytics.py  # dbt orchestration
├── dbt/
│   └── stock_analytics/
│       ├── models/
│       │   ├── stg_stock_data.sql
│       │   └── fct_stock_metrics.sql
│       ├── snapshots/
│       │   └── stock_prices_snapshot.sql
│       ├── tests/
│       ├── schema.yml
│       ├── dbt_project.yml
│       └── profiles.yml
├── docker-compose.yaml
├── Dockerfile
└── README.md
```

---

## 🔧 Components Summary

### 1️⃣ **Airflow – ETL Pipeline**
**File:** `airflow/dags/lab_etl.py`

- Calls Yahoo Finance API for stock symbols
- Loads merged & cleaned data to Snowflake RAW table
- **Uses:**
  - Snowflake connection (`snowflake_conn`)
  - Variables:
    - `stock_symbols = "AAPL,MSFT"`
    - `lookback_days = 180`

### 2️⃣ **Airflow – dbt ELT Pipeline**
**File:** `airflow/dags/dbt_stock_analytics.py`

Runs dbt in this order:
```bash
dbt run
dbt test
dbt snapshot
```

Triggered after ETL DAG completes using `TriggerDagRunOperator`.

### 3️⃣ **dbt Models**
**Folder:** `dbt/stock_analytics/models/`

**Staging Model:** `stg_stock_data.sql`
- Cleans raw loaded data
- Standardizes column names
- Removes duplicates

**Fact Model:** `fct_stock_metrics.sql`
Computes:
- 7-day moving average
- 30-day moving average
- Daily percent change

**Tests:** `schema.yml`
- `not_null` checks
- `unique` checks
- Custom test: `unique_symbol_date`

**Snapshots:** `stock_prices_snapshot.sql`
- Tracks historical changes using Type 2 SCD

### 4️⃣ **Visualization – Preset/Superset**

Dashboard includes:
- Daily close price + moving averages line chart
- Rolling 7-day moving average
- Monthly percent change
- Symbol selection filter

---

## ☁️ Snowflake Configuration

### **Account Details**
- **Account ID:** `sfedu02-lvb17920`
- **Database:** `USER_DB_PLATYPUS`
- **Warehouse:** `PLATYPUS_QUERY_WH`

### **Required Schemas**
```sql
-- Create schemas if they don't exist
CREATE SCHEMA IF NOT EXISTS USER_DB_PLATYPUS.RAW;
CREATE SCHEMA IF NOT EXISTS USER_DB_PLATYPUS.ANALYTICS;
```

### **Required Tables**
The ETL pipeline creates:
- `USER_DB_PLATYPUS.RAW.TWO_STOCK_V2`

dbt creates:
- `USER_DB_PLATYPUS.ANALYTICS.STG_STOCK_DATA`
- `USER_DB_PLATYPUS.ANALYTICS.FCT_STOCK_METRICS`
- `USER_DB_PLATYPUS.ANALYTICS.STOCK_PRICES_SNAPSHOT`

---

## ▶️ How to Run the Project

### **Prerequisites**
- Docker & Docker Compose
- Snowflake account access to `USER_DB_PLATYPUS`
- Proper roles and permissions for `RAW` and `ANALYTICS` schemas

### **1. Start Airflow**
```bash
docker-compose up -d
```

### **2. Access Airflow UI**
Navigate to: `http://localhost:8081`

**Login credentials:**
- Username: `airflow`
- Password: `airflow`

### **3. Add Airflow Variables**
In Airflow UI → Admin → Variables:
```
stock_symbols = AAPL,MSFT
lookback_days = 180
```

### **4. Add Snowflake Connection**
In Airflow UI → Admin → Connections → Add Connection:
- **Conn ID:** `snowflake_conn`
- **Conn Type:** `Snowflake`
- **Account:** `sfedu02-lvb17920`
- **User:** `your_username`
- **Password:** `your_password`
- **Warehouse:** `PLATYPUS_QUERY_WH`
- **Database:** `USER_DB_PLATYPUS`
- **Schema:** `RAW`
- **Role:** `your_role` (e.g., `ACCOUNTADMIN` or assigned role)

### **5. Configure dbt Profile**
Update `dbt/stock_analytics/profiles.yml`:
```yaml
stock_analytics:
  target: dev
  outputs:
    dev:
      type: snowflake
      account: sfedu02-lvb17920
      user: your_username
      password: your_password
      role: your_role
      database: USER_DB_PLATYPUS
      warehouse: PLATYPUS_QUERY_WH
      schema: ANALYTICS
      threads: 4
      client_session_keep_alive: False
```

### **6. Trigger ETL DAG**
In Airflow UI → DAGs → `two_stock_dag` → Trigger DAG

This will:
1. Extract stock data from Yahoo Finance
2. Transform and clean the data
3. Load into `USER_DB_PLATYPUS.RAW.TWO_STOCK_V2`

### **7. Trigger dbt DAG**
The dbt DAG runs automatically after ETL completes, or manually trigger:
- `dbt_stock_analytics` DAG

This will:
1. Run dbt models (staging + fact)
2. Execute data quality tests
3. Create snapshot for change tracking

### **8. Validate in Snowflake**
```sql
-- Check raw data
SELECT * FROM USER_DB_PLATYPUS.RAW.TWO_STOCK_V2
LIMIT 100;

-- Check staging data
SELECT * FROM USER_DB_PLATYPUS.ANALYTICS.STG_STOCK_DATA
LIMIT 100;

-- Check fact table with metrics
SELECT * FROM USER_DB_PLATYPUS.ANALYTICS.FCT_STOCK_METRICS
WHERE symbol = 'AAPL'
ORDER BY date DESC
LIMIT 100;

-- Check snapshot (Type 2 SCD)
SELECT * FROM USER_DB_PLATYPUS.ANALYTICS.STOCK_PRICES_SNAPSHOT
WHERE symbol = 'MSFT'
ORDER BY dbt_valid_from DESC
LIMIT 50;
```

### **9. Visualize in Preset**
1. Connect to Snowflake:
   - Account: `sfedu02-lvb17920`
   - Database: `USER_DB_PLATYPUS`
   - Schema: `ANALYTICS`
   - Warehouse: `PLATYPUS_QUERY_WH`
2. Create dataset from `FCT_STOCK_METRICS`
3. Build charts:
   - Line chart: Close price + moving averages over time
   - Bar chart: Monthly percent change
4. Add filters for symbol selection
5. Compile into dashboard

---

## 🐳 Docker Configuration

### **Dockerfile**
```dockerfile
FROM apache/airflow:2.10.1

USER root
RUN apt-get update && \
    apt-get install -y --no-install-recommends \
    gcc g++ python3-dev libpq-dev && \
    apt-get clean && \
    rm -rf /var/lib/apt/lists/*

USER airflow

# Install dependencies with pinned versions
RUN pip install --no-cache-dir \
    apache-airflow-providers-snowflake==5.7.0 \
    snowflake-connector-python==3.12.1 \
    yfinance==0.2.66 \
    dbt-core==1.8.8 \
    dbt-snowflake==1.8.4 \
    sentence-transformers==3.3.1 \
    pinecone-client==5.0.1 \
    alphavantage==0.0.12
```

### **docker-compose.yaml Environment Variables**
```yaml
environment:
  &airflow-common-env
  AIRFLOW__CORE__EXECUTOR: LocalExecutor
  AIRFLOW__DATABASE__SQL_ALCHEMY_CONN: postgresql+psycopg2://airflow:airflow@postgres/airflow
  # Snowflake connection (optional - can also configure via UI)
  AIRFLOW_CONN_SNOWFLAKE_CONN: 'snowflake://username:password@sfedu02-lvb17920/USER_DB_PLATYPUS/RAW?warehouse=PLATYPUS_QUERY_WH&role=your_role'
```

### **Key Points:**
- Uses `platform: linux/amd64` for M1/M2/M3/M4 Mac compatibility
- Pinned package versions prevent dependency resolution issues
- Includes system dependencies for compiling native extensions

---

## 📊 Key SQL Queries

### **Staging Model Query**
```sql
-- models/stg_stock_data.sql
SELECT
    symbol,
    date,
    open,
    high,
    low,
    close,
    volume,
    CURRENT_TIMESTAMP() as loaded_at
FROM {{ source('raw', 'two_stock_v2') }}
WHERE date IS NOT NULL
  AND symbol IS NOT NULL
```

### **Fact Model Query**
```sql
-- models/fct_stock_metrics.sql
SELECT
    symbol,
    date,
    close,
    AVG(close) OVER (
        PARTITION BY symbol 
        ORDER BY date 
        ROWS BETWEEN 6 PRECEDING AND CURRENT ROW
    ) as ma_7day,
    AVG(close) OVER (
        PARTITION BY symbol 
        ORDER BY date 
        ROWS BETWEEN 29 PRECEDING AND CURRENT ROW
    ) as ma_30day,
    (close - LAG(close) OVER (PARTITION BY symbol ORDER BY date)) 
        / LAG(close) OVER (PARTITION BY symbol ORDER BY date) * 100 
        as pct_change
FROM {{ ref('stg_stock_data') }}
```

### **Snapshot Query**
```sql
-- snapshots/stock_prices_snapshot.sql
{% snapshot stock_prices_snapshot %}

{{
    config(
      target_schema='analytics',
      unique_key='symbol || date',
      strategy='check',
      check_cols=['close', 'volume']
    )
}}

SELECT * FROM {{ ref('stg_stock_data') }}

{% endsnapshot %}
```

---

## 📘 Deliverables

✅ **Code:**
- ETL DAG with proper structure
- dbt DAG with run/test/snapshot tasks
- dbt models with transformations
- dbt tests and snapshots

✅ **Documentation:**
- Clear README with setup instructions
- Architecture diagram
- SQL query examples
- Snowflake configuration details

✅ **Screenshots:**
- Airflow DAG runs (successful)
- dbt run/test/snapshot logs
- Snowflake tables with data in `USER_DB_PLATYPUS`
- Preset dashboard with visualizations

✅ **GitHub:**
- Clean repository structure
- Only required files (no `.env`, no credentials)
- Proper `.gitignore`

---

## 🚀 Troubleshooting

### **Issue: dbt command not found**
**Solution:** Rebuild Docker image with pinned versions:
```bash
docker-compose down
docker-compose build --no-cache
docker-compose up -d
```

### **Issue: No active warehouse selected**
**Solution:** Update `dbt/stock_analytics/profiles.yml`:
```yaml
warehouse: PLATYPUS_QUERY_WH  
```

### **Issue: Connection to Snowflake fails**
**Solution:** Verify credentials and test connection:
```bash
docker-compose exec airflow bash
dbt debug --profiles-dir /opt/airflow/dbt/stock_analytics
```

### **Issue: Permission denied on schemas**
**Solution:** Grant proper permissions in Snowflake:
```sql
GRANT USAGE ON DATABASE USER_DB_PLATYPUS TO ROLE your_role;
GRANT USAGE ON SCHEMA USER_DB_PLATYPUS.RAW TO ROLE your_role;
GRANT USAGE ON SCHEMA USER_DB_PLATYPUS.ANALYTICS TO ROLE your_role;
GRANT CREATE TABLE ON SCHEMA USER_DB_PLATYPUS.ANALYTICS TO ROLE your_role;
GRANT SELECT, INSERT ON ALL TABLES IN SCHEMA USER_DB_PLATYPUS.RAW TO ROLE your_role;
```

### **Issue: Connection refused on M1/M2/M3/M4 Mac**
**Solution:** Add to docker-compose.yaml services:
```yaml
platform: linux/amd64
```

### **Issue: Dependency resolution hangs**
**Solution:** Use pinned versions in Dockerfile (as shown above)


---

## 📝 License

This project is for educational purposes as part of DATA 226 coursework.

---

## 👤 Author

**Akash Kumar**  
San Jose State University  
DATA 226 - Fall 2025

**Snowflake Environment:**
- Account: `sfedu02-lvb17920`
- Database: `USER_DB_PLATYPUS`
- Warehouse: `PLATYPUS_QUERY_WH`
