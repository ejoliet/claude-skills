---
name: roman-space-telescope
description: >
  Nancy Grace Roman Space Telescope skill for Emmanuel at IPAC Caltech. Apply
  whenever a task involves Roman mission data products, the WFI instrument, ASDF
  pipeline outputs, the SOC file exchange system, romancal pipeline operations,
  romanisim/STPSF/Pandeia/STIPS simulations, Roman IRSA/MAST data access, ObsCore
  queries for Roman observations, EKS deployment of Roman SOC services, Roman
  science planning (HLWAS, HLTDS, GBTDS, GPS surveys), roman_photoz, rad (Roman
  Attribute Dictionary), PySIAF aperture files, or Roman Research Nexus workflows.
  Trigger for: "roman", "roman wfi", "roman soc", "roman pipeline", "romancal",
  "romanisim", "roman asdf", "roman file exchange", "wfi filter", "roman survey",
  "roman calibration", "level 2 roman", "pandeia roman", "stpsf roman", "stips roman",
  "roman_photoz", "rad roman", "roman nexus", "roman notebooks". Do NOT trigger for
  generic ASDF/FITS questions unrelated to Roman.
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
| Galactic Plane Survey | GPS | Stellar populations, variables, star formation |

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

## SOC Monitor — Operational Runbook

### Health check queries (run first when something looks wrong)

```sql
-- Overall status breakdown (last 24h)
SELECT status, COUNT(*) AS n
FROM soc_files
WHERE received_at > NOW() - INTERVAL '24 hours'
GROUP BY status ORDER BY n DESC;

-- Stalled files: received but not validated after 30 min
SELECT id, s3_key, delivery_id, received_at,
       EXTRACT(EPOCH FROM (NOW() - received_at))/60 AS minutes_waiting
FROM soc_files
WHERE status = 'received'
  AND received_at < NOW() - INTERVAL '30 minutes'
ORDER BY received_at;

-- Latest delivery summary
SELECT delivery_id, product_level, COUNT(*) AS n_files,
       SUM(CASE WHEN status='validated' THEN 1 ELSE 0 END) AS validated,
       SUM(CASE WHEN status='failed'    THEN 1 ELSE 0 END) AS failed
FROM soc_files
WHERE delivery_id = (SELECT delivery_id FROM soc_files ORDER BY received_at DESC LIMIT 1)
GROUP BY delivery_id, product_level ORDER BY product_level;

-- Recent errors
SELECT id, s3_key, error_message, received_at
FROM soc_files
WHERE status = 'failed'
ORDER BY received_at DESC LIMIT 20;
```

### Triage decision tree

```
Files not arriving?
├─ Check SQS queue depth (CloudWatch → SQS → soc-events)
│   └─ Queue depth > 0 → messages arriving but processor not consuming
│       → Check FastMCP pod logs: kubectl logs -n roman-soc -l app=soc-monitor
│   └─ Queue depth = 0 → no S3 events firing
│       → Check S3 bucket notification config: aws s3api get-bucket-notification-configuration
│
Files arriving but stuck in 'received'?
├─ Check Aurora connectivity from pod:
│   kubectl exec -it <pod> -n roman-soc -- python -c "import asyncpg; ..."
├─ Check Aurora CPU/connections (CloudWatch → RDS)
│   └─ High connections → check for connection pool leak; restart pod
│
Files failing validation?
├─ Check error_message column for pattern
│   └─ Checksum mismatch → re-delivery needed from SOC
│   └─ Missing required keys → ASDF schema change; check romancal version
│   └─ S3 access denied → check pod IAM role / IRSA binding
│
FastMCP pod crash-looping?
├─ kubectl describe pod <pod> -n roman-soc
├─ Check OOMKilled → increase memory limit in Helm values
└─ Check DB secret rotation → update secret in AWS Secrets Manager + restart pod
```

### Recovery procedures

```bash
# Re-queue a specific failed file for reprocessing
kubectl exec -it <soc-monitor-pod> -n roman-soc -- python -c "
import asyncio, asyncpg
async def requeue(file_id):
    conn = await asyncpg.connect(dsn='...')
    await conn.execute(\"UPDATE soc_files SET status='received', error_message=NULL WHERE id=\$1\", file_id)
asyncio.run(requeue($FILE_ID))
"

# Force-restart the SOC monitor pod
kubectl rollout restart deployment/soc-monitor -n roman-soc

# Check IRSA role binding (if S3 access denied)
kubectl describe sa soc-monitor -n roman-soc
aws iam get-role --role-name roman-soc-monitor-role

# Tail live pod logs
kubectl logs -f -l app=soc-monitor -n roman-soc --tail=100
```

### SNS / alerting wiring

```python
# Expected CloudWatch alarms (defined in CloudFormation)
alarms = {
    "SOCQueueDepthHigh":     "SQS ApproximateNumberOfMessagesVisible > 100 for 5m",
    "SOCValidationFailRate":  "% failed files > 5% over 15m window (custom metric)",
    "SOCAuroraConnHigh":      "DatabaseConnections > 80% of max_connections",
    "SOCPodRestartHigh":      "kube_pod_container_status_restarts_total > 2 in 10m",
}
# All route to SNS: roman-soc-alerts → email + optional Slack
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
| Roman docs portal | https://roman-docs.stsci.edu |
| Roman @ IPAC | https://roman.ipac.caltech.edu |
| Roman @ STScI | https://www.stsci.edu/roman |
| roman_datamodels | https://roman-datamodels.readthedocs.io |
| RAD schemas | https://rad.readthedocs.io |
| romancal pipeline | https://roman-pipeline.readthedocs.io |
| romancal notebooks | https://github.com/spacetelescope/romancal-notebooks |
| roman_notebooks | https://github.com/spacetelescope/roman_notebooks |
| roman-data-workshop | https://github.com/spacetelescope/roman-data-workshop |
| romanisim | https://romanisim.readthedocs.io |
| STPSF | https://stpsf.readthedocs.io |
| Pandeia ETC | https://outerspace.stsci.edu/display/PEN |
| STIPS | https://stips.readthedocs.io |
| PySIAF | https://pysiaf.readthedocs.io |
| roman_photoz | https://github.com/spacetelescope/roman_photoz |
| Roman Research Nexus | https://nexus.stsci.edu |
| CRDS Roman | https://roman-crds.stsci.edu |
| ASDF Standard | https://asdf-standard.readthedocs.io |
| WFI instrument page | https://roman-docs.stsci.edu/display/RDox/Wide+Field+Instrument |
| soc_roman_tools (WFI imaging) | https://github.com/spacetelescope/soc_roman_tools |

---

## RAD — Roman Attribute Dictionary

`rad` defines the canonical ASDF schema and shared metadata attributes used
across all Roman pipeline software. It is the authoritative source for Roman
datamodel structure.

```bash
pip install rad
```

```python
# Inspect available schemas
import rad
import importlib.resources as pkg

# Schemas live in rad/resources/schemas/roman_datamodels/
# Key schemas: wfi_image, wfi_mode, wfi_science_raw, common, etc.

# Validate a Roman ASDF file against the rad schema
import asdf

with asdf.open("r0000101_cal.asdf", custom_schema="rad:roman_datamodels/wfi_image-1.0.0") as af:
    af.validate()   # raises asdf.ValidationError if non-conformant

# Inspect what fields are defined for WFI image
from rad.resources import schemas
schema = schemas.load_schema("roman_datamodels/wfi_image-1.0.0")
print(schema["properties"].keys())
```

- **ejoliet fork**: `ejoliet/rad` — used for SOC schema validation work
- Upstream: https://github.com/spacetelescope/rad
- Docs: https://rad.readthedocs.io

---

## Simulation Ecosystem

### Pandeia — Exposure Time Calculator (ETC)

```python
# pip install pandeia.engine
# Requires synphot data + Pandeia reference data
import os
os.environ["pandeia_refdata"] = "/data/pandeia_refdata"

from pandeia.engine.perform_calculation import perform_calculation

calc = {
    "telescope": "roman",
    "instrument": {
        "instrument": "wfi",
        "filter": "f129",
        "mode": "imaging"
    },
    "scene": [{
        "position": {"x_offset": 0, "y_offset": 0, "unit": "arcsec"},
        "shape": {"geometry": "point"},
        "spectrum": {
            "normalization": {"type": "abmag", "norm_flux": 25.0, "norm_fluxunit": "abmag",
                              "bandpass": "roman,wfi,f129"},
            "sed": {"sed_type": "flat", "unit": "fnu"}
        }
    }],
    "strategy": {
        "detector_readout": {"ramp_param": {"ngroups": 6, "nints": 1}},
    }
}

result = perform_calculation(calc)
print(f"SNR = {result['scalar']['sn']:.1f}")
print(f"Exposure time = {result['scalar']['exposure_time']:.1f} s")
```

- Pandeia docs: https://outerspace.stsci.edu/display/PEN
- Reference data: download from STScI box link in Pandeia docs

---

### STPSF — Space Telescope Point Spread Functions

STPSF (successor to WebbPSF) generates optical PSFs for Roman WFI detectors.

```python
# pip install stpsf
# Requires: export STPSF_PATH=/data/stpsf_data
import stpsf

# Roman WFI PSF
wfi = stpsf.RomanWFI()
wfi.filter = "F129"
wfi.detector = "SCA01"           # WFI01 in stpsf naming
wfi.detector_position = (2048, 2048)   # pixel position on SCA

# Compute PSF
psf = wfi.calc_psf(
    fov_arcsec=5.0,
    oversample=4,
    add_distortion=True,
)

# Access PSF image
import matplotlib.pyplot as plt
psf_array = psf[0].data            # oversampled PSF
plt.imshow(psf_array, origin="lower", norm="log")

# PSF FWHM estimate
from astropy.modeling.models import Gaussian2D
from astropy.modeling.fitting import LevMarLSQFitter
# ... (fit Gaussian to measure FWHM)
```

- STPSF docs: https://stpsf.readthedocs.io
- Data download: `stpsf.utils.download_stpsf_data()`

---

### STIPS — Scene-Level Image Simulator

STIPS generates realistic scenes with many sources (galaxies + stars) for
survey-level simulations. Complements romanisim (single-exposure level).

```python
# pip install stips
import stips
from stips.scene_module import SceneModule
from stips.observation_module import ObservationModule

obs_params = {
    "instrument": "WFI",
    "filters": ["F106"],
    "detectors": 1,
    "oversample": 1,
    "pupil_mask": "",
    "background": "avg",
    "observations_id": 1,
    "exptime": 300,
    "offsets": [{"id": 1, "delay": 0, "move": False, "ra": 0, "dec": 0, "pa": 0}]
}

obs = ObservationModule(obs_params, ra=150.0, dec=2.2, pa=0.0,
                        out_path="./output/", prefix="roman_sim")
obs.nextObservation()

src_file = obs.addCatalog("galaxy_catalog.fits")
obs.addError()
obs.finalize(mosaic=False)
```

- STIPS docs: https://stips.readthedocs.io
- GitHub: https://github.com/spacetelescope/STScI-STIPS

---

### PySIAF — Science Instrument Aperture Files

PySIAF provides access to Roman WFI aperture geometry, detector positions,
and pixel-to-sky transforms for pointing and dithering calculations.

```python
# pip install pysiaf
import pysiaf

# Load Roman WFI SIAF
siaf = pysiaf.Siaf("roman")

# List all apertures
for ap_name in siaf.apertures:
    print(ap_name)

# Get a specific SCA aperture
sca01 = siaf["ROMAN_WFI_SCA01_FULL"]

# Corner positions on sky (for a given V2V3 pointing)
corners_tel = sca01.corners("tel")   # V2, V3 in arcsec

# Pixel → sky (requires attitude)
v2ref, v3ref = sca01.V2Ref, sca01.V3Ref
print(f"SCA01 reference: V2={v2ref:.2f}, V3={v3ref:.2f} arcsec")
```

- PySIAF docs: https://pysiaf.readthedocs.io
- GitHub: https://github.com/spacetelescope/pysiaf

---

## roman_photoz — Photometric Redshifts

```bash
# ejoliet fork: ejoliet/roman_photoz
# upstream: https://github.com/spacetelescope/roman_photoz
pip install roman_photoz
```

```python
# roman_photoz wraps photo-z codes (EAZY, LePhare) for Roman band combinations
from roman_photoz import PhotozRunner

runner = PhotozRunner(
    catalog="roman_catalog.fits",
    filters=["F062", "F087", "F106", "F129", "F158", "F184", "W146"],
    method="eazy",              # or "lephare"
    output_dir="./photoz_output/",
)
runner.run()

results = runner.load_results()
# results["z_phot"] — best-fit photo-z
# results["z_phot_l68"], results["z_phot_u68"] — 68% credible interval
```

- GitHub: https://github.com/spacetelescope/roman_photoz
- Designed for HLWAS weak-lensing and galaxy clustering photo-z requirements

---

## Notebook Resources

### roman_notebooks

Official analysis notebooks for Roman data and tools, maintained by STScI.

```bash
git clone https://github.com/spacetelescope/roman_notebooks
cd roman_notebooks
pip install -r requirements.txt
jupyter lab
```

Key notebooks:
- `data_discovery/` — MAST archive query, exposure search
- `data_products/` — opening L1/L2 ASDF files, reading metadata
- `wcs/` — GWCS pixel↔sky transforms, WCS footprints
- `simulation/` — romanisim, Pandeia ETC examples
- `calibration/` — running romancal pipeline steps manually

---

### romancal-notebooks

Pipeline-focused notebooks. Run pipeline steps interactively, inspect
intermediate products, tune parameters.

```bash
git clone https://github.com/spacetelescope/romancal-notebooks
```

Key notebooks:
- Individual step notebooks (`dq_init`, `jump`, `ramp_fit`, etc.)
- Full L1→L2 pipeline walkthrough
- Custom step parameter tuning

---

### Roman Research Nexus

Cloud-based JupyterHub platform hosted by STScI for Roman data analysis.
Provides pre-installed Roman software stack + access to MAST data without downloading.

- URL: https://nexus.stsci.edu (requires STScI account)
- Pre-installed: `roman_datamodels`, `romancal`, `romanisim`, `stpsf`, `pandeia`, `jdaviz`
- Storage: persistent home directory; S3-backed MAST data access
- Use instead of local installs for interactive exploration of real Roman data

```python
# On Roman Nexus — MAST data access without download
from astroquery.mast import Observations
import roman_datamodels as rdm

# Files are accessible via s3:// URIs on the Nexus
obs = Observations.query_criteria(obs_collection="ROMAN", ...)
products = Observations.get_product_list(obs)
uri = products["dataURI"][0]   # s3://mast:roman/...

with rdm.open(uri) as dm:
    data = dm.data
```

---

### roman-data-workshop

Workshop materials (slides + notebooks) for Roman data analysis training.

```bash
git clone https://github.com/spacetelescope/roman-data-workshop
```

Covers: data product overview, pipeline walkthrough, simulation hands-on,
archive access, jdaviz visualization.

---

## Integration with Other Skills

- Co-triggers with `emmanuel-engineering` for SOC monitor architecture, EKS deployments, Aurora PG schema, FastMCP tools
- Co-triggers with `vo-explorer` for Roman TAP/ObsCore/SIA queries
- Co-triggers with `context7-docs-lookup` for live roman_datamodels, romancal, asdf API docs
- Co-triggers with `emmanuel-markdown` for Roman ADRs, runbooks, pipeline design docs
