# Signals BI — Auction Pipeline

A data pipeline that ingests UK property auction data from multiple sources into BigQuery, enabling market intelligence for the Signals BI platform.

## Overview

Scrapes from Auction House UK and Savills are stored in Google Cloud Storage and processed through a multi-stage pipeline that deduplicates lots, tracks status changes, and produces clean aggregated tables for the frontend.

## Pipeline

```
GCS (raw CSVs)
    ↓
auctions_raw        — append-only, every row from every scrape
    ↓
auctions_status_log — changelog: tracks ACTIVE → SOLD → WITHDRAWN transitions
    ↓
auctions_clean      — one row per lot, highest-precedence status
    ↓
auction_sector_monthly  — monthly aggregation per postcode sector
auction_sector_summary  — all-time summary per postcode sector
```

## Key features

- **Multi-source normalisation** — unified schema across Auction House and Savills (different column structures, property type labels, ID formats)
- **Status lifecycle tracking** — lots move through ACTIVE → SOLD / UNSOLD / WITHDRAWN; the pipeline detects and logs every change, preventing duplicates
- **Status precedence** — SOLD always wins over ACTIVE regardless of scrape order
- **SDL exclusion** — SDL Auctions filtered at ingest
- **File deduplication** — `ingested_files` log prevents re-processing the same GCS file
- **Dev/prod separation** — `auctions_dev` for testing, `auctions` for production; one-click promotion

## Tech stack

- **Python** (Colab notebook)
- **Google Cloud Storage** — raw file landing zone
- **BigQuery** — all tables in `signalsbi-new` project, EU region
- **Apify** — scraper platform (webhook trigger ready, currently manual)

## Usage

| Scenario | Settings |
|---|---|
| First run | `ENV = 'dev'`, `FIRST_RUN = True` |
| Monthly update | `ENV = 'prod'`, `FIRST_RUN = False` |
| Promote dev → prod | Run Cell 9 with `PROMOTE = True` |

## Data sources

- **Auction House UK** — auctionhouse.co.uk (~15,000 lots)
- **Savills** — auctions.savills.co.uk (~650 lots)

*Contains third-party auction data used for market intelligence purposes.*
