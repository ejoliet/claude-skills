# Astronomy Data Patterns Reference

Canonical patterns for pyvo, astroquery, file I/O, and VO discovery at IPAC.
Sources: pyvo docs (v1.8+), astroquery IRSA/MAST docs, IRSA TAP guide, NASA-NAVO notebooks.

---

## Table of Contents
1. [pyvo TAP patterns](#pyvo-tap)
2. [pyvo Registry discovery](#pyvo-registry)
3. [astroquery IRSA](#astroquery-irsa)
4. [astroquery MAST](#astroquery-mast)
5. [FITS I/O](#fits-io)
6. [ASDF I/O](#asdf-io)
7. [Parquet / S3](#parquet-s3)
8. [VOTable handling](#votable)
9. [VO service URLs](#service-urls)
10. [ADQL cheat sheet](#adql)

---

## pyvo TAP <a name="pyvo-tap"></a>

```python
import pyvo as vo
from astropy.coordinates import SkyCoord
from astropy.units import Quantity

# Direct TAP service (when you know the URL)
tap = vo.dal.TAPService("https://irsa.ipac.caltech.edu/TAP")

# Synchronous query (small/medium results)
results = tap.run_sync("""
    SELECT TOP 1000 designation, ra, dec, w1mpro, w2mpro
    FROM allwise_p3as_psd
    WHERE CONTAINS(POINT('ICRS', ra, dec),
                   CIRCLE('ICRS', 83.82, -5.39, 0.5)) = 1
""")
tbl = results.to_table()

# Async query (large results, >10k rows)
with tap.submit_job(query) as job:
    job.run()
    job.wait(phases=["COMPLETED", "ERROR"], timeout=120)
    if job.phase == "COMPLETED":
        result = job.fetch_result().to_table()

# ObsCore query pattern
obscore_tap = vo.dal.TAPService("https://irsa.ipac.caltech.edu/TAP")
obs = obscore_tap.run_sync("""
    SELECT obs_id, s_ra, s_dec, dataproduct_type, access_url
    FROM ivoa.obscore
    WHERE obs_collection = 'SPHEREx'
      AND 1=CONTAINS(POINT('ICRS', s_ra, s_dec), CIRCLE('ICRS', 83.82, -5.39, 1.0))
""").to_table()

# Datalink traversal
for dl in results.iter_datalinks():
    previews = dl[dl["semantics"] == "#preview"]
```

---

## pyvo Registry Discovery <a name="pyvo-registry"></a>

```python
import pyvo as vo

# Find all TAP services with ObsCore
for svc_rec in vo.registry.search(datamodel="obscore", servicetype="tap"):
    print(svc_rec["ivoid"], svc_rec["res_title"])

# Get a service from a registry record (preferred over access_url)
svc = svc_rec.get_service(service_type="tap", lax=True)

# Find image services in a waveband
image_svcs = vo.registry.search(
    servicetype="sia",
    waveband="infrared"
)

# Find by keyword
results = vo.registry.search(keywords=["NEOWISE", "moving objects"])
```

---

## astroquery IRSA <a name="astroquery-irsa"></a>

IRSA hosts: WISE/NEOWISE, Spitzer, 2MASS, SPHEREx, ZTF, Euclid, SOFIA, IRTF, Herschel, IRAS

```python
from astroquery.ipac.irsa import Irsa
from astropy.coordinates import SkyCoord
import astropy.units as u

# List catalogs (filter= available in astroquery >= 0.4.10)
Irsa.list_catalogs(filter="wise")

# Cone search — fastest for spatial-only queries
coord = SkyCoord(83.82, -5.39, unit="deg")
tbl = Irsa.query_region(coord, catalog="allwise_p3as_psd",
                         spatial="Cone", radius=5 * u.arcmin)

# Polygon search
from astropy.coordinates import SkyCoord
poly = [SkyCoord(ra=r, dec=d, unit="deg") for r, d in
        [(83.8, -5.4), (83.9, -5.4), (83.9, -5.3), (83.8, -5.3)]]
tbl = Irsa.query_region("", catalog="allwise_p3as_psd",
                         spatial="Polygon", polygon=poly)

# Complex TAP/ADQL query
tbl = Irsa.query_tap("""
    SELECT designation, ra, dec, w1mpro, w1sigmpro, w2mpro
    FROM allwise_p3as_psd
    WHERE CONTAINS(POINT('ICRS', ra, dec),
                   CIRCLE('ICRS', 83.82, -5.39, 0.5)) = 1
      AND w1mpro < 14.0
      AND cc_flags LIKE '0%'
""").to_table()

# Async mode (large queries, available in astroquery >= 0.4.11)
job = Irsa.query_tap(query, async_job=True)
tbl = job.get_results().to_table()
```

---

## astroquery MAST <a name="astroquery-mast"></a>

MAST hosts: HST, JWST, TESS, Kepler, K2, Roman (future), GALEX

```python
from astroquery.mast import Observations, Mast, Catalogs

# Cone search
obs_table = Observations.query_region("M51", radius="0.2 deg")

# Filter by mission
jwst = obs_table[obs_table["obs_collection"] == "JWST"]
hst = obs_table[obs_table["obs_collection"] == "HST"]

# Get downloadable products
products = Observations.get_product_list(jwst[:5])
science = Observations.filter_products(products, productType="SCIENCE",
                                        extension="fits")
manifest = Observations.download_products(science)

# Cloud access (S3 URIs) — preferred for AWS-based workflows
cloud_uris = Observations.get_cloud_uris(products)
# Returns s3://mast-hst-products/... URIs

# Resolve object name
coords = Mast().resolve_object("NGC 4993", resolver="NED")

# Catalog queries (e.g. Gaia from MAST)
gaia = Catalogs.query_region("M31", catalog="Gaia", radius=0.1)
```

---

## FITS I/O <a name="fits-io"></a>

```python
from astropy.io import fits
from astropy.table import Table
from astropy.wcs import WCS
import numpy as np

# Inspect HDU structure always first
with fits.open("image.fits", memmap=True) as hdul:
    hdul.info()
    # HDU 0: PRIMARY  (image data or empty)
    # HDU 1: SCI      (science data)
    # HDU 2: ERR      (error array)

# Read image + WCS
with fits.open("image.fits", memmap=True) as hdul:
    data = hdul["SCI"].data          # use name, not index
    header = hdul["SCI"].header
    wcs = WCS(header)

# Pixel → sky coordinate
from astropy.coordinates import SkyCoord
sky = wcs.pixel_to_world(512, 512)   # returns SkyCoord

# Read binary table
tbl = Table.read("catalog.fits")     # astropy handles FITS_1 ext
df = tbl.to_pandas()                 # → pandas DataFrame

# Write FITS table
tbl.write("output.fits", format="fits", overwrite=True)

# MEF (Multi-Extension FITS) — SPHEREx pattern
with fits.open("spherex_image.fits", memmap=True) as hdul:
    sci_data = hdul["SCI"].data
    # SPHEREx: spectral WCS in separate extension, SIP must be disabled
    spectral_wcs = WCS(hdul["SCI"].header, key="W")  # alt WCS
    spectral_wcs.sip = None  # required for SPHEREx spectral WCS
    wl, bw = spectral_wcs.pixel_to_world(x, y)
```

---

## ASDF I/O <a name="asdf-io"></a>

ASDF is used for Roman Space Telescope and JWST pipeline products.

```python
import asdf
import asdf_astropy  # noqa: F401 — registers astropy types

# Read an ASDF file
with asdf.open("roman_l2.asdf") as af:
    # af.tree is a dict-like structure
    data = af["roman"]["data"]          # numpy array
    meta = af["roman"]["meta"]
    wcs = af["roman"]["meta"]["wcs"]    # gwcs.WCS object

# Inspect tree structure
with asdf.open("file.asdf") as af:
    print(af.info())                     # prints tree summary

# Write ASDF
tree = {
    "data": np.zeros((100, 100), dtype=np.float32),
    "meta": {"telescope": "Roman", "filter": "F106"},
}
with asdf.AsdfFile(tree) as af:
    af.write_to("output.asdf")
```

---

## Parquet / S3 <a name="parquet-s3"></a>

```python
import pandas as pd
import pyarrow as pa
import pyarrow.parquet as pq
from astropy.table import Table

# Write astropy Table → Parquet (via pandas)
tbl = Table.read("catalog.fits")
df = tbl.to_pandas()
df.to_parquet("catalog.parquet", engine="pyarrow", compression="snappy")

# Read Parquet → astropy Table
df = pd.read_parquet("catalog.parquet")
tbl = Table.from_pandas(df)

# S3 Parquet with column pruning (awswrangler)
import awswrangler as wr
df = wr.s3.read_parquet(
    path="s3://my-bucket/catalog/",
    columns=["ra", "dec", "w1mpro"],
    dataset=True,
    partition_filter=lambda x: x["filter"] == "W1"
)

# Write partitioned Parquet to S3
wr.s3.to_parquet(
    df=df,
    path="s3://my-bucket/output/",
    dataset=True,
    partition_cols=["obs_date"],
    compression="snappy"
)
```

---

## VOTable Handling <a name="votable"></a>

```python
from astropy.io.votable import parse, parse_single_table
from astropy.table import Table

# Parse VOTable from file or URL
votable = parse("results.vot")
for resource in votable.resources:
    for table in resource.tables:
        tbl = table.to_table()

# Single-table shortcut
tbl = parse_single_table("results.vot").to_table()

# pyvo results are already VOTable-backed; just call .to_table()
result = tap.run_sync("SELECT ...")
tbl = result.to_table()

# Write VOTable
tbl.write("output.vot", format="votable", overwrite=True)
```

---

## VO Service URLs <a name="service-urls"></a>

| Service | TAP URL |
|---------|---------|
| IRSA | `https://irsa.ipac.caltech.edu/TAP` |
| MAST | `https://mast.stsci.edu/api/v0.1/invoke` (use astroquery) |
| GAVO/GAIA | `https://gea.esac.esa.int/tap-server/tap` |
| SIMBAD | `https://simbad.u-strasbg.fr/simbad/tap/sync` |
| VizieR | `https://tapvizier.u-strasbg.fr/TAPVizieR/tap` |
| NED | `https://ned.ipac.caltech.edu/tap` |

Use `pyvo.registry.search()` to discover URLs dynamically rather than hardcoding.

---

## ADQL Cheat Sheet <a name="adql"></a>

```sql
-- Cone search
WHERE 1=CONTAINS(POINT('ICRS', ra, dec), CIRCLE('ICRS', 83.82, -5.39, 0.5))

-- Box search
WHERE 1=CONTAINS(POINT('ICRS', ra, dec),
                 BOX('ICRS', 83.82, -5.39, 1.0, 1.0))

-- Polygon
WHERE 1=CONTAINS(POINT('ICRS', ra, dec),
                 POLYGON('ICRS', 83.0,-5.0, 84.0,-5.0, 84.0,-6.0, 83.0,-6.0))

-- Top N results
SELECT TOP 1000 ...

-- Cross-match pattern (do server-side when possible)
SELECT a.*, b.w1mpro
FROM catalog1 AS a
JOIN catalog2 AS b ON (1=CONTAINS(POINT('ICRS', a.ra, a.dec),
                                   CIRCLE('ICRS', b.ra, b.dec, 1.0/3600.0)))

-- ObsCore standard columns
SELECT obs_id, obs_collection, s_ra, s_dec, s_fov,
       t_min, t_max, em_min, em_max, dataproduct_type, access_url
FROM ivoa.obscore
WHERE obs_collection = 'SPHEREx'
```
