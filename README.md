# european-aviation-analytics
IATA/InformationHub-perspective benchmark of European aviation performance with Medallion architecture on Microsoft Fabric (Bronze/Silver/Gold) with PySpark, Power BI and incremental loading

#structure
│
├── README.md
├── .gitignore
├── LICENSE
│
├── data/
│   ├── raw/                 
│   └── samples/  
│
├── notebooks/
│   ├── 01_bronze_ingest/
│   ├── 02_silver_clean/
│   ├── 03_gold_model/
│   └── 04_exploration/
│
├── src/
│   ├── ingestion/
│   ├── transformation/
│   └── utils/
│
├── config/
│   └── pipeline_config.json  # URLs sources, watermarks, parameters
│
├── docs/
│   ├── architecture.md
│   ├── data_sources.md
│   ├── incremental_strategy.md
│   └── runbook.md
│
├── powerbi/
│   └── european_aviation.pbix
│
└── scrum/
    └── product_backlog.md
