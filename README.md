# Signals BI — HMLR Price Paid Ingest Pipeline

## Overview
This notebook downloads HM Land Registry Price Paid Data and loads it into BigQuery as a raw transactions table (`transactions_raw`). Aggregation queries are built and maintained separately in BigQuery.

## How it works
Data is sourced from HMLR's public endpoint:
```
https://price-paid-data.publicdata.landregistry.gov.uk
```

The notebook downloads yearly CSV files (1995 → present) on first run, then switches to the monthly update file for subsequent runs. All data lands in `transactions_raw` via a staging → MERGE pattern that handles HMLR's Add / Change / Delete record statuses correctly.

## Setup
Run in **Google Colab** — not BigQuery Notebooks (GCP datacenter IPs are blocked by HMLR).

Add this cell at the very top before running:
```python
from google.colab import auth
auth.authenticate_user()
```

## Settings (Cell 1)
The only cell you need to change between runs:

```python
ENV       = 'dev'   # 'dev' or 'prod'
FIRST_RUN = True    # True = full backfill | False = monthly update only
```

| Scenario | Settings |
|---|---|
| First ever run | `ENV = 'dev'`, `FIRST_RUN = True` |
| Monthly update | `ENV = 'prod'`, `FIRST_RUN = False` |

## Environments
| Environment | BigQuery Dataset |
|---|---|
| dev | `signalsbi-new.uk_sold_prices_dev` |
| prod | `signalsbi-new.uk_sold_prices` |

## Table created
### `transactions_raw`
One row per property sale. Source of truth for all downstream aggregations.

| Column | Type | Description |
|---|---|---|
| `transaction_id` | STRING | HMLR unique transaction reference |
| `transaction_date` | DATE | Date of sale |
| `price` | INT64 | Sale price in GBP |
| `postcode` | STRING | Full postcode |
| `postcode_sector` | STRING | Derived e.g. `SW1A 1` |
| `postcode_district` | STRING | Derived e.g. `SW1A` |
| `property_type` | STRING | Detached / Semi-Detached / Terraced / Flat / Other |
| `tenure` | STRING | Freehold / Leasehold |
| `new_build` | BOOL | Whether property is a new build |
| `street` | STRING | Street name |
| `town` | STRING | Town or city |
| `local_authority` | STRING | Local authority district |
| `county` | STRING | County |
| `year_month` | STRING | Derived e.g. `2024-03` |
| `year` | INT64 | Derived from transaction date |
| `record_status` | STRING | A = Add, C = Change, D = Delete |
| `ingested_at` | TIMESTAMP | When row was loaded |

Partitioned by `transaction_date`, clustered by `postcode_sector` and `property_type`.

## Schedule
Run on the **25th of each month** with `FIRST_RUN = False`. HMLR publishes on the 20th working day of each month — the 25th is a safe proxy.

## Methodology notes
- Sales counts include all registered transactions regardless of price
- Price-based filtering (e.g. `price > 10,000`) is applied in downstream BigQuery aggregation queries, not at ingest
- Duplicate `transaction_id` values within a single CSV are deduplicated at ingest (`keep='last'`)
- HMLR amendments (C records) and deletions (D records) are handled via the MERGE statement

## Data source attribution
Contains HM Land Registry data © Crown copyright and database right 2021. Licensed under the Open Government Licence v3.0.
