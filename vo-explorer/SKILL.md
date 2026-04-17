---
name: vo-explorer
description: >
  Expert guide for all Virtual Observatory (VO) software development: protocols
  (TAP, SODA, SCS, SIA, SSA, DataLink, RegTAP, ObsCore, ADQL, MOC, HiPS, UWS,
  SAMP), spatial operations (cone search, cutout, cross-match, TAP UPLOAD join,
  MOC filtering), modern formats (Parquet, HATS, LSDB, Zarr, VOTable, ASDF),
  data lakes (Iceberg, pyiceberg, AWS Glue), distributed compute (Dask, Slurm,
  EKS), science platforms (Rubin RSP, NOIRLab, ESCAPE), visualization (Firefly,
  jdaviz, ipyaladin, glue-jupyter, datashader), multi-mission science (Rubin,
  Euclid, SPHEREx, Roman, JWST), SED fitting (RAIL, LePHARE, CIGALE, EAZY,
  Bagpipes), source extraction (sep, photutils, scarlet), alert streams,
  provenance, and software publication (DaCHS, JOSS, FAIR). Trigger for: pyvo,
  astroquery, lsdb, hats, mocpy, firefly_client, jdaviz, ipyaladin, zarr,
  pyiceberg, ADQL queries, TAP cross-match, SODA cutout, catalog partitioning,
  HPC pipeline, S3 catalog, or any multi-mission overlap workflow.
---

# VO Explorer Skill

Full-stack guide: IVOA protocol stack → spatial operations → modern formats →
distributed compute → data lakes → visualization. Covers the complete path from
VO service discovery to science-ready interactive analysis.

> **MANDATORY FIRST STEP** — Before writing any code for a library covered
> here, call Context7 MCP to fetch current docs. See §0 for the procedure.

---

## §0 — Context7 Live Documentation Protocol

**Always call Context7 before generating code for any library in this skill.**
APIs in this space evolve fast (pyvo, lsdb, hats, jdaviz, firefly_client, zarr,
pyiceberg). Training data goes stale. Context7 prevents hallucinated signatures.

### MCP endpoint
```
URL:   https://mcp.context7.com/mcp
Tools: resolve-library-id  →  get-library-docs
```

### Two-step procedure (mandatory before any library code)

**Step 1 — resolve the library ID:**
```
tool:  resolve-library-id
input: { "libraryName": "<library name e.g. pyvo>" }
```

**Step 2 — fetch relevant docs:**
```
tool:  get-library-docs
input: {
  "context7CompatibleLibraryID": "<id from step 1>",
  "topic": "<specific topic e.g. 'TAP async query' or 'crossmatch'>",
  "tokens": 5000
}
```

### Library resolve names

| Library | Resolve name | Topic examples |
|---|---|---|
| pyvo | `"pyvo"` | `"TAP async"`, `"DataLink SODA"`, `"registry search"` |
| astropy | `"astropy"` | `"SkyCoord"`, `"Table VOTable"`, `"units"` |
| lsdb | `"lsdb"` | `"crossmatch"`, `"cone_search"`, `"to_hats"` |
| hats | `"hats"` | `"catalog metadata"`, `"partition"` |
| hats-import | `"hats-import"` | `"ImportArguments"`, `"pipeline_with_client"` |
| jdaviz | `"jdaviz"` | `"Imviz"`, `"Cubeviz"`, `"Specviz"`, `"load_data"` |
| firefly_client | `"firefly_client"` | `"show_table"`, `"show_fits"`, `"make_lab_client"` |
| zarr | `"zarr"` | `"open store"`, `"chunking"`, `"S3FileStore"` |
| pyiceberg | `"pyiceberg"` | `"load_catalog"`, `"scan"`, `"append"` |
| mocpy | `"mocpy"` | `"from_cone"`, `"intersection"`, `"contains_lonlat"` |
| specutils | `"specutils"` | `"Spectrum1D"`, `"line fitting"` |
| ipyaladin | `"ipyaladin"` | `"Aladin"`, `"add_table"`, `"add_moc"` |
| lightkurve | `"lightkurve"` | `"search_lightcurve"`, `"TessLightCurveFile"` |

### Fallback if Context7 returns nothing
Use the official docs URL from §12 References, then web search for release
notes of the specific version. Always note which version was assumed.

### When NOT to call Context7
- Pure ADQL query writing (SQL dialect, no library API)
- IVOA standard conceptual descriptions (stable specs)
- Architecture questions not requiring method signatures

---

## §1 — Protocol Selection

```
What do you need?
│
├─ Find what services exist globally → §2 RegTAP registry
│
├─ Query a catalog table → §3 TAP + ADQL
│   ├─ Fast / small result → sync TAP
│   ├─ Long-running / large → async TAP (UWS)
│   └─ Join against your own table → TAP UPLOAD cross-match
│
├─ Find and download images → §4 SIA 2.0 + ObsCore
│   └─ Cut a spatial/spectral region → SODA via DataLink
│
├─ Single-position catalog query → §5 SCS
│
├─ Spectrum discovery → §5 SSA
│
├─ Sky coverage / footprint logic → §6 MOC + HEALPix
│
├─ Billion-row catalog, scalable cross-match → §7 HATS + LSDB
│
├─ N-D array cubes (images, IFU, spectral) → §7 Zarr + xarray
│
├─ Versioned pipeline output table → §8 Apache Iceberg
│
├─ Distributed compute setup → §9 Platform / Dask
│
└─ Visualize interactively → §10 Visualization tools
```

---

## §2 — Service Discovery via RegTAP

RegTAP 1.2 (IVOA 2024) is the standard way to find VO services.
**Prefer registry lookup over hardcoding access URLs.**

```python
import pyvo as vo
from astropy.coordinates import SkyCoord
import astropy.units as u

# All TAP services exposing ObsCore
svcs = vo.registry.search(
    vo.registry.Datamodel("obscore"),
    vo.registry.Servicetype("tap")
)
svcs.to_table()["short_name", "res_title", "access_url"]

# By waveband + keyword
svcs = vo.registry.search(
    vo.registry.Waveband("infrared"),
    vo.registry.Freetext("galaxy clusters")
)

# Spatially (RegTAP 1.2 — not all registries support this yet)
svcs = vo.registry.search(
    vo.registry.Servicetype("sia2"),
    vo.registry.Spatial(SkyCoord(202.48, 47.23, unit="deg"), radius=5*u.deg)
)

# Get a usable service object
tap = svcs[0].get_service("tap")
```

**Service types:** `"tap"` `"sia2"` `"ssa"` `"conesearch"` `"datalink"`

See `references/protocols.md` §Registry for archive-specific stable access URLs
(IRSA, MAST, ESA Gaia, CDS VizieR, ESO, NOIRLab).

---

## §3 — TAP + ADQL

→ **Call Context7: `resolve-library-id("pyvo")` → `get-library-docs` topic `"TAP"`**

### Synchronous query
```python
tap = vo.dal.TAPService("https://irsa.ipac.caltech.edu/TAP")
result = tap.run_sync("""
    SELECT TOP 1000 ra, dec, j_m, h_m, k_m
    FROM fp_psc
    WHERE CONTAINS(
        POINT('ICRS', ra, dec),
        CIRCLE('ICRS', 202.48, 47.23, 0.5)
    ) = 1
    AND j_m < 15.0
""")
tbl = result.to_table()         # astropy Table
df  = tbl.to_pandas()           # pandas DataFrame
```

### Asynchronous query (UWS — for long jobs / large results)
```python
job = tap.submit_job("""
    SELECT source_id, ra, dec, phot_g_mean_mag
    FROM gaiadr3.gaia_source
    WHERE CONTAINS(POINT('ICRS', ra, dec),
                   CIRCLE('ICRS', 266.4, -29.0, 1.0)) = 1
""", maxrec=1000000)
job.run()
job.wait(phases=["COMPLETED", "ERROR"], timeout=600)
result = job.fetch_result().to_table()
job.delete()   # clean up server-side job
```

### TAP UPLOAD — server-side cross-match
Upload a local target list and JOIN it against a remote catalog in ADQL.
Most efficient pattern for positional cross-matching without downloading full catalog.

```python
from astropy.table import Table

targets = Table({"ra": [83.82, 84.05], "dec": [-5.39, -5.12]})

if tap.upload_methods:   # check support first
    result = tap.run_sync("""
        SELECT t.ra AS t_ra, t.dec AS t_dec,
               c.j_m, c.h_m, c.k_m
        FROM TAP_UPLOAD.mine AS t
        JOIN fp_psc AS c
          ON 1=CONTAINS(
              POINT('ICRS', c.ra, c.dec),
              CIRCLE('ICRS', t.ra, t.dec, 0.0028)
          )
    """, uploads={"mine": targets})
```

### ADQL geometry reference
```sql
-- Circle
CONTAINS(POINT('ICRS', ra, dec), CIRCLE('ICRS', 202.48, 47.23, 0.5)) = 1

-- Polygon
CONTAINS(POINT('ICRS', ra, dec),
         POLYGON('ICRS', 10.0,-1.0, 10.5,-1.0, 10.5,-0.5, 10.0,-0.5)) = 1

-- Region overlap (use INTERSECTS when CONTAINS is not selective enough)
INTERSECTS(s_region, CIRCLE('ICRS', 202.48, 47.23, 0.5)) = 1
```

---

## §4 — SIA 2.0 + ObsCore → DataLink → SODA Cutouts

Standard 3-step chain for archive image access.

→ **Call Context7: `resolve-library-id("pyvo")` → topic `"DataLink SODA cutout"`**

```python
# Step 1: Discover images
sia = vo.dal.SIA2Service("https://irsa.ipac.caltech.edu/SIA")
imgs = sia.search(
    pos=SkyCoord.from_name("M42"),
    size=Quantity(0.5, "deg"),
    format="image/fits"
)

# Step 2: Navigate DataLink to find the SODA cutout service
for dl in imgs.iter_datalinks():
    soda = dl.get_service("#cutout")
    if soda:
        break

# Step 3: Request cutout
if soda:
    cutout = soda.search(
        ID=dl.original_row["obs_publisher_did"],
        POS="CIRCLE 83.82 -5.39 0.05",
        # BAND="2.0e-6 2.5e-6",   # spectral range in metres (optional)
    )
    with open("cutout.fits", "wb") as f:
        f.write(cutout[0].getdataset().read())
```

**Critical pitfall:** Many archives expose DataLink but do NOT implement SODA.
Always guard with `if soda:` and fall back to direct `access_url` download.
See `references/pitfalls.md` §SODA for per-archive status.

---

## §5 — SCS and SSA

```python
# Simple Cone Search
scs = vo.dal.SCSService(
    "https://vizier.cds.unistra.fr/viz-bin/conesearch/I/355/gaiadr3"
)
result = scs.search(
    pos=SkyCoord(202.48, 47.23, unit="deg"),
    radius=Quantity(0.1, "deg")
)

# Simple Spectral Access
ssa = vo.dal.SSAService("http://archive.eso.org/ssap")
spectra = ssa.search(
    pos=SkyCoord.from_name("NGC 1068"),
    diameter=Quantity(0.05, "deg"),
    band=Quantity([3e-7, 1e-6], "m")
)
spec_url = spectra[0].getdataurl()
# → feed to specutils or jdaviz Specviz
```

---

## §6 — MOC + HEALPix

→ **Call Context7: `resolve-library-id("mocpy")` → topic matching your operation**

```python
from mocpy import MOC
import astropy.units as u

# Create
moc = MOC.from_cone(lon=202.48*u.deg, lat=47.23*u.deg,
                    radius=1.0*u.deg, max_depth=10)
moc = MOC.from_polygon_skycoord(skycoords, max_depth=12)
sdss = MOC.from_vizier_table("V/147/sdss12", max_depth=10)

# Set operations
overlap  = moc1.intersection(moc2)
union    = moc1.union(moc2)
diff     = moc1.difference(moc2)

# Filter a table to footprint
mask     = moc.contains_lonlat(table["ra"]*u.deg, table["dec"]*u.deg)
filtered = table[mask]

# RegTAP service discovery within a MOC
svcs = vo.registry.search(vo.registry.Spatial(moc))

# Serialize / deserialize
moc.save("coverage.fits", format="fits")
moc2 = MOC.load("coverage.fits", format="fits")
```

**Space-Time MOC (STMOC):** combine spatial + temporal coverage for time-domain surveys.
See `references/formats.md` §MOC for serialization formats and order/nside conversion.

---

## §7 — Modern Formats: Parquet, HATS, LSDB, Zarr

→ **Call Context7 for `lsdb`, `hats`, and `zarr` before writing format code.**

### Parquet
```python
import pyarrow.parquet as pq
import pyarrow.fs as pafs

fs = pafs.S3FileSystem(region="us-east-1")   # preferred over s3fs for throughput

tbl = pq.read_table(
    "bucket/catalog/",
    columns=["ra", "dec", "mag_g"],
    filters=[("mag_g", "<", 22.0)],   # row-group predicate pushdown
    filesystem=fs
)

# Dask lazy read
import dask.dataframe as dd
ddf = dd.read_parquet(
    "s3://bucket/catalog/",
    engine="pyarrow",
    columns=["ra", "dec", "mag_g"],
    filters=[[("mag_g", "<", 22.0)]],
)
```

### HATS + LSDB
```python
import lsdb

gaia = lsdb.read_hats("s3://data.lsdb.io/hats/gaia_dr3/")
ztf  = lsdb.read_hats("s3://data.lsdb.io/hats/ztf_dr22/")

# Cone filter (partition-aware, very fast)
field = gaia.cone_search(ra=202.48, dec=47.23, radius_arcsec=3600.0)

# Cross-match (nearest neighbour)
matched = ztf.crossmatch(gaia, n_neighbors=1, radius_arcsec=1.0)
result  = matched.compute()   # triggers Dask execution

# Save result as HATS
matched.to_hats("s3://bucket/output/ztf_x_gaia/", catalog_name="ztf_x_gaia")
```

### HATS import pipeline (external catalogs → HATS)
```python
from hats_import.pipeline import pipeline_with_client
from hats_import.catalog.arguments import ImportArguments
from dask.distributed import Client

args = ImportArguments(
    input_path="s3://bucket/raw/catalog.parquet",
    output_path="s3://bucket/hats/my_catalog/",
    ra_column="ra", dec_column="dec",
    catalog_name="my_catalog",
    file_reader="parquet",
)
with Client() as client:
    pipeline_with_client(args, client)
```

### Zarr (N-D cubes)
```python
import zarr, xarray as xr

# Open from S3
ds = xr.open_zarr("s3://bucket/cube.zarr", chunks="auto")

# Labeled cutout
cutout = ds["flux"].sel(ra=slice(202.0, 203.0), dec=slice(47.0, 48.0))

# Write with tuned chunking
store = zarr.open("output.zarr", mode="w",
    shape=(1000, 4096, 4096), chunks=(100, 512, 512),
    dtype="float32",
    compressor=zarr.Blosc(cname="zstd", clevel=5)
)
```

**⚠ Zarr v3:** `zarr-python` 3.x changed the store interface. Call Context7 for
current API. Always pin version in `requirements.txt`.

See `references/formats.md` for full conversion recipes.

---

## §8 — Apache Iceberg Data Lake

Use Iceberg when VO pipeline results need ACID writes, schema evolution,
time travel, or downstream SQL engines (Athena, Trino, Spark, DuckDB).

→ **Call Context7: `resolve-library-id("pyiceberg")` before writing Iceberg code.**

```python
from pyiceberg.catalog import load_catalog
import pyarrow as pa

# AWS Glue catalog (standard for AWS/IPAC deployments)
catalog = load_catalog("glue", **{"type": "glue", "s3.region": "us-east-1"})

# Create table
from pyiceberg.schema import Schema
from pyiceberg.types import NestedField, DoubleType, LongType
schema = Schema(
    NestedField(1, "source_id", LongType(), required=True),
    NestedField(2, "ra",        DoubleType()),
    NestedField(3, "dec",       DoubleType()),
    NestedField(4, "mag_g",     DoubleType()),
)
catalog.create_namespace("roman_soc")
table = catalog.create_table("roman_soc.catalog_v1", schema=schema)

# Append (from TAP result → pandas → Arrow)
table.append(pa.Table.from_pandas(result_df))

# Read with predicate pushdown
df = table.scan(row_filter="mag_g < 22.0").to_arrow().to_pandas()

# Time travel
df_old = table.scan(snapshot_id=<snap_id>).to_arrow().to_pandas()
```

### HATS vs Iceberg decision

| Need | Format |
|---|---|
| Spatial cross-match, HEALPix-aware, astronomy-native | **HATS + LSDB** |
| ACID writes, schema evolution, time travel, SQL engines | **Iceberg** |
| Ingest → process → serve | HATS for compute → Iceberg for result table |

---

## §9 — Platform Layer: Dask Backends

Same LSDB/Parquet/Zarr code on all platforms — only the client setup changes.

### Local / JupyterLab
```python
from dask.distributed import Client, LocalCluster
client = Client(LocalCluster(n_workers=4, threads_per_worker=2))
```

### Slurm HPC
```python
from dask_jobqueue import SLURMCluster
cluster = SLURMCluster(
    cores=8, memory="32GB", walltime="02:00:00",
    job_extra_directives=["--partition=science"],
)
cluster.scale(jobs=10)
client = Client(cluster)
# ⚠ Pre-stage data to GPFS/NFS — compute nodes usually block S3
```

### Kubernetes / EKS (Emmanuel's default)
```python
from dask_kubernetes.operator import KubeCluster
cluster = KubeCluster(
    image="ipac/jupyter:dev",
    resources={"requests": {"memory": "8Gi", "cpu": "4"}},
    n_workers=20,
)
client = Client(cluster)
```

### Rubin RSP / NOIRLab Astro Data Lab
```python
# RSP: built-in token-authenticated TAP
from lsst.rsp import get_tap_service
tap = get_tap_service()

# External pyvo with RSP token
import pyvo
cred = pyvo.auth.CredentialStore()
cred.set_password("x-oauth-basic",
                  open("~/.rsp-tap.token").read().strip())
tap = pyvo.dal.TAPService(
    "https://data.lsst.cloud/api/tap",
    session=cred.get("ivo://ivoa.net/sso#BasicAA")
)
```

### S3 performance rules
- Use `pyarrow.fs.S3FileSystem` directly (avoids Python GIL bottleneck vs s3fs)
- Always pass `filters=` for Parquet predicate pushdown on row groups
- Co-locate compute in same AWS region as S3 bucket
- HATS partitions are pre-sized ~100–300 MB — no repartitioning needed

See `references/platforms.md` for full Dask gateway configs and HPC checklist.

---

## §10 — Visualization Tools

→ **Call Context7 for `jdaviz`, `firefly_client`, `ipyaladin` before coding.**

### Tool selection

| Tool | Best for | Notebook? | Notes |
|---|---|---|---|
| **Firefly** | Archive-quality: linked tables+images+charts+SEDs | ✅ JupyterLab | IPAC/IRSA native; RSP Portal Aspect |
| **jdaviz** | JWST/Roman: 2D images, spectra, IFU cubes, MOS | ✅ cell widget | Imviz, Specviz, Cubeviz, Mosviz |
| **ipyaladin** | HiPS sky browser, MOC/catalog overlays | ✅ anywidget | CDS data; survey footprints |
| **glue-jupyter** | Linked multi-dataset brushing-and-linking | ✅ | Foundation of jdaviz |
| **pywwt** | 3D all-sky WWT, cross-wavelength comparison | ✅ | AAS tool |
| **datashader + bokeh** | Billion-row scatter (RSP Rubin standard) | ✅ | Large LSDB results |
| **specutils** | 1D spectral I/O, line fitting, EW | library only | Feeds jdaviz Specviz |
| **lightkurve** | Kepler/TESS time series, MAST-integrated | ✅ | Time-domain pipelines |
| **TOPCAT** (external) | TAP, ADQL, large tables, SAMP hub | via SAMP | `astropy.samp` bridge |

### Firefly
```python
from firefly_client import FireflyClient
import urllib.parse

# In JupyterLab (needs jupyter-firefly-extensions + $FIREFLY_URL)
fc = FireflyClient.make_lab_client()
# Outside JupyterLab
fc = FireflyClient.make_client(url="https://irsa.ipac.caltech.edu/irsaviewer")
fc.reinit_viewer()

# Show TAP result as table (auto-overlays on sky if coords present)
q = urllib.parse.quote_plus("SELECT ra,dec,j_m FROM fp_psc WHERE ...")
fc.show_table(f"https://irsa.ipac.caltech.edu/TAP/sync?QUERY={q}",
              tbl_id="cat", title="2MASS Query")

# Display FITS image
fval = fc.upload_file("cutout.fits")
fc.show_fits(fval, plot_id="img", title="SODA Cutout")
fc.set_stretch("img", "sigma", "log", lower_value=-2, upper_value=30)

# Run Firefly locally
# docker run -p 8080:8080 -e MAX_JVM_SIZE=8G --rm ipac/firefly
# export FIREFLY_URL=http://localhost:8080/firefly
```

### jdaviz
```python
import jdaviz

imviz   = jdaviz.Imviz();   imviz.load_data("roman.asdf");    imviz.show()
cubeviz = jdaviz.Cubeviz(); cubeviz.load_data("cube.fits");   cubeviz.show()
specviz = jdaviz.Specviz(); specviz.load_data(spectrum1d_obj); specviz.show()
mosviz  = jdaviz.Mosviz();  mosviz.show()   # NIRSpec MOS
```

### ipyaladin
```python
import ipyaladin as ipyal
aladin = ipyal.Aladin(target="M42", fov=1.0, survey="CDS/P/DSS2/color")
display(aladin)
aladin.add_table(astropy_table)  # table must have ra/dec columns
aladin.add_moc(moc_object)
```

### datashader (billion-row catalogs)
```python
import datashader as ds, holoviews as hv
from holoviews.operation.datashader import rasterize
hv.extension("bokeh")
points     = hv.Points(dask_df, kdims=["ra", "dec"])
rasterized = rasterize(points, aggregator=ds.count())
rasterized.opts(colorbar=True, width=800, height=500)
```

### SAMP bridge to external tools
```python
from astropy.samp import SAMPIntegratedClient
client = SAMPIntegratedClient()
client.connect()
client.notify_all({
    "samp.mtype": "table.load.votable",
    "samp.params": {"url": votable_url, "table-id": "my_table"}
})
```

---

## §11 — Key Pitfalls

1. **SODA not widely implemented** — <50% of archives. Guard with
   `soda = dl.get_service("#cutout"); if not soda: fall back to access_url`.

2. **TAP UPLOAD not universal** — check `tap.upload_methods` before UPLOAD JOIN.

3. **Zarr v3 API break** — `zarr-python` 3.x changed store interfaces vs 2.x.
   Always call Context7 for current API; pin version.

4. **HATS incremental updates** — new in hats-import 0.3+; API still evolving.
   Call Context7 before using update/append workflows.

5. **Slurm + no internet** — HPC compute nodes typically block outbound HTTP.
   Pre-stage to GPFS/NFS; never rely on S3 from inside job unless explicitly allowed.

6. **RSP token auth** — use `pyvo.auth.CredentialStore`; never hardcode tokens.

7. **`INTERSECTS` vs `CONTAINS`** — `s_region` support is optional in ObsCore.
   Fall back to `CONTAINS(POINT, CIRCLE)` on center coords if zero results.

8. **Firefly needs a server** — `firefly_client` is a thin Python client.
   Requires a running Firefly instance (Docker or `https://irsa.ipac.caltech.edu/irsaviewer`).

9. **ADQL `TOP` vs `maxrec`** — `TOP N` in ADQL and `maxrec=N` in pyvo are
   independent limits; the more restrictive one applies.

---

## §13 — Multi-Mission Software Patterns

This section covers the engineering concerns specific to building software that
spans multiple concurrent surveys — the defining challenge of the 2025–2030 era
(Rubin/LSST DP1+, Euclid DR1, SPHEREx, Roman, JWST).

→ **Call Context7 for any library listed below before writing code.**

### Active mission landscape (2025–2026)

| Mission | Archive | Key data | VO access | Python library |
|---|---|---|---|---|
| Rubin LSST DP1 | RSP / IRSA | ugrizy imaging + catalogs | TAP, ObsCore, SODA | `lsst.daf.butler`, `lsst.rsp` |
| Euclid Q1+ | ESA / ENSCI | VIS, NISP-P, NISP-S | TAP, DataLink | `astroquery.esa.euclid` (beta) |
| SPHEREx | IRSA | 102-band all-sky spectra | TAP, SODA cutout | `astroquery.ipac.irsa` |
| Roman (sims now) | MAST / IPAC | WFI imaging, grism | ASDF-native | `roman_datamodels`, `romancal` |
| JWST | MAST | NIRCam/MIRI/NIRSpec | TAP, DataLink, SODA | `astroquery.mast`, `jdaviz` |
| Gaia DR3 | ESA | astrometry + BP/RP spectra | TAP | `astroquery.gaia` |

### Canonical cross-mission cross-match workflow

```python
import lsdb
import astropy.units as u
from astropy.coordinates import SkyCoord

# Step 1: load catalogs (HATS format where available)
rubin  = lsdb.read_hats("s3://data.lsdb.io/hats/rubin_dp1/")
gaia   = lsdb.read_hats("s3://data.lsdb.io/hats/gaia_dr3/")
euclid = lsdb.read_hats("s3://data.lsdb.io/hats/euclid_q1/")  # when available

# Step 2: restrict to science footprint using MOC intersection
from mocpy import MOC
footprint = MOC.from_fits("euclid_wide_dr1_moc.fits")
rubin_overlap = rubin.cone_search(  # or .moc_search() when lsdb supports it
    ra=150.0, dec=2.2, radius_arcsec=3600.0
)

# Step 3: propagate proper motions to common epoch before matching
# (critical for stars; negligible for galaxies at z > 0.1)
# See references/pitfalls.md §Epochs

# Step 4: sequential cross-match (chain)
rubin_x_gaia   = rubin_overlap.crossmatch(gaia,   n_neighbors=1, radius_arcsec=0.5)
rubin_x_euclid = rubin_x_gaia.crossmatch(euclid,  n_neighbors=1, radius_arcsec=0.5)
result = rubin_x_euclid.compute()
```

### Catalog authority by parameter type

| Parameter | Authoritative mission | Why |
|---|---|---|
| Astrometry (RA/Dec) | Gaia DR3 | Best proper motions, μas precision |
| u/g/r/i/z/y photometry | Rubin LSST | Deep, wide, 6-band |
| NIR photometry (Y/J/H) | Euclid NISP-P or Roman WFI | Space resolution |
| Spectral shape (102 bands) | SPHEREx | All-sky low-res spectra |
| Galaxy morphology / shape | Euclid VIS or Roman WFI | Diffraction-limited |
| Photo-z (galaxies) | Run RAIL/LePHARE on merged catalog | No single mission |
| Stellar classification | Gaia BP/RP + SPHEREx | |
| Variability / time-domain | Rubin LSST | Cadence survey |

### Photometric system reconciliation

Different missions use different zeropoints, filter curves, and apertures.
Always document and propagate the photometric system:

```python
# Standard: work in AB magnitudes throughout
# Gaia: Vega-based internally; use DR3 synthetic AB mags (phot_g_mean_mag_ab)
# 2MASS: Vega; apply corrections from Cohen et al. 2003 for AB

# Filter curve access via SVO Filter Profile Service
import requests
r = requests.get(
    "http://svo2.cab.inta-csic.es/theory/fps/api.php",
    params={"ID": "Euclid/NISP.Y", "FORMAT": "votable"}
)

# astropy synthetic photometry
from astropy.modeling.models import BlackBody
from synphot import SpectralElement, SourceSpectrum, Observation
# See references/science-tools.md §SED for full synphot patterns
```

### Astrometric epoch propagation (critical for stellar cross-matches)

```python
from astropy.coordinates import SkyCoord, Distance
from astropy.time import Time
import astropy.units as u

# Gaia DR3 epoch is J2016.0; Rubin/SPHEREx catalogs reference J2000 or obs epoch
coord_2016 = SkyCoord(
    ra=83.82*u.deg, dec=-5.39*u.deg,
    pm_ra_cosdec=1.5*u.mas/u.yr,
    pm_dec=-0.3*u.mas/u.yr,
    distance=Distance(parallax=5.0*u.mas),
    obstime=Time("J2016.0")
)
coord_2000 = coord_2016.apply_space_motion(new_obstime=Time("J2000.0"))
# Use coord_2000.ra.deg, coord_2000.dec.deg for cross-matching with J2000 catalogs

# Rule of thumb: epoch correction matters when:
# - proper motion > 10 mas/yr (high PM stars)
# - cross-matching epoch gap > 5 years
# - required match radius < 0.5 arcsec
```

### Mission-specific data access patterns

See `references/missions.md` for per-mission details. Key entry points:

**SPHEREx (IRSA, live since July 2025):**
```python
from astroquery.ipac.irsa import Irsa
# SPHEREx spectral image cutout (SODA-compatible service)
# Table names: spherex_s1_l2_sp_v01 (initial release — check IRSA for current)
results = Irsa.query_tap("""
    SELECT * FROM spherex_s1_l2_sp_v01
    WHERE CONTAINS(POINT('ICRS', ra, dec),
                   CIRCLE('ICRS', 150.0, 2.2, 0.1)) = 1
""").to_table()
```

**Euclid (ESA + ENSCI, Q1 March 2025):**
```python
# astroquery.esa.euclid is in active development — always call Context7 first
from astroquery.esa.euclid import Euclid   # may require astroquery >= 0.4.8
Euclid.login(username="...", password="...")
results = Euclid.query_tap("""
    SELECT * FROM euclid_dr1.mer_catalogue
    WHERE CONTAINS(POINT('ICRS', ra, dec),
                   CIRCLE('ICRS', 53.1, -28.0, 0.5)) = 1
""")
```

**Rubin Butler (internal pipeline access, not VO):**
```python
import lsst.daf.butler as daf_butler
butler = daf_butler.Butler("/repo/main", collections=["LSSTComCam/runs/DRP/..."])
calexp = butler.get("calexp", dataId={"visit": 1234, "detector": 0})
src    = butler.get("src",    dataId={"visit": 1234, "detector": 0})
# Butler is NOT a VO interface; use RSP TAP for VO-compatible access
```

**Roman (simulations / RomanCAL pipeline):**
```python
import roman_datamodels as rdm
model = rdm.open("roman_wfi_image_2_wcs.asdf")
data  = model.data          # 4096×4096 array
wcs   = model.meta.wcs      # GWCS object
# Feed to jdaviz Imviz or Firefly for visualization
```

### SED fitting and photometric redshifts

See `references/science-tools.md` §SED for full patterns. Key tools:

| Tool | Use case | Language |
|---|---|---|
| `RAIL` | Rubin/DESC photo-z pipeline framework; compare estimators | Python |
| `LePHARE` | SED template fitting; Euclid/COSMOS standard | Python wrapper |
| `CIGALE` | Bayesian SED fitting, AGN + galaxy | Python |
| `EAZY` | Fast photo-z, grism-friendly | Python |
| `Bagpipes` | Bayesian galaxy SED fitting + star formation histories | Python |
| `AGNfitter` | Bayesian AGN SED decomposition (radio→X-ray) | Python |

### Alert stream and time-domain integration

See `references/science-tools.md` §Alerts. Key patterns:

```python
# Rubin alert broker subscription (via community brokers)
# ALeRCE, Fink, ANTARES, LASAIR receive Rubin alerts via Kafka
import confluent_kafka
from lsst.alert.packet import SchemaRegistry

# TOM Toolkit for follow-up coordination
from tom_targets.models import Target
from tom_observations.facilities import get_service_class
```

### Provenance and reproducibility

See `references/publishing.md` §Provenance. Core principle:

```python
# Every pipeline output should record:
# 1. Input catalog names + versions + query parameters
# 2. Pipeline code version (git SHA)
# 3. Processing date + environment
# 4. Output DOI or persistent identifier

# Minimal provenance dict for ASDF output
provenance = {
    "inputs": {
        "gaia": {"version": "DR3", "query": adql_query, "date": "2025-07-01"},
        "rubin": {"collection": "LSSTComCam/runs/DRP/DP1", "butler_run": "..."},
    },
    "software": {"lsdb": lsdb.__version__, "git_sha": git_sha},
    "produced": datetime.utcnow().isoformat(),
}
# Embed in ASDF, Iceberg table metadata, or Parquet file-level metadata
```

---

## §14 — References (Updated)

| File | Read when generating code for |
|---|---|
| `references/protocols.md` | TAP/ADQL patterns, SIA/DataLink/SODA chains, archive stable URLs |
| `references/formats.md` | Format conversion recipes, Zarr chunking, MOC serialization |
| `references/platforms.md` | Dask backends, RSP/Slurm/K8s auth, S3 performance, workflow orchestration |
| `references/pitfalls.md` | SODA gaps, archive quirks, version caveats, epoch/quality flag patterns |
| `references/missions.md` | Per-mission access: SPHEREx, Euclid, Roman, Rubin Butler, JWST |
| `references/science-tools.md` | SED fitting, photo-z, source extraction, alert streams, deblending |
| `references/publishing.md` | Provenance, FAIR data, DaCHS TAP server, JOSS/ASCL, software citation |

**Context7 resolve names for new libraries:**

| Library | Resolve name | Key topics |
|---|---|---|
| RAIL | `"RAIL"` or `"LSSTDESC/RAIL"` | `"photo-z estimator"`, `"pipeline"` |
| LePHARE | `"LePHARE"` | `"SED fitting"`, `"photometric redshift"` |
| CIGALE | `"CIGALE"` | `"SED fitting"`, `"AGN"` |
| roman_datamodels | `"roman_datamodels"` | `"open"`, `"WFI"` |
| astroquery.esa.euclid | `"astroquery"` | `"euclid"`, `"query_tap"` |
| tom_toolkit | `"tom_toolkit"` | `"Target"`, `"observation"` |
| photutils | `"photutils"` | `"source extraction"`, `"aperture"` |
| sep | `"sep"` | `"extract"`, `"background"` |
| scarlet | `"scarlet"` | `"deblend"`, `"multiband"` |

**Official docs fallback:**

| Library | URL |
|---|---|
| pyvo | https://pyvo.readthedocs.io |
| lsdb | https://docs.lsdb.io |
| hats / hats-import | https://hats.readthedocs.io / https://hats-import.readthedocs.io |
| jdaviz | https://jdaviz.readthedocs.io |
| firefly_client | https://caltech-ipac.github.io/firefly_client |
| zarr | https://zarr.readthedocs.io |
| pyiceberg | https://py.iceberg.apache.org |
| mocpy | https://cds-astro.github.io/mocpy |
| specutils | https://specutils.readthedocs.io |
| ipyaladin | https://cds-astro.github.io/ipyaladin |
| lightkurve | https://lightkurve.github.io/lightkurve |
| Rubin RSP | https://rsp.lsst.io |
| roman_datamodels | https://roman-datamodels.readthedocs.io |
| RAIL | https://lsstdescrail.readthedocs.io |
| photutils | https://photutils.readthedocs.io |
| scarlet | https://pmelchior.github.io/scarlet |
| tom_toolkit | https://tom-toolkit.readthedocs.io |
| DaCHS | https://dachs-doc.readthedocs.io |
