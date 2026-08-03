# european-aviation-analytics
IATA/InformationHub-perspective benchmark of European aviation performance with Medallion architecture on Microsoft Fabric (Bronze/Silver/Gold) with PySpark, Power BI and incremental loading

## Repository Structure

```
european-aviation-analytics/
│
├── README.md                          # Project overview and quick start
├── .gitignore                         # Excludes raw data, credentials, temp files
├── LICENSE                            # MIT
│
├── config/
│   └── pipeline_config.json           # Source URLs, watermarks, Fabric schema names
│
├── data/
│   ├── raw/                           # Downloaded source files (gitignored)
│   └── samples/                       # Small data extracts for testing and CI
│
├── notebooks/
│   ├── 01_bronze_ingest/
│   │   ├── bronze_atfm_delay.ipynb    # EUROCONTROL ATFM delay CSV → Bronze
│   │   ├── bronze_airport_traffic.ipynb # EUROCONTROL airport traffic CSV → Bronze
│   │   ├── bronze_opdi_flights.ipynb  # OPDI flight list Parquet → Bronze
│   │   ├── bronze_our_airports.ipynb  # OurAirports reference CSV → Bronze
│   │   └── bronze_iata_airlines.ipynb # IATA airlines reference CSV → Bronze
│   │   └── bronze_mapping_airlines.ipynb # manually creating mapping of ICAO airlines and airlines names → Bronze
│   │
│   ├── 02_silver_clean/
│   │   ├── silver_atfm_delay.ipynb    # Clean + validate ATFM delay
│   │   ├── silver_airport_traffic.ipynb # Clean + validate airport traffic
│   │   ├── silver_opdi_flights.ipynb  # Clean + validate flight list
│   │   └── silver_dim_airport.ipynb   # SCD2 airport dimension
│   │   └── silver_dim_airline.ipynb   # Carrier_type
│   │
│   ├── 03_gold_model/
│   │   ├── gold_network_performance.ipynb  # fact_network_performance (airport × month)
│   │   ├── gold_airline_punctuality.ipynb  # fact_airline_punctuality (airline × route)
│   │   ├── gold_dim_airport.ipynb          # dim_airport (current snapshot)
│   │   ├── gold_dim_airline.ipynb          # dim_airline with LCC/Network type
│   │   └── gold_dim_date.ipynb             # dim_date
│   │
│   └── 04_exploration/
│       ├── eda_atfm_delay.ipynb       # Exploratory analysis — delays
│       ├── eda_traffic_recovery.ipynb # Exploratory analysis — COVID recovery
│       └── eda_airline_mix.ipynb      # Exploratory analysis — LCC vs network
│
├── src/
│   ├── ingestion/
│   │   ├── eurocontrol_loader.py      # Generic EUROCONTROL CSV/Parquet downloader
│   │   └── reference_loader.py        # OurAirports + IATA airlines loader
│   │
│   ├── transformation/
│   │   ├── atfm_cleaner.py            # ATFM delay cleaning logic
│   │   ├── flight_cleaner.py          # OPDI flight list cleaning logic
│   │   └── scd2_handler.py            # SCD2 merge logic for dim_airport
│   │
│   └── utils/
│       ├── config_loader.py           # Reads pipeline_config.json
│       ├── watermark.py               # Read/write watermark per source
│       └── run_logger.py              # Run log write to Gold layer
│
├── docs/
│   ├── architecture.md                # Source → Bronze → Silver → Gold → Power BI
│   ├── data_sources.md                # All 5 sources with links and grain description
│   ├── incremental_strategy.md        # Watermark logic, batch design, rerun behavior
│   └── runbook.md                     # How to run and refresh the full pipeline
│
├── powerbi/
│   └── european_aviation.pbix         # Main Power BI report (Gold as source)
│
└── scrum/
    └── product_backlog.md             # PBIs, owners, status, Sprint Goal, DoD
```
