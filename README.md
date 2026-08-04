# European Aviation Analytics

IATA Benchmark Report on European Aviation — built on Microsoft Fabric with PySpark and Power BI.

> **Team:** 4 Data Engineers · **Dry run:** Aug 12, 2026 · **Final presentation:** Aug 15, 2026

Full architecture documentation: [`docs/architecture.md`](docs/architecture.md)

---

## Overview

This project implements a **Medallion architecture** (Bronze → Silver → Gold) to analyse European aviation performance from 2019 to 2024, answering five business questions on airport delays, traffic, route efficiency, and airline-type distribution.

---

## Business Questions

| # | Question | Gold table | Coverage |
|---|----------|------------|----------|
| Q1 | ATFM delay trend per airport over time | fact_airport_performance_month | 2019–2024 |
| Q2 | Airports with the most traffic | fact_airport_performance_month | 2019–2024 |
| Q3 | LCC / Network / Regional split by airline | fact_airline_month | 2022–2024 |
| Q4 | Most delayed routes (ADEP + ADES perspective) | fact_route_month | 2022–2024 |
| Q5 | Correlation between traffic volume and delay | fact_airport_performance_month | 2019–2024 |

---

## Repository Structure

```
european-aviation-analytics/
│
├── README.md
├── docs/
│   └── architecture.md              # Full architecture, data model, ADRs, glossary
│
├── config/
│   ├── pipeline_config.json         # Source URLs, watermarks, table names
│   └── lcc_network_lookup.csv       # ICAO airline → LCC / Network / Regional
│
└── notebooks/
    │
    ├── utils/                        # Shared helpers (%run in every pipeline notebook)
    │   ├── config_loader.ipynb       # load_config(), get_source(), update_last_loaded()
    │   └── eurocontrol_loader.ipynb  # ingest(source, url, format, table, batch_id)
    │
    ├── bronze/                       # Raw ingestion — lands data as-is, all columns as strings
    │   ├── bronze_atfm_delay.ipynb          # → bronze.apt_dly          (incremental, year)
    │   ├── bronze_airport_traffic.ipynb     # → bronze.airport_traffic  (incremental, year)
    │   ├── bronze_opdi_flight_list.ipynb    # → bronze.opdi_flight_list (incremental, month)
    │   ├── bronze_airport_ref.ipynb         # → bronze.airport_raw      (one-time)
    │   └── bronze_airline_ref.ipynb         # → bronze.airline_raw      (one-time)
    │
    ├── silver/                       # Cleaning & typing — casts, renames, deduplicates
    │   ├── silver_atfm_delay.ipynb          # bronze.apt_dly          → silver.atfm_delay
    │   ├── silver_airport_traffic.ipynb     # bronze.airport_traffic  → silver.airport_traffic
    │   ├── silver_opdi_flights.ipynb        # bronze.opdi_flight_list → silver.opdi_flights
    │   ├── silver_airport.ipynb             # bronze.airport_raw      → silver.airport  (SCD2)
    │   └── silver_airline.ipynb             # bronze.airline_raw + lcc_lookup → silver.airline
    │
    └── gold/                         # Star schema in EAA_Gold Warehouse
        ├── gold_fact_airport_performance_month.ipynb   # Q1, Q2, Q5
        ├── gold_fact_route_month.ipynb                 # Q4
        ├── gold_fact_airline_month.ipynb               # Q3
        ├── gold_dim_airport.ipynb                      # silver.airport → gold.dim_airport
        ├── gold_dim_airline.ipynb                      # silver.airline → gold.dim_airline
        └── gold_dim_date.ipynb                         # generated calendar table
```

---

## Naming Convention

| Layer | Rule | Example |
|-------|------|---------|
| Bronze | No prefix — raw table name | `bronze.airport_raw`, `bronze.apt_dly` |
| Silver | No prefix — descriptive name | `silver.airport`, `silver.atfm_delay` |
| Gold facts | `fact_` prefix | `gold.fact_airport_performance_month` |
| Gold dimensions | `dim_` prefix | `gold.dim_airport`, `gold.dim_date` |

The `dim_` prefix is **reserved for Gold only** — it signals a Kimball-ready denormalized dimension. Bronze and Silver reference tables do not carry it (see [ADR-07](docs/architecture.md#adr-07-dim_-prefix-reserved-for-the-gold-layer-only)).

---

## Data Sources

| ID | Name | Provider | Format | Grain | Load type |
|----|------|----------|--------|-------|-----------|
| S1 | ATFM Delay | EUROCONTROL PRU | CSV.BZ2 / year | Airport × day | Incremental (year) |
| S2 | Airport Traffic | EUROCONTROL PRU | CSV / year | Airport × day | Incremental (year) |
| S3 | OPDI Flight List | EUROCONTROL OPDI v002 | Parquet / month | 1 flight | Incremental (month) |
| S4 | Airport Reference | OurAirports | CSV | 1 airport | One-time |
| S5 | Airline Reference | IATA (benct/iata-utils) | CSV | 1 airline | One-time |
| S6 | LCC/Network Lookup | `config/lcc_network_lookup.csv` | CSV | 1 airline | Static |

Source URLs and watermarks are managed in `config/pipeline_config.json`.

---

## Incremental Strategy

- **S1 & S2 (EUROCONTROL):** watermark on `year` — initial load 2019, then yearly batches up to 2024.
- **S3 (OPDI):** watermark on `YYYYMM` — initial load January 2022, then monthly batches up to December 2024.
- **S4, S5, S6 (references):** one-time load; reload manually only if the source changes.

Re-runs are safe: the loader uses `append` mode with `batch_id` and `loaded_at` audit columns on every row.

---

## Stack

| Component | Technology |
|-----------|------------|
| Platform | Microsoft Fabric |
| Bronze & Silver storage | EAA_LakeHouse (Delta Lake, schemas `bronze` / `silver`) |
| Gold storage | EAA_Gold Warehouse (SQL endpoint for Power BI) |
| Transformations | PySpark (Fabric notebooks) |
| Reporting | Power BI (DirectQuery / Import on EAA_Gold) |
| Ingestion utilities | Python — `requests`, `bz2`, `json`, `uuid` |
| Version control | Git |
