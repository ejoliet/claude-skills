# CLAUDE.md — claude-skills

This is Emmanuel Joliet's personal skills repository for Claude Code.
Skills encode engineering style, document templates, and workflow patterns.
Read this file at the start of every session.

---

## Skills Index

| Skill | Path | Trigger |
|-------|------|---------|
| `emmanuel-engineering` | `engineering/SKILL.md` | Architecture, AWS/EKS, Python, MCP, pipelines, astronomy, LocalStack, Grafana |
| `emmanuel-markdown` | `markdown/SKILL.md` | Any `.md` doc, README, ADR, runbook, RFC, post-mortem |
| `readme-driven-dev` | `readme-driven-dev/SKILL.md` | "RDD for X", "agent-ready README", bootstrapping a new tool |
| `context7-docs-lookup` | `context7/SKILL.md` | Any coding task involving a named library or framework |
| `vo-explorer` | `vo-explorer/SKILL.md` | TAP/ADQL/IVOA/ObsCore/MOC/HiPS/pyvo VO queries, multi-archive |
| `roman-space-telescope` | `roman-space-telescope/SKILL.md` | Roman WFI, SOC system, romancal, romanisim, ASDF data products |
| `data-tools` | `data-tools/SKILL.md` | DuckDB, inspect parquet/CSV/FITS, schema sniff, quick data CLI |
| `astropy` | `astropy/SKILL.md` | Coordinates, units, FITS, WCS, tables, cosmology, astropy.time |
| `scientific-visualization` | `scientific-visualization/SKILL.md` | Publication figures, matplotlib/seaborn, journal formatting, colorblind palettes |
| `paper-lookup` | `paper-lookup/SKILL.md` | arXiv, Semantic Scholar, OpenAlex, Crossref, PubMed, DOI lookup |
| `zarr-python` | `zarr-python/SKILL.md` | Chunked N-D arrays, cloud (S3) storage, Dask/Xarray integration |
| `statistical-analysis` | `statistical-analysis/SKILL.md` | Test selection, t-test, ANOVA, regression, effect sizes, power analysis |
| `polars` | `polars/SKILL.md` | Fast DataFrames (1–100 GB), lazy evaluation, Arrow backend, pandas replacement |

---

## How to Load a Skill

Before responding to any task, check if a skill applies:

```bash
# Read a skill
cat engineering/SKILL.md
cat markdown/SKILL.md
cat readme-driven-dev/SKILL.md
cat context7/SKILL.md
```

**Multiple skills can apply.** For example, "RDD for a new FastMCP server" triggers
`readme-driven-dev` + `emmanuel-engineering` + `context7-docs-lookup`. Read all that apply.

---

## Skill Trigger Rules

### `emmanuel-engineering` — read when:
- Architecture decisions, system design, infrastructure planning
- Any AWS/EKS/Airflow/Jenkins/Aurora PostgreSQL work
- Python backends, FastAPI, FastMCP server development
- CI/CD pipelines, Docker, Kubernetes, Helm
- Astronomical data access (pyvo, astroquery, IRSA, MAST)
- VO protocols (TAP, SIA, ADQL, ObsCore, VOTable, FITS, ASDF)
- IaC (CloudFormation, Custodian, Terraform, CDK)
- Casual: "design X", "build me Z", "spin this up"
- **Do NOT trigger** for pure writing, trivia, non-technical requests

### `emmanuel-markdown` — read when:
- Any `.md` document requested
- README, ADR, Runbook, RFC/Design Doc, Meeting Notes, Post-Mortem
- "Write a markdown", "create a doc", "document this", "draft a README"
- **Do NOT trigger** for inline chat answers, code-only responses, slides

### `readme-driven-dev` — read when:
- "RDD for X", "create a README-driven spec", "agent-ready README"
- Bootstrapping a new tool, CLI, service, or library with no existing spec
- Any request where README should also be the build blueprint
- **Do NOT trigger** for post-hoc docs of already-built code, ADRs, runbooks

### `vo-explorer` — read when:
- Any task involving IVOA-compliant services, TAP, SIA, SCS, SSA, ObsCore, DataLink
- Writing ADQL queries or debugging ADQL syntax
- VO registry service discovery (pyvo `registry.search`)
- Cross-matching catalogs across archives (IRSA, MAST, VizieR, Gaia, NED)
- MOC / HiPS / Aladin-related spatial coverage work
- VOTable parsing, pyvo result handling
- "VO service", "cone search", "tap query", "multi-archive", "cross-match via TAP"
- **Do NOT trigger** for non-VO REST API queries or astroquery calls that don't involve VO protocols

### `roman-space-telescope` — read when:
- Any Roman Space Telescope data, pipeline, simulation, or SOC work
- Reading/writing Roman ASDF files with `roman_datamodels`
- Running or extending the `romancal` calibration pipeline
- Generating simulated WFI exposures with `romanisim`
- SOC file exchange monitor (EKS + FastMCP + Aurora PG)
- Roman WFI filters, detectors, survey programs (HLWAS, HLTDS, GBTDS)
- Roman GWCS coordinate transforms
- Roman MAST / IRSA archive access
- "roman", "roman wfi", "roman soc", "roman pipeline", "romancal", "romanisim"
- **Do NOT trigger** for generic ASDF/FITS questions unrelated to Roman

### `context7-docs-lookup` — read when:
- Any coding task, system design, or technical question involving a named library
- FastAPI, SQLAlchemy, boto3, Airflow, FastMCP, Helm, Kubernetes, Terraform, Pydantic, etc.
- "How do I configure X", "what's the syntax for Y", version migration questions
- **Trigger even when you think you know the answer** — training data may be stale

### `data-tools` — read when:
- Inspecting an unknown file (Parquet, CSV, JSON, FITS, ASDF)
- "What columns does this have?", "schema of", "what's in this file", "quick look"
- Ad-hoc SQL / GROUP BY against local or S3 files without a full pipeline
- Building a small data inspection CLI
- DuckDB, pyarrow, Parquet metadata, FITS table exploration
- **Do NOT trigger** for billion-row cross-match (vo-explorer), full pipeline (engineering), or science analysis (roman-space-telescope)

### `astropy` — read when:
- Coordinate system conversions (ICRS, Galactic, FK5, AltAz, etc.)
- Physical units and quantities (`astropy.units`, `.to()`, equivalencies)
- Reading, writing, or manipulating FITS files (`astropy.io.fits`)
- Cosmological calculations (luminosity distance, lookback time, redshift)
- Time handling with multiple scales/formats (UTC, TAI, TT, TDB, JD, MJD)
- Catalog cross-matching with `SkyCoord.match_to_catalog_sky`
- WCS pixel↔world transformations (`astropy.wcs`)
- "astropy", "SkyCoord", "astropy.units", "FITS header", "Planck18"
- **Do NOT trigger** for non-astropy VO protocol work (use vo-explorer), Roman-specific data (use roman-space-telescope)

### `scientific-visualization` — read when:
- Creating figures for papers, proposals, or presentations
- Multi-panel figures with consistent journal-specific styling
- Colorblind-safe palettes or grayscale-compatible plots
- Publication DPI/format requirements (Nature 89 mm, PDF/EPS/TIFF)
- Significance annotations, error bars, panel labels (A, B, C)
- "plot for paper", "publication figure", "matplotlib style", "seaborn", "journal figure"
- **Do NOT trigger** for quick exploratory plots or interactive dashboards

### `paper-lookup` — read when:
- Searching for papers by topic, author, DOI, PMID, or arXiv ID
- "find papers on X", "look up this DOI", "papers citing Y"
- Fetching open-access PDFs or full text
- Citation graph / author metrics queries
- Cross-referencing preprints (arXiv, bioRxiv) with published versions
- Any mention of arXiv, Semantic Scholar, OpenAlex, Crossref, PubMed, Unpaywall
- **Do NOT trigger** for non-scholarly web search

### `zarr-python` — read when:
- Storing or reading large N-D arrays with chunking and compression
- Cloud-native array workflows (S3, GCS) with zarr stores
- Dask + Zarr or Xarray + Zarr pipeline integration
- Roman or ASDF-adjacent large array I/O
- "zarr", "chunked array", "s3 zarr store", "da.from_zarr", "xr.open_zarr"
- **Do NOT trigger** for HDF5-only workflows or FITS-only pipelines (use data-tools)

### `statistical-analysis` — read when:
- Choosing the right statistical test for a dataset
- Checking normality, homogeneity of variance, or regression assumptions
- t-tests, ANOVA, chi-square, Mann-Whitney, Kruskal-Wallis
- Correlation (Pearson/Spearman), linear/logistic regression
- Effect sizes (Cohen's d, η², r) and confidence intervals
- Power analysis / sample-size planning
- APA-format result reporting
- **Do NOT trigger** for pure ML modeling tasks (use scikit-learn / pytorch)

### `polars` — read when:
- DataFrames that are slow in pandas or need lazy evaluation
- ETL pipelines with 1–100 GB in-memory datasets
- "polars", "LazyFrame", "pl.col", "scan_csv", "group_by().agg()"
- Parquet I/O, Arrow-native processing, fast joins
- Replacing pandas in a data pipeline
- **Do NOT trigger** for out-of-core (>RAM) data — use dask or DuckDB instead

---

## Active Systems Context

| System | Stack | Key details |
|--------|-------|-------------|
| Roman SOC file exchange | FastMCP + Aurora PG + EKS | `soc_files_table`; GraphQL layer |
| ADS MCP Server v2 | FastMCP + pypdf + httpx | ADS search, ArXiv PDF parsing |
| Transient broker dashboard | HTML + Aladin Lite | GCN, ALeRCE, Fink, ANTARES, LASAIR |
| Jenkins EC2 | AL2023; CloudWatch CPU alarms | SNS: `jenkins-cpu-alerts`; 40% threshold |
| Airflow on EKS | REST API configured | Pipeline orchestration |
| Missions | Roman, SPHEREx, Euclid, NEOWISE | IPAC/Caltech; IRSA access patterns |

**AWS account**: `765894972596` / `us-east-1`
**Author**: Emmanuel Joliet (`ejoliet`) / IPAC Caltech

---

## Output Standards

- **Architecture first** — never jump to code without a design
- **Trade-off tables** whenever multiple approaches exist
- **Phased plans** with clear exit criteria per phase
- **Docker artifacts** on every new service/API/notebook (goal: `docker compose up` works)
- **Production-grade defaults** — no toy examples
- **Honest scope** — state what's out of scope for v1

---

## Repo Layout

```
claude-skills/
├── CLAUDE.md                        ← this file
├── skills.json                      ← machine-readable manifest
├── engineering/
│   ├── SKILL.md
│   └── references/
│       ├── mcp-patterns.md
│       ├── aws-defaults.md
│       ├── kubernetes-manifests.md
│       └── astronomy-patterns.md
├── markdown/
│   └── SKILL.md
├── readme-driven-dev/
│   ├── SKILL.md
│   └── references/
│       ├── rdd-template.md
│       └── examples.md
├── context7/
│   └── SKILL.md
├── vo-explorer/
│   ├── SKILL.md
│   └── references/
│       ├── vo-service-catalog.md
│       ├── protocols.md
│       ├── formats.md
│       ├── platforms.md
│       ├── pitfalls.md
│       ├── missions.md
│       ├── science-tools.md
│       └── publishing.md
├── data-tools/
│   └── SKILL.md
├── roman-space-telescope/
│   └── SKILL.md
├── astropy/
│   └── SKILL.md
├── scientific-visualization/
│   └── SKILL.md
├── paper-lookup/
│   └── SKILL.md
├── zarr-python/
│   └── SKILL.md
├── statistical-analysis/
│   └── SKILL.md
└── polars/
    └── SKILL.md
```

---

## Adding New Skills

1. Create `<skill-name>/SKILL.md` with YAML frontmatter (`name`, `description`)
2. Add trigger rules to the **Skills Index** table above
3. Update `skills.json` with the new entry
4. If the skill has reference files, add them under `<skill-name>/references/`
