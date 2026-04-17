---
name: roman-space-telescope
description: >
  Nancy Grace Roman Space Telescope skill for Emmanuel at IPAC Caltech. Apply
  whenever a task involves Roman mission data products, the WFI instrument, ASDF
  pipeline outputs, the SOC file exchange system, romancal pipeline operations,
  romanisim simulations, Roman IRSA/MAST data access, ObsCore queries for Roman
  observations, EKS deployment of Roman SOC services, or any Roman-specific science
  planning (HLWAS, HLTDS, GBTDS surveys). Trigger for: "roman", "roman wfi", "roman
  soc", "roman pipeline", "romancal", "romanisim", "roman asdf", "roman file exchange",
  "wfi filter", "roman survey", "roman calibration", "level 2 roman". Do NOT trigger
  for generic ASDF/FITS questions unrelated to Roman.
---

# Roman Space Telescope Skill

Reference guide for working with the Nancy Grace Roman Space Telescope at IPAC Caltech.
Covers the SOC file exchange system, WFI data products, pipeline operations, simulations,
and archive access patterns.

---

## Mission Overview

| Property | Value |
|----------|-------|
| Launch | Late 2026 (current estimate) |
| Orbit | L2 (same as JWST) |
| Primary instrument | Wide Field Instrument (WFI) |
| Secondary instrument | Coronagraph Instrument (CGI) |
| Field of view | ~0.28 deg² (18 H4RG-10 detectors) |
| Pixel scale | 0.11 arcsec/pixel |
| Wavelength range | 0.48–2.3 µm |
| Survey programs | HLWAS, HLTDS, GBTDS |

### WFI Filters

| Filter | Pivot λ (µm) | Notes |
|--------|-------------|-------|
| F062 | 0.62 | Blue optical |
| F087 | 0.87 | z-band |
| F106 | 1.06 | Y-band |
| F129 | 1.29 | J-band |
| F158 | 1.58 | H-band |
| F184 | 1.84 | H+J red |
| F213 | 2.13 | K-band |
| W146 | 1.46 | Wide filter (survey speed) |
| Prism | 0.75–1.8 | Slitless low-res spectroscopy |
| Grism | 1.0–1.93 | Slitless grism spectroscopy |

### Core Surveys

| Survey | Abbrev | Science |
|--------|--------|---------|
| High Latitude Wide Area Survey | HLWAS | Weak lensing, galaxy clustering |
| High Latitude Time Domain Survey | HLTDS | SNe Ia, transients |
| Galactic Bulge Time Domain Survey | GBTDS | Microlensing, exoplanets |

---

## Data Product Levels

| Level | Name | Content | File format |
|-------|------|---------|-------------|
| 0 | Raw telemetry | Detector reads, uncalibrated | ASDF |
| 1 | Ramp | Up-the-ramp samples, basic corrections | ASDF |
| 2 | Rate image | CR-rejected, flat-fielded, calibrated | ASDF |
| 2.5 | Resampled | WCS-aligned to common grid | ASDF |
| 3 | Mosaic / catalog | Co-added images, source catalogs | ASDF + FITS |

---

## ASDF Roman Data Access

Roman pipeline products use the **ASDF** format with `roman_datamodels` wrappers.
Always use `roman_datamodels` rather than raw `asdf` for structured access.

### Reading Roman Level 2 Files

```python
import roman_datamodels as rdm
import asdf
from astropy.wcs import WCS
import numpy as np

# Preferred: roman_datamodels wrapper
with rdm.open("r0000101001001001001_01101_0001_WFI01_cal.asdf") as dm:
    sci   = dm.data                    # SCI array (DN/s)
    err   = dm.err                     # ERR array
    dq    = dm.dq                      # DQ bitfield array
    wcs   = dm.meta.wcs                # GWCS object
    filt  = dm.meta.instrument.filter  # e.g. "F129"
    exp   = dm.meta.exposure.exposure_time  # seconds

    # Convert pixel to sky (GWCS)
    sky = wcs(1024, 1024)   # (ra, dec) at pixel (x, y)

# Raw ASDF access (when roman_datamodels unavailable)
with asdf.open("file.asdf") as af:
    data = af["roman"]["data"]
    meta = af["roman"]["meta"]
    filter_name = meta["instrument"]["filter"]
```

### Key ASDF Tree Paths (Level 2)

```
roman/
  data                    # science array [y, x] in DN/s
  err                     # uncertainty array
  dq                      # data quality flags (bitmask)
  var_poisson             # Poisson variance
  var_rnoise              # readnoise variance
  meta/
    instrument/
      detector            # e.g. "WFI01" ... "WFI18"
      filter              # filter name string
      optical_element     # same as filter for WFI
    exposure/
      exposure_time       # effective exposure time (s)
      start_time          # MJD
    wcs                   # GWCS object
    photometry/
      conversion_megajansky  # MJy/sr per DN/s
    target/
      ra, dec             # boresight
```

---

## SOC File Exchange System (IPAC)

Emmanuel's active system: monitors Roman SOC file deliveries and tracks them in Aurora PG.

### Architecture

```
SOC delivery bucket (S3)
    ↓  S3 event → SQS
Roman SOC Monitor (FastMCP + EKS)
    ↓
Aurora PostgreSQL: soc_files_table
    ↓
GraphQL API layer
    ↓
Jenkins CI (validation pipelines)
```

### `soc_files_table` Schema

```sql
CREATE TABLE soc_files (
    id              SERIAL PRIMARY KEY,
    s3_key          TEXT        NOT NULL,
    delivery_id     TEXT        NOT NULL,
    product_level   SMALLINT,           -- 0, 1, 2, 2.5, 3
    detector        TEXT,               -- WFI01 … WFI18
    filter          TEXT,               -- F062 … Grism
    exposure_id     TEXT,
    visit_id        TEXT,
    file_size_bytes BIGINT,
    checksum_md5    TEXT,
    received_at     TIMESTAMPTZ NOT NULL DEFAULT now(),
    validated_at    TIMESTAMPTZ,
    status          TEXT        NOT NULL DEFAULT 'received',  -- received|validated|failed
    error_message   TEXT
);

CREATE INDEX idx_soc_files_delivery ON soc_files(delivery_id);
CREATE INDEX idx_soc_files_status   ON soc_files(status);
CREATE INDEX idx_soc_files_received ON soc_files(received_at DESC);
```

### FastMCP Tool Patterns for SOC Monitor

```python
from fastmcp import FastMCP
from pydantic import BaseModel
from typing import Optional
import asyncpg

mcp = FastMCP("roman-soc-monitor")

class FileQuery(BaseModel):
    delivery_id: Optional[str] = None
    status: Optional[str] = None
    detector: Optional[str] = None
    limit: int = 100

@mcp.tool()
async def list_soc_files(query: FileQuery) -> dict:
    """List Roman SOC file deliveries matching the given filters."""
    async with get_pool().acquire() as conn:
        rows = await conn.fetch("""
            SELECT id, s3_key, delivery_id, product_level,
                   detector, filter, status, received_at
            FROM soc_files
            WHERE ($1::text IS NULL OR delivery_id = $1)
              AND ($2::text IS NULL OR status = $2)
              AND ($3::text IS NULL OR detector = $3)
            ORDER BY received_at DESC
            LIMIT $4
        """, query.delivery_id, query.status, query.detector, query.limit)
        return {"files": [dict(r) for r in rows], "count": len(rows)}

@mcp.tool()
async def get_delivery_summary(delivery_id: str) -> dict:
    """Summarize a SOC delivery: counts by level, detector, status."""
    async with get_pool().acquire() as conn:
        rows = await conn.fetch("""
            SELECT product_level, detector, status, COUNT(*) AS n
            FROM soc_files
            WHERE delivery_id = $1
            GROUP BY product_level, detector, status
            ORDER BY product_level, detector
        """, delivery_id)
        return {"delivery_id": delivery_id, "breakdown": [dict(r) for r in rows]}
```

---

## romancal Pipeline Operations

```python
# Run a single exposure through the WFI Image pipeline (Level 1 → Level 2)
from romancal.pipeline import ExposurePipeline

result = ExposurePipeline.call(
    "r0000101001001001001_01101_0001_WFI01_uncal.asdf",
    save_results=True,
    output_dir="./output/",
    steps={
        "dq_init":        {"skip": False},
        "saturation":     {"skip": False},
        "refpix":         {"skip": False},
        "linearity":      {"skip": False},
        "dark_current":   {"skip": False},
        "jump":           {"rejection_threshold": 4.0},
        "ramp_fit":       {"skip": False},
        "assign_wcs":     {"skip": False},
        "flat_field":     {"skip": False},
        "photom":         {"skip": False},
    }
)

# Access result as roman_datamodels object
dm = result  # ExposurePipeline returns the datamodel directly
```

### Reference Files

```python
import crds
import os

# CRDS (Calibration Reference Data System)
os.environ["CRDS_PATH"] = "/data/crds_cache"
os.environ["CRDS_SERVER_URL"] = "https://roman-crds.stsci.edu"

# Get best reference file for a given exposure
from crds import getreferences
refs = getreferences(
    parameters={
        "roman.meta.instrument.filter": "F129",
        "roman.meta.instrument.detector": "WFI01",
        "roman.meta.exposure.start_time": "2027-01-15T00:00:00.0",
    },
    reftypes=["flat", "dark", "linearity"],
    observatory="roman"
)
```

---

## romanisim Simulation

```python
# Generate a simulated Roman WFI exposure
from romanisim import image, bandpass, catalog

# Build a simple point-source catalog
cat = catalog.make_stars_catalog(
    ra=202.484, dec=47.230, radius=0.2,  # degrees
    filter_name="F129",
    ab_mag_range=(18, 25)
)

# Simulate Level 1 exposure
result = image.make_l1(
    catalog=cat,
    filter_name="F129",
    detector="WFI07",
    date="2027-06-15",
    ra=202.484,
    dec=47.230,
    pa_aper=0.0,
    exptime=140.0,
    level=2,           # generate L2 directly
    seed=42
)
result.save("simulated_l2.asdf")
```

---

## Archive Access — MAST and IRSA

### MAST (primary Roman archive, STScI)

```python
from astroquery.mast import Observations

# Roman observations near a coordinate
obs = Observations.query_criteria(
    obs_collection="ROMAN",
    coordinates="202.484 +47.230",
    radius="0.1 deg",
    dataproduct_type="image"
)

# Download calibrated L2 files
products = Observations.get_product_list(obs)
l2_files = products[products["productType"] == "SCIENCE"]
manifest = Observations.download_products(l2_files)
```

### IRSA (Roman HLWAS / community access)

```python
from astroquery.ipac.irsa import Irsa

# Once Roman data is in IRSA
tbl = Irsa.query_tap("""
    SELECT obs_id, s_ra, s_dec, t_exptime, filter, access_url
    FROM ivoa.obscore
    WHERE obs_collection = 'ROMAN'
      AND 1=CONTAINS(
          POINT('ICRS', s_ra, s_dec),
          CIRCLE('ICRS', 202.484, 47.230, 0.5)
      )
""").to_table()
```

---

## Coordinate & WCS Patterns

Roman uses **GWCS** (generalized WCS) not the traditional FITS WCS.

```python
import roman_datamodels as rdm
from astropy.coordinates import SkyCoord
import astropy.units as u

with rdm.open("file_cal.asdf") as dm:
    gwcs = dm.meta.wcs

    # Pixel → Sky
    ra, dec = gwcs(512.0, 512.0)
    coord = SkyCoord(ra, dec, unit="deg")

    # Sky → Pixel
    x, y = gwcs.invert(coord.ra.deg, coord.dec.deg)

    # Bounding box of the exposure
    bb = gwcs.bounding_box
    print(f"Pixel extent: {bb}")

    # Full WCS footprint
    from astropy.wcs.utils import skycoord_to_pixel
    corners = gwcs([[0,4088,4088,0],[0,0,4096,4096]])
```

---

## File Naming Convention

Roman file names follow this pattern:

```
r{program_id}{obs_id}{visit_id}{exposure_id}_{sca}_{seq}_{detector}_{suffix}.asdf

Example:
r0000101001001001001_01101_0001_WFI01_cal.asdf
│         │          │     │    │     └── suffix: uncal|ramp|cal|i2d|cat
│         │          │     │    └── detector: WFI01–WFI18
│         │          │     └── sequence: 0001
│         │          └── visit: 01101
│         └── obs/visit/exposure chain
└── program id
```

Suffixes:

| Suffix | Level | Description |
|--------|-------|-------------|
| `uncal` | 0/1 | Raw ramp, uncalibrated |
| `ramp` | 1 | Intermediate ramp-fit |
| `cal` | 2 | Calibrated rate image |
| `i2d` | 2.5/3 | Resampled/mosaicked image |
| `cat` | 3 | Source catalog |

---

## Environment & Dependencies

```bash
# Core Roman Python stack
pip install \
    roman-datamodels \
    romancal \
    romanisim \
    asdf \
    asdf-astropy \
    astropy \
    pyvo \
    astroquery \
    gwcs \
    crds \
    webbpsf    # PSF modeling (shared JWST/Roman)
```

### Docker for Roman work

```dockerfile
FROM python:3.11-slim AS runtime
WORKDIR /app
RUN pip install --no-cache-dir \
    roman-datamodels romancal romanisim \
    asdf asdf-astropy astropy \
    pyvo astroquery gwcs crds
ENV CRDS_PATH=/data/crds_cache
ENV CRDS_SERVER_URL=https://roman-crds.stsci.edu
```

---

## Key Resources

| Resource | URL |
|----------|-----|
| Roman @ IPAC | https://roman.ipac.caltech.edu |
| Roman @ STScI | https://roman.gsfc.nasa.gov / https://www.stsci.edu/roman |
| roman_datamodels | https://roman-datamodels.readthedocs.io |
| romancal pipeline | https://roman-pipeline.readthedocs.io |
| romanisim | https://romanisim.readthedocs.io |
| CRDS Roman | https://roman-crds.stsci.edu |
| ASDF Standard | https://asdf-standard.readthedocs.io |
| WFI filters (STScI) | https://www.stsci.edu/roman/instrumentation/wfi |

---

## Integration with Other Skills

- Co-triggers with `emmanuel-engineering` for SOC monitor architecture, EKS deployments, Aurora PG schema, FastMCP tools
- Co-triggers with `vo-explorer` for Roman TAP/ObsCore/SIA queries
- Co-triggers with `context7-docs-lookup` for live roman_datamodels, romancal, asdf API docs
- Co-triggers with `emmanuel-markdown` for Roman ADRs, runbooks, pipeline design docs
