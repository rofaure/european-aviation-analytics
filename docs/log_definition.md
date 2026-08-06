# Log Definition - What to be logged during ingestion

The script/method for logging will take these parameters in the above order and create or append to a file with the same name as the resulting file name but as a .log.json or .log.csv

Do a "%run log" in a separate cell before anything else in your notebook
 
Fields to be passed to WriteLog:
        source_name : Some identifying string of the source like "OurAirports"
        file_name : The file name, like airports.csv
        source_url : the url used to download the file
        lakehouse_path : where the file will end up (probably /lakehouse/default/Files/data/raw/... for bronze)
        bronze_table_name : the name of the resulting table name (   .saveAsTable(f"bronze.{table_name}")   )
        load_type : instead of incremental or full I've gone for reference_loader or eurocontrol_loader, name defined in the loader with the call to write_log(...) 
        ingestion_started_at_utc : Start of the ingestion script
        ingestion_completed_at_utc : Ingestion completed
        download_time_seconds : time it took to download (don't log every part, log the whole dl process for the resulting csv)
        write_time_seconds : time it took to write to .csv and table
        size_bytes : bytes downloaded (measured on resulting text-content using len(file_content.encode("utf-8")))
        status : success if all went OK, else fail.
        error_message : cumulative error string for the process, but atleast give the any resulting error from the download
        log_format: either "json" or "csv", default is json and makes the logs visible in the logviewer script.
        log_folder=where to put the logs - i've set it to /Files/data/log ( "/lakehouse/default/Files/data/log" )

Example from reference_loader log call

log_path = write_log(
        source_name,
        file_name,
        url,
        f"{folder_path}/{file_name}",
        table_name,
        "reference_loader",
        ingestion_start_time,
        ingestion_stop_time,
        download_time,
        write_time,
        size_bytes,        
        dl_success, # TODO: needs to also account for write errors
        dl_error, # TODO: needs to be cumulative error for dl and wr
        "json",
        "/lakehouse/default/Files/data/log"
