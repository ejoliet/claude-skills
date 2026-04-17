# claude-skills

Personal Claude Code skills repository for [Emmanuel Joliet](https://github.com/ejoliet) at IPAC Caltech.

Skills are structured prompt libraries that Claude Code loads on demand to encode engineering style, reference patterns, workflow templates, and domain expertise. Each skill lives in its own directory with a `SKILL.md` and optional `references/` folder.

---

## What This Repo Is

Claude Code reads `CLAUDE.md` at the start of every session. That file indexes all skills and defines **trigger rules** — conditions under which Claude should load a specific skill before responding. Skills avoid repetition of context across sessions: once a skill exists, you never have to re-explain your preferred patterns.

Skills here cover:

- **Engineering style** — architecture-first, production defaults, IPAC Caltech AWS/EKS stack
- **Astronomy domain** — VO protocols, Roman Space Telescope, astropy, data inspection
- **Research workflows** — paper search, scientific writing, statistical analysis
- **Data engineering** — fast DataFrames, cloud-native array storage, visualization

---

## Repo Layout

```
claude-skills/
├── CLAUDE.md                     ← master index + trigger rules (loaded every session)
├── skills.json                   ← machine-readable manifest
├── engineering/SKILL.md          ← architecture, AWS/EKS, Python, MCP
├── markdown/SKILL.md             ← README, ADR, Runbook, RFC templates
├── readme-driven-dev/SKILL.md    ← README-as-spec / RDD workflow
├── context7/SKILL.md             ← live library docs via Context7 MCP
├── vo-explorer/SKILL.md          ← VO protocols, TAP/ADQL, pyvo, cross-match
├── roman-space-telescope/SKILL.md← Roman WFI, romancal, romanisim, SOC system
├── data-tools/SKILL.md           ← DuckDB, schema sniff, quick data CLI
├── astropy/SKILL.md              ← coordinates, units, FITS, WCS, cosmology
├── scientific-visualization/SKILL.md ← publication figures, matplotlib/seaborn
├── paper-lookup/SKILL.md         ← arXiv, Semantic Scholar, DOI, OpenAlex
├── zarr-python/SKILL.md          ← chunked N-D arrays, S3, Dask/Xarray
├── statistical-analysis/SKILL.md ← test selection, effect sizes, power analysis
└── polars/SKILL.md               ← fast DataFrames, lazy eval, Arrow backend
```

---

## How to Use

### Automatic (recommended)

Claude Code reads `CLAUDE.md` automatically at session start. When your prompt matches a trigger rule, Claude loads the skill before responding — no extra steps needed.

### Manual

If you want Claude to apply a specific skill explicitly, just reference it in your prompt:

```
Read roman-space-telescope/SKILL.md and then help me write a romancal step.
```

### Adding a New Skill

1. Create `<skill-name>/SKILL.md` with YAML frontmatter (`name`, `description`)
2. Add an entry to the **Skills Index** table in `CLAUDE.md`
3. Add trigger rules under `## Skill Trigger Rules` in `CLAUDE.md`
4. Add the entry to `skills.json`
5. Optionally add reference files under `<skill-name>/references/`

---

## Skills

### `emmanuel-engineering`

Personal engineering style for IPAC Caltech work. Enforces architecture-first responses, trade-off tables, phased plans, Docker artifacts, and production-grade defaults across AWS/EKS, Python backends, MCP servers, and astronomical pipelines.

**Triggers:** architecture, system design, FastAPI, FastMCP, AWS/EKS, Airflow, Docker, IaC, CI/CD, pyvo, astroquery

**Example prompts:**

> "Design a FastMCP server that monitors Roman SOC file deliveries and exposes a GraphQL endpoint on EKS."

> "Build me a Jenkins CPU alarm using CloudWatch + SNS with a 40% threshold — give me the CloudFormation YAML."

---

### `emmanuel-markdown`

Markdown layout guide for all structured written deliverables at IPAC Caltech: README, ADR, Runbook, RFC, Meeting Notes, Post-Mortem. Enforces consistent headings, metadata blocks, and section order.

**Triggers:** any `.md` doc requested, "write a runbook", "create an ADR", "draft a README", "post-mortem"

**Example prompts:**

> "Write an ADR for switching our Aurora PostgreSQL schema to use JSONB columns for the SOC files metadata."

> "Create a runbook for the Jenkins EC2 CPU alarm — include SNS notification steps and escalation path."

---

### `readme-driven-dev`

README-Driven Development: produces a single `README.md` that is simultaneously a polished project README **and** a complete agent build specification. The agent reading the README should be able to implement the tool without asking follow-up questions.

**Triggers:** "RDD for X", "agent-ready README", "README as spec", bootstrapping a new service with no existing spec

**Example prompts:**

> "RDD for a new FastMCP server that wraps the ADS search API and returns PDF text via httpx + pypdf."

> "Write an agent-ready README for a CLI tool that cross-matches two FITS catalogs using pyvo TAP UPLOAD."

---

### `context7-docs-lookup`

Fetches live, version-accurate documentation via the Context7 MCP server before generating any library code. Prevents hallucinated API signatures from stale training data.

**Triggers:** any coding task involving a named library — FastAPI, pyvo, astropy, boto3, Airflow, Helm, Pydantic, etc.

**Example prompts:**

> "How do I configure retry logic in boto3's S3 client? Check current docs first."

> "What's the correct way to register a custom romancal pipeline step in the latest version?"

---

### `vo-explorer`

Expert guide for the full IVOA protocol stack: TAP, SIA, SCS, SSA, ObsCore, DataLink, MOC, HiPS, ADQL. Covers pyvo service discovery, ADQL authoring, multi-archive cross-matching, VOTable I/O, and catalog partitioning with HATS/LSDB.

**Triggers:** VO service, TAP query, ADQL, cone search, pyvo, cross-match, ObsCore, MOC, HiPS, multi-archive

**Example prompts:**

> "Write an ADQL query against the IRSA TAP service to find all SPHEREx observations within 0.5° of RA=150, Dec=2.5 with an overlap in the F062 filter."

> "Discover all active IVOA TAP services that serve ObsCore tables and cross-match their coverage with a MOC of the Roman HLWAS footprint."

---

### `roman-space-telescope`

Reference guide for Nancy Grace Roman Space Telescope work at IPAC Caltech. Covers WFI data products (L0–L3), `roman_datamodels`, `romancal` pipeline, `romanisim` simulations, the SOC file exchange system (FastMCP + Aurora PG + EKS), CRDS reference files, and GWCS coordinate handling.

**Triggers:** roman, roman wfi, romancal, romanisim, roman asdf, roman soc, roman pipeline, wfi filter, HLWAS, HLTDS, GBTDS

**Example prompts:**

> "Show me how to open a Roman Level 2 ASDF file with `roman_datamodels`, read the WFI image array, and extract the WCS to convert pixel (512, 512) to RA/Dec."

> "Write a `romanisim` script to simulate a 100-second WFI F106 exposure at a pointing 10° off the galactic plane."

---

### `data-tools`

Fast-path patterns for inspecting, profiling, and querying data files before committing to a full pipeline. Primary tool: DuckDB (zero-setup, in-process). Also covers pandas, astropy Table, pyarrow, and S3 direct query.

**Triggers:** "what's in this file", "schema of", "inspect parquet", duckdb, "quick look", "what columns", FITS table, data CLI

**Example prompts:**

> "I have a 40 GB Parquet file on S3 at `s3://my-bucket/catalog.parquet`. Show me the schema, row count, and a GROUP BY on the `filter` column — use DuckDB."

> "Read this FITS binary table and print the column names, dtypes, and first 5 rows without loading it all into memory."

---

### `astropy`

Core Python astronomy library: celestial coordinates and frame transforms, physical units and equivalencies, FITS file I/O, WCS pixel↔world transforms, cosmological calculations, precise time handling, and catalog cross-matching.

**Triggers:** astropy, SkyCoord, astropy.units, astropy.io.fits, astropy.wcs, astropy.cosmology, Planck18, coordinate transform, FITS header, catalog cross-match

**Example prompts:**

> "Convert a list of (RA, Dec) in ICRS to Galactic (l, b), then compute angular separations to a reference source at RA=10.5°, Dec=+41.2° — use `SkyCoord` with vectorised operations."

> "Open `observation.fits`, read the WCS from the primary header, and return the sky coordinates of all pixels within 30 arcsec of a given target — include units."

---

### `scientific-visualization`

Publication-quality figures using matplotlib and seaborn. Covers multi-panel layouts, journal-specific sizing (Nature 89 mm, Science 55 mm), colorblind-safe palettes (Okabe-Ito), error bars, significance markers, and PDF/EPS/TIFF export at correct DPI.

**Triggers:** publication figure, plot for paper, matplotlib style, multi-panel figure, colorblind palette, journal figure, seaborn, error bars

**Example prompts:**

> "Create a 2-panel publication figure (Nature single-column, 89 mm wide): left panel shows a scatter plot of photometric redshift vs spectroscopic redshift with 1-σ error bars; right panel shows the residual histogram. Use colorblind-safe colors and export as PDF."

> "Apply Okabe-Ito palette and a clean ticks style to this seaborn violin plot comparing flux distributions across 4 WFI filters. Remove top and right spines, add panel label 'A', and save as 300-DPI PNG."

---

### `paper-lookup`

Searches 10 academic databases via REST APIs: arXiv, Semantic Scholar, OpenAlex, Crossref, PubMed, PMC, bioRxiv, medRxiv, CORE, and Unpaywall. Handles DOI/PMID/arXiv ID lookup, citation graphs, open-access PDF retrieval, and author publication lists.

**Triggers:** "find papers on X", "look up this DOI", arXiv, Semantic Scholar, OpenAlex, Crossref, "papers citing Y", citation graph, open access PDF

**Example prompts:**

> "Search arXiv and Semantic Scholar for papers on 'Roman Space Telescope weak lensing' published after 2023. Return titles, authors, and arXiv IDs."

> "Given DOI `10.3847/1538-4365/ac2154`, fetch metadata from Crossref and check Unpaywall for an open-access PDF link."

---

### `zarr-python`

Chunked N-D array storage with compression for cloud-native scientific workflows. Supports local, S3, and GCS backends. Integrates with NumPy, Dask, and Xarray for parallel, out-of-core I/O. Covers chunking strategies, sharding, codec selection, and metadata consolidation.

**Triggers:** zarr, zarr store, chunked array, S3 zarr, da.from_zarr, xr.open_zarr, zarr compression, cloud array

**Example prompts:**

> "Store a (10 000, 2048, 2048) float32 array in an S3 Zarr store with Blosc/Zstandard compression and chunks of (1, 512, 512). Consolidate metadata afterwards so Xarray can open it efficiently."

> "Load a Zarr store from `s3://roman-data/wfi-mosaic.zarr` as a Dask array, compute the median along the time axis in parallel using 8 workers, and write the result back as a new Zarr array."

---

### `statistical-analysis`

Guided statistical workflow: test selection, assumption checking (Shapiro-Wilk, Levene), hypothesis testing (t-test, ANOVA, Mann-Whitney, chi-square), regression, correlation, effect sizes (Cohen's d, η², r), power analysis, and APA-formatted result reporting. Also covers Bayesian alternatives via PyMC.

**Triggers:** statistical test, t-test, ANOVA, effect size, power analysis, assumption check, Cohen's d, normality test, APA statistics, correlation

**Example prompts:**

> "I have flux measurements from two detector arrays (n=48 each). Check normality and variance homogeneity, then run the appropriate test. Report the result in APA format with effect size and 95% CI."

> "How many observations per group do I need to detect a medium effect (Cohen's d = 0.5) with 80% power at α = 0.05 for an independent-samples t-test? Show the power curve."

---

### `polars`

Lightning-fast DataFrame library built on Apache Arrow. Lazy evaluation, automatic parallelism, expression-based API. Best for 1–100 GB in-memory ETL pipelines where pandas is too slow. Covers lazy scanning, group-by aggregations, joins, Parquet I/O, and migration from pandas.

**Triggers:** polars, LazyFrame, pl.col, scan_csv, group_by agg, fast DataFrame, "pandas too slow", Arrow backend, Parquet ETL

**Example prompts:**

> "Scan a 20 GB Parquet catalog with `pl.scan_parquet`, filter on `magnitude < 22` and `filter == 'F106'`, group by `detector_id` to get mean/std of flux, and collect — keep it lazy until the final step."

> "Migrate this pandas pipeline to Polars: read a CSV, add a computed column `flux_ratio = flux_auto / flux_psf`, join on `source_id` to a reference catalog, and write the result to Parquet."

---

## Active Systems Context

| System | Stack |
|--------|-------|
| Roman SOC file exchange | FastMCP + Aurora PG + EKS |
| ADS MCP Server v2 | FastMCP + pypdf + httpx |
| Transient broker dashboard | HTML + Aladin Lite |
| Jenkins EC2 | AL2023 + CloudWatch CPU alarms |
| Airflow on EKS | REST API orchestration |

**AWS:** `765894972596` / `us-east-1` · **Author:** `ejoliet` / IPAC Caltech

---

## Sources

New skills sourced from [K-Dense-AI/claude-scientific-skills](https://github.com/K-Dense-AI/claude-scientific-skills) (MIT/BSD-3):
`astropy`, `scientific-visualization`, `paper-lookup`, `zarr-python`, `statistical-analysis`, `polars`
