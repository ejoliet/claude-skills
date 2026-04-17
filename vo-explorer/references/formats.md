# formats.md — Astronomical Format Conversion Recipes

## Format Decision Tree

```
Result type            Recommended format   Library
─────────────────────────────────────────────────────
VO query result        VOTable → Parquet    pyvo + pyarrow
Small catalog (<1M)    astropy Table        astropy.io
Large catalog (>100M)  HATS (Parquet)       hats-import + lsdb
Image / 2D array       FITS or Zarr         astropy / zarr
Spectral cube (IFU)    FITS or Zarr         astropy / zarr
Pipeline product       ASDF                 asdf + asdf-astropy
Versioned table        Iceberg              pyiceberg
Sky coverage           MOC (FITS/JSON)      mocpy
```

## VOTable → Parquet

```python
import pyvo as vo
import pyarrow as pa
import pyarrow.parquet as pq

# From pyvo result
result = tap.run_sync(query)
tbl    = result.to_table()          # astropy Table
df     = tbl.to_pandas()            # pandas DataFrame
arrow  = pa.Table.from_pandas(df)   # PyArrow Table
pq.write_table(arrow, "output.parquet",
               compression="snappy",
               row_group_size=100_000)

# With spatial partitioning hint (for later Dask filtering)
pq.write_to_dataset(arrow, "output_partitioned/",
                    partition_cols=["healpix_k5"])  # pre-computed HEALPix
```

## astropy Table ↔ All Formats

```python
from astropy.table import Table

# Read
tbl = Table.read("data.fits")
tbl = Table.read("data.vot", format="votable")
tbl = Table.read("data.parquet")            # requires pyarrow
tbl = Table.read("data.csv", format="ascii.csv")

# Write
tbl.write("out.fits", overwrite=True)
tbl.write("out.vot",  format="votable")
tbl.write("out.parquet")
tbl.write("out.ecsv", format="ascii.ecsv")  # lossless text with metadata

# To/from pandas
df  = tbl.to_pandas()
tbl = Table.from_pandas(df)
```

## FITS Binary Table → HATS

```python
from hats_import.pipeline import pipeline_with_client
from hats_import.catalog.arguments import ImportArguments
from dask.distributed import Client

# Step 1: Convert FITS table to Parquet (if not already)
import astropy.table as at
t = at.Table.read("catalog.fits")
t.write("/tmp/catalog.parquet")

# Step 2: Import to HATS
args = ImportArguments(
    input_path="/tmp/catalog.parquet",
    output_path="s3://bucket/hats/my_catalog/",
    ra_column="ra",
    dec_column="dec",
    catalog_name="my_catalog",
    file_reader="parquet",
    pixel_threshold=1_000_000,   # max rows per partition
)
with Client() as client:
    pipeline_with_client(args, client)
```

## Parquet → HATS (direct, large catalogs)

```python
args = ImportArguments(
    input_path="s3://bucket/raw/*.parquet",
    output_path="s3://bucket/hats/gaia_dr3_subset/",
    ra_column="ra", dec_column="dec",
    catalog_name="gaia_dr3_subset",
    file_reader="parquet",
    dask_tmp="s3://bucket/tmp/",   # for shuffling on S3
    pixel_threshold=2_000_000,
    highest_healpix_order=7,
)
```

## Zarr Chunking Strategy

Chunk layout governs access performance. Tune to your dominant access pattern.

| Access pattern | Recommended chunk layout | Rationale |
|---|---|---|
| Spatial cutouts (RA/Dec) | `(1, 256, 256)` | Read one spectral slice over a spatial patch |
| Spectral extraction (spectrum at a point) | `(full_spec, 1, 1)` | Entire spectrum contiguous on disk |
| Temporal analysis | `(T, small_spatial, small_spatial)` | Time axis contiguous |
| General / mixed | `(100, 512, 512)` for 3D | Balanced |

```python
import zarr
from numcodecs import Blosc

# Write with explicit chunking and compression
z = zarr.open_array(
    "cube.zarr", mode="w",
    shape=(2000, 4096, 4096),      # (spectral, dec, ra)
    chunks=(200, 512, 512),
    dtype="float32",
    compressor=Blosc(cname="zstd", clevel=5, shuffle=Blosc.BITSHUFFLE),
    fill_value=float("nan"),
)

# Rechunk existing Zarr store (use rechunker or dask)
import dask.array as da
arr = da.from_zarr("input.zarr")
arr_rechunked = arr.rechunk({0: 200, 1: 512, 2: 512})
arr_rechunked.to_zarr("output_rechunked.zarr", overwrite=True)
```

## ASDF (Roman / JWST pipeline products)

```python
import asdf

# Read Roman WFI Level 2
with asdf.open("roman_wfi_l2.asdf") as af:
    data   = af["roman"]["data"]           # numpy array
    wcs    = af["roman"]["meta"]["wcs"]    # gwcs WCS object
    header = af["roman"]["meta"]

# Read JWST pipeline product
with asdf.open("jw01234_cal.asdf") as af:
    sci = af["data"]
    err = af["err"]

# Write ASDF
tree = {
    "data": numpy_array,
    "meta": {"instrument": "WFI", "filter": "F106"},
}
af = asdf.AsdfFile(tree)
af.write_to("output.asdf")
```

## MOC Serialization

```python
from mocpy import MOC

# Save
moc.save("coverage.fits", format="fits")    # FITS (standard, most compact)
moc.save("coverage.json", format="json")    # JSON (human-readable)
moc.save("coverage.ascii", format="ascii")  # ASCII string

# Load
moc = MOC.load("coverage.fits", format="fits")
moc = MOC.from_str("2/4-5 3/16-19 ...")    # inline ASCII

# Serialize to string (for embedding in ADQL or VOTable)
moc_str = moc.serialize(format="ascii")

# S-MOC area
import astropy.units as u
import numpy as np
sky_frac = moc.sky_fraction
area_deg2 = sky_frac * 4 * np.pi * (180/np.pi)**2

# STMOC (space-time)
from mocpy import TimeMOC, STMOC
tmoc = TimeMOC.from_time_ranges(t_start, t_end, delta_t=Quantity(1, "day"))
stmoc = STMOC.from_spatial_coverages(tmoc, [moc1, moc2])
```

## HEALPix Order ↔ Resolution Reference

| Order | Pixel size | Total pixels | Typical use |
|---|---|---|---|
| 0 | 58.6° | 12 | All-sky coarse |
| 5 | 1.83° | 12288 | Survey footprints |
| 7 | 27.5' | 196608 | Field-level |
| 10 | 3.44' | 12.6M | Catalog partitioning |
| 12 | 51.5" | 201M | Dense catalog MOCs |
| 15 | 6.4" | 12.9B | High-precision |

```python
import astropy_healpix as ah
import astropy.units as u

# Order ↔ nside
nside = ah.level_to_nside(order=10)   # → 1024
order = ah.nside_to_level(nside=1024) # → 10

# Pixel resolution
res = ah.nside_to_pixel_resolution(nside=1024)  # → Quantity in arcmin

# Sky coordinates → HEALPix index
idx = ah.lonlat_to_healpix(lon*u.deg, lat*u.deg, nside=1024, order="nested")
```
