# Movies — Medallion Architecture

A Bronze → Silver → Gold data pipeline built in **Databricks** that ingests a raw movies dataset (Parquet) from a Unity Catalog volume, cleans and standardizes it, and produces an analytics-ready star schema optimized for Power BI reporting.

## Architecture Overview

```
Parquet file (UC Volume)
        │
        ▼
┌─────────────────────────────────┐
│  BRONZE (movies_bronze)          │  Raw ingest
│  movies                           │  (as-loaded Parquet data)
└──────────────┬──────────────────┘
               │ deduplicate · trim · fill nulls · type casting
               ▼
┌─────────────────────────────────┐
│  SILVER (movies_silver)          │  Cleaned & standardized
│  movies                          │  (normalized genres delimiter,
│                                  │   typed numeric metrics)
└──────────────┬──────────────────┘
               │ feature engineering · build star schema
               ▼
┌─────────────────────────────────┐
│  GOLD (movies_gold)              │  Analytics-ready star schema
│  fact_movies                     │  (profit & ROI metrics)
│  dim_genre                       │  (unique genres with surrogate key)
│  dim_country                     │  (unique countries with surrogate key)
│  dim_data_source                 │  (dataset origins with surrogate key)
│  dim_time                        │  (year / decade / century)
└─────────────────────────────────┘
```

## Repository Structure

The pipeline is split into one notebook per medallion layer:

```
01_bronze_ingestion.ipynb   # Parquet from landing volume -> raw Delta table (movies_bronze.movies)
02_silver_layer.ipynb      # Bronze -> cleaned & standardized table (movies_silver.movies)
03_gold_layer.ipynb        # Silver -> star schema (movies_gold) for Power BI
```

## Pipeline Stages

### Bronze Layer

* Reads the raw Parquet file from `/Volumes/workspace/default/movies_landing/movies_complete_dataset.parquet`
* Saves it as-is as the `movies_bronze.movies` Delta table (overwrite mode for idempotency)

### Silver Layer

* **movies** — Deduplicated rows, trimmed whitespace in `title`, nulls in `budget` / `revenue` / `vote_count` filled with defaults, numeric metrics type-cast (`double` / `float`), and the `genres` delimiter normalized from `|` to `,`

### Gold Layer

Star schema for analytics and Power BI reporting:

* **fact_movies** — Enriched facts with `profit` (revenue − budget) and `roi` (`CASE WHEN budget > 0 THEN (revenue - budget) / budget ELSE 0 END`)
* **dim_genre** — Distinct genres exploded from the comma-separated `genres` column, with `genre_id` surrogate key
* **dim_country** — Distinct countries with `country_id` surrogate key
* **dim_data_source** — Distinct data sources with `source_id` surrogate key
* **dim_time** — Year / decade / century dimension built from `release_year` and `decade`

## Getting Started

### Prerequisites

* Databricks workspace with Unity Catalog
* The source Parquet file in a UC volume (adjust `file_path` in the bronze notebook if your volume path differs)
* Spark compute (Serverless or interactive cluster)

### Running the Pipeline

1. Clone this repo into a Databricks Git folder (or import the three notebooks into your workspace)
2. Run them in order: **01_bronze_ingestion** → **02_silver_layer** → **03_gold_layer** — each layer depends on the previous layer's tables
3. The pipeline creates three schemas: `movies_bronze`, `movies_silver`, `movies_gold`

## Technologies

* **Databricks** — Compute and Delta Lake storage
* **PySpark** — Distributed data processing
* **Delta Lake** — ACID transactions for all tables
* **Unity Catalog Volumes** — Landing storage for the raw Parquet file
* **Power BI** — Reporting on top of the gold star schema