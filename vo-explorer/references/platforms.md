# platforms.md — Science Platform & Distributed Compute Patterns

## Platform Comparison

| Platform | Scheduler | Auth | S3? | Internet in compute? |
|---|---|---|---|---|
| Local / laptop | LocalCluster | n/a | via boto3 | yes |
| JupyterHub (generic) | Dask gateway | token | yes | yes |
| Rubin RSP | Dask gateway (roadmap) | RSP JWT | n/a (IDF storage) | restricted |
| NOIRLab Astro Data Lab | Local + Dask | API token | yes | yes |
| HPC (Slurm/PBS) | dask-jobqueue | SSH/module | often blocked | no (compute nodes) |
| AWS EKS | dask-kubernetes | IRSA/IAM | native | yes |
| AWS EMR | Spark / Dask | IAM | native | yes |

---

## Dask: Full Config Examples

### Local (development)
```python
from dask.distributed import Client, LocalCluster

cluster = LocalCluster(
    n_workers=4,
    threads_per_worker=2,
    memory_limit="8GB",
    dashboard_address=":8787",
)
client = Client(cluster)
print(client.dashboard_link)
```

### Slurm HPC
```python
from dask_jobqueue import SLURMCluster

cluster = SLURMCluster(
    queue="science",
    cores=8,
    memory="32 GB",
    walltime="04:00:00",
    job_script_prologue=[
        "module load python/3.11",
        "source /path/to/venv/bin/activate",
        # Pre-stage data to local scratch:
        "cp /nfs/data/catalog.parquet $TMPDIR/",
    ],
    worker_extra_args=["--lifetime", "3h50m", "--lifetime-stagger", "4m"],
    log_directory="/nfs/logs/dask/",
)
cluster.scale(jobs=20)
client = Client(cluster)
```

**HPC pre-staging pattern** (when S3 blocked on compute nodes):
```bash
# Submit script — stage before Dask starts
#SBATCH --nodes=1 --ntasks=1
aws s3 sync s3://bucket/hats/gaia_dr3/ /scratch/$USER/gaia_dr3/ \
    --quiet --no-progress
python my_analysis.py --catalog /scratch/$USER/gaia_dr3/
```

### Kubernetes / EKS (Emmanuel's primary)
```python
from dask_kubernetes.operator import KubeCluster, make_cluster_spec

spec = make_cluster_spec(
    name="vo-analysis",
    image="ipac/jupyter:dev",
    n_workers=20,
    resources={
        "requests": {"memory": "8Gi", "cpu": "4"},
        "limits":   {"memory": "16Gi", "cpu": "8"},
    },
    env={"AWS_DEFAULT_REGION": "us-east-1"},
    # Mount AWS credentials via IRSA (preferred over env vars)
    service_account_name="dask-worker-sa",
)
cluster = KubeCluster(custom_cluster_spec=spec)
client  = Client(cluster)
```

### Dask Gateway (managed JupyterHub platforms)
```python
from dask_gateway import Gateway

gateway = Gateway()
options = gateway.cluster_options()
options.worker_cores   = 4
options.worker_memory  = 8      # GB
options.image          = "pangeo/pangeo-notebook:latest"

cluster = gateway.new_cluster(options)
cluster.scale(10)
client = Client(cluster)
print(cluster.dashboard_link)
```

---

## Rubin RSP: Auth and Access Patterns

### Internal (from RSP notebook — zero config)
```python
from lsst.rsp import get_tap_service
tap = get_tap_service("tap")      # → pre-authenticated TAPService
results = tap.search("SELECT TOP 10 * FROM dp02_dc2_catalogs.Object")
```

### External (pyvo from laptop or other platform)
```python
import pyvo, os

token = open(os.path.expanduser("~/.rsp-tap.token")).read().strip()
# NEVER commit token; rotate regularly

cred = pyvo.auth.CredentialStore()
cred.set_password("x-oauth-basic", token)
credential = cred.get("ivo://ivoa.net/sso#BasicAA")

tap = pyvo.dal.TAPService(
    "https://data.lsst.cloud/api/tap",
    session=credential,
)
tap.search("SELECT TOP 5 objectId, ra, dec FROM dp02_dc2_catalogs.Object")
```

### RSP-specific table names
```sql
-- DP1 (commissioning camera, 7 fields)
dp01_dc2_catalogs.Object
dp01_dc2_catalogs.Source

-- DP2 (simulated DC2 data)
dp02_dc2_catalogs.Object
dp02_dc2_catalogs.ForcedSource
dp02_dc2_catalogs.CcdVisit
dp02_dc2_catalogs.TruthSummary

-- ObsTAP images
ivoa.ObsCore
```

---

## NOIRLab Astro Data Lab

```python
# Option 1: Direct pyvo with Data Lab TAP
tap = pyvo.dal.TAPService("https://datalab.noirlab.edu/tap")
# Public access; no token needed for public tables

# Option 2: dl Python package (Data Lab native)
# pip install datalab
from dl import queryClient as qc
result = qc.query(sql="SELECT ra,dec,g FROM ls_dr10.tractor LIMIT 100",
                  fmt="pandas")
```

---

## S3 Performance Optimisation

### Use pyarrow native S3 filesystem (avoid s3fs for bulk reads)
```python
import pyarrow.fs as pafs
import pyarrow.parquet as pq

fs = pafs.S3FileSystem(
    region="us-east-1",
    # Credentials from env / IAM role / IRSA — don't hardcode
)

tbl = pq.read_table(
    "bucket/catalog/",
    filesystem=fs,
    columns=["ra", "dec", "mag_g"],
    filters=[("mag_g", "<", 22.0)],   # row-group predicate pushdown
    use_threads=True,
)
```

### Dask Parquet from S3 — key settings
```python
import dask.dataframe as dd

ddf = dd.read_parquet(
    "s3://bucket/catalog/",
    engine="pyarrow",
    filesystem="arrow",               # use pyarrow's native S3, not fsspec
    columns=["ra", "dec", "mag_g"],
    filters=[[("mag_g", "<", 22.0)]],
    calculate_divisions=False,        # skip for large datasets
)
```

### Zarr from S3
```python
import zarr
import s3fs   # Zarr still uses s3fs; watch zarr v3 for native pyarrow fs

fs = s3fs.S3FileSystem(anon=False, region_name="us-east-1")
store = s3fs.S3Map(root="bucket/cube.zarr", s3=fs)
z = zarr.open(store, mode="r")

# xarray preferred interface
import xarray as xr
ds = xr.open_zarr("s3://bucket/cube.zarr",
                  storage_options={"anon": False, "region_name": "us-east-1"})
```

### Performance rules of thumb
1. **Predicate pushdown first** — `filters=` can skip 90%+ of I/O on sorted data
2. **Column pruning** — never read columns you don't need
3. **Right-size partitions** — aim for 100–300 MB uncompressed per Parquet file
4. **Same region** — compute and S3 in same AWS region (huge latency difference)
5. **HATS is already right-sized** — don't repartition HATS catalogs after reading
6. **Avoid row-by-row operations** — always use vectorised/Dask operations

---

## Docker: Astronomy JupyterLab Stack

Standard container for local development and CI that mirrors science platform envs:

```dockerfile
FROM jupyter/scipy-notebook:python-3.11

USER root
RUN pip install --no-cache-dir \
    astropy \
    pyvo \
    astroquery \
    asdf asdf-astropy \
    pyarrow \
    lsdb \
    hats hats-import \
    zarr xarray \
    mocpy \
    ipyaladin \
    jdaviz \
    firefly-client \
    specutils lightkurve \
    dask[distributed,dataframe] dask-jobqueue dask-kubernetes \
    pyiceberg[s3,glue] \
    boto3 s3fs \
    datashader holoviews bokeh \
    && jupyter lab build --minimize=False

USER ${NB_UID}
WORKDIR /home/jovyan/work
```

```yaml
# docker-compose.astro.yml
services:
  jupyter:
    build: .
    ports:
      - "8888:8888"
    environment:
      - JUPYTER_ENABLE_LAB=yes
      - FIREFLY_URL=http://firefly:8080/firefly
      - AWS_DEFAULT_REGION=us-east-1
    env_file: .env.local
    volumes:
      - ./notebooks:/home/jovyan/work/notebooks
      - ${HOME}/.aws:/home/jovyan/.aws:ro

  firefly:
    image: ipac/firefly:latest
    ports:
      - "8080:8080"
    environment:
      - MAX_JVM_SIZE=8G
```

---

## Workflow Orchestration for Science Pipelines

### Parsl — parallel scripting for HPC and cloud

Parsl is widely used in DESC, CTA, and other large astronomy collaborations.
It decorates Python functions as parallel tasks that run on any backend.

```python
import parsl
from parsl.app.app import python_app, bash_app
from parsl.configs.htex_local import fresh_config

parsl.load(fresh_config())  # local threads for dev; swap for Slurm/K8s

@python_app
def tap_query(service_url, adql, maxrec=100000):
    import pyvo as vo
    tap = vo.dal.TAPService(service_url)
    return tap.run_sync(adql, maxrec=maxrec).to_table()

@python_app
def crossmatch(catalog_a, catalog_b, radius_arcsec=1.0):
    import lsdb
    a = lsdb.from_astropy_table(catalog_a)
    b = lsdb.from_astropy_table(catalog_b)
    return a.crossmatch(b, n_neighbors=1, radius_arcsec=radius_arcsec).compute()

# Submit parallel TAP queries
futures = [
    tap_query("https://irsa.ipac.caltech.edu/TAP", q)
    for q in adql_queries
]
results = [f.result() for f in futures]
```

### Prefect — modern workflow for production pipelines

Prefect is used in Keck pipeline redesign and similar observatory systems.
Better observability and retry logic than Parsl; good for long-running production.

```python
from prefect import flow, task
from prefect.tasks import task_input_hash
from datetime import timedelta

@task(cache_key_fn=task_input_hash, cache_expiration=timedelta(hours=24))
def query_catalog(service_url: str, adql: str) -> dict:
    import pyvo as vo
    tap = vo.dal.TAPService(service_url)
    result = tap.run_sync(adql, maxrec=1000000)
    return result.to_table().to_pandas().to_dict()

@task
def crossmatch_catalogs(catalog_a: dict, catalog_b: dict) -> dict:
    import lsdb, pandas as pd
    a = lsdb.from_dataframe(pd.DataFrame(catalog_a), ra_col="ra", dec_col="dec")
    b = lsdb.from_dataframe(pd.DataFrame(catalog_b), ra_col="ra", dec_col="dec")
    return a.crossmatch(b, radius_arcsec=1.0).compute().to_dict()

@flow(name="multi-mission-pipeline")
def run_pipeline(field_ra: float, field_dec: float, radius_deg: float):
    gaia   = query_catalog("https://gea.esac.esa.int/tap-server/tap",
                           f"SELECT ... WHERE CONTAINS(...) = 1")
    rubin  = query_catalog("https://data.lsst.cloud/api/tap",
                           f"SELECT ... WHERE CONTAINS(...) = 1")
    merged = crossmatch_catalogs(gaia, rubin)
    return merged

if __name__ == "__main__":
    run_pipeline(field_ra=150.0, field_dec=2.2, radius_deg=0.5)
```

### Snakemake — reproducible file-based pipelines

Common in radio astronomy and multi-step reduction pipelines.

```python
# Snakefile
rule query_gaia:
    output: "data/{field}_gaia.parquet"
    run:
        import pyvo as vo, pyarrow.parquet as pq
        field_ra, field_dec = float(wildcards.field.split("_"))
        tap = vo.dal.TAPService("https://gea.esac.esa.int/tap-server/tap")
        result = tap.run_sync(f"SELECT ... CIRCLE('ICRS', {field_ra}, {field_dec}, 0.5)")
        pq.write_table(result.to_table().to_pyarrow(), output[0])

rule crossmatch:
    input:
        gaia="data/{field}_gaia.parquet",
        rubin="data/{field}_rubin.parquet"
    output: "data/{field}_matched.parquet"
    run:
        import lsdb, pyarrow.parquet as pq
        g = lsdb.read_hats(input.gaia)
        r = lsdb.read_hats(input.rubin)
        pq.write_table(
            g.crossmatch(r, radius_arcsec=1.0).compute().to_arrow(),
            output[0]
        )
```

### Pipeline retry patterns for VO queries

VO archives have transient failures. Always wrap with retry logic:

```python
import time, pyvo as vo
from functools import wraps

def with_retry(n=3, backoff=2.0):
    def decorator(fn):
        @wraps(fn)
        def wrapper(*args, **kwargs):
            for attempt in range(n):
                try:
                    return fn(*args, **kwargs)
                except Exception as e:
                    if attempt == n - 1:
                        raise
                    wait = backoff ** attempt
                    print(f"Attempt {attempt+1} failed: {e}. Retrying in {wait}s")
                    time.sleep(wait)
        return wrapper
    return decorator

@with_retry(n=3, backoff=5.0)
def safe_tap_query(tap, adql, **kwargs):
    return tap.run_sync(adql, **kwargs).to_table()
```
