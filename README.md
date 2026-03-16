# Marine Route Optimization

[![Kaggle](https://kaggle.com/static/images/open-in-kaggle.svg)](https://www.kaggle.com/code/kevinmathewsgeorge/marine-route-optimization)

## Overview

**Marine Route Optimization** is an end-to-end data science project that fuses real-world AIS vessel-tracking data with CMEMS oceanographic forecasts to predict maritime risk and compute safer, more efficient shipping routes. A Random Forest model quantifies location-based risk and a Dijkstra shortest-path algorithm uses that risk to find optimal ocean corridors.

---

## Table of Contents

- [Key Results at a Glance](#key-results-at-a-glance)
- [Project Pipeline](#project-pipeline)
  - [1. Data Collection](#1-data-collection)
  - [2. Data Preprocessing](#2-data-preprocessing)
  - [3. Exploratory Data Analysis](#3-exploratory-data-analysis)
  - [4. Risk Modeling](#4-risk-modeling)
  - [5. Route Optimization](#5-route-optimization)
  - [6. Visualization](#6-visualization)
- [Metrics & Results](#metrics--results)
- [Feature Importance](#feature-importance)
- [Correlation Analysis](#correlation-analysis)
- [Technologies & Libraries](#technologies--libraries)
- [Project Structure](#project-structure)
- [Installation & Usage](#installation--usage)
- [Notebooks](#notebooks)
- [Requirements](#requirements)

---

## Key Results at a Glance

| Metric | Value |
|--------|-------|
| **Dataset size** | 305,786 records × 13 features |
| **Dataset file size** | 30.3 MB |
| **Missing values** | 0 (100% complete after preprocessing) |
| **Model type** | Random Forest Regressor |
| **R² Score** | **0.7514** (75.14% of variance explained) |
| **Mean Squared Error (MSE)** | **0.0068** |
| **Root Mean Squared Error (RMSE)** | **~0.082** |
| **Geographic coverage** | US Atlantic Coast, Caribbean & Gulf of Mexico |
| **Latitude range** | 20°N – 43°N |
| **Longitude range** | 87.5°W – 72.5°W |
| **Processing throughput** | ~95 iterations / second |
| **Estimated full-dataset runtime** | ~65 minutes |

---

## Project Pipeline

### 1. Data Collection

Two independent data sources were downloaded and combined:

#### AIS Data (NOAA)
- **Source:** NOAA Coast Survey AIS Data Handler (`coast.noaa.gov`)
- **Dates collected:** 01 Jan 2023, 01 Apr 2023, 01 Jul 2023, 01 Oct 2023
- **Format:** ZIP → CSV
- **Key columns:** `MMSI`, `BaseDateTime`, `LAT`, `LON`, `SOG` (Speed Over Ground), `COG` (Course Over Ground), `Heading`, `VesselType`, `Status`, `Length`, `Width`

#### CMEMS Oceanographic Data (Copernicus Marine Service)
- **Source:** `cmems_mod_glo_phy_anfc` (physics) and `cmems_mod_glo_wav_anfc` (wave) products at 0.083° spatial resolution
- **Physics variables (PT1H / P1D):** Salinity (`so`), Sea-water potential temperature (`thetao`), Layer thickness
- **Wave variables (PT3H):** Significant wave height (`VHM0`), Mean wave direction (`VMDR`), Peak wave period (`VTPK`)
- **Global grid dimensions:** 2,041 latitude × 4,320 longitude cells
- **Routing data dates:** 01–03 Jun 2023

---

### 2. Data Preprocessing

All filtering was applied to ensure data quality before merging:

| Filter | Criterion |
|--------|-----------|
| Vessel navigational status | Keep only Status = 0 (under way using engine) or Status = 8 (under way sailing) |
| Speed Over Ground | 7 knots < SOG < 102.2 knots |
| Latitude bounds | 20°N ≤ LAT ≤ 43°N |
| Longitude bounds | 87.5°W ≤ LON ≤ 72.5°W |
| Heading validity | Heading < 361° |

**Feature engineering:**
- `GrossTonnage = 0.67 × Length × Width` (Lloyd's simplified tonnage formula)
- All continuous features normalised to [0, 1] range (`StandardScaler`)
- AIS timestamps rounded to nearest CMEMS grid time step before spatial join

**Final preprocessed dataset:** `preprocessed_data.csv` — 305,786 rows, 13 columns, zero missing values

---

### 3. Exploratory Data Analysis

Six distribution plots and a correlation heat-map were produced for every numeric feature.

**Notable distribution findings:**

| Feature | Observation |
|---------|-------------|
| LAT | Multi-peaked — ships concentrate at specific latitude corridors |
| LON | Sharp single peak — dominant shipping lane longitude |
| Heading | Multi-modal — vessels travelling in multiple dominant directions |
| SOG_norm | Right-skewed — most vessels operate at lower speeds |
| GrossTonnage_norm | Multi-modal — dataset spans small to very large vessels |
| VHM0_norm | High concentration near 0.2 — predominantly calm seas; occasional high-wave outliers |
| Temperature_norm | Wide spread — dataset covers warm tropical and cooler mid-latitude regions |
| Salinity_norm | Sharp high-end peak — majority of records from high-salinity open-ocean areas |

---

### 4. Risk Modeling

#### Risk Score Definition

A composite **risk score** was derived from three safe (non-leaking) oceanographic features to avoid data leakage:

```
risk_score = 0.5 × (VMDR_norm + VTPK_norm + GrossTonnage_norm)
```

This combines wave direction, peak wave period, and vessel gross tonnage into a single continuous risk target.

#### Model Architecture

| Hyperparameter | Value |
|----------------|-------|
| Algorithm | `RandomForestRegressor` (scikit-learn) |
| Number of trees (`n_estimators`) | 100 |
| Maximum tree depth (`max_depth`) | 8 |
| Minimum samples to split (`min_samples_split`) | 10 |
| Random seed | 42 |
| Train / test split | 80% / 20% |
| Feature scaling | `StandardScaler` (inside a `Pipeline`) |

#### Input Features (4 total)

| # | Feature | Description |
|---|---------|-------------|
| 1 | `LAT` | Vessel latitude |
| 2 | `LON` | Vessel longitude |
| 3 | `Heading` | True heading (degrees) |
| 4 | `COG_norm` | Normalised course over ground |

---

### 5. Route Optimization

1. The trained model predicts a `predicted_risk` score for every row in the dataset.
2. A **directed graph** is built using **NetworkX** where each consecutive AIS position pair forms an edge:
   ```
   edge weight = geodesic_distance_km × (1 + predicted_risk)
   ```
   This penalises segments with high predicted risk.
3. **Dijkstra's shortest-path algorithm** (`nx.shortest_path`) finds the minimum-weight path between the start and end coordinates.
4. The resulting optimised path avoids high-risk sea areas (rough waves, unfavourable currents) while minimising travel distance.

**Example optimised waypoints (US East Coast → Caribbean):**

| # | Latitude | Longitude | Notes |
|---|----------|-----------|-------|
| 1 | 32.667°N | 78.335°W | Offshore start |
| 2 | 30.500°N | 78.800°W | Offshore waypoint |
| 3 | 28.284°N | 79.636°W | |
| 4 | 26.500°N | 80.500°W | Additional offshore waypoint |
| 5 | 25.887°N | 80.053°W | |
| 6 | 24.030°N | 81.709°W | |
| 7 | 23.870°N | 83.772°W | South endpoint |

---

### 6. Visualization

Six visualization types were produced:

| # | Visualization | Tool |
|---|---------------|------|
| 1 | Feature distribution histograms (KDE overlays) | Matplotlib / Seaborn |
| 2 | Numeric correlation heat-map | Seaborn |
| 3 | Residual analysis scatter plot | Seaborn |
| 4 | Feature importance horizontal bar chart | Matplotlib |
| 5 | Interactive risk-aware route map (scatter + polyline) | Plotly |
| 6 | Risk analysis along route (bar + cumulative line) | Plotly |

---

## Metrics & Results

### Model Performance

| Metric | Value | Interpretation |
|--------|-------|----------------|
| **R² Score** | **0.7514** | Model explains **75.14%** of variance in risk scores |
| **MSE** | **0.0068** | Very low average squared prediction error |
| **RMSE** | **~0.082** | Average prediction deviation in risk-score units |
| Residual pattern | Random scatter around zero | No systematic bias; well-fitted model |

> The MSE of 0.0068 indicates relatively low overall error. The R² score of 0.7514 suggests that the model is a good fit — residuals are randomly scattered around zero with no patterns of systematic bias.

### Processing Performance

| Metric | Value |
|--------|-------|
| Processing speed | ~95.12 iterations / second |
| Full-dataset runtime | ~65 minutes |
| Train set size (80%) | ~244,629 records |
| Test set size (20%) | ~61,157 records |

---

## Feature Importance

Latitude (geographic position) dominates model decisions:

| Rank | Feature | Approximate Importance | Interpretation |
|------|---------|----------------------|----------------|
| 1 | **LAT** | 50–60% | Strongest predictor — risk varies significantly by latitude |
| 2 | **LON** | 30–40% | Geographic longitude is the second most influential feature |
| 3 | Heading | 5–10% | Minor contribution to risk prediction |
| 4 | COG_norm | 1–5% | Minimal contribution |

> Geographic location (latitude + longitude combined) accounts for **~85–100% of feature importance**, confirming that sea conditions and shipping risk are primarily driven by where a vessel is rather than how it is oriented.

---

## Correlation Analysis

Key pairwise Pearson correlations in the preprocessed dataset:

| Feature Pair | Correlation | Interpretation |
|--------------|-------------|----------------|
| LAT ↔ LON | **+0.83** | Strong geographic pattern — latitude and longitude co-vary along coastal shipping lanes |
| COG_norm ↔ (directional features) | **~+0.91** | High consistency in directional data |
| Temperature ↔ LAT | **−0.86** | Strong negative — colder water at higher latitudes (classical climate gradient) |
| Temperature ↔ LON | **−0.63** | Moderate negative — regional thermal gradients |
| Temperature ↔ Salinity | **+0.63** | Positive — similar oceanographic forcing on both variables |
| VHM0_norm ↔ LON | **+0.61** | Moderate positive — geography affects wave behaviour |

---

## Technologies & Libraries

| Category | Technology |
|----------|------------|
| Language | Python 3.8+ |
| ML / Statistics | scikit-learn (`RandomForestRegressor`, `StandardScaler`, `Pipeline`, `train_test_split`, `r2_score`, `mean_squared_error`) |
| Graph / Optimization | NetworkX (Dijkstra's shortest path via `nx.shortest_path`) |
| Geospatial distance | GeoPy (`geodesic`) |
| Data manipulation | Pandas, NumPy |
| Multi-dimensional arrays | xarray (NetCDF CMEMS data) |
| Static visualization | Matplotlib, Seaborn |
| Interactive visualization | Plotly (`scatter_mapbox`, `Scattermapbox`) |
| Map display | Folium (referenced), OpenStreetMap tiles |
| Notebook environment | Jupyter / Kaggle Notebooks |
| HTTP data download | `requests`, `zipfile` |
| Cartographic projections | Cartopy (`ccrs.PlateCarree`, Natural Earth features) |

---

## Project Structure

```
Marine-route-optimization/
├── 1-data-collection-marine-route.ipynb     # Data collection, merging & preprocessing pipeline
├── 1-data-collection-marine-route (2).ipynb # Annotated version of the data collection notebook
├── marine-route-optimization.ipynb          # EDA, risk modeling, Dijkstra optimization & visualization
├── Project documentation.pdf               # Full project report with methodology and results
├── archive (9).zip                          # Archived data / previous notebook versions
└── README.md                                # This file
```

---

## Installation & Usage

1. **Clone the repository:**
   ```bash
   git clone https://github.com/KevinG45/Marine-route-optimization.git
   cd Marine-route-optimization
   ```

2. **Create a virtual environment (recommended):**
   ```bash
   python -m venv venv
   source venv/bin/activate   # On Windows: venv\Scripts\activate
   ```

3. **Install dependencies:**
   ```bash
   pip install pandas numpy matplotlib seaborn scikit-learn networkx geopy xarray plotly folium cartopy
   ```

4. **Open Jupyter:**
   ```bash
   jupyter notebook
   ```

5. Run notebooks **in order**:
   - `1-data-collection-marine-route.ipynb` → produces `preprocessed_data.csv`
   - `marine-route-optimization.ipynb` → loads the preprocessed CSV, trains the model, and generates optimised routes

> **Note:** Raw AIS and CMEMS NetCDF files are sourced from Kaggle datasets linked inside the notebooks. You will need Kaggle API access or the pre-uploaded datasets to reproduce the full pipeline from scratch.

---

## Notebooks

| Notebook | Purpose |
|----------|---------|
| `1-data-collection-marine-route.ipynb` | Downloads NOAA AIS ZIP files; loads January & April 2023 CMEMS physics/wave NetCDF files; applies bounding-box and quality filters; merges datasets; normalises features; exports `preprocessed_data.csv` |
| `1-data-collection-marine-route (2).ipynb` | Annotated duplicate of the above with detailed inline comments explaining each code block |
| `marine-route-optimization.ipynb` | Loads preprocessed data; runs EDA (distributions, correlations); builds and trains the Random Forest risk model; evaluates with MSE/R²; performs feature-importance analysis; constructs NetworkX graph; runs Dijkstra's algorithm; renders interactive Plotly maps |

---

## Requirements

```
Python >= 3.8
pandas
numpy
matplotlib
seaborn
scikit-learn
networkx
geopy
xarray
plotly
folium
cartopy
requests
jupyter
```

Install all at once:

```bash
pip install pandas numpy matplotlib seaborn scikit-learn networkx geopy xarray plotly folium cartopy requests jupyter
```

---

For a full written report including methodology, extended figures, and discussion, see [`Project documentation.pdf`](Project%20documentation.pdf).
