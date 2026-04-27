# VO Service Catalog

Curated IVOA service endpoints by archive and waveband. Verified as of 2026-04.

---

## TAP Services

| Archive | Base URL | Tables of interest |
|---------|----------|--------------------|
| IRSA | `https://irsa.ipac.caltech.edu/TAP` | `fp_psc`, `allwise_p3as_psd`, `neowiser_p1bs_psd`, `ivoa.obscore` |
| MAST | `https://mast.stsci.edu/tap/sync` | `dbo.observations`, `ivoa.obscore` |
| VizieR | `https://tapvizier.cds.unistra.fr/TAPVizieR/tap` | thousands; use `TAP_SCHEMA.tables` to discover |
| NED | `https://ned.ipac.caltech.edu/tap` | `NEDTAP.ned_objdir`, `NEDTAP.crossids` |
| SIMBAD | `https://simbad.u-strasbg.fr/simbad/tap/sync` | `basic`, `ident`, `flux`, `allfluxes` |
| Gaia (ESA) | `https://gea.esac.esa.int/tap-server/tap` | `gaiadr3.gaia_source`, `gaiadr3.astrophysical_parameters` |
| XMM-Newton | `https://nxsa.esac.esa.int/nxsa-sl/tap` | `xsa.v_all_observations` |
| Chandra | `https://cda.harvard.edu/csctap` | `csc2.master_source` |

---

## SIA Services

| Archive | URL | Waveband |
|---------|-----|----------|
| IRSA | `https://irsa.ipac.caltech.edu/SIA` | Multi (2MASS, WISE, Spitzer, etc.) |
| MAST | `https://mast.stsci.edu/sia/v2` | UV/Optical/NIR (HST, JWST, TESS) |
| Herschel (ESA) | `https://archives.esac.esa.int/hsa/whsa-tap-server/sia` | FIR |
| GALEX (MAST) | `https://mast.stsci.edu/sia/v2` | UV (filter by collection) |

---

## HiPS Servers

| Survey | URL |
|--------|-----|
| CDS Aladin HiPS list | `https://aladin.cds.unistra.fr/hips/list` |
| DSS2 Red | `https://alasky.cds.unistra.fr/DSS/DSS2Merged` |
| 2MASS J | `https://alasky.cds.unistra.fr/2MASS/J` |
| WISE W1 | `https://alasky.cds.unistra.fr/WISE/WISE_W1` |
| Spitzer IRAC1 | `https://irsa.ipac.caltech.edu/data/hips/spitzer/irac1` |

---

## Registry Endpoint

```
https://voparis-registry.obspm.fr/vo/ivoa/1/vosi/capabilities
https://registry.ivoa.net/   (primary IVOA registry)
```

---

## Common pyvo Registry Keywords

```python
# datamodel values
"obscore"    # ObsCore 1.1+ compliant
"epntap"     # EPN-TAP (solar system)

# servicetype values
"tap"
"sia"
"ssa"
"scs"
"conesearch"
"datalink"

# waveband values (IVOA-standard)
"Radio", "Millimeter", "Infrared", "Optical", "UV", "EUV",
"X-ray", "Gamma-ray"
```
