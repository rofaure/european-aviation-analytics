# Architecture — European Aviation Analytics

> **Project:** IATA Benchmark Report on European Aviation  
> **Stack:** Microsoft Fabric · PySpark · Power BI  
> **Team:** 4 data engineers · Dry run Aug 12 · Final presentation Aug 15, 2026

---

## Table of Contents

1. [Overview](#1-overview)
2. [Data Sources](#2-data-sources)
3. [Medallion Layers](#3-medallion-layers)
   - [Bronze — Raw Ingestion](#31-bronze--raw-ingestion)
   - [Silver — Cleaned & Typed](#32-silver--cleaned--typed)
   - [Gold — Analytics-Ready](#33-gold--analytics-ready)
4. [Gold Data Model](#4-gold-data-model)
5. [Business Questions Mapping](#5-business-questions-mapping)
6. [Architecture Decision Records (ADRs)](#6-architecture-decision-records-adrs)
7. [Glossary](#7-glossary)

---

## 1. Overview

This project implements a **Medallion architecture** (Bronze → Silver → Gold) on **Microsoft Fabric** to produce a benchmarking report on European aviation performance for the years 2019–2024.

```
EUROCONTROL APIs          Microsoft Fabric Lakehouse (EAA_LakeHouse)
OurAirports CSV    ──►  Bronze (raw)  ──►  Silver (cleaned)  ──►  Gold Warehouse (EAA_Gold)  ──►  Power BI
IATA airline ref                                                    (EAA_Gold schema)
```

Key design choices at a glance:

- All transformations are written in **PySpark** inside Fabric notebooks.
- Bronze and Silver tables are stored as **Delta Lake** tables inside `EAA_LakeHouse` under the `bronze` and `silver` schemas.
- Gold tables are stored in **EAA_Gold**, a Fabric Warehouse, so Power BI can connect via a SQL endpoint.
- Incremental loading is controlled by watermarks stored in `Files/config/pipeline_config.json`.

---

## 2. Data Sources

| ID | Name | Provider | Format | Grain | Coverage | Load type |
|----|------|----------|--------|-------|----------|-----------|
| S1 | ATFM Delay | EUROCONTROL Performance Review Unit | CSV.BZ2 per year | Airport × day | 2019–2024 | Incremental (year) |
| S2 | Airport Traffic | EUROCONTROL Performance Review Unit | CSV per year | Airport × day | 2019–2024 | Incremental (year) |
| S3 | OPDI Flight List | EUROCONTROL OPDI v002 | Parquet per month | 1 flight | 2022–2024 | Incremental (month) |
| S4 | Airport Reference | OurAirports | CSV | 1 airport | Snapshot | One-time |
| S5 | Airline Reference | IATA (benct/iata-utils) | CSV | 1 airline | Snapshot | One-time |
| S6 | LCC/Network Lookup | Internal config | CSV | 1 airline | Static | One-time |

**URL patterns** are stored in `Files/config/pipeline_config.json`. Watermarks (`atfm_delay_year`, `airport_traffic_year`, `opdi_month`) are updated after each successful batch so re-runs are safe and idempotent.

---

## 3. Medallion Layers

### 3.1 Bronze — Raw Ingestion

**Location:** `EAA_LakeHouse` · schema `bronze`  
**Principle:** land data exactly as received, with three audit columns added by the loader.

| Bronze table | Source | Notes |
|---|---|---|
| `bronze.apt_dly` | S1 — ATFM Delay | All columns remain strings; BZ2 decompressed before Spark read |
| `bronze.airport_traffic` | S2 — Airport Traffic | All columns remain strings |
| `bronze.opdi_flight_list` | S3 — OPDI Flight List | Parquet; no type coercion |
| `bronze.airport_raw` | S4 — OurAirports | Full global airport list |
| `bronze.airline_raw` | S5 — IATA Airline Ref | Full global airline list |
| `bronze.lcc_network_lookup` | S6 — LCC Lookup | Maps ICAO airline code → carrier_type |

Every row carries:

```
source_name  STRING   -- which pipeline produced this row
batch_id     STRING   -- UUID shared by all rows in one run
loaded_at    STRING   -- UTC ISO timestamp of ingestion
```

**Notebooks:** `bronze_atfm_delay`, `bronze_airport_traffic`, `bronze_opdi_flight_list`, `bronze_airport_ref`, `bronze_airline_ref` (shared utilities: `config_loader`, `eurocontrol_loader`).

---

### 3.2 Silver — Cleaned & Typed

**Location:** `EAA_LakeHouse` · schema `silver`  
**Principle:** cast types, rename columns to snake_case, deduplicate, filter to European scope.

| Silver table | Built from | Key transformations |
|---|---|---|
| `silver.atfm_delay` | `bronze.apt_dly` | Cast all columns (INT/DATE), rename (APT_ICAO → airport_icao), deduplicate on (airport_icao, flight_date) |
| `silver.airport_traffic` | `bronze.airport_traffic` | Cast, rename, deduplicate on (airport_icao, flight_date) |
| `silver.opdi_flights` | `bronze.opdi_flight_list` | Cast, parse DATE, join `silver.airline` on icao_airline to add `lcc_flag` and `carrier_type` |
| `silver.airport` | `bronze.airport_raw` | Filter to European airports, apply **SCD Type 2** |
| `silver.airline` | `bronze.airline_raw` + `bronze.lcc_network_lookup` | Merge on icao_airline → add `carrier_type` (LCC / Network / Regional) |

---

### 3.3 Gold — Analytics-Ready

**Location:** `EAA_Gold` Warehouse · SQL endpoint consumed by Power BI  
**Principle:** all fact tables at **month grain**, all dimensions denormalized and stable.

See [Section 4](#4-gold-data-model) for the full schema.

---

## 4. Gold Data Model

### Fact Tables

#### `gold.fact_airport_performance_month`

> **Grain:** 1 row = 1 airport × 1 calendar month  
> **Coverage:** 2019–2024  
> **Built from:** `silver.atfm_delay` (primary) + `silver.airport_traffic` (primary)

| Column | Type | Source |
|---|---|---|
| airport_icao | STRING | join key → dim_airport |
| year_month | STRING (YYYYMM) | join key → dim_date |
| atfm_delay_min | DOUBLE | SUM(DLY_APT_ARR_1) |
| delayed_arr | BIGINT | SUM(FLT_ARR_1_DLY) |
| delay_cause_weather | DOUBLE | SUM(DLY_APT_ARR_W_1) |
| delay_cause_capacity | DOUBLE | SUM(DLY_APT_ARR_C_1) |
| delay_cause_staffing | DOUBLE | SUM(DLY_APT_ARR_S_1) |
| *(other cause columns)* | DOUBLE | SUM(DLY_APT_ARR_*_1) |
| arr_flights | BIGINT | SUM(FLT_ARR_1) from airport_traffic |
| dep_flights | BIGINT | SUM(FLT_DEP_1) |
| total_flights | BIGINT | SUM(FLT_TOT_1) |

**Answers:** Q1 (ATFM delay trend), Q2 (traffic ranking), Q5 (delay ↔ traffic correlation)

---

#### `gold.fact_route_month`

> **Grain:** 1 row = 1 ADEP × 1 ADES × 1 calendar month  
> **Coverage:** 2022–2024 (OPDI availability)  
> **Built from:** `silver.opdi_flights` (aggregated) + `silver.atfm_delay` (two lookups)

| Column | Type | Source |
|---|---|---|
| adep_icao | STRING | join key → dim_airport (active relationship) |
| ades_icao | STRING | join key → dim_airport (inactive, USERELATIONSHIP) |
| year_month | STRING (YYYYMM) | join key → dim_date |
| flight_count | BIGINT | COUNT(*) from opdi_flights |
| lcc_flights | BIGINT | SUM(lcc_flag) |
| network_flights | BIGINT | SUM(network_flag) |
| lcc_share_pct | DOUBLE | lcc_flights / flight_count |
| avg_atfm_delay_adep | DOUBLE | LEFT JOIN atfm_delay ON ADEP = airport_icao |
| avg_atfm_delay_ades | DOUBLE | LEFT JOIN atfm_delay ON ADES = airport_icao |

**Answers:** Q4 (most delayed routes)

---

#### `gold.fact_airline_month`

> **Grain:** 1 row = 1 airline × 1 calendar month  
> **Coverage:** 2022–2024 (OPDI availability)  
> **Built from:** `silver.opdi_flights` (aggregated) + `silver.atfm_delay` (lookup on ADEP per flight)

| Column | Type | Source |
|---|---|---|
| icao_airline | STRING | join key → dim_airline |
| year_month | STRING (YYYYMM) | join key → dim_date |
| flight_count | BIGINT | COUNT(*) |
| carrier_type | STRING | from dim_airline (LCC / Network / Regional) |
| avg_atfm_delay_experienced | DOUBLE | weighted avg of atfm_delay at each flight's ADEP |

**Answers:** Q3 (LCC / Network / Regional split)

---

### Dimension Tables

#### `gold.dim_airport`

Sourced from `silver.airport` (SCD2). European airports only.  
Columns: `airport_icao`, `airport_iata`, `airport_name`, `country`, `latitude`, `longitude`, `is_current`.  
**Role-playing:** used twice in `fact_route_month` — active on `adep_icao`, inactive on `ades_icao`.

#### `gold.dim_airline`

Sourced from `silver.airline`.  
Columns: `icao_airline`, `iata_airline`, `airline_name`, `country`, `carrier_type`.  
Connected to `fact_airline_month` only (see [ADR-04](#adr-04-dim_airline-connects-only-to-fact_airline_month)).

#### `gold.dim_date`

Generated table (no external source). Columns: `year_month`, `year`, `month`, `quarter`, `month_name`, `semester`.  
Shared foreign key across all three fact tables.

---

### Entity-Relationship Summary

```
dim_date ──────────────────────────────────────────────────────────
  │ year_month                                                       │
  ├──► fact_airport_performance_month (airport_icao → dim_airport)   │
  ├──► fact_route_month (adep_icao / ades_icao → dim_airport ×2)    │
  └──► fact_airline_month (icao_airline → dim_airline)               │
```

---

## 5. Business Questions Mapping

| # | Business Question | Primary Table | Key Measures | Dimensions Needed |
|---|---|---|---|---|
| Q1 | What is the ATFM delay trend per airport over time? | fact_airport_performance_month | atfm_delay_min, delayed_arr | dim_airport, dim_date |
| Q2 | Which airports handle the most traffic? | fact_airport_performance_month | arr_flights, dep_flights, total_flights | dim_airport, dim_date |
| Q3 | What is the LCC / Network / Regional split by airline? | fact_airline_month | flight_count, carrier_type, lcc_share_pct | dim_airline, dim_date |
| Q4 | What are the most delayed routes? | fact_route_month | avg_atfm_delay_adep, avg_atfm_delay_ades, lcc_share_pct | dim_airport (×2), dim_date |
| Q5 | Is there a correlation between traffic volume and ATFM delay? | fact_airport_performance_month | atfm_delay_min vs. total_flights | dim_airport, dim_date |

> **Coverage note:** Q1, Q2, Q5 cover 2019–2024. Q3 and Q4 cover 2022–2024 only (OPDI availability constraint). This must be communicated clearly in Power BI tooltips and report captions.

---

## 6. Architecture Decision Records (ADRs)

### ADR-01: Month grain for all fact tables

**Context:** EUROCONTROL sources (S1, S2) are available at day grain. OPDI (S3) is a flight-level dataset.  
**Decision:** Aggregate all three sources to **month grain** in the Gold layer.  
**Rationale:** OPDI produces very sparse route-level data at day grain (many ADEP/ADES pairs operate only a few times per week). Mixing day-grain and month-grain tables would require bridge tables in Power BI and would produce misleading comparisons. Month grain produces dense, comparable rows across all fact tables and allows a shared `dim_date` key.  
**Trade-off:** Day-level drill-down is lost in Gold; it remains available in Silver for ad-hoc queries.

---

### ADR-02: Merge ATFM delay and airport traffic into one fact table

**Context:** ATFM delay (S1) and airport traffic (S2) share the same grain (airport × day → month) and the same natural key (`airport_icao + year_month`).  
**Decision:** Combine them into `fact_airport_performance_month` with both measure groups in a single table.  
**Rationale:** Keeping them in two separate fact tables would force every Q5 visual (correlation analysis) to perform a DAX CROSSJOIN or use a virtual bridge table — both fragile and slow. A single fact table with both measure groups is the correct Kimball pattern when grain and key are identical.  
**Trade-off:** The fact table has more columns, but all are additive aggregates with clear naming conventions (`atfm_*`, `arr_*`, `dep_*`).

---

### ADR-03: OPDI kept in separate fact tables (route and airline)

**Context:** OPDI (S3) covers only 2022–2024. EUROCONTROL sources cover 2019–2024.  
**Decision:** Do **not** add LCC metrics from OPDI into `fact_airport_performance_month`. Instead, maintain two separate OPDI-derived facts: `fact_route_month` and `fact_airline_month`.  
**Rationale:** If OPDI columns were added to `fact_airport_performance_month`, every 2019–2021 row would carry NULL for all LCC metrics. Power BI measures that don't explicitly handle NULLs would silently produce wrong totals. Separate fact tables with explicit coverage noted in the grain description make the data boundary visible and intentional.  
**Trade-off:** The analyst must be aware that Q3/Q4 and Q1/Q2 cannot be directly compared across the 2019–2021 period. This is documented in the Power BI report tooltips.

---

### ADR-04: `dim_airline` connects only to `fact_airline_month`

**Context:** `fact_route_month` has a route grain (ADEP × ADES × month). Multiple airlines operate the same route in the same month.  
**Decision:** `dim_airline` is **not** connected to `fact_route_month`.  
**Rationale:** A relationship between `dim_airline` and `fact_route_month` would be many-to-many — each route row corresponds to flights from multiple airlines. Power BI resolves many-to-many relationships with bidirectional cross-filtering, which causes measure double-counting and unpredictable filter propagation. The LCC/Network breakdown on routes is embedded directly in `fact_route_month` as pre-aggregated columns (`lcc_flights`, `network_flights`, `lcc_share_pct`).  
**Trade-off:** You cannot filter `fact_route_month` by a specific airline in Power BI via `dim_airline`. That analysis belongs in `fact_airline_month`.

---

### ADR-05: `dim_airport` role-playing in `fact_route_month`

**Context:** `fact_route_month` has two airport foreign keys: `adep_icao` (departure) and `ades_icao` (arrival), both referencing `gold.dim_airport`.  
**Decision:** Use one **active** relationship (`adep_icao → dim_airport.airport_icao`) and one **inactive** relationship (`ades_icao → dim_airport.airport_icao`). Inactive relationship is activated in DAX measures with `USERELATIONSHIP()`.  
**Rationale:** Power BI does not allow two simultaneous active relationships from the same fact table to the same dimension — it would create ambiguous filter propagation. The role-playing pattern is the standard Kimball approach: a single shared dimension table serves both roles, keeping the model compact and consistent.  
**Implementation note:** Create two DAX measures sets — one for ADEP perspective (e.g., "Avg ATFM Delay at Departure") and one for ADES perspective (e.g., "Avg ATFM Delay at Arrival") — each using `USERELATIONSHIP()` on the appropriate key.

---

### ADR-07: `dim_` prefix reserved for the Gold layer only

**Context:** Reference tables exist at every layer (Bronze: raw CSVs; Silver: cleaned and typed; Gold: denormalized star-schema dimensions).  
**Decision:** Tables named with the `dim_` prefix appear **only in Gold**. Bronze and Silver reference tables use plain descriptive names (`airport_raw`, `airline_raw`, `silver.airport`, `silver.airline`).  
**Rationale:** The `dim_` prefix carries a specific Kimball meaning: a denormalized, analytics-ready dimension suitable for direct use in a star schema. Bronze tables are raw snapshots with no type guarantees; Silver tables are cleaned but not yet modelled for BI. Applying `dim_` to them would misrepresent their readiness and create confusion when browsing the Lakehouse catalog. The `dim_` prefix serves as an implicit quality signal: only Gold tables carry it.  
**Trade-off:** The lineage (`airport_raw` → `silver.airport` → `gold.dim_airport`) requires knowing the convention. This document and notebook naming make it explicit.

---

### ADR-06: Gold stored in Fabric Warehouse (EAA_Gold), not Lakehouse

**Context:** Both Lakehouse (Delta tables) and Warehouse (SQL engine) are available in Microsoft Fabric.  
**Decision:** Gold tables are written into **EAA_Gold Warehouse** via its SQL endpoint.  
**Rationale:** Power BI's DirectQuery and Import modes perform best against a SQL Warehouse endpoint, which supports columnar storage and query optimisation natively. The Lakehouse SQL endpoint is read-only and less optimised for BI workloads. Writing Gold from PySpark notebooks is still possible via the Warehouse connector.  
**Trade-off:** Gold tables cannot be queried with PySpark `spark.table("gold.xxx")` directly; they require JDBC or the Warehouse SQL endpoint. All Gold notebooks use the Warehouse connector for writes.

---

## 7. Glossary

| Term | Definition |
|---|---|
| **ATFM** | Air Traffic Flow Management. A system operated by EUROCONTROL's Network Manager that issues pre-departure slot delays (ground delays) to aircraft when destination or en-route airspace capacity is insufficient. |
| **ADEP** | Aerodrome of Departure. The ICAO 4-letter code of the departure airport. |
| **ADES** | Aerodrome of Destination. The ICAO 4-letter code of the arrival airport. |
| **LCC** | Low-Cost Carrier. Airlines operating with a no-frills, point-to-point model (e.g., Ryanair, easyJet). |
| **Network carrier** | Full-service airlines operating hub-and-spoke networks (e.g., Air France, Lufthansa). |
| **OPDI** | Open Performance Data Initiative. EUROCONTROL's open dataset of individual flight records. |
| **IFR** | Instrument Flight Rules. All commercial flights operate under IFR; IFR movement counts are used as the standard traffic metric. |
| **SCD Type 2** | Slowly Changing Dimension Type 2. A historisation strategy that keeps previous versions of a dimension row by adding `valid_from`, `valid_to`, and `is_current` columns instead of overwriting. |
| **Medallion architecture** | A layered data architecture pattern (Bronze / Silver / Gold) that progressively refines raw data into analytics-ready tables. |
| **Role-playing dimension** | A single dimension table used multiple times in the same fact table under different aliases and relationships (e.g., `dim_airport` as both ADEP and ADES). |
| **Grain** | The precise definition of what one row in a fact table represents (e.g., "one airport, one calendar month"). |
| **Watermark** | A persisted value (e.g., last loaded year or month) used to resume incremental ingestion from where the previous run stopped. |
| **USERELATIONSHIP()** | A DAX function that activates an inactive relationship in a Power BI measure, used here to switch `dim_airport` between ADEP and ADES roles. |
