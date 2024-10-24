# dbt_bitcoin

## Project Overview
**dbt_bitcoin** is a dbt project designed to analyze Bitcoin trading data. It transforms raw data into meaningful insights, providing key metrics related to trading performance.

### Features
- Hourly and daily trading analysis
- Profit and Loss (PnL) calculations
- Flexible configuration for entry days
- Comprehensive data validation tests

## Getting Started

### Prerequisites
- **Homebrew**: Ensure you have Homebrew installed on macOS for package management.
- **Anaconda**: This project uses Anaconda for managing Python environments.
- **PostgreSQL**: This project is configured to work with PostgreSQL.

### Installation Steps

#### Step 1: Install Homebrew
Homebrew is a package manager for macOS, which simplifies installing PostgreSQL and other utilities.
```bash
/bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/HEAD/install.sh)"
```
#### Step 2: Install PostgreSQL
Using Homebrew, install PostgreSQL:
```bash
brew install postgresql
brew services start postgresql
```
Check that PostgreSQL is running:
```bash
psql --version
```
#### Step 3: Install Anaconda
Download and install Anaconda from their official website.
#### Step 4: Create a New Conda Environment
```bash
conda create -n my_dbt_env python=3.9
conda activate my_dbt_env
```
#### Step 5: Install dbt and PostgreSQL Adapter
```bash
pip install dbt-core dbt-postgres psycopg2
```
Verify the installation:
```bash
dbt --version
```
#### Step 6: Install Jupyter
```bash
conda install jupyter
```
#### Step 7: Initialize the dbt Project
Create a new dbt project named dbt_bitcoin:
```bash
dbt init dbt_bitcoin
```

### Configuration
Set up your profiles.yml file:

Navigate to your dbt profiles directory (usually located in ~/.dbt/).
Create or edit the profiles.yml file to include your database connection details:

```yaml
dbt_bitcoin:
  outputs:
    dev:
      type: postgres
      host: localhost
      user: postgres
      password: # Your PostgreSQL password
      port: 5432
      dbname: bitcoin_data
      schema: analytics
  target: dev
```
### Database Setup
Connect to PostgreSQL
Start the PostgreSQL service:
```bash
brew services start postgresql
```

Connect to PostgreSQL:
```bash
psql -U postgres -h localhost
```
***Create the Database and Schema***

Run the following commands in the PostgreSQL shell:
```bash
CREATE DATABASE bitcoin_data;
\q
```
Connect to the bitcoin_data database:
```bash
psql -U postgres -h localhost -d bitcoin_data
```
Create the analytics schema:
```bash
CREATE SCHEMA IF NOT EXISTS analytics;
```

### Load Data
Create the Main Table
Create the btc_usdt_1s table:
```bash
CREATE TABLE IF NOT EXISTS analytics.btc_usdt_1s (
    "Open Time" TIMESTAMPTZ PRIMARY KEY,
    "Open" NUMERIC,
    "High" NUMERIC,
    "Low" NUMERIC,
    "Close" NUMERIC,
    "Volume" NUMERIC,
    "Close Time" TIMESTAMPTZ,
    "Quote Asset Volume" NUMERIC,
    "Number of Trades" INTEGER,
    "Taker Buy Base Asset Volume" NUMERIC,
    "Taker Buy Quote Asset Volume" NUMERIC,
    "Ignore" TEXT
);
```
Handle Duplicates on "Open Time" Column
To handle duplicates, use a temporary table:
```sql
CREATE TEMP TABLE temp_btc_usdt_1s (
    "Open Time" TIMESTAMPTZ,
    "Open" NUMERIC,
    "High" NUMERIC,
    "Low" NUMERIC,
    "Close" NUMERIC,
    "Volume" NUMERIC,
    "Close Time" TIMESTAMPTZ,
    "Quote Asset Volume" NUMERIC,
    "Number of Trades" INTEGER,
    "Taker Buy Base Asset Volume" NUMERIC,
    "Taker Buy Quote Asset Volume" NUMERIC,
    "Ignore" TEXT
);
```
Load the data into the temporary table:
```bash
\copy temp_btc_usdt_1s FROM 'half2_BTCUSDT_1s.csv' DELIMITER ',' CSV HEADER;
```
Insert data into the main table, handling duplicates:
```bash
INSERT INTO analytics.btc_usdt_1s (
    "Open Time", "Open", "High", "Low", "Close", "Volume",
    "Close Time", "Quote Asset Volume", "Number of Trades",
    "Taker Buy Base Asset Volume", "Taker Buy Quote Asset Volume", "Ignore"
)
SELECT * FROM temp_btc_usdt_1s
ON CONFLICT ("Open Time") DO NOTHING;
```

### Model Lineage Overview

This section describes the lineage and dependencies between the models in the `dbt_bitcoin` project.

## 1. `stg_btc` (Staging Model)
- **Path**: `dbt_bitcoin/models/trading/stg_btc.sql`
- **Description**: Loads and prepares the raw Bitcoin OHLC data from the `btc_usdt_1s` table in the `analytics` schema.
- **Downstream Dependencies**:
  - `daily_trading`

---

## 2. `daily_trading` (Hourly Trading Model)
- **Path**: `dbt_bitcoin/models/trading/daily_trading.sql`
- **Description**: Aggregates trades executed every hour. Tracks hourly buy and sell prices and calculates the total trades per hour.
- **Upstream Dependencies**:
  - `stg_btc`
- **Downstream Dependencies**:
  - `returns_losses`

---

## 3. `returns_losses` (Returns and Losses Model)
- **Path**: `dbt_bitcoin/models/trading/returns_losses.sql`
- **Description**: Calculates total returns and identifies maximum losses for each hour.
- **Upstream Dependencies**:
  - `daily_trading`
- **Downstream Dependencies**:
  - `max_return`
  - `min_maxLoss`

---

## 4. `max_return` (Max Returns Model)
- **Path**: `dbt_bitcoin/models/trading/max_return.sql`
- **Description**: Identifies the hour of the day with the highest returns.
- **Upstream Dependencies**:
  - `returns_losses`

---

## 5. `min_maxLoss` (Min-Max Loss Model)
- **Path**: `dbt_bitcoin/models/trading/min_maxLoss.sql`
- **Description**: Identifies the hour with the lowest maximum loss.
- **Upstream Dependencies**:
  - `returns_losses`


### Running dbt
Finally, run dbt with your specified variables:
```bash
dbt build --vars '{"entry_days": ['Monday', 'Tuesday', 'Wednesday', 'Thursday', 'Friday', 'Saturday', 'Sunday'],"start_date": '2024-01-01', "end_date": '2024-08-01'}'
--target dev
```
