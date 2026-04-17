---
name: data-tools
description: >
  Quick data inspection and CLI exploration skill for Emmanuel at IPAC Caltech.
  Apply whenever the task involves inspecting an unknown file, sniffing schema,
  profiling a Parquet/CSV/JSON/FITS dataset, running ad-hoc SQL against local or
  S3 files, or building a lightweight data CLI before committing to a full pipeline.
  Trigger for: "what's in this file", "schema of", "inspect parquet", "duckdb",
  "quick look", "count rows", "profile data", "explore this dataset", "data CLI",
  "read parquet", "check the schema", "what columns", "head of file", "sample rows".
  Do NOT trigger for full pipeline development (use emmanuel-engineering), billion-row
  catalog cross-match (use vo-explorer), or FITS/ASDF science analysis (use roman-space-telescope).
---

# Data Tools Skill

Fast-path patterns for inspecting, profiling, and querying data files before
committing to a full pipeline. Primary tool: **DuckDB** (zero-setup, in-process).
Secondary: pandas, astropy Table, awswrangler for AWS-specific workflows.

---

## §0 — Inspection Decision Tree

```
What do you have?
│
├─ Parquet / CSV / JSON (local or S3)
│   └─ Schema sniff → DuckDB DESCRIBE / SUMMARIZE
│   └─ Ad-hoc filter/group → DuckDB SQL
│   └─ Need astropy Table → DuckDB → pandas → Table.from_pandas()
│
├─ FITS table
│   └─ astropy.table.Table.read(f, format='fits')
│   └─ or: fitsio for large binary tables (faster than astropy for raw access)
│
├─ ASDF (Roman/JWST)
│   └─ roman_datamodels.open() or asdf.open()
│   └─ See roman-space-telescope skill
│
├─ VOTable (VO query result)
│   └─ pyvo result.to_table() or astropy.io.votable.parse()
│
└─ Unknown format
    └─ file <path>; python -m astropy.io.fits info <path>; duckdb auto-detect
```

---

## §1 — DuckDB: Core Patterns

### Install

```bash
pip install duckdb        # Python API
brew install duckdb       # CLI (macOS)
```

### Schema inspection

```python
import duckdb

con = duckdb.connect()    # in-memory, no setup

# Full schema + stats in one shot
con.sql("SUMMARIZE 'data.parquet'").show()
# Returns: column_name, column_type, min, max, approx_unique, null_percentage, mean, std

# Just column names + types
con.sql("DESCRIBE SELECT * FROM 'data.parquet' LIMIT 0").show()

# Row count (fast: reads metadata only for Parquet)
con.sql("SELECT COUNT(*) FROM 'data.parquet'").fetchone()[0]

# Peek at data
con.sql("SELECT * FROM 'data.parquet' LIMIT 10").show()
```

### Multi-file glob

```python
# All Parquet files in a directory (including subdirs)
con.sql("SELECT * FROM 'output/**/*.parquet' LIMIT 5").show()

# Count rows across all shards
con.sql("SELECT COUNT(*) FROM 'soc-files/*.parquet'").show()
```

### Ad-hoc aggregation

```python
con.sql("""
    SELECT
        status,
        detector,
        COUNT(*)                          AS n_files,
        SUM(file_size_bytes) / 1e6        AS total_mb,
        MIN(received_at)                  AS earliest,
        MAX(received_at)                  AS latest
    FROM 'soc_delivery.parquet'
    GROUP BY status, detector
    ORDER BY n_files DESC
""").show()
```

### Convert to pandas / astropy

```python
# → pandas
df = con.sql("SELECT * FROM 'catalog.parquet' WHERE mag_g < 22").df()

# → astropy Table (via pandas)
from astropy.table import Table
tbl = Table.from_pandas(
    con.sql("SELECT ra, dec, mag_g FROM 'catalog.parquet'").df()
)

# → polars (fast, no pandas dependency)
pl_df = con.sql("SELECT * FROM 'data.parquet'").pl()
```

### Export / transform

```python
# Filtered export to new Parquet
con.sql("""
    COPY (
        SELECT * FROM 'big.parquet' WHERE status = 'failed'
    ) TO 'failed_only.parquet' (FORMAT PARQUET, COMPRESSION SNAPPY)
""")

# Export to CSV
con.sql("""
    COPY (SELECT * FROM 'results.parquet' LIMIT 1000)
    TO 'sample.csv' (HEADER, DELIMITER ',')
""")
```

---

## §2 — S3 Direct Query

Query files on S3 without downloading. Requires the `httpfs` extension.

```python
import duckdb, os

con = duckdb.connect()
con.sql("INSTALL httpfs; LOAD httpfs;")

# Auth via env vars (preferred — picks up ~/.aws or instance role)
con.sql(f"""
    SET s3_region = 'us-east-1';
    SET s3_access_key_id     = '{os.environ["AWS_ACCESS_KEY_ID"]}';
    SET s3_secret_access_key = '{os.environ["AWS_SECRET_ACCESS_KEY"]}';
""")

# Or use credential_chain (instance role / SSO / ~/.aws — no hardcoded keys)
con.sql("SET s3_use_credential_chain = true; SET s3_region = 'us-east-1';")

# Query directly
con.sql("""
    SELECT detector, COUNT(*) AS n
    FROM 's3://my-bucket/soc-delivery/2025-*/**.parquet'
    GROUP BY detector
""").show()

# Schema of S3 file
con.sql("DESCRIBE SELECT * FROM 's3://my-bucket/catalog/part-0000.parquet' LIMIT 0").show()
```

---

## §3 — CLI One-Liners

```bash
# Schema sniff
duckdb -c "DESCRIBE SELECT * FROM 'data.parquet' LIMIT 0;"

# Row count
duckdb -c "SELECT COUNT(*) FROM 'data.parquet';"

# Quick head
duckdb -c "SELECT * FROM 'data.parquet' LIMIT 5;"

# SUMMARIZE (all stats)
duckdb -c "SUMMARIZE 'data.parquet';"

# Group-by from CLI
duckdb -c "SELECT status, COUNT(*) FROM 'soc.parquet' GROUP BY 1 ORDER BY 2 DESC;"

# Export filtered slice
duckdb -c "COPY (SELECT * FROM 'big.parquet' WHERE mag < 20) TO 'bright.csv' (HEADER);"

# Multi-file glob
duckdb -c "SELECT COUNT(*) FROM 'output/*.parquet';"
```

---

## §4 — FITS Quick Inspection

```python
from astropy.io import fits
from astropy.table import Table

# Structure overview (always run first on unknown FITS)
with fits.open("file.fits", memmap=True) as hdul:
    hdul.info()
    # Prints: No., Name, Ver, Type, Cards, Dimensions, Format

# Read binary table HDU
tbl = Table.read("catalog.fits", hdu=1)
print(tbl.colnames)
print(tbl[:5])

# Column stats
import numpy as np
for col in tbl.colnames:
    arr = tbl[col]
    print(f"{col}: dtype={arr.dtype}, min={np.nanmin(arr):.4g}, max={np.nanmax(arr):.4g}")

# Image HDU quick look
with fits.open("image.fits") as hdul:
    data = hdul[0].data
    header = hdul[0].header
    print(f"Shape: {data.shape}, dtype: {data.dtype}")
    print(f"NAXIS1={header['NAXIS1']}, NAXIS2={header['NAXIS2']}")
```

---

## §5 — Parquet Metadata (without DuckDB)

```python
import pyarrow.parquet as pq

# Schema only (reads footer, not data)
schema = pq.read_schema("data.parquet")
print(schema)

# File-level metadata
meta = pq.read_metadata("data.parquet")
print(f"Rows: {meta.num_rows}, Row groups: {meta.num_row_groups}")
print(f"Serialized size: {meta.serialized_size} bytes")

# Column stats per row group (useful for predicate pushdown planning)
for rg in range(meta.num_row_groups):
    for col in range(meta.num_columns):
        col_meta = meta.row_group(rg).column(col)
        print(f"RG{rg} {col_meta.path_in_schema}: "
              f"min={col_meta.statistics.min}, max={col_meta.statistics.max}")

# Partition discovery (Hive-style)
dataset = pq.ParquetDataset("partitioned_dir/", use_legacy_dataset=False)
print(dataset.partitioning.schema)
```

---

## §6 — Quick Data CLI Pattern

When building a small CLI to inspect a dataset (common pattern from ChatGPT sessions):

```python
#!/usr/bin/env python3
"""inspect.py — quick data file inspector"""
import argparse, sys
import duckdb

def main():
    p = argparse.ArgumentParser(description="Inspect a data file")
    p.add_argument("path", help="File path or glob (local or s3://)")
    p.add_argument("--sql", default=None, help="Custom SQL (use 'tbl' as table name)")
    p.add_argument("--head", type=int, default=10, help="Number of rows to show")
    p.add_argument("--summarize", action="store_true", help="Show column stats")
    p.add_argument("--count", action="store_true", help="Row count only")
    args = p.parse_args()

    con = duckdb.connect()
    con.sql("INSTALL httpfs; LOAD httpfs;")
    con.sql("SET s3_use_credential_chain=true; SET s3_region='us-east-1';")
    con.execute(f"CREATE VIEW tbl AS SELECT * FROM '{args.path}'")

    if args.count:
        print(con.sql("SELECT COUNT(*) AS n_rows FROM tbl").fetchone()[0])
    elif args.summarize:
        con.sql("SUMMARIZE tbl").show()
    elif args.sql:
        con.sql(args.sql).show()
    else:
        con.sql(f"SELECT * FROM tbl LIMIT {args.head}").show()

if __name__ == "__main__":
    main()
```

```bash
# Usage
python inspect.py data.parquet
python inspect.py data.parquet --summarize
python inspect.py data.parquet --count
python inspect.py 's3://bucket/prefix/*.parquet' --sql "SELECT status, COUNT(*) FROM tbl GROUP BY 1"
```

---

## §7 — When to Graduate Out of This Skill

| Need | Next tool |
|---|---|
| Billion-row spatial cross-match | `lsdb` + HATS (vo-explorer skill) |
| Full ETL pipeline with orchestration | Airflow DAG (engineering skill) |
| Roman/JWST ASDF science analysis | roman-space-telescope skill |
| VO archive query (TAP/ADQL) | vo-explorer skill |
| Production data lake (ACID writes) | Apache Iceberg + pyiceberg (vo-explorer skill) |

---

## References

- DuckDB docs: https://duckdb.org/docs
- pyarrow Parquet: https://arrow.apache.org/docs/python/parquet.html
- astropy Table I/O: https://docs.astropy.org/en/stable/io/unified.html
- awswrangler (S3 + pandas): https://aws-sdk-pandas.readthedocs.io
