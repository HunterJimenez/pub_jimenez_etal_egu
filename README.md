# Modeling the Combined Effects of the 2023 Türkiye-Syria Earthquake and an Atmospheric River Event on Landslide Hazard - Data and Code Repository

## Overview

This repository contains the data and code used to model probabilistic landslide hazard
under combined seismic and atmospheric-river (AR) forcing in the 2023 Türkiye-Syria
earthquake region. The workflow consists of two sequential modeling stages: (1) a daily
soil moisture and recharge model that translates gridded precipitation and temperature
into combined with soil and vegetation parameters to create spatially distributed root-zone 
leakage, and (2) a probabilistic landslide failure model that ingests rainfall drivers and 
earthquake legacy effects with land surface and soils data to estimate the probability of 
failure P(F) at each grid node. Five scenarios are evaluated, ranging from a no-legacy AR 
baseline to coseismic dry and pre-seismic AR conditions.

The repository is organized into three top-level folders: `Models/`, `Components/`, and
`Data/`. All models are implemented in Python using the
[Landlab](https://landlab.readthedocs.io) modeling framework.

## Citation

Jimenez, H. N., Istanbulluoglu, E., Gorum, T., Stanley, T. A., Amatya, P. M., Tanyas, H.,
Demirel, M. C., Akgun, A., and Bozkurt, D.: Modeling the combined effects of the 2023
Türkiye-Syria Earthquake and an Atmospheric River event on landslide hazard, EGUsphere
[preprint], https://doi.org/10.5194/egusphere-2025-3011, 2025.

## Repository Structure

```
repository/
│
├── Model/
│   ├── SoilMoisture_Recharge.ipynb
│   └── LandslideProbability.ipynb
│
├── Components/
│   ├── landslide_probability_slope_dist.py
│   ├── landslide_probability_newmark_sliding_block.py
│   ├── soil_moisture_dynamics.py
│   ├── radiation.py
│   └── potential_evapotranspiration_field.py
│
└── Data/
    ├── soilmoisture_input/
    ├── recharge/             ← soil moisture output, used as input for landslide model
    ├── landslide_input/
    └── landslide_output/
```

## Model Notebooks (`Models/`)

### 1. `SoilMoisture_Recharge.ipynb`

Computes daily root-zone water balance and recharge for a storm event of interest,
producing the spatially distributed maximum recharge composite grids (.asc) used as input to the
landslide probability model.

The workflow covers a 7-day event window and includes:

- Daily radiation (Landlab `Radiation` component, grid method)
- Priestley-Taylor potential evapotranspiration (`PotentialEvapotranspiration`)
- Single-layer root-zone water balance (`SoilMoisture`)
- Flow-accumulation routing of peak recharge (`FlowAccumulator`, D8)

**Input data**: elevation, field capacity, wilting point, leaf area index, saturated water content, plant functional type, and daily IMERG V7 precipitation and ERA5 min/max temperature rasters (see `Data/soilmoisture_input/` and `Data/soilmoisture_input/<event_month> <event_year>`).

**Output**: a routed peak-event recharge raster (`IMERG_Recharge_<event>_routed.asc`)
written to `Data/recharge/`. To model a different event, replace the
precipitation and temperature rasters in Section 7.

---

### 2. `LandslideProbability.ipynb`

Computes probabilistic landslide failure P(F) by incorporating rainfall drivers and earthquake legacy effects with land surface and soils data, into a probabilistic implementation of the infinite slope stability theorem through a Monte Carlo approach (2000 iterations).  

**Scenarios:**

| # | Name | Component | Recharge | Cohesion |
|---|------|-----------|----------|----------|
| 1a | No Legacy | `landslide_probability_slope_dist` | March 2023 AR | No adjustment |
| 1b | Legacy | `landslide_probability_slope_dist` | March 2023 AR | Legacy-adjusted |
| 2 | Historic Extreme | `landslide_probability_slope_dist` | Cell-wise max of historical events | Legacy-adjusted |
| 3 | Dry Coseismic | `landslide_probability_newmark_sliding_block` | Near-zero | No adjustment |
| 4 | AR before earthquake | `landslide_probability_newmark_sliding_block` | March 2023 AR | No adjustment |

**Input data**: elevation, slope (min/mean/max), soil thickness, root cohesion (min/mean/max), lithology (used to derive rock/soil cohesion), saturated hydraulic conductivity, peak ground acceleration, and routed recharge grids. See `Data/landslide_input/` for input fields necessary to upload for a model run in your domain. Note that some inputs used by the landslide components (e.g., soil density) are assumed to be uniform values in the notebook. 

**Output**: one P(F) raster per scenario written to `Data/landslide_output/`.

---

## Components (`Components/`)

All components extend the Landlab `LandslideProbability` framework.

### Landslide Probability Components

| File | Description |
|------|-------------|
| `landslide_probability_slope_dist.py` | Modified component for large-scale model domains to account for local slope variability. Samples slope each Monte Carlo iteration from a triangular distribution defined by minimum, maximum, and mean slope fields. Used for Scenarios 1a, 1b, and 2. |
| `landslide_probability_newmark_sliding_block.py` | Modified component for coseismic scenarios. Replaces the static FS ≤ 1.0 threshold with the Newmark (1965) critical-acceleration criterion: FS ≤ `(PGA / sin(arctan(θ))) + 1`, computed per node from the mean local slope and `peak__ground_acceleration`. Used for Scenarios 3 and 4. |

### Ecohydrology Components

| File | Description |
|------|-------------|
| `soil_moisture_dynamics.py` | Single-layer root-zone water balance model. Driven by effective precipitation and PET; produces `soil_moisture__root_zone_leakage` (recharge) and `surface__evapotranspiration`. |
| `radiation.py` | Computes spatially distributed net radiation from gridded Tmin/Tmax, latitude, albedo, and day-of-year. |
| `potential_evapotranspiration_field.py` | Estimates PET using the Priestley-Taylor method applied to radiation and temperature fields. |

---

## Data (`Data/`)

### `Data/soilmoisture_input/` — Soil Moisture Model Inputs

| File | Description | Source |
|------|-------------|--------|
| `landslide_run_dem.asc` | Digital elevation model (m), 90 m resolution | COPERNICUS GLO-30 |
| `IMERGV7PrecipitationMM<YYYYMMDD>.asc` | Daily precipitation (mm), one file per day | NASA IMERG V7 |
| `ERA5TemperatureMinC<YYYYMMDD>.asc` | Daily minimum temperature (°C, 2m), one file per day | ECMWF ERA5 |
| `ERA5TemperatureMaxC<YYYYMMDD>.asc` | Daily maximum temperature (°C, 2m), one file per day | ECMWF ERA5 |
| `soil__thickness.asc` | Soil thickness (m) | ISRIC SoilGrids |
| `field_capacity.asc` | Volumetric water content at field capacity (×0.0001 m³/m³) | FutureWater HiHydro |
| `saturated_water_content.asc` | Saturated volumetric water content / porosity (×0.0001 m³/m³) | FutureWater HiHydro |
| `wilting_point.asc` | Volumetric water content at wilting point (×0.0001 m³/m³) | FutureWater HiHydro |
| `vegetation__plant_functional_type.asc` | Plant functional type integer classes | Derived from Sentinel-2 LULC |
| `lai_cleaned.asc` | Leaf area index | NASA MODIS MOD15A2H |

### `Data/recharge/` — Soil Moisture Model Outputs / Landslide Model Recharge Inputs

Routed peak-event recharge grids produced by `SoilMoisture_Recharge.ipynb`.
This folder is shared between the two models; it is the output destination for the soil
moisture notebook and the recharge input for the landslide probability notebook.

| File | Description |
|------|-------------|
| `IMERG_Recharge_March_2023_routed.asc` | Peak routed recharge, March 2023 AR event |
| `IMERG_Recharge_December_2001_routed.asc` | Peak routed recharge, December 2001 historical event |
| `IMERG_Recharge_November_2004_routed.asc` | Peak routed recharge, November 2004 historical event |
| `IMERG_Recharge_February_2011_routed.asc` | Peak routed recharge, February 2011 historical event |
| `IMERG_Recharge_December_2018_routed.asc` | Peak routed recharge, December 2018 historical event |

### `Data/landslide_input/` — Landslide Model Inputs

| File | Description | Source |
|------|-------------|--------|
| `landslide_run_dem.asc` | Digital elevation model (m), 90 m | COPERNICUS GLO-30 |
| `slope_min_90m.asc` | Minimum slope (m/m) (stored × 1×10⁷) | Derived from COPERNICUS GLO-30 |
| `slope_mean_90m.asc` | Mean slope (m/m) (stored × 1×10⁷) | Derived from COPERNICUS GLO-30 |
| `slope_max_90m.asc` | Maximum slope (m/m) (stored × 1×10⁷) | Derived from COPERNICUS GLO-30 |
| `soil__thickness.asc` | Soil thickness (m) | ISRIC SoilGrids |
| `Ksat_topsoil.asc` | Saturated hydraulic conductivity, topsoil (cm/day) | FutureWater HiHydro |
| `C_min.asc` | Minimum root cohesion (kPa) | Derived from Sentinel-2 LULC |
| `C_mode.asc` | Mode root cohesion (kPa) | Derived from Sentinel-2 LULC |
| `C_max.asc` | Maximum root cohesion (kPa) | Derived from Sentinel-2 LULC |
| `MTA_Lithology.asc` | Lithology class raster | MTA Geological Map of Türkiye |
| `pga_new_potential.asc` | Peak ground acceleration (fraction of g) | USGS ShakeMap |
| `Data/recharge/` | Routed recharge grids (mm/d) | SoilMoisture_Recharge model |

### `Data/landslide_output/` — Landslide Model Outputs

One P(F) raster per scenario, produced by `LandslideProbability.ipynb`.

| File | Scenario |
|------|----------|
| `AR_march2023_no_legacy_landslide_probability.asc` | Scenario 1a — AR, no legacy |
| `AR_march2023_legacy_landslide_probability.asc` | Scenario 1b — Post-EQ AR, March 2023 |
| `AR_historical_composite_landslide_probability.asc` | Scenario 2 — Post-EQ AR, historical composite |
| `coseismic_dry_landslide_probability.asc` | Scenario 3 — Coseismic, dry |
| `AR_before_EQ_march2023_landslide_probability.asc` | Scenario 4 — AR before earthquake |

---

## Software and Dependencies

| Package |
|---------|
| Python |
| Landlab |
| NumPy |
| Jupyter |
| Matplotlib (optional, plotting) |

Install dependencies via:

```bash
pip install landlab numpy jupyter
```

---

## Spatial Reference

All raster inputs and outputs are ESRI ASCII grids (.asc) in geographic coordinates
(EPSG:32637) unless otherwise noted. The Landlab model grid is constructed at 90 m
resolution. The co-seismic landslide inventory point shapefile used for validation is
in EPSG:4326, reprojected to EPSG:32637 (UTM Zone 37N) for comparison with model outputs.

---

## References

Görüm, T., Tanyas, H., Karabacak, F., Yılmaz, A., Girgin, S., Allstadt, K. E., et al. (2023). Preliminary documentation of coseismic ground failure triggered by the February 6, 2023 Türkiye earthquake sequence. Engineering Geology, 327, 107315. https://doi.org/https://doi.org/10.1016/j.enggeo.2023.107315

Hobley, D. E. J., Adams, J. M., Siddhartha Nudurupati, S., Hutton, E. W. H., Gasparini, N. M., Istanbulluoglu, E., & Tucker, G. E. (2017). Creative computing with Landlab: An open-source toolkit for building, coupling, and exploring two-dimensional numerical models of Earth-surface dynamics. Earth Surface Dynamics, 5(1). https://doi.org/10.5194/esurf-5-21-2017

Nudurupati, S. S., Istanbulluoglu, E., Tucker, G. E., Gasparini, N. M., Hobley, D. E. J.,
Hutton, E. W. H., et al. (2023). On transient semi-arid ecosystem dynamics using Landlab:
Vegetation shifts, topographic refugia, and response to climate. *Water Resources Research*,
59, e2021WR031179. https://doi.org/10.1029/2021WR031179

Strauch, R., Istanbulluoglu, E., Nudurupati, S., Bandaragoda, C., Gasparini, N., Tucker, G.
(2018). A hydroclimatological approach to predicting regional landslide probability using
Landlab. *Earth Surface Dynamics*, 6(1), 49–75. https://doi.org/10.5194/esurf-6-49-2018

