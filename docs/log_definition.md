# Log Definition - What to be logged during ingestion

Fields to fill in for each full download of a file/set of files from data source:
        "batch_uid" - Unique identifier for the download
        "source_name" - Source name like "Eurocontrol" or "OpenAirport - GitHub"
        "file_name" - The resulting files name
        "lakehouse_path" - The path where the file is stored
        "source_url" - The URL where the data is fetched from
        "file_format" - resulting file format
        "bronze_table" - name of the resulting bronze table
        "load_type" - load_type as in what loading method was used; eurocontrol_loader, reference_loader etc.
        "ingestion_started_at_utc" - when the ingestion process starts
        "ingestion_completed_at_utc" - when the ingestion process stops or is complete
        "download_time_seconds" - time it took to download
        "write_time_seconds" - time it took to write to lakehouse
        "size_bytes" - size of the source data
        "status" - succeeded or failed
        "error_message" - if failed the reason should be in here

  The script/method for logging will take these parameters in the above order and create or append to a file with the same name as the resulting file name but as a .log.json or .log.csv
