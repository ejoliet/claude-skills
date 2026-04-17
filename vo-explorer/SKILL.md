---
name: vo-explorer
description: >
  Virtual Observatory (VO) exploration skill for Emmanuel at IPAC Caltech. Apply
  whenever a task involves discovering or querying IVOA-compliant services, writing
  ADQL, using pyvo or astroquery for VO protocols (TAP, SIA, SCS, SSA, MOC, HiPS,
  ObsCore), VO registry lookups, multi-archive cross-matching, or building VO-aware
  Python pipelines. Trigger for: "VO service", "TAP query", "cone search", "ADQL",
  "IVOA", "registry search", "ObsCore", "VOTable", "HiPS", "MOC", "Aladin", service
  discovery, multi-catalog cross-match, or any pyvo/astroquery VO workflow. Do NOT
  trigger for non-VO archive queries or pure HTTP REST API calls to non-IVOA services.
---

# VO Explorer Skill

Patterns and reference for discovering, querying, and integrating IVOA-compliant
Virtual Observatory services. Covers service discovery, ADQL authoring, result
handling, and multi-archive workflows at IPAC Caltech scale.

---

## IVOA Protocol Reference

| Protocol | Class | Use case |
|----------|-------|----------|
| TAP | `pyvo.dal.TAPService` | ADQL table queries; catalog cross-matches; ObsCore |
| SIA v1/v2 | `pyvo.dal.SIAService` / `SIA2Service` | Image footprint discovery |
| SCS | `pyvo.dal.SCSService` | Simple cone search (position + radius) |
| SSA | `pyvo.dal.SSAService` | Spectral data access |
| MOC | `mocpy.MOC` | Multi-Order Coverage maps for spatial overlap |
| HiPS | Aladin Lite / `hips2fits` | Progressive sky surveys |
| ObsCore | TAP table `ivoa.obscore` | Unified obs metadata across archives |
| DataLink | `pyvo.dal.adhoc.DatalinkResults` | Product navigation from a row result |

---

## Service Discovery

### Registry Search (preferred over hardcoded URLs)

```python
import pyvo as vo

# Find all TAP services exposing ObsCore
services = vo.registry.search(datamodel="obscore", servicetype="tap")
for svc in services:
    print(svc.res_title, svc.access_url)

# Find SIA services for a specific waveband
sia_services = vo.registry.search(
    servicetype="sia",
    waveband="infrared",
    keywords=["IRSA"]
)

# Get a specific service object directly
tap = services[0].get_service("tap")
sia = services[0].get_service("sia")
```

### Known IVOA Service Endpoints

| Archive | Protocol | URL |
|---------|----------|-----|
| IRSA TAP | TAP | `https://irsa.ipac.caltech.edu/TAP` |
| MAST TAP | TAP | `https://mast.stsci.edu/tap/sync` |
| VizieR TAP | TAP | `https://tapvizier.cds.unistra.fr/TAPVizieR/tap` |
| NED TAP | TAP | `https://ned.ipac.caltech.edu/tap` |
| CDS SIMBAD | TAP | `https://simbad.u-strasbg.fr/simbad/tap/sync` |
| ESA Gaia | TAP | `https://gea.esac.esa.int/tap-server/tap` |
| IRSA SIA | SIA | `https://irsa.ipac.caltech.edu/SIA` |
| MAST SIA | SIA | `https://mast.stsci.edu/sia/v2` |

---

## TAP / ADQL Patterns

### Synchronous Query

```python
import pyvo as vo
from astropy.coordinates import SkyCoord

tap = vo.dal.TAPService("https://irsa.ipac.caltech.edu/TAP")

result = tap.run_sync("""
    SELECT TOP 200
        designation, ra, dec, j_m, h_m, k_m,
        j_msigcom, h_msigcom, k_msigcom
    FROM fp_psc
    WHERE CONTAINS(
        POINT('ICRS', ra, dec),
        CIRCLE('ICRS', 202.484, 47.230, 0.1)
    ) = 1
      AND j_m IS NOT NULL
    ORDER BY j_m
""")
tbl = result.to_table()  # astropy Table
```

### Asynchronous TAP (large queries)

```python
with tap.submit_job("""
    SELECT source_id, ra, dec, pmra, pmdec, parallax
    FROM gaiadr3.gaia_source
    WHERE parallax > 10.0
    AND ruwe < 1.4
""") as job:
    job.run()
    job.wait(phases=["COMPLETED", "ERROR"], timeout=300)
    if job.phase == "COMPLETED":
        result = job.fetch_result()
        tbl = result.to_table()
    else:
        raise RuntimeError(f"TAP job failed: {job.url}")
```

### ObsCore Query

```python
tap = vo.dal.TAPService("https://irsa.ipac.caltech.edu/TAP")

obscore = tap.run_sync("""
    SELECT
        obs_id, obs_collection, dataproduct_type,
        s_ra, s_dec, s_fov, s_region,
        t_min, t_max, t_exptime,
        em_min, em_max,
        access_url, access_format
    FROM ivoa.obscore
    WHERE 1=CONTAINS(
        POINT('ICRS', s_ra, s_dec),
        CIRCLE('ICRS', 202.484, 47.230, 0.5)
    )
    AND dataproduct_type = 'image'
    AND em_min < 2.2e-6   -- K-band cutoff
""").to_table()
```

### Table Discovery

```python
# List tables in a TAP service
tables = tap.tables
for name, tbl_meta in tables.items():
    print(name, tbl_meta.description)

# Columns for a specific table
cols = tables["fp_psc"].columns
for col in cols.values():
    print(col.name, col.ucd, col.unit)
```

### Upload TAP (cross-match local catalog)

```python
from astropy.table import Table
import pyvo as vo

local = Table.read("my_sources.fits")
tap = vo.dal.TAPService("https://irsa.ipac.caltech.edu/TAP")

result = tap.run_sync("""
    SELECT u.id, p.designation, p.ra, p.dec, p.j_m,
           DISTANCE(POINT('ICRS', u.ra, u.dec),
                    POINT('ICRS', p.ra, p.dec)) AS sep_deg
    FROM TAP_UPLOAD.sources AS u
    JOIN fp_psc AS p
      ON 1=CONTAINS(POINT('ICRS', p.ra, p.dec),
                    CIRCLE('ICRS', u.ra, u.dec, 3.0/3600.0))
""", uploads={"sources": local})
matched = result.to_table()
```

---

## SIA Image Search

```python
import pyvo as vo
from astropy.coordinates import SkyCoord
from astropy.units import Quantity

# SIA v1
sia = vo.dal.SIAService("https://irsa.ipac.caltech.edu/SIA")
coord = SkyCoord.from_name("NGC 1068")
images = sia.search(pos=coord, size=Quantity(0.2, "deg"))
for row in images:
    print(row.getdataurl(), row["bandpass_id"])

# SIA v2 (multi-pos, multi-band)
sia2 = vo.dal.SIA2Service("https://mast.stsci.edu/sia/v2")
results = sia2.search(
    pos=[(202.484, 47.230, 0.1)],  # ra, dec, radius in deg
    band=(1e-6, 2.5e-6),           # wavelength range in meters
    maxrec=50
)
```

---

## Cone Search (SCS)

```python
scs = vo.dal.SCSService("https://irsa.ipac.caltech.edu/cgi-bin/Gator/nph-svo?catalog=fp_psc&")
result = scs.search(pos=SkyCoord(202.484, 47.230, unit="deg"), radius=2/60.0)
tbl = result.to_table()
```

---

## MOC — Multi-Order Coverage

```python
from mocpy import MOC
from astropy.coordinates import SkyCoord
import astropy.units as u

# Load MOC from CDS
moc = MOC.from_vizier_table("II/246/out")  # 2MASS PSC footprint

# Check if positions are covered
coords = SkyCoord([10.0, 20.0], [30.0, 40.0], unit="deg")
covered = moc.contains(coords.ra, coords.dec)

# Intersection of two MOCs
moc_a = MOC.from_fits("survey_a.fits")
moc_b = MOC.from_fits("survey_b.fits")
overlap = moc_a.intersection(moc_b)
sky_fraction = overlap.sky_fraction  # 0.0–1.0

# Create MOC from cone
moc_cone = MOC.from_cone(
    lon=202.484 * u.deg, lat=47.230 * u.deg,
    radius=1.0 * u.deg, max_depth=10
)
```

---

## DataLink — Product Navigation

```python
# After a TAP/SIA query, follow DataLink to get actual data products
tap = vo.dal.TAPService("https://mast.stsci.edu/tap/sync")
result = tap.run_sync("""
    SELECT obs_id, dataURL, access_url
    FROM dbo.observations
    WHERE obs_collection='JWST'
    AND t_min > 59945.0
    LIMIT 10
""")

for row in result:
    dl = row.getdataset()     # fetches DataLink document
    for link in dl:
        if link.semantics == "#this":
            link.cachedataset(filename=f"{row['obs_id']}.fits")
```

---

## VOTable I/O

```python
from astropy.io.votable import parse, writeto
from astropy.table import Table

# Read VOTable
votable = parse("result.vot")
tbl = votable.get_first_table().to_table(use_names_over_ids=True)

# Write astropy Table as VOTable
writeto(Table.from_pandas(df), "output.vot")

# Round-trip via pyvo result
result = tap.run_sync("SELECT * FROM fp_psc LIMIT 10")
result.votable.to_xml("output.vot")        # raw VOTable XML
tbl = result.to_table()                    # astropy Table
df  = result.to_table().to_pandas()        # pandas DataFrame
```

---

## Multi-Archive Cross-Match Pattern

When cross-matching across IRSA, MAST, and external archives:

```python
import pyvo as vo
from astropy.table import Table, join
from astropy.coordinates import SkyCoord, match_coordinates_sky
import astropy.units as u

# 1. Query archive A via TAP with upload
irsa_tap = vo.dal.TAPService("https://irsa.ipac.caltech.edu/TAP")
mast_tap  = vo.dal.TAPService("https://mast.stsci.edu/tap/sync")

source_list = Table.read("targets.fits")

irsa_matches = irsa_tap.run_sync("""
    SELECT u.id, p.designation, p.j_m, p.h_m, p.k_m,
           DISTANCE(POINT('ICRS',u.ra,u.dec), POINT('ICRS',p.ra,p.dec)) AS sep
    FROM TAP_UPLOAD.tgts AS u
    JOIN fp_psc AS p
      ON 1=CONTAINS(POINT('ICRS',p.ra,p.dec), CIRCLE('ICRS',u.ra,u.dec,0.00083))
""", uploads={"tgts": source_list}).to_table()

# 2. Sky-coordinate match for local cross-match step
catalog_coords = SkyCoord(irsa_matches["ra"], irsa_matches["dec"], unit="deg")
target_coords  = SkyCoord(source_list["ra"], source_list["dec"], unit="deg")
idx, sep, _ = match_coordinates_sky(target_coords, catalog_coords)
good = sep < 1.0 * u.arcsec
```

---

## VO Service Quality Checks

Before relying on a service in production:

```python
# Check availability
try:
    tap.run_sync("SELECT TOP 1 * FROM ivoa.obscore")
    status = "ok"
except Exception as e:
    status = f"unavailable: {e}"

# Validate ObsCore compliance
tables = tap.tables
assert "ivoa.obscore" in tables, "Service is not ObsCore-compliant"
required_cols = {"obs_id","s_ra","s_dec","dataproduct_type","access_url"}
present = {c for c in tables["ivoa.obscore"].columns}
missing = required_cols - present
assert not missing, f"Non-compliant ObsCore: missing {missing}"
```

---

## ADQL Cheat Sheet

```sql
-- Spatial cone
CONTAINS(POINT('ICRS', ra, dec), CIRCLE('ICRS', 202.4, 47.2, 0.5)) = 1

-- Spatial box
CONTAINS(POINT('ICRS', ra, dec),
         BOX('ICRS', 202.4, 47.2, 1.0, 0.5)) = 1   -- ra_center, dec_center, width, height

-- Great-circle distance
DISTANCE(POINT('ICRS', ra1, dec1), POINT('ICRS', ra2, dec2)) < 0.01

-- Wavelength range (SI meters in ObsCore)
em_min < 2.2e-6 AND em_max > 2.0e-6

-- Time range (MJD in ObsCore)
t_min > 59000.0 AND t_max < 60000.0

-- Cross-match join
JOIN catalog2 AS c
  ON 1=CONTAINS(POINT('ICRS',c.ra,c.dec), CIRCLE('ICRS',a.ra,a.dec,3/3600.0))

-- String filter (case-sensitive in most TAP)
obs_collection IN ('WISE', 'NEOWISE-R')
```

---

## Integration with Other Skills

- Co-triggers with `emmanuel-engineering` for any VO-based Python service or pipeline
- Co-triggers with `context7-docs-lookup` for live pyvo/astroquery/mocpy API docs
- Co-triggers with `roman-space-telescope` when querying Roman ObsCore or WFI image data via TAP/SIA
- Co-triggers with `emmanuel-markdown` when producing VO query runbooks or ADRs

---

## References

- `references/vo-service-catalog.md` — curated list of IVOA service endpoints by archive and waveband
- IVOA standards: https://www.ivoa.net/documents/
- pyvo docs: https://pyvo.readthedocs.io/
- IRSA TAP: https://irsa.ipac.caltech.edu/docs/program_interface/tap.html
- ADQL 2.1 standard: https://www.ivoa.net/documents/ADQL/
