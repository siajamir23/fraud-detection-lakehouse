# Fraud Detection & Risk Analytics Dashboard — Documentation

Generated 2026-11-17. Documents the AI/BI dashboard built on the `fraud_detection` lakehouse.

## 1. Overview

The dashboard monitors portfolio-wide fraud KPIs, customer risk distribution, merchant-category and country comparisons, daily fraud trends, and transaction-level fraud patterns. It has two canvas pages plus a global-filters sidebar that applies the same selections on every page.

## 2. Data sources (Unity Catalog)

| Table | Grain | Used for |
| --- | --- | --- |
| `fraud_detection.silver.transactions` | one row per transaction | KPIs, daily trends, patterns |
| `fraud_detection.silver.merchants` | one row per merchant | merchant-category attribute (join) |
| `fraud_detection.gold.customer_risk_metrics` | one row per customer | risk categories, age groups, top customers |
| `fraud_detection.gold.fraud_summary_by_category` | one row per category | category bar chart |
| `fraud_detection.gold.fraud_summary_by_country` | one row per country | country bar chart |

## 3. Pages and widgets

### Fraud Overview
* KPI counters: total transactions, fraudulent transactions, fraud rate (%), total transaction amount, fraudulent transaction amount, fraud amount share (%)
* Customer risk distribution pie (HIGH / MEDIUM / LOW customer counts)
* Fraud rate by merchant category (bar)
* Fraudulent transactions by country (bar)
* Daily fraud trends (line, fraud rate by day)

### Fraud Patterns
* Fraud rate by payment method (bar)
* Fraud rate by channel (bar)
* Fraud rate by customer age group (bar)
* Top 10 highest-risk customers (table)

## 4. Datasets

| Dataset | Kind | Source |
| --- | --- | --- |
| Overall Fraud KPIs | SQL | silver.transactions joined to silver.merchants |
| Daily Fraud Trends | SQL | silver.transactions joined to silver.merchants |
| Customer Risk Distribution | SQL | gold.customer_risk_metrics |
| Transaction Patterns | local metric view | silver.transactions joined to silver.merchants |
| Customer Risk Profiles | local metric view | gold.customer_risk_metrics |
| Top High-Risk Customers | local metric view | gold.customer_risk_metrics (top 10 by fraudulent transactions) |
| Fraud by Merchant Category | SQL | gold.fraud_summary_by_category |
| Fraud by Country | SQL | gold.fraud_summary_by_country |
| Transaction Attributes | SQL | distinct payment methods and channels (filter option lists) |
| Customer Age Groups | SQL | distinct age-group buckets (filter option list) |

## 5. Global filters (sidebar)

Six synchronized sidebar filters, all applied BEFORE aggregation:

1. **Transaction Date** (date-range picker, default last 5 years to today) — drives Overall Fraud KPIs, Daily Fraud Trends, Transaction Patterns
2. **Transaction Country** (multi-select) — drives Overall Fraud KPIs, Customer Risk Distribution, Daily Fraud Trends, Transaction Patterns, Customer Risk Profiles, Top High-Risk Customers
3. **Merchant Category** (multi-select) — drives Overall Fraud KPIs, Daily Fraud Trends, Transaction Patterns
4. **Payment Method** (multi-select) — drives Overall Fraud KPIs, Daily Fraud Trends, Transaction Patterns
5. **Channel** (multi-select) — drives Overall Fraud KPIs, Daily Fraud Trends, Transaction Patterns
6. **Age Group** (multi-select) — drives Customer Risk Distribution, Customer Risk Profiles, Top High-Risk Customers

The country and category bar charts on Fraud Overview filter through field bindings on their own gold datasets. Filters are grain-scoped: date, payment method, and channel apply only to transaction-grain data; age group applies only to customer-grain data, because those attributes do not exist on the other grain.

## 6. Parameterization pattern

Every filter drives a per-dataset parameter so predicates are applied before aggregation:

* SQL datasets declare multi parameters (`countries`, `categories`, `payment_methods`, `channels`, `age_groups`) guarded as `COALESCE(SIZE(:param), 0) = 0 OR array_contains(:param, column)`, and a `date_range` range parameter guarded with `:date_range.min IS NULL OR ...` / `:date_range.max IS NULL OR ...` predicates. Default date range: `now-5y/y` to `now/d`.
* Metric-view datasets declare `ARRAY<STRING>` parameters with `param IS NULL OR array_contains(param, column)` guards; the date filter binds as a `STRUCT<min DATE, max DATE>` parameter with NULL guards on `date_range`, `date_range.min`, and `date_range.max`.
* Empty defaults are required: without a default selection an "All" state arrives unbound and the query fails with UNBOUND_SQL_PARAMETER.

## 7. Business rules and conventions

* Fraud rate = `SUM(fraud_label) * 100.0 / COUNT(*)` at the filtered grain.
* Age groups derive from `age` via CASE buckets: 18-24, 25-34, 35-44, 45-54, 55-64, 65+.
* Parameterized datasets are not materialization-eligible; the gold summary datasets are materialized on publish.
* Risk categories (HIGH / MEDIUM / LOW) are assigned by the gold aggregation layer, not by the dashboard.