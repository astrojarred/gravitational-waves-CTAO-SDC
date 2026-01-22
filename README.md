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

### 3. Metadata Table

The metadata files provide a comprehensive index of all events and their associated data products.

The metadata is provided in both Parquet and CSV formats:

#### Parquet File

If possible, we recommend using the Parquet file for an easier inference of types and missing values. The file can be loaded with:

```python
import pandas as pd
meta = pd.read_parquet("metadata/CTAO-SDC-metadata.parquet")
```

> NOTE: you might need to install the `pyarrow` package into your Python environment to read the Parquet file.

#### CSV File

The CSV file provides a comprehensive index of all events and their associated data products. Because the table contains lists and dictionaries, the file should be loaded with caution like so:

```python
import json
import pandas as pd
meta = pd.read_csv("metadata/CTAO-SDC-metadata.csv", converters={'tilepy_pointings': json.loads, 'alert_ifos': json.loads})
```

#### Columns

- **`sdc_event_id`**: Sequential event ID for the data challenge [**not sequential!**]
- **`superevent_id`**: Dummy gravitational wave superevent identifier (format: GWYYMMDDa/b/c...)
- **`model_id`**: Original model ID from the O5 catalog (this is for reference only and not needed for the SDC)
- **`model_filepath`**: Relative path to the gammapy model FITS file
- **`gcn_filepath`**: Relative path to the GCN JSON file
- **`timestamp_utc`**: Event trigger time in UTC (ISO format)
- **`distance_mpc`**: Distance to the source in Megaparsecs (Mpc)
- **`z`**: Cosmological redshift
- **`alert_ifos`**: List of interferometers that alerted on the event (JSON string)
- **`alert_area90`**: Area of the 90% containment radius of the event (square degrees)
- **`alert_distance90`**: Distance to the event in Mpc (90% containment radius)
- **`tilepy_n_observations`**: Number of follow-up pointings calculated by tilepy
- **`tilepy_prob_covered`**: Probability that the event is covered by the follow-up pointings
- **`tilepy_first_ra`**: Right Ascension of the first pointing in degrees
- **`tilepy_first_dec`**: Declination of the first pointing in degrees
- **`tilepy_first_latency`**: Latency of the first pointing in seconds
- **`tilepy_first_utc`**: Start time of the first pointing in UTC (ISO format)  
- **`tilepy_pointings`**: List of telescope pointings:
  - `ra`: Right Ascension of the pointing in degrees
  - `dec`: Declination of the pointing in degrees
  - `latency`: Latency of the pointing after the event onset in seconds
  - `duration`: Duration of the pointing in seconds (fixed to 5 mins for the SDC)
  - `obs_time_utc`: Start time of the pointing in UTC (ISO format)
  - `observatory`: "north" or "south", indicating the CTAO observatory site

#### Pointings

The pointings array is generated with the open-source `tilepy` software package ([read more here](https://tilepy.com/)). Each observation consists of multiple five-minute pointings covering the localization region of the skymap. The order of the pointings is optimized by `tilepy`.

#### Event Statistics

- Each original spectral model is used **twice** (with different timestamps) to increase statistics
- Events are randomly distributed between 2028-01-01 00:00:00 UTC and 2034-12-31 00:00:00 UTC
- Superevent IDs are assigned based on the date, with sequential letters (a, b, c...) for multiple events on the same day

### 4. Tiling Config (`config/sdc.ini`)

The configuration of the tiling algorithm is presented in the sdc.ini file. The tiling derived for the SDC can be re-generated using this precise configuration. Else, a modified tiling set can be obtained by tuning those parameters. The software used to derived the pointings is `tilepy`. The precise code used to obtained the tilings can be found in ([read more here](https://github.com/mseglar/science-data-challenge-gw))  
