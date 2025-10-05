# Retail-ETL-pipeline-airflow-gcp
End-to-end ETL pipeline for retail sales and merchant data using Apache Airflow, Google Cloud Storage, and BigQuery. Automates ingestion, transformation, and upsert logic for analytics-ready data in the GCP cloud.
# Overview
This pipeline demonstrates core data engineering skills:
Orchestrates multi-stage ETL flow using Airflow.
Loads JSON sales and merchant data from GCS to BigQuery staging tables.
Joins and upserts enriched data via BigQuery MERGE.
Implements modular, reusable tasks with robust error handling and schema enforcement.
# Architecture
GCS Bucket (bigquery_projects)
├── walmart_ingestion/merchants/merchants.json
└── walmart_ingestion/sales/walmart_sales.json
                    ↓
            Airflow DAG (Daily)
                    ↓
    ┌───────────────────────────────┐
    │  Create Dataset & Tables      │
    │  - walmart_dwh (dataset)      │
    │  - merchants_tb (dimension)   │
    │  - walmart_sales_stage        │
    │  - walmart_sales_tgt (fact)   │
    └───────────────────────────────┘
                    ↓
    ┌───────────────────────────────┐
    │  Load Data from GCS           │
    │  - merchants.json → merchants_tb     │
    │  - walmart_sales.json → stage table  │
    └───────────────────────────────┘
                    ↓
    ┌───────────────────────────────┐
    │  MERGE Operation              │
    │  JOIN stage + merchants       │
    │  UPSERT into walmart_sales_tgt│
    └───────────────────────────────┘
# Data Model
Dimensional Model (Star Schema)
Dimension Table: merchants_tb
- merchant_id (STRING, PRIMARY KEY)
- merchant_name (STRING)
- merchant_category (STRING)
- merchant_country (STRING)
- last_update (TIMESTAMP)

# Staging Table: walmart_sales_stage
- sale_id (STRING, PRIMARY KEY)
- sale_date (DATE)
- product_id (STRING)
- quantity_sold (INT64)
- total_sale_amount (FLOAT64)
- merchant_id (STRING, FOREIGN KEY)
- last_update (TIMESTAMP)

# Fact Table: walmart_sales_tgt (Enriched with merchant details)
- sale_id (STRING, PRIMARY KEY)
- sale_date (DATE)
- product_id (STRING)
- quantity_sold (INT64)
- total_sale_amount (FLOAT64)
- merchant_id (STRING, FOREIGN KEY)
- merchant_name (STRING)
- merchant_category (STRING)
- merchant_country (STRING)
- last_update (TIMESTAMP)

**Design Rationale**:
- **Star Schema**: Optimized for analytical queries with simple joins
- **Denormalization**: Merchant attributes in fact table reduce join overhead
- **Date Dimension**: sale_date enables time-series analysis
- **Staging Layer**: Decouples ingestion from transformation

---

## Pipeline Components

### 1. Dataset Creation
**Operator**: `BigQueryCreateEmptyDatasetOperator`
- **Task ID**: `create_dataset`
- **Purpose**: Creates the `walmart_dwh` dataset if it doesn't exist
- **Location**: US region
- **Idempotent**: Safe to run multiple times

### 2. Table Creation (Runtime)
**Operator**: `BigQueryCreateTableOperator`
- **Tasks**: 
  - `create_merchants_table` - Dimension table for merchant reference data
  - `create_walmart_sales_table` - Staging table for daily sales data
  - `create_target_table` - Fact table for final enriched sales data
- **Schema Definition**: Defined dynamically in DAG code

### 3. Data Loading from GCS
**Operator**: `GCSToBigQueryOperator`
- **Task Group**: `load_data`
  - `gcs_to_bq_merchants`: Loads merchant data
  - `gcs_to_bq_walmart_sales`: Loads sales data
- **Source Format**: Newline-delimited JSON (NDJSON)
- **Write Mode**: `WRITE_TRUNCATE`

### 4. MERGE Operation (Upsert)
**Operator**: `BigQueryInsertJobOperator`
- **Task ID**: `merge_walmart_sales`
- **Logic**: Joins staging with dimension, performs UPSERT

## DAG Configuration
DAG ID: walmart_sales_etl_gcs
Schedule: @daily (runs once per day at midnight UTC)
Start Date: days_ago(1)
Catchup: False (only processes current data, no backfilling)
Retries: 1 (automatically retries failed tasks once)
Max Active Runs: 1 (prevents concurrent executions)

## Task Dependencies
create_dataset
    ↓
[create_merchants_table, create_walmart_sales_table, create_target_table]
    ↓
load_data (Task Group)
    ├── gcs_to_bq_merchants
    └── gcs_to_bq_walmart_sales
    ↓
merge_walmart_sales



