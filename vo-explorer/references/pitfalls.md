# pitfalls.md — Known Pitfalls and Archive Quirks

## SODA: Per-Archive Implementation Status

SODA is the most commonly missing protocol. Always guard before use.

| Archive | SODA implemented? | Notes |
|---|---|---|
| ESO | ✅ Yes | Positional + spectral cutouts since 2020 |
| CADC | ✅ Yes | Full SODA 1.0 |
| MAST / STScI | ⚠ Partial | Image cutout service, not full SODA |
| Rubin RSP | ✅ Yes | Image cutout service via DataLink |
| IRSA | ⚠ Partial | Service-dependent; check per collection |
| CDS / VizieR | ❌ No | No SODA; use direct access_url |
| ESA Gaia | ❌ No | Download full datalink products |

**Defensive pattern:**
```python
soda = None
for dl in imgs.iter_datalinks():
    soda = dl.get_service("#cutout")
    if soda:
        break

if soda:
    result = soda.search(ID=pub_did, POS=f"CIRCLE {ra} {dec} {r}")
else:
    # Fall back to direct download of full file
    import requests
    r = requests.get(imgs[0].getdataurl(), stream=True)
    with open("full_file.fits", "wb") as f:
        f.write(r.content)
```

## TAP: Common Quirks

### IRSA
- Table names are case-sensitive: `fp_psc` not `FP_PSC`
- Large async queries timeout at 1 hour; split by spatial region if needed
- `TAP_UPLOAD` is supported; max upload size ~50MB
- NEOWISE: use `neowiser_p1ba_mch`, not deprecated `wise_allwise`

### ESA Gaia
- Must use `TOP` or `maxrec`; service rejects unbounded queries on large tables
- Gaia DR3 epoch is J2016.0; apply proper motion for other epochs:
  ```sql
  SELECT source_id,
    ra  + pmra  / 3600000.0 * (2024.0 - 2016.0) AS ra_2024,
    dec + pmdec / 3600000.0 * (2024.0 - 2016.0) AS dec_2024
  FROM gaiadr3.gaia_source WHERE ...
  ```
- `gaiadr3.gaia_source` has 1.8B rows — always use spatial WHERE clause

### CDS VizieR
- Table IDs use slash notation: `I/355/gaiadr3`, `II/246/out`
- TAP URL: `https://tapvizier.cds.unistra.fr/TAPVizieR/tap`
- No UPLOAD support
- VOTable output only (no Parquet)

### MAST
- Prefer `astroquery.mast` over raw TAP for mission-specific workflows
- DataLink returns many product types; filter by `productType="SCIENCE"` or `productSubGroupDescription`
- Token required for proprietary data (not yet public)

### Rubin RSP
- All queries run against the RSP-internal database; no public access without token
- 5GB intermediate result limit; 2GB final result limit
- `ivoa.ObsCore` table available for ObsTAP image discovery
- Async queries preferred for anything > 1M rows

## ADQL: Common Mistakes

```sql
-- WRONG: CONTAINS returns 1, not true
WHERE CONTAINS(POINT, CIRCLE) = TRUE  -- fails on some services

-- CORRECT:
WHERE CONTAINS(POINT, CIRCLE) = 1

-- WRONG: mixing ICRS and icrs (case matters on some services)
POINT('icrs', ra, dec)  -- may fail

-- CORRECT:
POINT('ICRS', ra, dec)

-- WRONG: using DISTANCE for cross-match with large tables (very slow)
WHERE DISTANCE(POINT('ICRS', c.ra, c.dec), POINT('ICRS', 202.48, 47.23)) < 0.5

-- CORRECT: use CONTAINS with CIRCLE (uses spatial index)
WHERE CONTAINS(POINT('ICRS', c.ra, c.dec), CIRCLE('ICRS', 202.48, 47.23, 0.5)) = 1
```

## pyvo: Version-Sensitive Patterns

### DataLink iteration (pyvo ≥ 1.4)
```python
# New preferred pattern
for dl in result.iter_datalinks():
    ...

# Old pattern (still works but deprecated in some versions)
for row in result:
    dl = row.getdatalink()
```

Always call Context7 for current pyvo DataLink API.

### Registry Spatial constraint (requires RegTAP 1.2)
```python
# This may fail on older registry implementations
svcs = vo.registry.search(vo.registry.Spatial(skycoord, radius=1*u.deg))

# Safer fallback: search by datamodel + service type, then filter locally
svcs = vo.registry.search(
    vo.registry.Datamodel("obscore"),
    vo.registry.Servicetype("tap")
)
# Then test each service manually
```

## LSDB / HATS: Known Issues

### Version pinning
LSDB and HATS move fast. Always pin:
```
lsdb>=0.7,<1.0
hats>=0.3,<1.0
hats-import>=0.3,<1.0
```
And call Context7 before each coding session to get current API.

### crossmatch result column names
Column name collisions: if both catalogs have `ra`, `dec` etc., lsdb suffixes
them with `_left` and `_right`. Access as:
```python
matched["ra_left"]   # from the calling catalog (ztf in ztf.crossmatch(gaia))
matched["ra_right"]  # from the argument catalog (gaia)
```

### S3 paths
lsdb `read_hats()` accepts `s3://` paths directly — no need to set up fsspec
separately as long as boto3 credentials are configured in environment.

### Memory on local machines
LSDB is lazy but `.compute()` loads into RAM. For billion-row catalogs on a
laptop, always use `cone_search()` or `polygon_search()` to reduce before
computing.

## Zarr: Version 2 vs Version 3

`zarr-python` 3.x (released 2024) changed the store interface:

| Feature | v2 | v3 |
|---|---|---|
| Open store | `zarr.open("path")` | `zarr.open("path")` (same) |
| S3 store | `zarr.storage.FSStore(...)` | `zarr.storage.FsspecStore(...)` |
| Compressor | `zarr.Blosc(...)` | `numcodecs.Blosc(...)` (separate package) |
| Chunk key sep | `.` default | `/` default (configurable) |

Always call Context7 for current zarr API. Always pin: `zarr>=2.18,<3` or
`zarr>=3.0,<4` (don't mix major versions).

## Iceberg: pyiceberg Gotchas

- `load_catalog("glue")` requires `pyiceberg[glue]` extra install
- Schema IDs must be unique positive integers — manage carefully across versions
- Parquet files written by pandas/pyarrow are compatible but may have metadata
  differences; use `table.overwrite()` for full replacement, `table.append()`
  for incremental
- AWS Glue catalog: namespace = Glue database, table = Glue table. Naming
  constraints apply (no hyphens in names)

## Firefly: Common Setup Issues

- `make_lab_client()` requires `jupyter-firefly-extensions` JupyterLab extension
  AND the `FIREFLY_URL` environment variable set BEFORE JupyterLab starts
- If `FIREFLY_URL` is not set, client defaults to `http://localhost:8080/firefly`
  which silently fails if no local server is running
- Table brushing (linked selection between table and image) requires the table
  and image to share the same `tbl_id` coordinate column

## jdaviz: Common Issues

- `jdaviz.Imviz()` requires `ipywidgets` and JupyterLab widget extensions enabled
- ASDF files (Roman) load differently from FITS; use `load_data(path, data_label=...)`
- `cubeviz.app.get_data("Flux Viewer")` is the pattern for programmatic data access
  after interactive selection — API changes frequently; always check Context7

## MOC: Precision Gotchas

- `max_depth` affects query efficiency: too low → imprecise footprint; too high → large MOC
  Recommended defaults: order 10 for catalogs, order 7 for survey-level footprints
- `from_vizier_table()` fetches pre-computed MOC from MOCServer (order 10/11 max)
  For custom catalogs, build MOC from coordinates with `from_skycoords()`
- MOC intersection requires both MOCs at same or comparable depth; MOCPy handles
  this automatically but be aware of depth degradation in the result

---

## §Epochs — Astrometric Epoch Pitfalls in Cross-Mission Cross-Matching

### When epoch correction matters

| Scenario | Correction needed? |
|---|---|
| Galaxies at z > 0.1 across any epochs | ❌ No (proper motion negligible) |
| Stars, epoch gap < 2 yr, match radius > 1" | ❌ Usually not |
| Stars, Gaia × Rubin (2016→2025), match radius < 0.5" | ✅ Yes |
| High-PM stars (>100 mas/yr), any epoch gap | ✅ Yes, critical |
| Solar neighbourhood stars (d < 100 pc), any survey | ✅ Yes |

### Epoch reference points by mission

| Mission | Catalog epoch | Notes |
|---|---|---|
| Gaia DR3 | J2016.0 | Positions + PM available |
| Rubin DP1 | ~J2024.8 | Observation-date dependent |
| 2MASS | J2000 approx | Obs 1997–2001, avg ~J1999 |
| WISE/NEOWISE | varies | Obs 2010–present |
| SPHEREx | J2025+ | Weekly obs; use obs_date from table |
| Euclid Q1 | J2024–2025 | VIS astrometry tied to Gaia |
| Roman | J2026+ | Will be Gaia DR3 anchored |

### Proper motion correction pattern

```python
from astropy.coordinates import SkyCoord, Distance
from astropy.time import Time
import astropy.units as u
import numpy as np

def propagate_gaia_to_epoch(ra, dec, pmra, pmdec, parallax, target_epoch):
    """Propagate Gaia DR3 (J2016.0) positions to target epoch."""
    valid = np.isfinite(pmra) & np.isfinite(pmdec) & np.isfinite(parallax)

    coord = SkyCoord(
        ra=ra[valid]*u.deg,
        dec=dec[valid]*u.deg,
        pm_ra_cosdec=pmra[valid]*u.mas/u.yr,
        pm_dec=pmdec[valid]*u.mas/u.yr,
        distance=Distance(parallax=parallax[valid]*u.mas),
        frame="icrs",
        obstime=Time("J2016.0"),
    )
    propagated = coord.apply_space_motion(new_obstime=Time(target_epoch))
    return propagated.ra.deg, propagated.dec.deg

# Apply before cross-matching Gaia × Rubin
gaia_ra_2025, gaia_dec_2025 = propagate_gaia_to_epoch(
    gaia_df.ra, gaia_df.dec,
    gaia_df.pmra, gaia_df.pmdec,
    gaia_df.parallax,
    target_epoch="J2024.8"
)
```

### Rule of thumb for match radius

Proper motion introduces positional drift of:
- `0.5 mas/yr × 8 yr (2016→2024) = 4 mas` — negligible for most science
- `100 mas/yr × 8 yr = 800 mas = 0.8"` — significant, must correct

Always add quadrature correction to match radius:
`r_match = sqrt(r_intrinsic² + (pm_typical × Δt)²)`

---

## §QualityFlags — Data Quality Flags and Masking Patterns

### Gaia DR3 quality selection

```python
# Standard quality cuts (Lindegren et al. 2021)
gaia_clean = df[
    (df["ruwe"] < 1.4) &                              # astrometric quality
    (df["phot_g_mean_flux_over_error"] > 50) &         # G-band S/N > 50
    (df["astrometric_excess_noise_sig"] < 2.0) &       # no excess noise
    (~df["non_single_star"].astype(bool)) &             # not a binary flag
    (df["ipd_gof_harmonic_amplitude"] < 0.1)           # PSF asymmetry
]
```

### 2MASS quality flags

```sql
-- In ADQL: select only high-quality 2MASS detections
SELECT ra, dec, j_m, h_m, k_m
FROM fp_psc
WHERE ph_qual LIKE 'A%'        -- J-band grade A (SNR > 10, no contamination)
  AND rd_flg NOT LIKE '%0%'   -- not upper limit in any band
  AND cc_flg = '000'           -- no contamination/confusion
```

### Rubin LSST bitmask flags

```python
from astropy.nddata.bitmask import bitfield_to_boolean_mask, interpret_bit_flags

# Rubin flag bits (defined in lsst.meas.base.flagDefinitions)
BAD_FLAGS = [
    "base_PixelFlags_flag_saturated",
    "base_PixelFlags_flag_cr",
    "base_PixelFlags_flag_bad",
    "base_PixelFlags_flag_edge",
    "base_PixelFlags_flag_interpolated",
]

# Select clean sources
clean = src_catalog[
    ~src_catalog["base_PixelFlags_flag_saturated"] &
    ~src_catalog["base_PixelFlags_flag_cr"] &
    ~src_catalog["base_PixelFlags_flag_bad"] &
    (src_catalog["base_PsfFlux_snr"] > 5)
]
```

### Euclid MER catalog quality selection

```python
# Euclid quality flags (verify column names against current DR)
euclid_clean = df[
    (df["vis_mag_err"] < 0.2) &         # VIS photometry quality
    (df["y_mag_err"] < 0.2) &           # Y band
    (df["photo_z_flags"] == 0) &        # no photo-z issues
    (df["star_galaxy_sep"] < 0.5)       # extended source (0=star, 1=galaxy)
]
```

### Depth maps and completeness masks as MOCs

```python
from mocpy import MOC
import astropy.units as u

# Many surveys provide completeness depth maps
# Convert a survey completeness mask (boolean array + WCS) to MOC
from astropy.wcs import WCS
from astropy.io import fits

hdu = fits.open("rubin_completeness_mask.fits")[1]
wcs = WCS(hdu.header)
mask = hdu.data > 0   # complete pixels

# Create MOC from mask (uses astropy WCS to get sky coordinates)
moc = MOC.from_fits_image(hdu, max_norder=10, mask=mask)

# Use MOC to filter your catalog to complete regions
in_complete = moc.contains_lonlat(df["ra"]*u.deg, df["dec"]*u.deg)
df_complete = df[in_complete]
```

