# CTAO SDC - Gravitational Wave Follow-ups

This document describes the data products, models, and metadata for the CTAO Open Science Data Challenge focusing on gravitational wave (GW) event follow-ups.

## Overview

This dataset contains simulated gamma-ray emission models for gravitational wave events, along with corresponding GCN (Gamma-ray Coordinates Network) notices and metadata. The models are based on GRB (Gamma-Ray Burst) spectral templates that have been converted to a gammapy-compatible format and assigned new timestamps within the timeframe of the SDC window (00:00:00 UTC January 1st, 2028, until 00:00:00 December 31st, 2034.).

## Context

The multi-messenger discovery of the binary neutron stare merger GW170817 demonstrated
the promise of follow-up efforts to identify and interpret counterparts of Gravitational Wave
(GW) sources. In addition, the recent discovery of TeV emission from GRBs confirmed that
emission form these transients can extend to the VHE range. GW event follow-up
observations by CTAO may be able to probe particle acceleration and high-energy emission
from binary black hole and neutron star mergers, although the former is not expected to
have significant EM emission.

From the perspective of CTAO capability testing, this science case is identical to that of
Gamma-ray Bursts. The main difference between a GW follow-up and GRB follow-up is the
size of the localization region. GRBs have localization region either on the size of the CTAO
FoV or smaller, while GW localization region can be up to 100 square degrees.

Due to this, in addition the science case of short Gamma-ray Bursts, this dataset can also be used to test for the detection of sources in a tiled pointing strategy covering a localization region larger than that of the CTAO FoV

## Data Products

### 1. Spectral Models (`gammapy_models/`)

The spectral models are stored as FITS files in gammapy-compatible format using `LightCurveTemplateTemporalModel`.

#### File Naming Convention

- Format: `{superevent_id}_{original_model_filename}_gammapy.fits`
- Example: `GW280102a_catO5_1856_gammapy.fits`

#### Model Structure

The model structure outlined below was originally provided by Fabio Pintore (INAF/IASF Palermo).

> **Note: The included models are *intrinsic* models,** so the EBL absorption should be added in with gammapy before any processing. Please use the distance/redshift values from the metadata table or FITS headers to calculate the EBL absorption.

**Spatial Model:**

- Point source models (fixed position)
- Position defined by RA/Dec from original FITS header (LAT/LONG keywords)

**Spectral Model:**

- Intrinsic source flux (before EBL absorption)
- Time-dependent differential flux spectra
- Stored as `RegionNDMap` with energy and time axes
- Flux unit: `cm^-2 s^-1 GeV^-1`

**Temporal Model:**

- `LightCurveTemplateTemporalModel` format
- Reference time (`t_ref`) set to the event timestamp (UTC)
- Interpolation method: `linear`
- Values scale: `log`
- Three time bins are prepended before T0 with zero flux

**Metadata in FITS Header:**
Some metadata is added to the FITS header for reference:

- `DISTANCE`: Distance to the source in Mpc
- `REDSHIFT`: Cosmological redshift (z)


### 2. GCN Notices (`GCNs/`)

Dummy GCN notices are provided in JSON format for each event.

#### GCN File Naming Convention

- Format: `{superevent_id}_GCN.json`
- Example: `GW280102a_GCN.json`

#### GCN Structure

Each GCN contains:

- `alert_type`: "PRELIMINARY"
- **`time_created`**: Alert creation time (here fixed to 10 seconds after event time)
- **`superevent_id`**: Unique identifier (format: GWYYMMDDa/b/c...)
- `urls`: Dummy GraceDB URL
- **`event`**: Event information including:
  - **`time`**: Event trigger time (UTC)
  - **`instruments`**: List of interferometers, this is actually  (fixed to ["H1", "L1", "V1", "K1"])

**Note:** Any other fields not mentioned here are dummy/default values.

### 3. Metadata Table (`CTAO-SDC-GW-metadata.csv`)

The metadata CSV file provides a comprehensive index of all events and their associated data products.

#### Columns

- **`sdc_event_id`**: Sequential event ID (0-indexed) for the data challenge
- **`superevent_id`**: Dummy gravitational wave superevent identifier (format: GWYYMMDDa/b/c...)
- **`model_id`**: Original model ID from the O5 catalog (this is for reference only and not needed for the SDC)
- **`model_filepath`**: Relative path to the gammapy model FITS file
- **`gcn_filepath`**: Relative path to the GCN JSON file
- **`timestamp_utc`**: Event trigger time in UTC (ISO format)
- **`distance_mpc`**: Distance to the source in Megaparsecs (Mpc)
- **`z`**: Cosmological redshift
- **`pointings`**: List of telescope pointings:
  - `pointing_id`: Sequential pointing ID (0-indexed) for the data challenge
  - `start_time`: Start time of the pointing in UTC (ISO format)
  - `duration`: Duration of the pointing in seconds
  - `ra`: Right Ascension of the pointing in degrees
  - `dec`: Declination of the pointing in degrees

#### Pointings

The pointings array is generated with the open-source `tilepy` software package ([read more here](https://tilepy.com/)). Each observation consists of multiple five-minute pointings covering the localization region of the skymap. The order of the pointings is optimized by `tilepy`.

#### Event Statistics

- Each original spectral model is used **twice** (with different timestamps) to increase statistics
- Events are randomly distributed between 2028-01-01 00:00:00 UTC and 2034-12-31 00:00:00 UTC
- Superevent IDs are assigned based on the date, with sequential letters (a, b, c...) for multiple events on the same day


## Caveats and Limitations


## Usage with Gammapy

To load and use a model in gammapy:

```python
from gammapy.modeling.models import LightCurveTemplateTemporalModel
from astropy.time import Time

# Load the temporal model
temporal_model = LightCurveTemplateTemporalModel.read(
    "gammapy_models/GW280102a_catO5_1856_gammapy.fits",
    format="map"
)

# Evaluate flux at specific time and energy
time = Time("2028-01-02T00:03:39.819055", scale="utc")
energy = 1.0 * u.TeV  # Adjust to your energy range

# The model can be integrated into a full SkyModel
from gammapy.modeling.models import (
    PointSpatialModel,
    ConstantSpectralModel,
    SkyModel
)
from astropy.coordinates import SkyCoord

# Get position from metadata or FITS header
position = SkyCoord(ra=273.93*u.deg, dec=-0.86*u.deg, frame="icrs")
spatial_model = PointSpatialModel(lon_0=position.ra, lat_0=position.dec)

# Create spectral model (constant normalization, temporal variation from template)
spectral_model = ConstantSpectralModel(const="1 cm-2 s-1 GeV-1")

# Combine into full model
sky_model = SkyModel(
    spatial_model=spatial_model,
    spectral_model=spectral_model,
    temporal_model=temporal_model,
    name="GW_event"
)
```

## File Structure

```text
.
├── CTAO-SDC-GW-metadata.csv      # Main metadata table
├── gammapy_models/                # Spectral models (FITS files)
│   ├── GW280102a_catO5_1856_gammapy.fits
│   └── ...
└── GCNs/                          # GCN notices (JSON files)
    ├── GW280102a_GCN.json
    └── ...

```

