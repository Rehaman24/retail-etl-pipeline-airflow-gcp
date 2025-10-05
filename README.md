# Retail-ETL-pipeline-airflow-gcp
End-to-end ETL pipeline for retail sales and merchant data using Apache Airflow, Google Cloud Storage, and BigQuery. Automates ingestion, transformation, and upsert logic for analytics-ready data in the GCP cloud.
# Overview
This pipeline demonstrates core data engineering skills:
Orchestrates multi-stage ETL flow using Airflow.
Loads JSON sales and merchant data from GCS to BigQuery staging tables.
Joins and upserts enriched data via BigQuery MERGE.
Implements modular, reusable tasks with robust error handling and schema enforcement.

# Architecture
       +------------------+
       |  GCS Bucket      | <-- Raw JSON files (merchants + sales)
       | bigquery_projects|
       +--------+---------+
                |
                v
       +------------------+
       |  Airflow DAG     | <-- Scheduled daily at midnight UTC
       | (walmart_sales_  |
       |  etl_gcs)        |
       +--------+---------+
                |
                v
       +------------------+
       | Create Dataset   | <-- Creates walmart_dwh dataset
       | & Tables Task    |     and staging/target tables
       +--------+---------+
                |
                v
       +------------------+
       | Load Data Tasks  | <-- Parallel: GCS → BigQuery
       | (Task Group)     |     - merchants.json → merchants_tb
       |                  |     - walmart_sales.json → stage table
       +--------+---------+
                |
                v
       +------------------+
       | Transform & Merge| <-- JOIN stage + merchants
       | Task             |     UPSERT → walmart_sales_tgt (fact)
       +------------------+
                |
                v
       +------------------+
       | BigQuery Tables  | <-- Final dimensional model
       | - merchants_tb   |     (dimension + fact tables)
       | - walmart_sales  |
       |   _tgt (fact)    |
       +------------------+

       
# Task Dependencies
create_dataset 
    ↓
[create_merchants_table, create_walmart_sales_table, create_target_table]
    ↓
load_data (Task Group)
    ├── gcs_to_bq_merchants
    └── gcs_to_bq_walmart_sales
    ↓
merge_walmart_sales




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



