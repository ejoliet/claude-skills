# rubin-lsst.md — Rubin Observatory / LSST Data Access Patterns

> Check current data release status at https://data.lsst.cloud and https://dp0.lsst.io.
> APIs and schema evolve with each Data Preview (DP0.2, DP0.3, DR1). Always call Context7
> for `lsst.daf.butler` before writing Butler code.

---

## Data Access Layers (choose by context)

| Layer | Tool | When to use |
|---|---|---|
| SQL/ADQL over catalogs | **TAP + pyvo** | Archive queries, cone search, cross-match |
| Processed image cutouts | **SODA (vo-cutouts)** | Image sub-region retrieval |
| Raw file access (exposures/coadds) | **daf_butler** | Pipeline dev, Rubin RSP, calibration |
| Billion-row catalog | **LSDB + HATS** | Large-scale spatial / lightcurve analysis |

---

## §1 — TAP Access (Rubin RSP + Public)

Rubin exposes catalogs via a TAP service on the Rubin Science Platform (RSP).

```python
import pyvo as vo
from astropy.coordinates import SkyCoord
import astropy.units as u

# RSP TAP endpoint (requires Rubin RSP token)
RSP_TAP = "https://data.lsst.cloud/api/tap"
svc = vo.dal.TAPService(RSP_TAP)

# List available tables
for t in svc.tables.values():
    if "object" in t.name.lower():
        print(t.name, t.description)

# Cone search on DP0.2 Object catalog
result = svc.run_sync("""
    SELECT objectId, coord_ra, coord_dec,
           mag_g_cModel, mag_r_cModel, mag_i_cModel,
           extendedness
    FROM dp02_dc2_catalogs.Object
    WHERE CONTAINS(
        POINT('ICRS', coord_ra, coord_dec),
        CIRCLE('ICRS', 62.0, -37.0, 0.1)
    ) = 1
      AND detect_isPrimary = 1
      AND mag_r_cModel < 25.0
""")
tbl = result.to_table()
```

### Authentication (RSP token)

```python
import pyvo as vo

# Set token in session
session = vo.auth.AuthSession()
session.credentials.set_password("x-oauth-basic", "YOUR_RSP_TOKEN", "data.lsst.cloud")

svc = vo.dal.TAPService("https://data.lsst.cloud/api/tap", session=session)
```

---

## §2 — Image Cutouts (SODA / vo-cutouts)

Rubin's `vo-cutouts` service implements IVOA SODA for cutout retrieval.

```python
import pyvo as vo
from astropy.coordinates import SkyCoord
import astropy.units as u

CUTOUT_URL = "https://data.lsst.cloud/api/cutout"

# SODA cutout via DataLink/SODA
result = svc.run_sync("""
    SELECT dataproduct_type, access_url, obs_id
    FROM dp02_dc2_catalogs.CcdVisit
    WHERE CONTAINS(
        POINT('ICRS', ra, dec),
        CIRCLE('ICRS', 62.0, -37.0, 0.05)
    ) = 1
""")

# Follow DataLink to get cutout URL
for row in result.to_table():
    print(row["access_url"])
```

Direct SODA pattern:

```python
import requests

params = {
    "ID": "butler://rubin-repo/ExposureF/...",
    "POS": "CIRCLE 62.0 -37.0 0.01",   # deg
    "FORMAT": "image/fits",
}
resp = requests.get(CUTOUT_URL, params=params,
                    headers={"Authorization": f"Bearer {RSP_TOKEN}"})
with open("cutout.fits", "wb") as f:
    f.write(resp.content)
```

---

## §3 — daf_butler (Rubin Data Access Framework)

Butler is Rubin's abstraction over data repositories. Required for pipeline
development and direct exposure/coadd access on the RSP.

```python
from lsst.daf.butler import Butler

# Open a Butler repo (RSP or local test repo)
butler = Butler("/repo/dp02", collections=["2.2i/runs/DP0.2"])

# List available dataset types
for dt in butler.registry.queryDatasetTypes():
    print(dt.name)

# Fetch a calibrated exposure (calexp)
dataId = {"visit": 192350, "detector": 175, "band": "r"}
calexp = butler.get("calexp", dataId=dataId)

# Access image data + WCS
image_array = calexp.image.array          # numpy ndarray
wcs = calexp.getWcs()                      # Rubin GWCS object

# Convert pixel to sky
from lsst.geom import Point2D
sky = wcs.pixelToSky(Point2D(512, 512))
print(f"RA={sky.getRa().asDegrees():.5f}, Dec={sky.getDec().asDegrees():.5f}")

# Query registry for available exposures
datasets = list(butler.registry.queryDatasets(
    "calexp",
    where="visit IN (192350, 192351) AND band = 'r'",
))
```

---

## §4 — LSDB for Rubin-Scale Catalogs

Use LSDB when working with the full Rubin/LSST object or source catalog
(billions of rows) outside the RSP.

```python
import lsdb

# DP0.2 object catalog (if converted to HATS — check data.lsdb.io)
cat = lsdb.read_hats("s3://stpubdata/hats/catalogs/dp02_objects/",
                      storage_options={"anon": True})

result = (
    cat
    .cone_search(ra=62.0, dec=-37.0, radius_arcsec=3600.0)
    .query("mag_r < 25 and detect_isPrimary == 1")
    [["objectId", "coord_ra", "coord_dec", "mag_g", "mag_r", "mag_i"]]
    .compute()
)
```

See `lsdb` skill for full LSDB patterns.

---

## §5 — Gaia Selection Functions (gaiaunlimited)

Use alongside Gaia catalogs to model completeness and selection effects.

```python
# pip install gaiaunlimited
from gaiaunlimited.selectionfunctions import DR3SelectionFunctionTCG
import numpy as np

sf = DR3SelectionFunctionTCG()

# Probability of a source being in Gaia DR3 given G magnitude + sky position
import astropy.units as u
from astropy.coordinates import SkyCoord

coords = SkyCoord(ra=62.0*u.deg, dec=-37.0*u.deg)
g_mag = np.array([18.0, 19.5, 20.5, 21.0])

prob = sf(coords, g_mag)
print(f"Selection probability at G=20.5: {prob[2]:.3f}")
```

---

## §6 — HEALPix / MOC Tools (cds-healpix)

Python bindings to the CDS HEALPix Rust library — faster than healpy for some operations.

```python
# pip install cdshealpix
from cdshealpix import healpix_to_lonlat, lonlat_to_healpix
import numpy as np

# Convert sky position to HEALPix index
ra = np.array([62.0, 150.0])
dec = np.array([-37.0, 2.2])
ipix = lonlat_to_healpix(np.deg2rad(ra), np.deg2rad(dec), depth=7)

# Reverse: HEALPix index to sky center
lon, lat = healpix_to_lonlat(ipix, depth=7)
print(np.rad2deg(lon), np.rad2deg(lat))
```

For MOC operations, prefer `mocpy`:

```python
from mocpy import MOC
moc = MOC.from_cone(lon=62.0*u.deg, lat=-37.0*u.deg,
                    radius=1.0*u.deg, max_depth=10)
print(f"MOC area: {moc.sky_fraction * 4 * np.pi * (180/np.pi)**2:.2f} sq deg")
```

---

## §7 — Fornax Cloud-Native Archive Access

The NASA Fornax project demonstrates cloud-native multi-archive workflows.
Demo notebooks at: https://github.com/nasa-fornax/fornax-demo-notebooks

Key patterns from Fornax notebooks:

```python
# Cloud-native cross-archive workflow
# 1. Discover services via VO registry
import pyvo as vo

services = vo.registry.search(
    keywords=["LSST", "Rubin"],
    servicetype="tap",
)

# 2. Query HATS catalog (LSDB) for spatial filter
import lsdb
cat = lsdb.read_hats("s3://stpubdata/hats/catalogs/gaia_dr3/",
                      storage_options={"anon": True})
cone = cat.cone_search(ra=62.0, dec=-37.0, radius_arcsec=7200.0).compute()

# 3. Cross-match with IRSA catalog via pyvo
irsa_tap = vo.dal.TAPService("https://irsa.ipac.caltech.edu/TAP")
irsa_result = irsa_tap.run_sync("""
    SELECT ra, dec, j_m, h_m, k_m
    FROM fp_psc
    WHERE CONTAINS(POINT('ICRS', ra, dec), CIRCLE('ICRS', 62.0, -37.0, 2.0)) = 1
""").to_table()

# 4. Match using astropy SkyCoord
from astropy.coordinates import SkyCoord
import astropy.units as u

c1 = SkyCoord(cone["ra"], cone["dec"], unit="deg")
c2 = SkyCoord(irsa_result["ra"], irsa_result["dec"], unit="deg")
idx, sep, _ = c1.match_to_catalog_sky(c2)
matched = cone[sep < 1 * u.arcsec]
```

---

## References

- Rubin RSP docs: https://nb.lsst.io
- DP0.2 schema: https://dm.lsst.org/sdm_schemas/browser/dp02_dc2.html
- daf_butler docs: https://pipelines.lsst.io/modules/lsst.daf.butler
- vo-cutouts (SODA): https://github.com/lsst-sqre/vo-cutouts
- lsst-tap-service: https://github.com/lsst-sqre/lsst-tap-service
- gaiaunlimited: https://github.com/gaia-unlimited/gaiaunlimited
- Fornax demo notebooks: https://github.com/nasa-fornax/fornax-demo-notebooks
- cdshealpix (Python): https://cds-astro.github.io/cds-healpix-python
