# science-tools.md — Science Analysis Software Patterns

> **Always call Context7 for each library before writing code.**
> These tools evolve faster than protocols — API changes between minor versions.

---

## §SED — SED Fitting and Photometric Redshifts

### RAIL — Rubin/DESC photo-z pipeline framework

RAIL (Redshift Assessment Infrastructure Layers) is the Rubin DESC standard for
running, comparing, and evaluating photo-z estimators. The framework separates
*data ingestion*, *estimation*, and *evaluation* — allowing any estimator to slot
in with the same interface.

```python
# pip install pz-rail lsstdesc-rail
import rail
from rail.core.stage import RailStage
from rail.core.data import TableHandle
from rail.estimation.algos.bpz_lite import BPZliteEstimator
from rail.evaluation.metrics.pit import PIT

# 1. Create data store
DS = RailStage.data_store
DS.__class__.allow_overwrite = True

# 2. Load training and test photometry (astropy Table or pandas DataFrame)
# Columns must include mag_u, mag_g, mag_r, mag_i, mag_z, mag_y
# plus magerr_* and redshift (for training set)

# 3. Run a photo-z estimator
estimator = BPZliteEstimator.make_stage(
    name="bpz",
    hdf5_groupname="photometry",
    output="output_bpz.hdf5"
)
estimator.fit_model()         # train on training set
results = estimator.estimate(test_data)   # run on test set

# 4. Evaluate
pit = PIT(results, true_redshifts)
pit_array, pit_out_rate, qc_mask = pit.evaluate()

# 5. Compare multiple estimators (RAIL's key feature)
# Run BPZ, EAZY, FlexZBoost, etc. with identical interface
# then compare PIT, sigma_68, outlier fraction
```

**Key RAIL estimators available:**
- `BPZliteEstimator` — template fitting
- `FZBoost` — boosted decision tree on colors
- `GPzEstimator` — Gaussian process
- `DNNEstimator` — deep neural network
- `EAZYEstimator` — EAZY wrapper

### LePHARE — SED template fitter

Standard for Euclid, COSMOS, CFHLS photo-z. Template-based, handles stars + galaxies + AGN.

```python
# pip install lephare
import lephare as lp
import numpy as np

# Configure: filter list + template libraries
config = {
    "FILTER_LIST": "euclid_vis,euclid_y,euclid_j,euclid_h",
    "ZPHOTLIB":    "BC03_CHAB",   # Bruzual & Charlot templates
    "ADD_EMLINES": 1,
    "PHYS_LIB":    "PEGASE2",
}

# Prepare photometric input (flux + flux_err in μJy, -99 for missing)
cat = lp.PhotoCat(
    fluxes=flux_array,      # (N_obj, N_bands)
    errors=err_array,
    filter_names=["EUCLID_VIS", "EUCLID_Y", "EUCLID_J", "EUCLID_H"],
)

# Run
runner = lp.Runner(config)
results = runner.run(cat)   # returns DataFrame with z_best, z68_low, z68_high

# Access full P(z) PDF
pdz = results.pdz   # (N_obj, N_z) array
```

### CIGALE — Bayesian galaxy SED fitting

CIGALE fits the full SED from UV to radio; good for stellar mass, SFR, AGN fraction.

```python
# pip install pcigale
# CIGALE uses a configuration file + command-line interface
# but can be called from Python

import pcigale

# Create config
config = pcigale.Configuration()
config.set_module_list([
    "sfhdelayed",       # star formation history
    "bc03",             # stellar population
    "nebular",          # emission lines
    "dustatt_modified_starburst",
    "dale2014",         # dust emission
    "redshifting",
])

# Input: multi-band photometry catalog (CSV)
# Output: best-fit parameters + probability distributions
# Run: pcigale run
```

### EAZY — Fast photo-z for galaxy surveys

```python
# pip install eazy
import eazy

# Initialize with filter set
photoz = eazy.PhotoZ(
    param_file="zphot.param",
    translate_file="zphot.translate",
)

# Set photometry (magnitudes or fluxes)
photoz.param["CATALOG_FILE"] = "photometry.cat"

# Run
photoz.fit_catalog(
    iz=np.arange(photoz.NOBJ),
    n_proc=4,
)

# Results
z_phot = photoz.zgrid[np.argmin(photoz.chi2_fit, axis=1)]
```

### BAGPIPES — Bayesian galaxy SED fitting

```python
# pip install bagpipes
import bagpipes as pipes
import numpy as np

# Define star formation history model
sfh = {
    "tau":     {"age": (0.1, 15.0),     # Gyr
                "tau": (0.1, 10.0),
                "massformed": (1., 12.),
                "metallicity": (0., 2.5)},
}

# Define dust + nebular
dust = {"type": "Calzetti", "Av": (0., 3.0)}
neb  = {"logU": (-4., -1.)}

fit_instructions = {"sfh": sfh, "dust": dust, "nebular": neb,
                    "redshift": (0., 5.)}

# Observation: fluxes in μJy + errors, filter list
obs = {"photometry": flux_with_errors,
       "photometry_units": "mJy",
       "filt_list": filter_paths}

# Run fitting
galaxy = pipes.fit(obs, fit_instructions)
galaxy.fit(verbose=True)
galaxy.plot_sfh_posterior()
```

### AGNfitter — Bayesian AGN SED decomposition

```python
# github.com/GabrielaCR/AGNfitter
# Input: multi-band photometry from radio to X-rays
# Components: accretion disk, torus, stellar, cold dust, X-ray corona
# Output: posterior distributions for all components

# Minimal call:
from AGNfitter.run import run
run(
    catalog_path="input_photometry.fits",
    output_dir="output/",
    NCPU=4,
    N_samplers=100,
    N_iterations=500,
)
```

---

## §Source — Source Extraction and Deblending

### sep — Fast source extraction (SExtractor wrapper)

```python
# pip install sep
import sep
import numpy as np
from astropy.io import fits

# Load image
data = fits.getdata("image.fits").astype(np.float64)
bkg  = sep.Background(data)
data_sub = data - bkg

# Detect sources (threshold = 3σ above background)
objects = sep.extract(data_sub, thresh=3.0, err=bkg.globalrms)
print(f"Detected {len(objects)} sources")

# Aperture photometry
flux, fluxerr, flag = sep.sum_circle(
    data_sub, objects["x"], objects["y"],
    r=5.0,        # aperture radius in pixels
    err=bkg.globalrms
)
```

### photutils — Astropy-affiliated photometry

```python
from photutils.detection import DAOStarFinder, IRAFStarFinder
from photutils.aperture import CircularAperture, aperture_photometry
from photutils.background import Background2D, MedianBackground
from photutils.segmentation import SourceExtractor

# Background estimation
bkg_estimator = MedianBackground()
bkg = Background2D(data, (50, 50), filter_size=(3, 3),
                   bkg_estimator=bkg_estimator)

# Source detection
daofind = DAOStarFinder(fwhm=3.0, threshold=5.*bkg.background_rms_median)
sources = daofind(data - bkg.background)

# Aperture photometry
positions = list(zip(sources["xcentroid"], sources["ycentroid"]))
apertures = CircularAperture(positions, r=5.0)
phot_table = aperture_photometry(data - bkg.background, apertures)

# Segmentation-based (better for crowded fields)
seg = SourceExtractor()
seg_result = seg.extract(
    data - bkg.background,
    threshold=5.0 * bkg.background_rms_median,
    npixels=10,
    deblend=True,
)
```

### scarlet / scarlet-lite — Multi-band deblending

Used in Rubin LSST pipeline for separating overlapping sources across bands.

```python
# pip install scarlet-lite
import scarlet

# scarlet deblending requires:
# 1. Multi-band image data (n_bands, ny, nx)
# 2. PSF for each band
# 3. Detection footprints

# Initialize observations
obs = scarlet.Observation(
    images=multiband_images,    # (n_bands, ny, nx)
    psfs=psfs,                  # (n_bands, psf_ny, psf_nx)
    weights=1./variance_images,
    channels=band_names,
)

# Initialize sources at detected positions
sources = [
    scarlet.ExtendedSource(
        model_frame,
        center=pos,
        observations=obs,
    )
    for pos in source_positions
]

# Fit
blend = scarlet.Blend(sources, obs)
blend.fit(200, e_rel=1e-4)   # 200 iterations

# Extract deblended models
models = [src.get_model() for src in blend.sources]
```

### PSF homogenization across missions

When building multi-mission photometric catalogs, all images should be convolved
to a common (worst) PSF before forced photometry:

```python
from astropy.convolution import convolve_fft
from photutils.psf import EPSFBuilder

# Estimate PSF from stars
# (using photutils PSF building from isolated stars)
from photutils.psf import extract_stars
from astropy.nddata import NDData

nddata = NDData(data=science_image)
stars  = extract_stars(nddata, star_catalog, size=25)
epsf_builder = EPSFBuilder(oversampling=4, maxiters=3)
epsf, fitted_stars = epsf_builder(stars)

# Homogenize PSF: convolve narrow-PSF image to match widest PSF
# Target PSF: Rubin seeing ~0.7"; Roman: 0.11" → convolve Roman to Rubin PSF
kernel = make_matching_kernel(source_psf, target_psf)
convolved = convolve_fft(roman_image, kernel)
```

---

## §Alerts — Alert Stream and Time-Domain Integration

### tom_toolkit — Target and Observation Manager

TOM Toolkit is the standard framework for multi-mission follow-up coordination.
It manages targets, observations, data products, and broker integrations.

```python
# pip install tomtoolkit
# Usually run as a Django web app, but can be used as a library

from tom_targets.models import Target, TargetExtra
from tom_observations.facilities import get_service_class
from tom_dataproducts.models import DataProduct

# Create a target from alert
target = Target.objects.create(
    name="AT2025abc",
    ra=83.82,
    dec=-5.39,
    type="SIDEREAL",
)
TargetExtra.objects.create(target=target, key="classification", value="SN Ia")

# Submit observation request
LCOService = get_service_class("LCO")
obs_params = {
    "target_id": target.id,
    "facility": "LCO",
    "observation_type": "PHOTOMETRIC_SEQUENCE",
    "filters": ["g", "r", "i"],
    "exposure_time": 300,
}
# LCOService.submit_observation(obs_params)
```

### Kafka alert stream (Rubin LSST / ZTF)

```python
# pip install confluent-kafka fastavro
from confluent_kafka import Consumer, KafkaError
import fastavro, io

conf = {
    "bootstrap.servers": "alert-stream.lsst.org:9092",
    "group.id":          "my_filter_group",
    "auto.offset.reset": "earliest",
}
consumer = Consumer(conf)
consumer.subscribe(["lsst.alerts.v1"])

while True:
    msg = consumer.poll(timeout=1.0)
    if msg is None: continue
    if msg.error(): break

    # Deserialize Avro alert packet
    reader = fastavro.reader(io.BytesIO(msg.value()))
    for alert in reader:
        ra, dec       = alert["ra"], alert["decl"]
        mag           = alert["diaSource"]["psFlux"]   # in nJy
        filter_       = alert["diaSource"]["band"]
        classification = alert.get("classifications", [])
        print(f"{ra:.4f} {dec:.4f} {filter_} {mag:.2f}")
```

### Cross-matching alerts against archival catalogs (TAP UPLOAD)

```python
# Real-time alert → TAP UPLOAD → archival cross-match
import pyvo as vo
from astropy.table import Table

irsa_tap = vo.dal.TAPService("https://irsa.ipac.caltech.edu/TAP")

# Batch of alerts (accumulate before querying)
alert_table = Table({
    "alert_id": ["AT2025abc", "AT2025def"],
    "ra":  [83.82, 84.05],
    "dec": [-5.39, -5.12],
})

result = irsa_tap.run_sync("""
    SELECT a.alert_id, a.ra, a.dec, c.cntr, c.j_m, c.k_m
    FROM TAP_UPLOAD.alerts AS a
    LEFT JOIN fp_psc AS c
      ON 1=CONTAINS(POINT('ICRS', c.ra, c.dec),
                    CIRCLE('ICRS', a.ra, a.dec, 0.0014))
""", uploads={"alerts": alert_table})
```

---

## §ML — Machine Learning with VO Data

### Building training sets from VO catalogs

```python
import lsdb
import numpy as np
from astropy.table import Table

# Cross-match labeled spectroscopic sample with photometric catalog
spec  = lsdb.read_hats("s3://data.lsdb.io/hats/sdss_specobj/")  # labeled
photo = lsdb.read_hats("s3://data.lsdb.io/hats/rubin_dp1/")

# Cross-match to get labels
training = photo.crossmatch(spec, n_neighbors=1, radius_arcsec=1.0)
df = training.compute()

# Feature matrix
features = df[["mag_u", "mag_g", "mag_r", "mag_i", "mag_z",
               "mag_g-mag_r", "mag_r-mag_i"]].values
labels   = df["z"].values  # spectroscopic redshifts

# Train/test split respecting spatial structure (avoid positional leakage)
# Use random HEALPix-based split rather than random row split
from sklearn.model_selection import train_test_split
healpix_ids = df["Norder"].values   # HEALPix partition ID
train_idx, test_idx = train_test_split(
    np.arange(len(df)), test_size=0.2,
    stratify=(healpix_ids % 5)  # spatial stratification
)
```

### Astronomy foundation models

```python
# AstroCLIP: contrastive image+spectrum model
# pip install astroclip
from astroclip import AstroCLIP

model = AstroCLIP.from_pretrained("astroclip-base")
# Embed galaxy images
image_embeddings = model.encode_image(galaxy_images)   # (N, 512)
# Embed spectra
spec_embeddings  = model.encode_spectrum(spectra)      # (N, 512)

# Use embeddings for downstream tasks: photo-z, morphology classification, anomaly detection
```

### Serving model outputs back to VO

When publishing ML results (photo-z PDFs, morphological classifications, etc.),
package them as VO-compatible products:

```python
from astropy.table import Table
from astropy.io.votable import from_table, writeto

results = Table({
    "source_id":   source_ids,
    "ra":          ra_array,
    "dec":         dec_array,
    "z_phot":      z_best,
    "z_phot_err":  z_err,
    "class_star":  star_prob,   # P(star)
    "class_galaxy": gal_prob,   # P(galaxy)
})

# Add UCDs for VO interoperability
results["ra"].unit    = "deg"
results["dec"].unit   = "deg"
results["z_phot"].description = "Photometric redshift (best fit)"

# Write as VOTable (VO-compatible)
writeto(from_table(results), "my_photo_z_catalog.vot")

# Also write as Parquet for pipeline use
results.write("my_photo_z_catalog.parquet")
```

### Quality flags and data quality masks

```python
# Gaia DR3 quality selection (common pattern)
result_df = result_df[
    (result_df["ruwe"] < 1.4) &                    # astrometric quality
    (result_df["phot_g_mean_flux_over_error"] > 50) &  # S/N
    (result_df["astrometric_excess_noise"] < 2.0)   # astrometric outliers
]

# Rubin LSST quality flags (bitmask interpretation)
from astropy.nddata.bitmask import bitfield_to_boolean_mask, interpret_bit_flags
good_mask = bitfield_to_boolean_mask(
    flag_array,
    ignore_flags=interpret_bit_flags("~BAD,~SATURATED,~CR,~INTERPOLATED"),
    good_mask_value=True,
)

# Euclid MER catalog quality selection
result_df = result_df[
    result_df["vis_mag_err"] < 0.2 &
    result_df["photo_z_quality"] == "GOOD"  # quality label varies by release
]
```
