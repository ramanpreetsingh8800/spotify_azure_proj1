# Spotify Azure Data Pipeline — Project 1

An **Azure Data Factory** pipeline that incrementally ingests Spotify dimensional data from Azure SQL Database into an ADLS Gen2 data lake (Bronze layer), using a timestamp-based CDC watermark pattern.

---

## Architecture Overview

```
Azure SQL Database
  (DimUser, DimArtist, DimTrack, DimDate, FactStream)
        │
        ▼
Azure Data Factory
  ├─ LoopIncrementalIngestion  ← orchestrator (ForEach over all tables)
  └─ IncrementalIngestion       ← single-table CDC pipeline
        │
        ▼
ADLS Gen2 — Bronze Container
  ├─ <table>/          ← Parquet files (timestamped)
  └─ <table>_cdc/      ← cdc.json (watermark state)
```

---

## How It Works

Each pipeline run follows this sequence for every table:

1. **LastCDC** — Reads `cdc.json` from the Bronze layer to retrieve the last successfully processed timestamp.
2. **Current** — Sets a `utcNow()` variable used to timestamp the output file.
3. **AzureSQLtoLake** — Copies rows from Azure SQL where `updated_at > last_cdc_value` into a Parquet file in `bronze/<table>/`.
4. **IfIncrementalData** — Checks whether any data was actually copied:
   - **No data** → deletes the empty Parquet file (avoids lake clutter).
   - **Data present** → runs `MaxCDC` to find the new high-water mark, then overwrites `cdc.json` via `UpdateLastCDC`.

The `LoopIncrementalIngestion` pipeline wraps this logic in a **ForEach** over all five tables and includes an **Alerts** activity that posts pipeline failure details to an Azure Logic App webhook.

---

## Tables Ingested

| Table | CDC Column | Schema |
|---|---|---|
| DimUser | updated_at | dbo |
| DimArtist | updated_at | dbo |
| DimTrack | updated_at | dbo |
| DimDate | updated_at | dbo |
| FactStream | updated_at | dbo |

---

## Repo Structure

```
├── dataset/
│   ├── AzureSql.json           # Source dataset (Azure SQL, schema-flexible)
│   ├── JsonDynamic.json        # Dynamic JSON dataset (ADLS Gen2, for CDC files)
│   └── ParquetDynamic.json     # Dynamic Parquet dataset (ADLS Gen2, Snappy)
│
├── linkedService/
│   ├── AzureSqlProj1.json      # Linked service → Azure SQL DB
│   └── DataLake.json           # Linked service → ADLS Gen2
│
├── pipeline/
│   ├── IncrementalIngestion.json       # Single-table CDC pipeline
│   └── LoopIncrementalIngestion.json   # Multi-table orchestrator
│
├── factory/
│   └── SpotifyProj1-DataFactory.json   # ADF factory metadata
│
└── publish_config.json                 # ADF Git integration config
```

---

## Key Design Decisions

- **Dynamic datasets** — `JsonDynamic` and `ParquetDynamic` accept `container`, `folder`, and `file` as parameters, making both pipelines fully table-agnostic.
- **Watermark-based CDC** — Uses a `cdc.json` file per table to store the last processed `updated_at` value, avoiding full reloads on every run.
- **Empty file cleanup** — If a CDC run returns zero rows, the resulting empty Parquet file is deleted immediately, keeping the Bronze layer clean.
- **Snappy compression** — All Parquet files are written with Snappy codec for a good balance of speed and compression ratio.
- **Failure alerting** — `LoopIncrementalIngestion` fires a Logic App HTTP webhook on pipeline failure, passing `pipeline_name` and `pipeline_runID` for triage.

---

## Prerequisites

- Azure Data Factory instance
- Azure SQL Database with the five Spotify tables and an `updated_at` column on each
- ADLS Gen2 storage account with a `bronze` container
- Each table must have a corresponding `bronze/<table>_cdc/cdc.json` initialised before the first run (set `cdc` to an epoch date, e.g. `1900-01-01`)
- An `empty.json` file at `bronze/<table>_cdc/empty.json` (used as a read source when overwriting the CDC watermark)

---

## Running the Pipeline

**Single table (IncrementalIngestion):**

Trigger manually with these parameters:

| Parameter | Example Value | Description |
|---|---|---|
| schema | dbo | SQL schema |
| table | DimTrack | Table name |
| cdc_col | updated_at | Timestamp column for CDC |
| from_date | *(leave blank)* | Override start date (optional) |

**All tables (LoopIncrementalIngestion):**

Trigger with no parameters — the `input_loop` default value covers all five tables. Override `input_loop` if you need to ingest a subset.

## Potential Improvements

- Enable **parallel ForEach** (`isSequential: false`) to ingest all tables concurrently once volume justifies it.
- Add **retry policies** (currently `retry: 0`) on Copy activities for transient network failures.
- Move the Logic App webhook URL to **Azure Key Vault** to avoid secrets in source control.
- Add a **Silver layer** transformation pipeline downstream of this Bronze ingestion.
- Consider **trigger scheduling** (e.g. tumbling window trigger) for automated daily runs.

---

## Tech Stack

- Azure Data Factory
- Azure SQL Database
- Azure Data Lake Storage Gen2
- Azure Logic Apps (alerting)
- Parquet (Snappy) · JSON
