---
name: lsdb
description: >
  Scalable astronomical catalog analysis using HATS (Hierarchically-partitioned
  Astronomical Time Series) format and LSDB. Apply for billion-row catalog operations:
  spatial filtering, cross-matching, time-series lightcurve access for survey data
  (LSST/Rubin, ZTF, Gaia, 2MASS). Lazy Dask execution — no full data load into memory.
  Trigger for: lsdb, hats, HATS catalog, catalog cross-match at scale, Rubin/LSST catalog
  analysis, ZTF lightcurves at scale, HiPS-partitioned catalog, dask spatial query,
  billion-row catalog, hats-cloudtests, astronomy-commons.
---

# LSDB Skill

LSDB (Large Survey Database) is a Python library for scalable analysis of
HATS-partitioned astronomical catalogs. Built on Dask + pandas. Handles
billion-row spatial queries, cross-matches, and lightcurve access without
loading data into memory.

> **MANDATORY FIRST STEP** — call Context7 MCP to fetch current lsdb/hats docs before writing code.
> APIs evolve fast. Library name: `lsdb` / `hats`.

---

## When to Use LSDB vs Other Tools

| Scale / need | Tool | Reason |
|---|---|---|
| < 10M rows, local file | DuckDB / pandas | Simpler, no Dask overhead |
| > 100M rows, spatial ops | **LSDB** | Dask + HATS partitioning |
| VO protocol query | pyvo TAP | Protocol-level; returns small result sets |
| Single-file Parquet inspection | DuckDB | No HATS format needed |
| Rubin/LSST DR release | **LSDB** | Native HATS format |
| ZTF / Gaia full-sky at scale | **LSDB** | Public HATS catalogs available |

---

## §0 — Install

```bash
pip install lsdb hats

# With Dask distributed (cluster use):
pip install "lsdb[distributed]"
```

---

## §1 — Load a HATS Catalog

```python
import lsdb

# From local HATS directory (lazy — no data read yet)
cat = lsdb.read_hats("path/to/hats_catalog/")

# From S3 (anonymous public bucket)
cat = lsdb.read_hats(
    "s3://stpubdata/hats/catalogs/gaia_dr3/",
    storage_options={"anon": True},
)

# With AWS credentials
cat = lsdb.read_hats(
    "s3://my-bucket/catalogs/my-catalog/",
    storage_options={"anon": False},  # uses ~/.aws or env vars
)

# Inspect schema (no data loaded)
print(cat.dtypes)
print(cat.hc_structure)   # HEALPix partition map
print(len(cat))           # total row count (from metadata)
```

---

## §2 — Spatial Filtering

```python
from astropy.coordinates import SkyCoord
import astropy.units as u

# Cone search (lazy)
cone = cat.cone_search(
    ra=150.0, dec=2.2,
    radius_arcsec=3600.0,   # 1 degree
)

# From SkyCoord
coord = SkyCoord.from_name("M31")
cone = cat.cone_search(
    ra=coord.ra.deg, dec=coord.dec.deg,
    radius_arcsec=1800.0,
)

# Box search
box = cat.box_search(ra=(149.5, 150.5), dec=(2.0, 2.5))

# MOC filter (precise footprint)
from mocpy import MOC
moc = MOC.from_fits("survey_footprint.fits")
filtered = cat.filter_by_moc(moc)

# Execute (triggers Dask compute → pandas DataFrame)
result_df = cone.compute()
```

---

## §3 — Cross-Matching

```python
import lsdb

cat1 = lsdb.read_hats("catalog_a/")
cat2 = lsdb.read_hats("catalog_b/")

# Nearest-neighbor cross-match
matched = cat1.crossmatch(
    cat2,
    radius_arcsec=1.0,
    suffixes=("_a", "_b"),
    algorithm=lsdb.algorithms.KdTreeCrossmatch,
)
result = matched.compute()   # pandas DataFrame with merged columns

# Left join (keep all cat1 rows; NaN where no match)
left = cat1.crossmatch(cat2, radius_arcsec=1.0, how="left").compute()

# Check match distance column
print(result["_dist_arcsec"].describe())
```

---

## §4 — Column Selection and Filtering

```python
# Select columns + filter before compute (reduces I/O)
result = (
    cat
    .cone_search(ra=150.0, dec=2.2, radius_arcsec=7200.0)
    .query("mag_g < 22 and mag_r > 18")
    [["object_id", "ra", "dec", "mag_g", "mag_r"]]
    .compute()
)

# All operations are lazy until .compute()
```

---

## §5 — Time-Series / Lightcurves (Nested Catalogs)

```python
# Load an object catalog + source (detection) catalog
object_cat = lsdb.read_hats("objects/")
source_cat = lsdb.read_hats("sources/")   # has object_id + mjd + flux

# Join sources into a nested column on the object catalog
nested = object_cat.join_nested(
    source_cat,
    left_on="object_id",
    right_on="object_id",
    nested_column_name="lightcurve",
)

# Compute lightcurves for a cone
lcs = nested.cone_search(ra=150.0, dec=2.2, radius_arcsec=3600.0).compute()

# Access a single lightcurve
first_lc = lcs["lightcurve"].iloc[0]   # pandas DataFrame: mjd, flux, flux_err
```

---

## §6 — Dask Distributed (Cluster)

```python
from dask.distributed import Client
import lsdb

# Local cluster (all cores)
client = Client()

# Remote cluster (EKS, Slurm)
client = Client("tcp://scheduler:8786")

# All lsdb calls automatically use the active Dask client
cat = lsdb.read_hats("s3://bucket/rubin-lsst/objects/",
                      storage_options={"anon": False})
result = (
    cat
    .cone_search(ra=150.0, dec=2.2, radius_arcsec=7200.0)
    .query("mag_g < 22")
    .compute()
)
client.close()
```

---

## §7 — Writing HATS Catalogs

```python
import pandas as pd
import hats

df = pd.read_parquet("my_catalog.parquet")

# Import a pandas DataFrame into HATS format
from hats_import.catalog.arguments import ImportArguments
from hats_import.pipeline import pipeline_with_client

args = ImportArguments(
    input_path="my_catalog.parquet",
    output_path="output/hats/",
    catalog_name="my-catalog",
    ra_column="ra",
    dec_column="dec",
    catalog_type="object",
    highest_healpix_order=7,    # ~0.5 sq deg pixels; use 5–10 depending on density
)
pipeline_with_client(args, client=None)   # pass Dask client for distributed
```

---

## §8 — Public HATS Catalogs

| Catalog | Access | Notes |
|---|---|---|
| Gaia DR3 | `s3://stpubdata/hats/catalogs/gaia_dr3/` (anon) | 1.8B sources |
| 2MASS PSC | `s3://stpubdata/hats/catalogs/2mass_psc/` (anon) | 470M sources |
| ZTF DR22 | `s3://stpubdata/hats/catalogs/ztf_dr22/` (anon) | Alert + source |
| AllWISE | Contact IRSA | HATS conversion in progress |
| Rubin LSST DR1 | TBD ~2026 | Via Rubin RSP or LSDB public buckets |

> Verify S3 paths at https://data.lsdb.io — bucket layouts change between releases.

---

## §9 — Integration with astropy and pyvo

```python
import lsdb
from astropy.table import Table
from astropy.coordinates import SkyCoord
import astropy.units as u
import pyvo as vo

# LSDB result → astropy Table
result_df = cat.cone_search(ra=10.0, dec=20.0, radius_arcsec=3600.0).compute()
tbl = Table.from_pandas(result_df)

# Cross-match LSDB result with VO TAP result using SkyCoord
tap = vo.dal.TAPService("https://irsa.ipac.caltech.edu/TAP")
vo_tbl = tap.run_sync("SELECT ra, dec, j_m FROM fp_psc WHERE ...").to_table()

c_lsdb = SkyCoord(tbl["ra"], tbl["dec"], unit="deg")
c_vo = SkyCoord(vo_tbl["ra"], vo_tbl["dec"], unit="deg")
idx, sep, _ = c_lsdb.match_to_catalog_sky(c_vo)
matched = tbl[sep < 1 * u.arcsec]
```

---

## §10 — When to Graduate Out of This Skill

| Need | Tool |
|---|---|
| Full VO archive query (TAP/ADQL) | pyvo (vo-explorer skill) |
| Single-file schema inspection | DuckDB (data-tools skill) |
| Rubin RSP interactive | Butler + Jupyter on RSP |
| LSST catalog + spectroscopic followup | daf_butler (Rubin RSP) |

---

## References

- LSDB docs: https://lsdb.readthedocs.io
- HATS docs: https://hats.readthedocs.io
- Public HATS catalogs browser: https://data.lsdb.io
- astronomy-commons GitHub: https://github.com/astronomy-commons/lsdb
- Fornax cloud-native notebooks: https://github.com/nasa-fornax/fornax-demo-notebooks
