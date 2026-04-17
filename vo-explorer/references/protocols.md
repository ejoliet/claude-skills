# protocols.md — VO Protocol Deep Patterns

## Registry: Stable Archive Access URLs

| Archive | TAP URL | Notes |
|---|---|---|
| IRSA (IPAC) | `https://irsa.ipac.caltech.edu/TAP` | WISE, 2MASS, ZTF, Spitzer, SPHEREx |
| MAST (STScI) | `https://mast.stsci.edu/tap/sync` | JWST, HST, TESS, Kepler |
| ESA Gaia | `https://gea.esac.esa.int/tap-server/tap` | Gaia DR3 |
| ESO | `https://archive.eso.org/tap_obs` | VLT, ALMA (ObsTAP) |
| CDS VizieR | `https://tapvizier.cds.unistra.fr/TAPVizieR/tap` | All VizieR catalogs |
| NOIRLab Astro Data Lab | `https://datalab.noirlab.edu/tap` | DECaLS, DES, etc. |
| Rubin RSP | `https://data.lsst.cloud/api/tap` | Token required |
| CADC | `https://ws.cadc-ccda.hia-iha.nrc-cnrc.gc.ca/argus` | CFHT, Gemini |

## TAP: Output Format Options

```python
# Default is VOTable; request Parquet for large results (where supported)
result = tap.run_sync(query, maxrec=1000000)
tbl = result.to_table()

# Some services support Parquet output (RSP, NOIRLab)
# → pass FORMAT=application/x-votable+parquet in query params (service-specific)
```

## TAP: VOSI Table Introspection

```python
tap = vo.dal.TAPService("https://irsa.ipac.caltech.edu/TAP")

# List all available tables
for schema in tap.tables.values():
    for table in schema.tables.values():
        print(table.name, table.description)

# Inspect columns of a specific table
table = tap.tables["fp_psc"]
for col in table.columns.values():
    print(col.name, col.datatype, col.ucd)
```

## TAP: Checking Service Capabilities

```python
cap = tap.get_tap_capability()
print("Default limit:", cap.outputlimit.default.content)
print("Hard limit:",    cap.outputlimit.hard.content)
print("Upload methods:", tap.upload_methods)
```

## ADQL: Advanced Patterns

### Nearest-neighbour (no UPLOAD needed for single target)
```sql
SELECT TOP 1 source_id, ra, dec,
       DISTANCE(POINT('ICRS', ra, dec), POINT('ICRS', 202.48, 47.23)) AS sep
FROM gaiadr3.gaia_source
WHERE CONTAINS(POINT('ICRS', ra, dec),
               CIRCLE('ICRS', 202.48, 47.23, 0.01)) = 1
ORDER BY sep ASC
```

### Multi-table JOIN within same service
```sql
SELECT g.source_id, g.ra, g.dec, g.phot_g_mean_mag,
       t.j_m, t.h_m, t.k_m
FROM gaiadr3.gaia_source AS g
JOIN gaiadr3.tmass_psc_xsc_best_neighbour AS x ON g.source_id = x.source_id
JOIN gaiadr3.tmass_psc_xsc_join AS t ON x.original_ext_source_id = t.original_psc_source_id
WHERE CONTAINS(POINT('ICRS', g.ra, g.dec),
               CIRCLE('ICRS', 83.82, -5.39, 0.5)) = 1
```

### Aggregate / statistics query
```sql
SELECT ROUND(mag_g/0.5)*0.5 AS mag_bin,
       COUNT(*) AS n,
       AVG(mag_r - mag_g) AS mean_color
FROM my_catalog
WHERE CONTAINS(POINT('ICRS', ra, dec),
               CIRCLE('ICRS', 150.0, 2.2, 1.0)) = 1
GROUP BY mag_bin
ORDER BY mag_bin
```

## DataLink: Service Descriptor Semantics

DataLink `#access` relations and standard semantic values:

| Semantic | Meaning |
|---|---|
| `#this` | Primary data product (full file) |
| `#cutout` | SODA cutout service |
| `#proc` | Generic processing service |
| `#preview` | Quick-look preview image |
| `#auxiliary` | Ancillary data (weight maps, masks) |
| `#progenitor` | Input data that produced this product |
| `#derivation` | Products derived from this one |

```python
# Full DataLink navigation pattern
for dl in imgs.iter_datalinks():
    for row in dl:
        print(row["semantics"], row["access_url"] or row["service_def"])
```

## SODA: Parameter Reference

| Parameter | Type | Example |
|---|---|---|
| `ID` | publisher DID | `ivo://irsa.ipac.caltech.edu/...` |
| `POS` | DALI geometry | `"CIRCLE 83.82 -5.39 0.1"` |
| `CIRCLE` | ra dec radius (deg) | `83.82 -5.39 0.1` |
| `POLYGON` | ra dec pairs | `83.0 -5.0 84.0 -5.0 84.0 -6.0 83.0 -6.0` |
| `BAND` | wavelength range (m) | `2.0e-6 2.5e-6` |
| `TIME` | MJD range | `58000 58500` |
| `POL` | Stokes | `"I"` or `"I Q"` |

## SIA 2.0: Common Search Parameters

```python
imgs = sia.search(
    pos=SkyCoord.from_name("Orion"),
    size=Quantity(1.0, "deg"),
    band=Quantity([1.0e-6, 3.0e-6], "m"),   # NIR
    time=Time(["2020-01-01", "2023-01-01"]),
    format="image/fits",
    calib_level=[2, 3],    # 2=reduced, 3=science-ready
    dptype="cube",         # or "image", "spectrum", "timeseries"
    maxrec=100,
)
```

## ObsCore: Key Column Reference

| Column | Description |
|---|---|
| `obs_id` | Observation identifier |
| `s_ra`, `s_dec` | Center coordinates (ICRS) |
| `s_region` | Sky coverage as STC-S or ADQL geometry |
| `t_min`, `t_max` | Time range (MJD) |
| `em_min`, `em_max` | Spectral range (metres) |
| `dataproduct_type` | `image`, `cube`, `spectrum`, `timeseries` |
| `calib_level` | 0=raw, 1=instrument, 2=reduced, 3=science, 4=contributed |
| `access_url` | Direct download URL |
| `access_format` | MIME type |
| `obs_publisher_did` | Publisher dataset identifier (used by SODA/DataLink) |
