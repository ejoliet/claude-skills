# missions.md — Mission-Specific Data Access Patterns

> All code here is subject to rapid change as missions mature.
> **Always call Context7 for the relevant library before writing mission-specific code.**
> Check mission news pages for current data release status.

---

## SPHEREx (IRSA, operational since May 2025)

**What it is:** All-sky survey, 102 near-infrared bands (0.75–5.0 μm), ~450M galaxies.
**Archive:** IRSA (`https://irsa.ipac.caltech.edu`)
**Data cadence:** Weekly quick releases; full-sky map after year 1.

### Key data products
| Product | Description | IRSA table (verify current name) |
|---|---|---|
| Spectral image MEF | 102-band FITS multi-extension | `spherex_s1_l2_sp_v*` |
| Point source catalog | Positions + 102-band photometry | `spherex_s1_l3_psc_v*` |
| Cutout service | SODA-compatible spatial+spectral | Via IRSA cutout API |

### Access patterns
```python
from astroquery.ipac.irsa import Irsa
import pyvo as vo

# TAP query for catalog
irsa_tap = vo.dal.TAPService("https://irsa.ipac.caltech.edu/TAP")

# Check available SPHEREx tables first (table names evolve with releases)
for t in irsa_tap.tables.values():
    if "spherex" in t.name.lower():
        print(t.name, t.description)

# Point source catalog query
result = irsa_tap.run_sync("""
    SELECT ra, dec, flux_102, flux_err_102, flux_001, flux_err_001
    FROM spherex_s1_l3_psc_v01
    WHERE CONTAINS(POINT('ICRS', ra, dec),
                   CIRCLE('ICRS', 150.0, 2.2, 0.5)) = 1
""")

# Cutout service (SODA-compatible)
# Use IRSA cutout API or pyvo DataLink — check IRSA docs for current URL
cutout_url = "https://irsa.ipac.caltech.edu/SIA/SPHEREx"
sia = vo.dal.SIA2Service(cutout_url)
imgs = sia.search(pos=SkyCoord.from_name("M51"), size=Quantity(0.1, "deg"))
```

### SPHEREx science notes
- 102 bands = 17 spectral channels × 6 filters per channel; resolution R~40
- Each pixel (6.2") has a full 102-point spectrum — not imaging in the usual sense
- Cross-matching to Gaia/Rubin gives redshifts for galaxy catalog; SPHEREx alone gives photometric redshifts to z~2
- Weekly reprocessing means table names increment; always check `irsa_tap.tables`

---

## Euclid (ESA + ENSCI/IPAC, Q1 March 2025)

**What it is:** Wide survey to z=2; VIS (optical), NISP-P (YJH photometry), NISP-S (grism spectra).
**Archives:** ESA Science Archive, ENSCI (NASA side at IPAC)
**Data releases:** Q1 (March 2025) = early science fields; DR1 planned ~2026.

### Access via ESA TAP
```python
import pyvo as vo

# ESA Euclid TAP (check ESA docs for current URL)
euclid_tap = vo.dal.TAPService("https://easotf.esac.esa.int/tap-server/tap")

# List available tables
for t in euclid_tap.tables.values():
    print(t.name)

# MER (Merged) catalog — primary multi-band photometric catalog
result = euclid_tap.run_sync("""
    SELECT object_id, ra, dec, vis_mag, y_mag, j_mag, h_mag,
           photo_z, photo_z_err
    FROM euclid_dr1.mer_catalogue
    WHERE CONTAINS(POINT('ICRS', ra, dec),
                   CIRCLE('ICRS', 53.1, -28.0, 0.5)) = 1
    AND photo_z BETWEEN 0.5 AND 1.0
""", maxrec=10000)
```

### Access via astroquery (beta — call Context7 first)
```python
# astroquery.esa.euclid requires astroquery >= 0.4.8
# API still stabilizing — always check Context7 for current method names
from astroquery.esa.euclid import Euclid

Euclid.login(username="your_username", password="your_password")

# Positional query
results = Euclid.query_region(
    SkyCoord(53.1, -28.0, unit="deg"),
    radius=0.1*u.deg,
    table="mer_catalogue"
)

# DataLink for spectral products (NISP-S grism spectra)
dl_results = Euclid.get_datalinks(observation_ids=["..."])
```

### Euclid data model key tables
| Table | Contents |
|---|---|
| `mer_catalogue` | Merged multi-band photometry + photo-z |
| `vis_catalogue` | VIS-only detection catalog (deepest) |
| `spe_catalogue` | NISP-S spectroscopic redshifts |
| `spe_spectra` | 1D spectra (via DataLink) |
| `ero_catalogue` | Early Release Observations |

### Euclid Q1 footprint
```python
from mocpy import MOC
# Download Q1 MOC from ESA
q1_moc = MOC.from_fits("https://easotf.esac.esa.int/.../euclid_q1_moc.fits")
# Intersect with your target field
overlap = q1_moc.intersection(my_field_moc)
```

---

## Roman Space Telescope (Launch ~2026–2027)

**What it is:** 2.4m wide-field NIR imager + grism; WFI covers 0.48–2.3 μm.
**Archive:** MAST (STScI) + IPAC Roman SSC
**Status (2025):** Simulations available; pipeline in development; no on-sky data yet.

### Simulation and pipeline access
```python
import roman_datamodels as rdm   # always call Context7 first

# Open WFI Level 2 simulated image
model = rdm.open("roman_wfi_image_2_wcs.asdf")

# Key attributes
data    = model.data              # (4096, 4096) float32 array
err     = model.err               # uncertainty array
dq      = model.dq                # data quality bitmask
wcs     = model.meta.wcs          # GWCS object (gwcs library)
filter_ = model.meta.instrument.optical_element   # e.g. "F106"
exptime = model.meta.exposure.effective_exposure_time

# Convert to astropy for downstream tools
from astropy.nddata import CCDData
import astropy.units as u
ccd = CCDData(data=model.data, uncertainty=model.err, unit=u.electron/u.s)
```

### RomanCAL pipeline (romancal)
```python
# romancal: the Roman calibration pipeline (mirrors JWST Calibration pipeline)
# Always call Context7 for romancal before using — API mirrors jwst but Roman-specific
import romancal
from romancal.pipeline import ExposurePipeline

result = ExposurePipeline.call("roman_uncal.asdf", save_results=True)
```

### Roman + jdaviz
```python
import jdaviz
imviz = jdaviz.Imviz()
imviz.load_data("roman_wfi_l2.asdf", data_label="Roman WFI")
imviz.show()
# Imviz natively handles Roman ASDF via roman_datamodels backend
```

### Roman science footprint overlap with Euclid
```python
# Euclid Galactic Bulge Survey and Roman GBTDS overlap
# Roman Wide Area Survey overlaps with Euclid Wide Survey
# Design your cross-match assuming:
#   Roman pixel scale: 0.11 arcsec/pix
#   Euclid VIS pixel scale: 0.1 arcsec/pix
#   Astrometric tie: Gaia DR3
```

---

## Rubin LSST (RSP DP1, operational)

**What it is:** 10-year ugrizy survey of the southern sky; 20B+ objects expected.
**Access layers:** RSP TAP (VO), `lsst.daf.butler` (internal pipeline).

### Butler vs TAP — when to use which

| Use case | Layer |
|---|---|
| Query released catalog data (DP1, DR1...) | RSP TAP (`lsst.rsp.get_tap_service`) |
| Build on top of calibrated images | Butler (`lsst.daf.butler`) |
| Run pipeline tasks on exposures | Butler + `lsst.ctrl.mpexec` |
| Cross-match with external catalogs | TAP UPLOAD or LSDB |
| Access from outside RSP | pyvo + token auth |

### Butler access (from inside RSP notebook)
```python
import lsst.daf.butler as daf_butler

butler = daf_butler.Butler("/repo/main",
    collections=["LSSTComCam/runs/DRP/DP1/20250101T000000Z"])

# List available dataset types
for dt in butler.registry.queryDatasetTypes():
    print(dt.name)

# Get calibrated exposure
calexp = butler.get("calexp",
    dataId={"visit": 2024110200001, "detector": 4,
            "instrument": "LSSTComCam"})

# Get source catalog
src = butler.get("src",
    dataId={"visit": 2024110200001, "detector": 4,
            "instrument": "LSSTComCam"})
src_df = src.asAstropy().to_pandas()
```

### Key DP1 TAP table names
```sql
-- Object catalog (static; coadd detections)
dp01_dc2_catalogs.Object        -- DP0.2 sim
dp01_lsstcomcam.Object          -- DP1 real data (verify exact name on RSP)

-- Forced photometry time series
dp01_lsstcomcam.ForcedSource

-- Single-visit catalogs
dp01_lsstcomcam.Source

-- Observation metadata
dp01_lsstcomcam.CcdVisit

-- ObsCore images
ivoa.ObsCore
```

### Rubin LSST alert stream (broker integration)
```python
# Rubin sends alerts via Kafka; community brokers redistribute
# Install: pip install lsst-alert-packet confluent-kafka fastavro

from lsst.alert.packet import SchemaRegistry
import fastavro, io

# Deserialize a Rubin alert packet
with open("alert.avro", "rb") as f:
    reader = fastavro.reader(f)
    for alert in reader:
        obj_id  = alert["objectId"]
        ra, dec = alert["ra"], alert["decl"]
        diaobj  = alert["diaObject"]   # object-level summary
        diasrc  = alert["diaSource"]   # this detection

# Community brokers with Rubin support:
# ALeRCE:  https://alerce.science
# Fink:    https://fink-portal.org
# ANTARES: https://antares.noirlab.edu
# LASAIR:  https://lasair.roe.ac.uk
```

---

## JWST (MAST, operational)

### Standard access pattern
```python
from astroquery.mast import Observations
from astroquery.mast import Mast

# Query for JWST observations
obs = Observations.query_criteria(
    obs_collection="JWST",
    instrument_name="NIRCAM*",
    target_name="HUDF",
    calib_level=[2, 3],
)

# Get product list
products = Observations.get_product_list(obs)
science  = Observations.filter_products(products,
    productType="SCIENCE",
    extension="fits")

# Download
manifest = Observations.download_products(science[:3])
```

### JWST + jdaviz pipeline
```python
import jdaviz

# NIRSpec IFU cube → Cubeviz
cubeviz = jdaviz.Cubeviz()
cubeviz.load_data("jw01234_s3d.fits")
cubeviz.show()

# NIRSpec MOS → Mosviz
mosviz = jdaviz.Mosviz()
mosviz.load_data(spectra2d=spectra_list, spectra1d=spectra1d_list,
                 images=image_list)
mosviz.show()

# NIRCam imaging → Imviz with multi-band comparison
imviz = jdaviz.Imviz()
for filt in ["F090W", "F150W", "F200W", "F277W", "F356W", "F444W"]:
    imviz.load_data(f"jw_nircam_{filt}.fits", data_label=filt)
imviz.show()
```

---

## Cross-Mission Footprint Reference (MOC-based)

```python
from mocpy import MOC
import astropy.units as u

# Fetch survey MOCs from CDS MOCServer
euclid_wide  = MOC.from_vizier_table("...", max_depth=7)  # when available
rubin_y1     = MOC.from_fits("rubin_y1_footprint.fits")
spherex_all  = MOC.from_str("0/0-11")  # all-sky

# Find overlap region
overlap = euclid_wide.intersection(rubin_y1)
area_deg2 = overlap.sky_fraction * 4 * 3.14159 * (180/3.14159)**2
print(f"Overlap: {area_deg2:.0f} deg²")

# IRSA hosts MOCs for many missions:
# https://irsa.ipac.caltech.edu/MOC/
```
