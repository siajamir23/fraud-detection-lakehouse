# fraud-detection-lakehouse

End-to-end fraud detection data engineering pipeline using Databricks, PySpark, Delta Lake and Medallion Architecture.

## Architecture

Synthetic banking data flows through a Medallion Architecture in the `fraud_detection` Unity Catalog:

| Layer | Location | Purpose |
| --- | --- | --- |
| Raw | `/Volumes/fraud_detection/default/raw_data` | Source CSV files |
| Bronze | `fraud_detection.bronze.transactions/customers/merchants` | Raw ingested data (all-string columns + ingestion metadata) |
| Silver | `fraud_detection.silver.transactions/customers/merchants` | Typed, deduplicated, validated data |
| Gold | `fraud_detection.gold.customer_risk_metrics`, `fraud_summary_by_category`, `fraud_summary_by_country` | Aggregated analytics tables |

## Repository structure

| Path | Contents |
| --- | --- |
| `notebooks/00_environment_setup` | Catalog, schema, and volume setup |
| `notebooks/01_generate_synthetic_data` | Generates synthetic transactions, customers, and merchants |
| `notebooks/02_bronze_ingestion` | Loads raw CSV data into bronze Delta tables |
| `notebooks/03_silver_transformation` | Cleans, deduplicates, and types data into silver |
| `notebooks/04_gold_aggregation` | Builds gold analytics tables |
| `notebooks/05_data_quality_testing` | Data quality test suite (bad-record quarantine checks, duplicate detection, silver verification) |
| `jobs/fraud_detection_pipeline.job.json` | Lakeflow Jobs pipeline configuration |
| `docs/dashboard-documentation.md` | AI/BI dashboard documentation |
| `sql/fraud_analytics.dbquery.ipynb` | Saved analytics query |

## Pipeline orchestration

The `fraud_detection_pipeline` Lakeflow Job (ID 90298076188764) chains five notebook tasks on serverless compute, each running only after the previous task succeeds:

generate_synthetic_data → bronze_ingestion → silver_transformation → gold_aggregation → data_quality_tests

* The final `data_quality_tests` task fails the run if silver contains invalid records or duplicate transaction IDs.
* Notebooks use full-refresh (overwrite) semantics: each run regenerates and replaces the data in every layer.
* The job is triggered manually; no schedule is configured yet.

## Dashboard

An AI/BI dashboard (Fraud Overview and Fraud Patterns pages) monitors fraud KPIs, customer risk distribution, merchant-category and country comparisons, daily fraud trends, and transaction patterns, with six synchronized global filters (date, country, merchant category, payment method, channel, age group). See `docs/dashboard-documentation.md`.

## Data quality

`notebooks/05_data_quality_testing` verifies that:

* Duplicate transaction IDs are identified
* Missing customer IDs are quarantined
* Invalid merchant IDs are quarantined
* Negative transaction amounts are quarantined
* Invalid fraud labels are quarantined
* Null transaction timestamps are quarantined

It applies the same checks to the full silver table (100,000 records) and asserts zero failures and zero duplicates, so any regression fails the pipeline run.

---

Last updated: 2026-11-17
