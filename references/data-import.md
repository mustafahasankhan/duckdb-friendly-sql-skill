# DuckDB Data Import Reference

Comprehensive reference for importing data from all supported sources into DuckDB.

---

## CSV Files

### Basic Usage (Auto-Detection)
```sql
-- DuckDB auto-detects delimiter, types, headers
SELECT * FROM 'data.csv';

-- Create table from CSV
CREATE OR REPLACE TABLE my_data AS FROM 'data.csv';

-- Multiple CSV files
SELECT * FROM 'data/part-*.csv';
```

### read_csv with Full Options
```sql
SELECT * FROM read_csv('data.csv',
    auto_detect   = true,          -- default: true
    delim         = ',',           -- delimiter character
    header        = true,          -- first row is header
    quote         = '"',           -- quote character
    escape        = '"',           -- escape character
    sample_size   = 20480,         -- rows to sample (-1 = full file)
    all_varchar   = false,         -- skip type inference, all VARCHAR
    dateformat    = '%Y-%m-%d',    -- strptime format for dates
    timestampformat = '%Y-%m-%d %H:%M:%S',
    columns       = {'col1': 'INTEGER', 'col2': 'DATE'},  -- override types
    auto_type_candidates = ['BIGINT', 'DATE', 'VARCHAR'],  -- candidate types
    null_padding  = false,         -- pad missing columns with NULL
    ignore_errors = false,         -- skip rows with errors
    filename      = false,         -- add source filename column
    hive_partitioning = false,     -- auto-detect hive partition columns
    union_by_name = false,         -- combine files by column name not position
    skip          = 0,             -- rows to skip before header
    max_line_size = 2097152        -- max bytes per line
);
```

### CSV Sniffer
```sql
-- Inspect what DuckDB detects about a CSV
FROM sniff_csv('mystery.csv');
-- Returns: Delimiter, Quote, Escape, HasHeader, Columns (struct), DateFormat,
--          TimestampFormat, and a Prompt column with ready-to-use SQL

-- With custom sample size
FROM sniff_csv('large_file.csv', sample_size = 10000);
```

### COPY Statement (for pre-defined tables)
```sql
CREATE TABLE sales (date DATE, amount DECIMAL(10,2), region VARCHAR);
COPY sales FROM 'sales.csv' (FORMAT csv, HEADER true, DELIMITER ',');

-- Export
COPY sales TO 'sales_export.csv' (FORMAT csv, HEADER true);
COPY sales TO 'sales_export.parquet' (FORMAT parquet);
```

### Handling Faulty CSV Files
```sql
-- Skip rows with errors
SELECT * FROM read_csv('messy.csv', ignore_errors = true);

-- Force all columns to VARCHAR to avoid cast errors
SELECT * FROM read_csv('messy.csv', all_varchar = true);

-- Override problematic column type
SELECT * FROM read_csv('data.csv', types = {'birth_date': 'VARCHAR'});
```

### Reading from stdin
```bash
cat data.csv | duckdb -c "SELECT * FROM read_csv('/dev/stdin')"
```

---

## Parquet Files

### Basic Usage
```sql
-- Direct query (very fast with columnar pushdown)
SELECT * FROM 'data.parquet';

-- Only reads metadata for count (no data scan!)
SELECT count(*) FROM 'data.parquet';

-- Column pruning — only downloads needed columns
SELECT name, email FROM 'large_dataset.parquet';
```

### Multiple Parquet Files
```sql
-- Glob pattern
SELECT * FROM 'warehouse/part-*.parquet';

-- Explicit list
SELECT * FROM read_parquet(['file1.parquet', 'file2.parquet']);

-- Recursive glob
SELECT * FROM 'data/**/*.parquet';

-- Add source filename as column
SELECT * FROM read_parquet('data/*.parquet', filename = true);

-- Hive partitioning
SELECT * FROM read_parquet('data/year=*/month=*/*.parquet',
    hive_partitioning = true);
-- Automatically adds year and month as columns

-- Union by name (different schemas)
SELECT * FROM read_parquet('data/*.parquet', union_by_name = true);
```

### Export to Parquet
```sql
COPY (SELECT * FROM my_table) TO 'output.parquet' (FORMAT parquet);

-- Partitioned export
COPY my_table TO 'output_dir/' (
    FORMAT parquet,
    PARTITION_BY (year, region),
    OVERWRITE_OR_IGNORE true
);
-- Creates: output_dir/year=2024/region=US/data_0.parquet, etc.
```

---

## JSON Files

### Basic Usage
```sql
SELECT * FROM 'data.json';
SELECT * FROM 'data.ndjson';    -- newline-delimited JSON
SELECT * FROM 'data.jsonl';     -- JSON Lines format
```

### read_json with Options
```sql
SELECT * FROM read_json('data.json',
    format        = 'auto',      -- 'auto', 'unstructured', 'newline_delimited', 'array'
    columns       = {'id': 'INTEGER', 'name': 'VARCHAR'},
    sample_size   = 20480,
    maximum_depth = -1,          -- max nesting depth to infer (-1 = unlimited)
    records       = 'auto',      -- 'auto', 'true', 'false'
    timestampformat = '%Y-%m-%dT%H:%M:%S'
);
```

### Automatic JSON → Nested Types Conversion
DuckDB automatically converts JSON to native LIST and STRUCT types:

```sql
-- Access nested JSON arrays and objects directly
SELECT
    data[1].name AS first_name,
    data[1].address.city AS city
FROM 'nested.json';

-- Or from a URL
SELECT starfleet[10].model AS ship
FROM 'https://raw.githubusercontent.com/example/data.json';
```

---

## Excel Files (.xlsx)

### Reading Excel
```sql
-- Simple read (auto-detects headers)
SELECT * FROM 'report.xlsx';

-- With options
SELECT * FROM read_xlsx('report.xlsx',
    header    = true,
    sheet     = 'Sheet2',         -- default: first sheet
    range     = 'A1:D100',        -- spreadsheet cell range
    all_varchar = false,
    ignore_errors = false,
    empty_as_varchar = false,
    stop_at_empty = true
);

-- Via COPY
CREATE TABLE sales (date DATE, amount DOUBLE, region VARCHAR);
COPY sales FROM 'sales.xlsx' WITH (FORMAT xlsx, HEADER);
```

### Writing Excel
```sql
COPY my_table TO 'output.xlsx' WITH (FORMAT xlsx, HEADER true, SHEET 'Results');
```

**Note:** Only `.xlsx` is supported. `.xls` files are not supported.

---

## Amazon S3 / S3-Compatible Storage

### Setup (requires httpfs extension — auto-loaded)
```sql
-- Option 1: Static credentials
CREATE OR REPLACE SECRET s3_secret (
    TYPE s3,
    PROVIDER config,
    KEY_ID     'AKIAIOSFODNN7EXAMPLE',
    SECRET     'wJalrXUtnFEMI/K7MDENG/bPxRfiCYEXAMPLEKEY',
    REGION     'us-east-1'
);

-- Option 2: Auto-detect from environment (IAM role, env vars, ~/.aws/credentials)
CREATE OR REPLACE SECRET s3_auto (
    TYPE s3,
    PROVIDER credential_chain
);

-- Option 3: Specific credential chain
CREATE OR REPLACE SECRET s3_chain (
    TYPE s3,
    PROVIDER credential_chain,
    CHAIN 'env;config'   -- tries env vars, then ~/.aws/credentials
);

-- Option 4: Specific AWS profile
CREATE OR REPLACE SECRET s3_profile (
    TYPE s3,
    PROVIDER credential_chain,
    CHAIN config,
    PROFILE 'my_profile'
);
```

### Reading from S3
```sql
SELECT * FROM 's3://bucket/path/data.parquet';
SELECT * FROM 's3://bucket/path/*.csv';
SELECT * FROM read_parquet('s3://bucket/folder*/100?/t[0-9].parquet');
SELECT * FROM read_parquet('s3://bucket/*.parquet', filename = true);
```

### Writing to S3
```sql
COPY my_table TO 's3://bucket/output.parquet';

-- Partitioned write
COPY my_table TO 's3://bucket/partitioned/' (
    FORMAT parquet,
    PARTITION_BY (year, region),
    OVERWRITE_OR_IGNORE true
);
-- Output: s3://bucket/partitioned/year=2024/region=US/data_0.parquet
```

### S3 with KMS Encryption
```sql
CREATE OR REPLACE SECRET s3_kms (
    TYPE s3,
    PROVIDER credential_chain,
    CHAIN config,
    REGION 'eu-west-1',
    KMS_KEY_ID 'arn:aws:kms:region:account_id:key/key_id',
    SCOPE 's3://secure-bucket'
);
```

---

## Google Cloud Storage (GCS)

```sql
CREATE OR REPLACE SECRET gcs_secret (
    TYPE gcs,
    KEY_ID 'my_key',
    SECRET 'my_secret'
);

-- Use gcs:// or gs:// prefix
SELECT * FROM 'gcs://my-bucket/data.parquet';
SELECT * FROM read_parquet('gs://my-bucket/*.parquet');
```

---

## Cloudflare R2

```sql
CREATE OR REPLACE SECRET r2_secret (
    TYPE r2,
    KEY_ID     'AKIAIOSFODNN7EXAMPLE',
    SECRET     'wJalrXUtnFEMI/K7MDENG/bPxRfiCYEXAMPLEKEY',
    ACCOUNT_ID 'my_cloudflare_account_id'
);

SELECT * FROM read_parquet('r2://my-bucket/data.parquet');
```

---

## Azure Blob Storage and ADLS

### Setup
```sql
INSTALL azure; LOAD azure;  -- auto-loaded on first use

-- Connection string auth
CREATE SECRET azure_secret (
    TYPE azure,
    CONNECTION_STRING 'DefaultEndpointsProtocol=https;AccountName=...'
);

-- Anonymous (public container)
CREATE SECRET azure_anon (
    TYPE azure,
    PROVIDER config,
    ACCOUNT_NAME 'my_storage_account'
);

-- Azure credential chain (CLI, managed identity, env vars)
CREATE SECRET azure_chain (
    TYPE azure,
    PROVIDER credential_chain,
    CHAIN 'cli;managed_identity;env',
    ACCOUNT_NAME 'my_storage_account'
);

-- Service Principal
CREATE SECRET azure_spn (
    TYPE azure,
    PROVIDER service_principal,
    TENANT_ID       'tenant_id',
    CLIENT_ID       'client_id',
    CLIENT_SECRET   'client_secret',
    ACCOUNT_NAME    'my_storage_account'
);
```

### Reading Azure Blob Storage
```sql
SELECT * FROM 'az://my_container/path/data.parquet';
SELECT * FROM 'azure://my_container/path/*.csv';
SELECT * FROM 'az://my_storage_account.blob.core.windows.net/my_container/data.csv';
```

### Reading Azure Data Lake Storage (ADLS)
```sql
-- ADLS is faster for complex glob patterns (understands directory hierarchy)
SELECT * FROM 'abfss://my_filesystem/path/data.parquet';
SELECT * FROM 'abfss://my_storage_account.dfs.core.windows.net/filesystem/*.csv';
```

**Performance tip:** Use `abfss://` (ADLS) over `az://` (Blob) when using complex glob patterns — ADLS is significantly faster because it understands directory structure.

---

## HTTP(S) Files

```sql
-- Query a file over HTTPS directly
SELECT * FROM 'https://example.com/data.parquet';
SELECT * FROM 'https://example.com/data.csv';

-- Parquet supports partial reads over HTTP (only fetches needed columns/rows)
SELECT name, email FROM 'https://example.com/large_dataset.parquet';

-- Count without downloading data (reads metadata only)
SELECT count(*) FROM 'https://example.com/data.parquet';

-- Multiple files
SELECT * FROM read_parquet([
    'https://example.com/file1.parquet',
    'https://example.com/file2.parquet'
]);
```

### HTTP Authentication
```sql
-- Bearer token
CREATE SECRET http_auth (
    TYPE http,
    BEARER_TOKEN 'my_bearer_token'
);

-- Custom headers
CREATE SECRET http_headers (
    TYPE http,
    EXTRA_HTTP_HEADERS MAP {
        'Authorization': 'Bearer my_token',
        'X-Custom-Header': 'value'
    }
);

-- HTTP proxy
CREATE SECRET http_proxy (
    TYPE http,
    HTTP_PROXY          'http://proxy.example.com:3128',
    HTTP_PROXY_USERNAME 'user',
    HTTP_PROXY_PASSWORD 'pass'
);
```

---

## Hive Partitioning

Automatically parse partition columns from directory names:

```sql
-- Directory structure: data/year=2024/month=01/data.parquet
SELECT * FROM read_parquet('data/year=*/month=*/*.parquet',
    hive_partitioning = true);
-- Adds 'year' and 'month' columns automatically from directory names

-- Also works with S3
SELECT * FROM read_parquet('s3://bucket/data/year=*/month=*/*.parquet',
    hive_partitioning = true);
```

---

## Secrets Management

```sql
-- List all secrets
FROM duckdb_secrets();

-- Delete a secret
DROP SECRET IF EXISTS my_secret;

-- Persistent secrets (survive restarts) — stored in ~/.duckdb/
CREATE PERSISTENT SECRET my_secret (...);

-- Temporary secret (default — session only)
CREATE TEMPORARY SECRET my_secret (...);

-- Scope secrets to specific paths
CREATE OR REPLACE SECRET prod_s3 (
    TYPE s3,
    PROVIDER credential_chain,
    SCOPE 's3://production-bucket/'  -- only applies to this path
);
```
