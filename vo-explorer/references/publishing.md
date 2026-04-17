# publishing.md — Provenance, FAIR Data, and Software Publication

---

## §Provenance — Pipeline Provenance Tracking

### IVOA Provenance Data Model

The IVOA ProvenanceDM (2020) is based on W3C PROV-DM. It tracks:
- **Entity** — data products (catalogs, images, spectra)
- **Activity** — pipeline steps, queries, processing runs
- **Agent** — software, person, organization

Practical implementation: embed provenance as metadata in your output files.

### Minimal provenance in Parquet (file-level metadata)

```python
import pyarrow as pa
import pyarrow.parquet as pq
import subprocess, datetime, json

def get_git_sha():
    return subprocess.check_output(
        ["git", "rev-parse", "HEAD"]
    ).decode().strip()

provenance = {
    "pipeline_version": "1.2.3",
    "git_sha":          get_git_sha(),
    "produced":         datetime.datetime.utcnow().isoformat() + "Z",
    "inputs": {
        "gaia":    {"release": "DR3", "tap_url": "https://gea.esac.esa.int/tap-server/tap",
                    "adql": adql_query, "query_date": "2025-07-01"},
        "rubin":   {"collection": "LSSTComCam/runs/DRP/DP1",
                    "butler_run": "20250101T000000Z"},
        "spherex": {"table": "spherex_s1_l3_psc_v01",
                    "adql": spherex_query, "query_date": "2025-07-01"},
    },
    "software": {
        "lsdb":    lsdb.__version__,
        "pyvo":    pyvo.__version__,
        "astropy": astropy.__version__,
    },
    "environment": {
        "platform": "AWS EKS us-east-1",
        "python":   "3.11.8",
    }
}

# Embed in Parquet as custom metadata
existing = arrow_table.schema.metadata or {}
new_meta = {**existing, b"provenance": json.dumps(provenance).encode()}
arrow_table = arrow_table.replace_schema_metadata(new_meta)
pq.write_table(arrow_table, "catalog_with_provenance.parquet")
```

### Provenance in ASDF (Roman/JWST products)

```python
import asdf

tree = {
    "data":   numpy_array,
    "wcs":    gwcs_object,
    "meta": {
        "provenance": {
            "software": {"name": "my_pipeline", "version": "1.0.0",
                         "git_sha": get_git_sha()},
            "inputs": [
                {"name": "roman_l2.asdf", "sha256": file_hash},
                {"name": "gaia_dr3_query.vot", "adql": query},
            ],
            "produced": datetime.datetime.utcnow().isoformat(),
        }
    }
}
af = asdf.AsdfFile(tree)
af.write_to("output_with_provenance.asdf")
```

### Iceberg snapshots as provenance primitives

Apache Iceberg's snapshot system provides built-in time travel and audit trail:

```python
from pyiceberg.catalog import load_catalog

catalog = load_catalog("glue", **{"type": "glue", "s3.region": "us-east-1"})
table   = catalog.load_table("roman_soc.catalog_v1")

# Every append creates a snapshot with timestamp and summary
for snapshot in table.history():
    print(snapshot.snapshot_id, snapshot.timestamp_ms,
          snapshot.summary)

# Pin a specific snapshot for reproducible analysis
pinned = table.scan(snapshot_id=1234567890).to_arrow()

# Add custom provenance summary to snapshot
table.append(
    data,
    snapshot_properties={"pipeline_version": "1.2.3",
                          "git_sha": git_sha,
                          "query_date": "2025-07-01"}
)
```

### ProvTAP — querying provenance metadata via VO

ProvTAP exposes provenance information as a queryable TAP service:

```python
# ProvTAP is available at select observatories (GAVO, CADC, others)
prov_tap = vo.dal.TAPService("https://dc.g-vo.org/tap")  # example
result = prov_tap.run_sync("""
    SELECT entity_id, activity_id, agent_id, generated_at
    FROM prov.entity
    WHERE entity_id = 'ivo://irsa.ipac.caltech.edu/catalog/fp_psc'
""")
```

---

## §FAIR — FAIR Data Practices for Astronomy Catalogs

FAIR = Findable, Accessible, Interoperable, Reusable.

### Findable: persistent identifiers

```python
# Register your catalog with a persistent DOI
# Options: Zenodo, IRSA (for NASA mission products), CDS/VizieR

# Zenodo (free, integrated with GitHub)
# 1. Tag a GitHub release
# 2. Zenodo auto-generates DOI
# 3. Cite as: doi:10.5281/zenodo.XXXXXXX

# VizieR submission: submit via https://cds.unistra.fr/submit.html
# → catalog becomes queryable via CDS TAP and SCS services

# Embed DOI in your data product metadata
provenance["doi"] = "10.5281/zenodo.XXXXXXX"
```

### Accessible: VO-queryable output

Make your catalog queryable via standard VO protocols using DaCHS:

```bash
# DaCHS = GAVO's Data Center Helper Suite — turn a CSV/Parquet into a TAP service
# pip install gavodachs  OR  use Docker image
docker run -p 8080:8080 -v $(pwd)/data:/data gavo/dachs

# Minimal resource descriptor (my_catalog.rd)
# → creates ivoa.obscore + TAP endpoint + SCS automatically
```

```xml
<!-- my_catalog.rd — DaCHS resource descriptor -->
<resource schema="my_survey">
  <table id="objects">
    <column name="ra"    unit="deg" ucd="pos.eq.ra;meta.main"/>
    <column name="dec"   unit="deg" ucd="pos.eq.dec;meta.main"/>
    <column name="z_phot" unit=""   ucd="src.redshift.phot"/>
    <column name="mag_g" unit="mag" ucd="phot.mag;em.opt.B"/>
  </table>

  <data id="import">
    <sources>my_catalog.parquet</sources>
    <parquetGrammar/>
    <make table="objects"/>
  </data>

  <service id="tap" allowed="tap,adql">
    <publish render="tap" sets="ivo_managed"/>
  </service>
</resource>
```

### Interoperable: use UCDs and standard column names

UCD (Unified Content Descriptor) — machine-readable semantics for columns:

```python
# Common UCDs for multi-mission catalogs
UCD_MAP = {
    "ra":           "pos.eq.ra;meta.main",
    "dec":          "pos.eq.dec;meta.main",
    "z_phot":       "src.redshift.phot",
    "z_spec":       "src.redshift",
    "mag_g":        "phot.mag;em.opt.B",
    "mag_r":        "phot.mag;em.opt.R",
    "flux_3p6":     "phot.flux.density;em.IR.3-4um",
    "class_star":   "src.class.starGalaxy",
    "parallax":     "pos.parallax",
    "pmra":         "pos.pm;pos.eq.ra",
    "pmdec":        "pos.pm;pos.eq.dec",
}

# Embed UCDs in astropy Table for VOTable output
from astropy.table import Table
from astropy.io.votable import from_table
tbl = Table(my_results)
for col, ucd in UCD_MAP.items():
    if col in tbl.colnames:
        tbl[col].meta["ucd"] = ucd
```

### Reusable: machine-readable documentation

```python
# Include README.md with:
# - Column descriptions with units and UCDs
# - Selection criteria (magnitude limits, quality flags)
# - Citation instructions (BibTeX entry)
# - Known caveats

# BibVO — IVOA bibliographic interface standard (2024)
# Embed citation metadata in VOTable INFO element
# <INFO name="CITATION" value="Doe et al. 2025 ApJ 123 456"/>
# <INFO name="DOI" value="10.3847/1538-4357/XXXXXX"/>
```

---

## §Software — Software Publication and Citation

### JOSS — Journal of Open Source Software

Best for Python packages with a test suite and documentation:

```markdown
<!-- paper.md template -->
# My Astronomy Pipeline

## Summary
Brief description of what the software does and why it matters.

## Statement of need
Which science problem does it solve? What gap does it fill?

## Installation
```bash
pip install my-pipeline
```

## Example usage
[code snippet]

## Acknowledgements
[funding sources]

## References
[bibliography]
```

Submit at: https://joss.theoj.org/  
Review criteria: functioning software, tests, documentation, clear scope.

### ASCL — Astrophysics Source Code Library

Lower barrier than JOSS; just registers existence and provides a citable record:

```
Submit at: https://ascl.net/code/add
Required: name, URL, description, language, license
Result: citable record like ascl:2507.001
```

### Zenodo software archiving

```bash
# Tag a GitHub release for automatic Zenodo archiving
git tag -a v1.0.0 -m "Release 1.0.0"
git push origin v1.0.0

# Then in Zenodo: enable GitHub → Zenodo sync
# DOI is auto-generated per release
# Cite specific version: doi:10.5281/zenodo.XXXXXXX

# Include CITATION.cff in your repo root
```

```yaml
# CITATION.cff
cff-version: 1.2.0
title: My Astronomy Pipeline
version: 1.0.0
doi: 10.5281/zenodo.XXXXXXX
date-released: "2025-07-01"
authors:
  - name: Doe, Jane
    orcid: https://orcid.org/0000-0000-0000-0000
license: MIT
repository-code: https://github.com/username/my-pipeline
```

### Software Management Plan (SMP)

Increasingly required by NASA and ESF funding agencies:

Key elements:
1. Version control (GitHub/GitLab)
2. Open license (MIT, Apache 2.0, or BSD preferred)
3. Persistent identifier (DOI via Zenodo)
4. Test suite + CI (GitHub Actions / Jenkins)
5. Documentation (ReadTheDocs preferred)
6. Contribution guidelines (CONTRIBUTING.md)
7. Long-term maintenance plan

---

## §DaCHS — Self-hosting a VO TAP Service

When you want your pipeline output to be queryable by the community via ADQL:

```bash
# Docker-based deployment (simplest for development)
docker run -d \
  --name dachs \
  -p 8080:8080 \
  -v $(pwd)/data:/var/gavo/inputs \
  -v $(pwd)/logs:/var/log/dachs \
  gavo/dachs

# Load a resource descriptor
docker exec -it dachs gavo imp my_catalog.rd

# Restart services
docker exec -it dachs gavo serve restart
```

```python
# Test your new TAP service from Python
tap = vo.dal.TAPService("http://localhost:8080/tap")
result = tap.run_sync("SELECT TOP 5 ra, dec, z_phot FROM my_survey.objects")
```

**DaCHS capabilities generated automatically:**
- TAP/ADQL queryable endpoint
- SCS (cone search) if ra/dec columns are present
- VOSI capabilities and tables endpoints
- Optional: ObsCore table, HiPS generation, SIA2

---

## §DataOrigin — IVOA DataOrigin Standard (2024–2025)

DataOrigin is the emerging IVOA lightweight provenance standard — lighter than
full ProvenanceDM, targeted at data products rather than full workflow graphs:

```xml
<!-- In VOTable response, as INFO elements -->
<INFO name="DATA_ORIGIN_TYPE" value="Catalog"/>
<INFO name="DATA_ORIGIN_IVOID" value="ivo://ipac.caltech.edu/irsa#fp_psc"/>
<INFO name="DATA_ORIGIN_BIBCODE" value="2003yCat.2246....0C"/>
<INFO name="DATA_ORIGIN_CURATION_DATE" value="2025-03-01"/>
```

When querying archives with DataOrigin support, check for these INFO elements
to automatically capture dataset provenance:

```python
result = tap.run_sync(query)
# pyvo exposes INFO elements from the VOTable response
for info in result.votable.infos:
    if info.name.startswith("DATA_ORIGIN"):
        print(info.name, ":", info.value)
```
