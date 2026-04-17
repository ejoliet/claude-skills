---
name: emmanuel-engineering
description: >
  Personal engineering style guide for Emmanuel at IPAC Caltech. Apply whenever
  Emmanuel asks for architecture, system design, infrastructure, code scaffolding,
  cloud/AWS/EKS, Python backends, MCP servers, CI/CD, data pipelines, database schemas,
  or any technical task needing a structured plan. Also trigger for: astronomical data
  access (pyvo, astroquery, IRSA, MAST), VO protocols (TAP, SIA, SCS, SSA, ObsCore,
  ADQL), file formats (FITS, ASDF, VOTable, Parquet), IaC (CloudFormation, Custodian,
  Terraform, CDK), Docker/container prototyping, docker-compose setups, JupyterLab
  notebook environments, or GitHub code search for patterns. Trigger for casual requests
  like "help me design X", "build me a Z", "spin this up locally", "find examples on
  GitHub" — especially for AWS, Kubernetes, Python, astronomy, VO standards, Docker,
  or distributed systems. Do NOT use for pure writing, trivia, or non-technical requests.
---

# Emmanuel Engineering Skill

Personal engineering style and execution guide. This skill encodes Emmanuel's
preferred workflow, tooling biases, and output standards across all technical work,
including astronomical data science at IPAC Caltech.

---

## Core Philosophy

1. **Architecture first** — Never jump to code before the design is clear.
2. **Evidence-backed** — Justify decisions with trade-offs, not opinions.
3. **Structured plans** — Break work into phases with clear exit criteria.
4. **Practical code paths** — Production-grade defaults; no toy examples.
5. **macOS/Linux bias** — Assume Darwin or Linux (Ubuntu/AL2023) unless told otherwise.
6. **Search before you build** — For astronomy patterns, scan GitHub (NASA-NAVO, caltech-ipac, astropy) for existing approaches before inventing new ones.
7. **Container-first prototyping** — For any web app, API, notebook, or service prototype, always include a `Dockerfile` + `docker-compose.yml`. Use JupyterLab for exploratory/notebook work. Aim for `docker compose up` → working environment in one step.

---

## Response Structure

For any non-trivial technical request, follow this order:

```
1. Problem restatement (1–2 sentences)
2. Architecture / design (diagram or structured description)
3. Trade-off table (when multiple approaches exist)
4. Phased implementation plan
5. Code (complete, runnable, production-standard)
6. Operational notes (monitoring, failure modes, runbook hints)
```

Skip sections only when the request is clearly narrow (e.g., "fix this one-liner").

---

## Architecture Defaults

### Cloud / Infrastructure

#### IaC Priority Order (use first applicable)
1. **CloudFormation** (YAML/JSON) — preferred for AWS-native resources; native rollback, drift detection, Change Sets, compliance integration; use for new greenfield stacks
2. **Cloud Custodian** (`c7n`) — for governance-as-code, compliance enforcement, tag policies, auto-remediation on existing resources; YAML DSL; CNCF project
3. **Terraform** (HCL) — for multi-provider resources (Kubernetes objects, Datadog, GitHub, etc.) or existing Terraform-managed estate; use `terraform plan` before all applies
4. **CDK** (Python/TypeScript) — only when high-level L2/L3 constructs offer clear complexity reduction; remember CDK synthesizes to CloudFormation under the hood

> **Decision rule**: If it's pure AWS and new → CloudFormation. If it's compliance/governance on existing AWS → Custodian. If it's multi-provider or existing Terraform → Terraform. If developer abstraction is the top priority on AWS → CDK.

#### General
- **Cloud**: AWS-first (us-east-1 default)
- **Compute**: EKS (Kubernetes) for services; EC2 (AL2023) for long-running agents
- **CI/CD**: Jenkins pipelines; GitHub Actions as fallback
- **Secrets**: AWS Secrets Manager; never hardcode credentials
- **Networking**: VPC with private subnets; ALB for ingress; NLB for TCP

### Data / Storage
- **Primary DB**: Aurora PostgreSQL (RDS-compatible)
- **Object store**: S3 with lifecycle policies and versioning on
- **Queues**: SQS (standard) or Kafka/MSK for streaming
- **Cache**: ElastiCache Redis when needed
- **Search**: OpenSearch for full-text; standard PG indexes first

### Orchestration / Pipelines
- **Workflow**: Apache Airflow on EKS
- **Event routing**: EventBridge for AWS-internal; Kafka for cross-service streaming
- **Container registry**: ECR

### Observability
- **Metrics**: CloudWatch; Prometheus/Grafana for K8s
- **Alarms**: Always wire to SNS topic first; email + optional Slack
- **Logging**: Structured JSON to CloudWatch Logs; log groups per service

---

## Language & Framework Defaults

### Python (preferred backend language)
```
- Python 3.11+
- FastAPI for HTTP APIs
- FastMCP for MCP servers
- SQLAlchemy 2.x (async) + asyncpg for PostgreSQL
- Pydantic v2 for validation/schemas
- pandas for tabular data manipulation
- boto3 / botocore for AWS
- httpx for async HTTP clients
- pytest + pytest-asyncio for tests
- uv or pip-tools for dependency management
- Black + isort + ruff for formatting/linting
```

**Standard astronomy Python stack** (add when domain-relevant):
```
- astropy             — core units, coordinates, FITS I/O, WCS, Table
- pyvo               — VO protocol access (TAP, SIA, SCS, SSA, registry)
- astroquery         — archive-specific clients (IRSA, MAST, SIMBAD, etc.)
- asdf + asdf-astropy — ASDF format for Roman/JWST pipeline products
- astropy.io.fits    — FITS read/write (prefer over raw pyfits)
- pandas + pyarrow   — Parquet I/O; interop with astropy Table via to_pandas()
```

### MCP Servers (Emmanuel's specialty)
- Always use **FastMCP** pattern
- Tool names: `snake_case`, descriptive
- Always include: input validation, structured error returns, docstrings
- Pattern: tool → service layer → data layer (never put DB calls in tool handlers)
- See `references/mcp-patterns.md` for boilerplate

### Shell / CLI
- Assume **bash** (not tcsh/zsh unless specified)
- Scripts: `#!/usr/bin/env bash`, `set -euo pipefail`
- macOS: assume Homebrew; Linux: assume dnf (AL2023) or apt (Ubuntu)

### Frontend / Dashboards
- Self-contained **HTML artifacts** for quick tools and dashboards
- React (JSX artifact) for interactive multi-state UIs
- Tailwind for styling; no external CSS frameworks
- Aladin Lite for sky/astronomical map overlays

---

## Astronomical Data Access Patterns

### VO Protocol Priority (for archive data retrieval)
1. **pyvo TAP** for catalog/table queries with ADQL — works on any IVOA-compliant service
2. **astroquery.ipac.irsa** for IRSA-hosted data (WISE/NEOWISE, Spitzer, SPHEREx, ZTF, Euclid, 2MASS)
3. **astroquery.mast** for HST, JWST, TESS, and STScI-hosted missions
4. **astroquery.* other** (SIMBAD, VizieR, NED, etc.) for specialized services

### pyvo Patterns (TAP-first)
```python
import pyvo as vo
from astropy.coordinates import SkyCoord
from astropy.units import Quantity

# Registry lookup — preferred over hardcoding access URLs
svc = vo.registry.search(datamodel="obscore", servicetype="tap")[0].get_service("tap")

# Synchronous TAP / ADQL query
result = svc.run_sync("""
    SELECT TOP 100 obs_id, s_ra, s_dec, dataproduct_type
    FROM ivoa.obscore
    WHERE 1=CONTAINS(POINT('ICRS', s_ra, s_dec),
                     CIRCLE('ICRS', 202.48, 47.23, 0.5))
""")
tbl = result.to_table()  # -> astropy Table

# Async TAP for long-running queries
with svc.submit_job("SELECT * FROM big_catalog WHERE ...") as job:
    job.run()
    result = job.fetch_result()

# SIA for image discovery
sia_svc = vo.dal.SIAService("https://irsa.ipac.caltech.edu/SIA")
images = sia_svc.search(SkyCoord.from_name("M31"), size=Quantity(0.5, "deg"))
```

### astroquery IRSA Patterns
```python
from astroquery.ipac.irsa import Irsa
from astropy.coordinates import SkyCoord
import astropy.units as u

# Spatial cone (simple) — preferred for most spatial queries
coord = SkyCoord(202.48417, 47.23056, unit="deg")
table = Irsa.query_region(coord, catalog="fp_psc", spatial="Cone",
                          radius=2 * u.arcmin)

# Complex ADQL via TAP — when spatial cone is insufficient
table = Irsa.query_tap("""
    SELECT designation, ra, dec, j_m, h_m, k_m
    FROM fp_psc
    WHERE CONTAINS(POINT('ICRS', ra, dec),
                   CIRCLE('ICRS', 202.48, 47.23, 0.5)) = 1
      AND j_m < 15.0
""").to_table()

# List catalogs (supports filter= kwarg in astroquery >= 0.4.10)
catalogs = Irsa.list_catalogs(filter="neowise")
```

### astroquery MAST Patterns
```python
from astroquery.mast import Observations, Mast

obs = Observations.query_region("M101", radius="0.2 deg")
jwst_obs = obs[obs["obs_collection"] == "JWST"]
products = Observations.get_product_list(jwst_obs)
manifest = Observations.download_products(products, productType="SCIENCE")
```

### File Format Conventions

| Format | Library | Use case |
|--------|---------|----------|
| FITS | `astropy.io.fits` | Images, spectra, legacy catalog tables |
| FITS Table | `astropy.table.Table.read(f, format='fits')` | Binary FITS tables |
| ASDF | `asdf` + `asdf-astropy` | Roman, JWST pipeline products; hierarchical metadata |
| VOTable | `astropy.io.votable` or pyvo result `.to_table()` | VO query results |
| Parquet | `pandas.read_parquet()` or `Table.read(f, format='parquet')` | Large catalogs, pipeline outputs |
| CSV/TSV | `astropy.table.Table.read(f, format='ascii')` or `pandas` | Simple interchange |

**FITS rules:**
- Always use `memmap=True` for large files (lazy loading)
- Use `astropy.wcs.WCS` for coordinate transforms, not manual header parsing
- For SPHEREx spectral WCS: disable SIP (`spectral_wcs.sip = None`)
- Call `hdul.info()` before assuming HDU structure

**ASDF rules:**
- Use `asdf.open(filename)` as a context manager
- Roman/JWST products use `asdf-astropy` extension for unit/coord serialization

**Parquet rules:**
- Prefer `pyarrow` backend; compress with `snappy`
- For large S3 catalogs, use `awswrangler.s3.read_parquet()` with partition pruning

---

## Quick Data Inspection (DuckDB)

Use DuckDB as a zero-setup scratchpad to inspect Parquet, CSV, JSON, and S3 files
before committing to a full pipeline. Faster than pandas for schema exploration.

```python
import duckdb

con = duckdb.connect()   # in-memory; no file needed

# Schema sniff — always start here
con.sql("DESCRIBE SELECT * FROM 'data.parquet' LIMIT 0").show()
con.sql("SUMMARIZE 'data.parquet'").show()     # stats: min/max/nulls per column

# Quick filter + count
con.sql("""
    SELECT status, COUNT(*) AS n
    FROM 'output/*.parquet'
    GROUP BY status ORDER BY n DESC
""").show()

# Convert to pandas (then astropy if needed)
df = con.sql("SELECT * FROM 'catalog.parquet' WHERE mag_g < 22").df()

# Read CSV (auto-detects delimiter, header, types)
con.sql("SELECT * FROM read_csv_auto('results.csv') LIMIT 10").show()

# Read JSON
con.sql("SELECT * FROM read_json_auto('events.json') LIMIT 5").show()
```

### S3 Direct Query (no download)

```python
import duckdb

con = duckdb.connect()
con.sql("INSTALL httpfs; LOAD httpfs;")
con.sql("""
    SET s3_region='us-east-1';
    SET s3_access_key_id=?;
    SET s3_secret_access_key=?;
""")

# Query a Parquet file directly from S3
con.sql("""
    SELECT detector, COUNT(*) AS n_files, SUM(file_size_bytes)/1e6 AS total_mb
    FROM 's3://my-bucket/soc-files/*.parquet'
    GROUP BY detector
""").show()
```

### CLI one-liners

```bash
# Quick schema + row count from terminal
duckdb -c "DESCRIBE SELECT * FROM 'data.parquet' LIMIT 0; SELECT COUNT(*) FROM 'data.parquet';"

# Export filtered slice to CSV
duckdb -c "COPY (SELECT * FROM 'big.parquet' WHERE status='failed') TO 'failed.csv' (HEADER);"
```

**When to use DuckDB vs pandas:**
- Schema sniff, quick filter, GROUP BY → **DuckDB** (faster, no memory load)
- Astropy Table interop, VO result manipulation → **pandas** (astropy `.to_pandas()`)
- Billion-row catalogs with spatial ops → **LSDB + Dask** (see vo-explorer skill)

---

## GitHub Search Guidance

When asked to find patterns, examples, or reference implementations:

**Search these orgs first:**
- `github.com/NASA-NAVO` — VO notebook tutorials (pyvo, TAP, SIA patterns)
- `github.com/caltech-ipac` — IRSA tools, Firefly, SPHEREx pipeline
- `github.com/astropy` — pyvo, astroquery, astropy core
- `github.com/spacetelescope` — JWST, Roman pipeline patterns
- `github.com/cloud-custodian` — c7n policy examples

**Search method:**
- Use `gh search code "PATTERN" --language python` for CLI search
- Use GitHub web search: `site:github.com org:astropy pyvo TAP example`
- Always check the most recent commits and release notes, not just READMEs
- For IPAC-specific patterns, check `caltech-ipac.github.io/irsa-tutorials` notebooks

---

## Code Quality Standards

Every code output must include:

| Standard | Requirement |
|---|---|
| Error handling | Try/except with typed exceptions; never bare `except:` |
| Logging | `structlog` or `logging` with level + context |
| Config | Environment variables via `pydantic-settings` or `os.environ`; no hardcoded values |
| Types | Full type annotations; `mypy`-compatible |
| Docstrings | Google style on all public functions/classes |
| Tests | At minimum: happy path + one failure case |
| README fragment | Usage snippet + env var table |

---

## Phased Planning Template

When producing an implementation plan, always use phases:

```
Phase 0: Prerequisites & scaffolding   (env, deps, repo layout)
Phase 1: Core data model / schema      (DB schema, Pydantic models)
Phase 2: Service layer                 (business logic, no I/O)
Phase 3: I/O adapters                  (API endpoints, MCP tools, Airflow DAGs)
Phase 4: Observability                 (logging, metrics, alarms)
Phase 5: Deployment                    (Dockerfile, Helm chart, CI/CD)
Phase 6: Validation                    (smoke tests, integration tests)
```

Adjust phases to match scope. Always state which phase the current response covers.

---

## Trade-off Format

When comparing options, use a concise table:

```markdown
| Option | Pros | Cons | Best when |
|--------|------|------|-----------|
| A      | ...  | ...  | ...       |
| B      | ...  | ...  | ...       |
```

Follow with a **Recommendation** line and 1–2 sentence rationale.

---

## Context: Emmanuel's Active Systems

| System | Stack | Notes |
|--------|-------|-------|
| Roman SOC file exchange monitor | FastMCP + Aurora PG + EKS | `soc_files_table`; GraphQL schema layer |
| ADS MCP Server v2 | FastMCP + pypdf + httpx | ADS search, citations, ArXiv PDF parsing |
| Astronomical transient broker dashboard | HTML + Aladin Lite | GCN, ALeRCE, Fink, TNS, ANTARES, LASAIR |
| Jenkins EC2 (master + big-executor) | AL2023; CloudWatch CPU alarms | SNS: `jenkins-cpu-alerts`; threshold 40% 7d avg |
| Airflow on EKS | REST API configured | Used for pipeline orchestration |
| GitHub workflows | SSH remotes, RC tagging | Standard GitOps; Jenkins triggers |
| Missions | Roman, SPHEREx, Euclid, NEOWISE | IPAC/Caltech; IRSA data access patterns apply |

---

## Docker Prototyping Defaults

Whenever producing code for a web app, API, MCP server, data pipeline, or notebook,
**always include Docker artifacts** unless the request is clearly a tiny one-liner fix.
Goal: `docker compose up` → fully working, isolated test environment in one step.

### Dockerfile Defaults

```dockerfile
# Multi-stage: builder + runtime
FROM python:3.11-slim AS builder
WORKDIR /app
COPY requirements.txt .
RUN pip install --no-cache-dir --user -r requirements.txt

FROM python:3.11-slim AS runtime
WORKDIR /app
# Non-root user — always
RUN useradd -m -u 1000 appuser
COPY --from=builder /root/.local /home/appuser/.local
COPY --chown=appuser:appuser . .
USER appuser
ENV PATH=/home/appuser/.local/bin:$PATH
EXPOSE 8000
CMD ["uvicorn", "app.main:app", "--host", "0.0.0.0", "--port", "8000"]
```

Key rules:
- Multi-stage build always (keep runtime image lean)
- Non-root user (`useradd -m -u 1000 appuser`)
- `COPY --chown` to fix ownership
- `ENV` for runtime config; never hardcode secrets
- Tag images as `{service}:{env}` e.g. `roman-monitor:dev`

### docker-compose.yml Defaults

```yaml
# docker-compose.yml — local dev / test
version: "3.9"
services:
  app:
    build:
      context: .
      target: runtime          # use builder stage during dev
    image: myservice:dev
    ports:
      - "8000:8000"
    environment:
      - DATABASE_URL=postgresql+asyncpg://user:pass@db:5432/mydb
      - LOG_LEVEL=debug
    env_file:
      - .env.local             # gitignored; .env.example committed
    volumes:
      - ./src:/app/src:ro      # hot-reload during dev
    depends_on:
      db:
        condition: service_healthy
    restart: unless-stopped

  db:
    image: postgres:15-alpine
    environment:
      POSTGRES_USER: user
      POSTGRES_PASSWORD: pass
      POSTGRES_DB: mydb
    volumes:
      - pgdata:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U user"]
      interval: 5s
      timeout: 3s
      retries: 5

volumes:
  pgdata:
```

### JupyterLab Container (Notebooks / Astronomy Exploration)

Use when the task involves exploratory data analysis, pyvo/astroquery queries,
FITS/ASDF file inspection, or algorithm prototyping.

```dockerfile
# Dockerfile.jupyter
FROM jupyter/scipy-notebook:python-3.11

USER root
RUN pip install --no-cache-dir \
    astropy \
    pyvo \
    astroquery \
    asdf \
    asdf-astropy \
    pyarrow \
    pandas \
    matplotlib \
    boto3 \
    awswrangler \
    ipywidgets \
    && jupyter lab build --minimize=False

USER ${NB_UID}
WORKDIR /home/jovyan/work
EXPOSE 8888
```

```yaml
# docker-compose.jupyter.yml
version: "3.9"
services:
  jupyter:
    build:
      context: .
      dockerfile: Dockerfile.jupyter
    image: ipac-jupyter:dev
    ports:
      - "8888:8888"
    volumes:
      - ./notebooks:/home/jovyan/work/notebooks
      - ./data:/home/jovyan/work/data
      - ${HOME}/.aws:/home/jovyan/.aws:ro   # mount AWS creds read-only
    environment:
      - JUPYTER_ENABLE_LAB=yes
      - GRANT_SUDO=no
    command: >
      start-notebook.sh
      --NotebookApp.token=''
      --NotebookApp.password=''
      --NotebookApp.open_browser=False
```

**Start:** `docker compose -f docker-compose.jupyter.yml up`
**Access:** http://localhost:8888/lab

### Compose Patterns by Service Type

| Service type | Base image | Port | Compose extras |
|---|---|---|---|
| FastAPI / HTTP API | `python:3.11-slim` | 8000 | `--reload` via `uvicorn` in dev |
| FastMCP server | `python:3.11-slim` | 8001 | stdio mode needs no port |
| JupyterLab (astronomy) | `jupyter/scipy-notebook:python-3.11` | 8888 | mount `~/.aws` read-only |
| Airflow (local test) | `apache/airflow:2.9` | 8080 | use official `docker-compose.yaml` from Airflow docs |
| PostgreSQL / Aurora local | `postgres:15-alpine` | 5432 | healthcheck required |
| Redis | `redis:7-alpine` | 6379 | `redis-server --save ""` for ephemeral dev |

### Quick Commands Reference

```bash
# Build and start (detached)
docker compose up --build -d

# Tail logs
docker compose logs -f app

# Exec into running container
docker compose exec app bash

# Run one-off command
docker compose run --rm app python -m pytest

# Rebuild only one service
docker compose up --build app

# Tear down (keep volumes)
docker compose down

# Tear down + wipe volumes
docker compose down -v
```

---

## Local AWS Development (LocalStack)

Use LocalStack to run Lambda, SQS, S3, and other AWS services locally before
deploying to `us-east-1`. Eliminates round-trips to AWS during development.

### docker-compose with LocalStack

```yaml
# docker-compose.localstack.yml
version: "3.9"
services:
  localstack:
    image: localstack/localstack:3
    ports:
      - "4566:4566"          # unified gateway
    environment:
      - SERVICES=s3,sqs,lambda,events,iam,logs
      - DEBUG=1
      - LAMBDA_EXECUTOR=docker   # run Lambdas in nested containers
      - DOCKER_HOST=unix:///var/run/docker.sock
      - AWS_DEFAULT_REGION=us-east-1
    volumes:
      - "/var/run/docker.sock:/var/run/docker.sock"
      - "./localstack-init:/etc/localstack/init/ready.d"  # init scripts
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:4566/_localstack/health"]
      interval: 10s
      timeout: 5s
      retries: 5
```

### AWS CLI against LocalStack

```bash
# Always use --endpoint-url or set AWS_ENDPOINT_URL
export AWS_ENDPOINT_URL=http://localhost:4566
export AWS_ACCESS_KEY_ID=test
export AWS_SECRET_ACCESS_KEY=test
export AWS_DEFAULT_REGION=us-east-1

# S3
aws s3 mb s3://my-local-bucket
aws s3 cp data.parquet s3://my-local-bucket/

# SQS
aws sqs create-queue --queue-name test-queue
aws sqs send-message --queue-url http://localhost:4566/000000000000/test-queue \
    --message-body '{"key": "value"}'

# Lambda: deploy and invoke
aws lambda create-function \
    --function-name my-func \
    --runtime python3.11 \
    --handler handler.lambda_handler \
    --zip-file fileb://function.zip \
    --role arn:aws:iam::000000000000:role/lambda-role
aws lambda invoke --function-name my-func --payload '{}' response.json
```

### boto3 against LocalStack

```python
import boto3

# Option 1: endpoint_url per client
s3 = boto3.client("s3", endpoint_url="http://localhost:4566",
                  region_name="us-east-1",
                  aws_access_key_id="test",
                  aws_secret_access_key="test")

# Option 2: AWS_ENDPOINT_URL env var (boto3 >= 1.28, recommended)
# Just set the env var; no code changes needed vs production boto3 usage
import os
os.environ["AWS_ENDPOINT_URL"] = "http://localhost:4566"
s3 = boto3.client("s3")  # picks up endpoint automatically
```

### Init script pattern (auto-create resources on startup)

```bash
# localstack-init/01-setup.sh  (runs when LocalStack is ready)
#!/bin/bash
set -e
awslocal s3 mb s3://soc-delivery-local
awslocal sqs create-queue --queue-name soc-events
awslocal s3api put-bucket-notification-configuration \
    --bucket soc-delivery-local \
    --notification-configuration '{
        "QueueConfigurations": [{
            "QueueArn": "arn:aws:sqs:us-east-1:000000000000:soc-events",
            "Events": ["s3:ObjectCreated:*"]
        }]
    }'
echo "LocalStack init complete"
```

### Key pitfalls
- `LAMBDA_EXECUTOR=docker` requires Docker socket mount; use `local` for simpler functions
- LocalStack free tier doesn't support all services (RDS, EKS) — use `localstack-pro` or mock at the DB layer instead
- Use `awslocal` CLI wrapper (pip install awscli-local) to skip `--endpoint-url` every time
- Terraform + LocalStack: set `skip_credentials_validation = true` and `skip_requesting_account_id = true`

---

## Grafana / Observability Dashboards

Use Grafana when CloudWatch alone isn't enough — K8s metrics, cross-service
dashboards, or sharing with non-AWS teams.

### docker-compose addition for local Grafana

```yaml
  grafana:
    image: grafana/grafana:10-ubuntu
    ports:
      - "3000:3000"
    environment:
      - GF_SECURITY_ADMIN_PASSWORD=admin
      - GF_USERS_ALLOW_SIGN_UP=false
    volumes:
      - ./grafana/provisioning:/etc/grafana/provisioning
      - grafana-data:/var/lib/grafana

volumes:
  grafana-data:
```

### Provisioning datasources (CloudWatch)

```yaml
# grafana/provisioning/datasources/cloudwatch.yaml
apiVersion: 1
datasources:
  - name: CloudWatch
    type: cloudwatch
    jsonData:
      defaultRegion: us-east-1
      authType: default    # uses instance role / env vars / ~/.aws
    isDefault: true
```

### Key dashboard panels for IPAC services

```json
// Jenkins CPU panel (CloudWatch metric)
{
  "type": "timeseries",
  "title": "Jenkins Master CPU",
  "targets": [{
    "datasource": "CloudWatch",
    "namespace": "AWS/EC2",
    "metricName": "CPUUtilization",
    "dimensions": { "InstanceId": "${jenkins_instance_id}" },
    "statistic": "Average",
    "period": 300
  }],
  "thresholds": [{"value": 40, "color": "red"}]
}
```

```python
# Programmatic dashboard via Grafana API
import httpx

GRAFANA = "http://localhost:3000"
HEADERS = {"Authorization": "Bearer <api-key>", "Content-Type": "application/json"}

# Push a dashboard JSON
resp = httpx.post(f"{GRAFANA}/api/dashboards/db",
                  headers=HEADERS,
                  json={"dashboard": dashboard_dict, "overwrite": True})

# List dashboards
dashboards = httpx.get(f"{GRAFANA}/api/search", headers=HEADERS).json()
```

### Prometheus + Grafana for EKS (kube-prometheus-stack)

```bash
# Install via Helm (Emmanuel's standard K8s stack)
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm upgrade --install kube-prometheus prometheus-community/kube-prometheus-stack \
    --namespace monitoring --create-namespace \
    --set grafana.adminPassword=admin \
    --set prometheus.prometheusSpec.retention=15d
```

### SOC monitor dashboard checklist

| Panel | Metric source | Alert threshold |
|---|---|---|
| Files received / hour | Aurora PG: `soc_files` | < 1/hr during expected delivery |
| Validation failure rate | Aurora PG: `status='failed'` | > 5% → page |
| SQS queue depth | CloudWatch SQS | > 100 messages |
| FastMCP pod restarts | Kubernetes | > 2 in 10 min |
| Aurora connections | CloudWatch RDS | > 80% of max |

---

## Operational Defaults

- **Dockerfile**: multi-stage build; non-root user; `COPY --chown`
- **Helm**: always set `resources.requests/limits`; `readinessProbe` required
- **AWS IAM**: least-privilege; use IRSA (IAM Roles for Service Accounts) on EKS
- **S3**: versioning on; block public access; lifecycle for old versions
- **RDS/Aurora**: connection pooling via PgBouncer or `asyncpg` pool; never open to 0.0.0.0/0
- **Secrets**: never in env files committed to git; use `.env.example` pattern
- **CloudFormation**: always use Change Sets before updating production stacks; enable termination protection on critical stacks
- **Custodian**: dry-run (`--dryrun`) before deploying any destructive action policy; store policy output to S3

---

## Additional Tooling (Starred / Validated)

### aws-vault — AWS Credential Management

```bash
brew install --cask aws-vault

# Store credentials (uses macOS Keychain)
aws-vault add my-profile

# Execute a command with temporary credentials (MFA-aware)
aws-vault exec my-profile -- aws s3 ls

# Open a browser console session
aws-vault login my-profile

# List stored profiles
aws-vault list
```

Use instead of long-lived keys in `~/.aws/credentials`. Supports MFA tokens,
assume-role chains, and session TTL. Pairs with IRSA on EKS.

---

### mountpoint-s3 — Mount S3 as a POSIX Filesystem

```bash
brew install --cask mountpoint-s3   # macOS
# Linux: download from https://github.com/awslabs/mountpoint-s3/releases

# Mount a bucket read-only
mount-s3 my-bucket /mnt/my-bucket

# With custom prefix (mount a "directory")
mount-s3 my-bucket /mnt/catalogs --prefix catalogs/

# Unmount
umount /mnt/my-bucket
```

Useful for EKS pods that need POSIX access to S3 objects (e.g., reading HATS
catalogs or large FITS files without streaming code). Backed by AWS's official
FUSE driver — outperforms `s3fs` for sequential read.

---

### apprise — Unified Push Notifications

Send alerts to Slack, PagerDuty, email, SNS, iOS (Bark), Telegram, and 70+
other platforms from a single Python call. Complement to CloudWatch SNS alarms.

```python
# pip install apprise
import apprise

apobj = apprise.Apprise()

# Add destinations (one or more)
apobj.add("slack://token/channel")
apobj.add("sns://AccessKeyID/SecretAccessKey/us-east-1/TopicARN")
apobj.add("bark://DeviceKey/")       # iOS Bark push
apobj.add("mailto://user:pass@gmail.com")

# Send
apobj.notify(
    title="SOC File Delivery Alert",
    body="5 files failed validation in the last hour",
)
```

CLI usage:

```bash
apprise -b "Deployment complete" \
    "slack://TOKEN/CHANNEL" \
    "sns://KEY/SECRET/us-east-1/TOPIC_ARN"
```

---

### datamodel-code-generator — Pydantic from OpenAPI/JSON Schema

```bash
pip install datamodel-code-generator

# Generate Pydantic v2 models from OpenAPI spec
datamodel-codegen \
    --input api.yaml \
    --input-file-type openapi \
    --output models.py \
    --target-python-version 3.11 \
    --use-annotated \
    --field-constraints

# From JSON Schema
datamodel-codegen \
    --input schema.json \
    --input-file-type jsonschema \
    --output models.py

# From URL (live spec)
datamodel-codegen \
    --url https://api.example.com/openapi.json \
    --output models.py
```

Use to bootstrap Pydantic models for FastAPI/FastMCP when an OpenAPI or JSON
Schema spec already exists. Avoids hand-writing dozens of model classes.

---

### astronomer/agents — Airflow AI Agent Tooling

Astronomer's agents library adds MCP-aware AI workflows to Apache Airflow DAGs.
Use when building agentic data pipelines that call LLMs or tools mid-DAG.

```python
# pip install astronomer-agents
from astronomer.agents import AgentTask
from airflow import DAG
from datetime import datetime

with DAG("ai_pipeline", start_date=datetime(2025, 1, 1)) as dag:
    classify = AgentTask(
        task_id="classify_files",
        agent_config={
            "model": "claude-sonnet-4-6",
            "tools": ["mcp://roman-soc-monitor/list_recent_files"],
        },
        prompt="Classify the following Roman SOC files by priority: {{ ti.xcom_pull('list_files') }}",
    )
```

- GitHub: https://github.com/astronomer/agents
- Pairs with FastMCP server pattern — expose tools via MCP, call from DAG agents

---

## References

- `references/mcp-patterns.md` — FastMCP boilerplate, tool patterns, error handling
- `references/aws-defaults.md` — AWS resource naming, tagging conventions, CloudFormation/Terraform/Custodian snippets
- `references/kubernetes-manifests.md` — Deployment, Service, HPA, Ingress templates for EKS
- `references/astronomy-patterns.md` — pyvo/astroquery patterns, FITS/ASDF/Parquet I/O, VO service discovery (**read for any astronomy task**)

Read the relevant reference file when generating code for that domain.
