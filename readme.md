# Gauge Gap Analysis & Prioritization

A two-notebook workflow for identifying ungauged HUC-10 watersheds and prioritizing them on the basis of socio-economic datasets.

---

## Repository Structure

```
gauge-gap/
├── gauge_gap_analysis.ipynb        # Notebook 1 — identify ungauged HUC-10s
├── gauge_prioritization.ipynb      # Notebook 2 — score and rank ungauged HUC-10s
├── data/
│   ├── Gap_input.gpkg              
│   └── prioritization.gpkg         # Input for prioritization (contain four socio-economic datasets and identified ungauged HUC-10 from 'gauge_gap_analysis.ipynb'.
├── prioritization_out/             # Auto-created on run
│   ├── prioritization_outputs.gpkg
│   ├── prioritization_outputs_all.csv
│   ├── prioritization_outputs_ungauged.csv
│   └── priority_map.png
└── README.md
```

---

## Notebooks

### 1. `gauge_gap_analysis.ipynb` — Gauge Gap Analysis

Identifies ungauged HUC-10 watersheds by propagating stream gage coverage through the hydrologic network.

**Pipeline:**
1. Load spatial layers (flowpaths, divides, gages, HUC-12, HUC-10, state boundary) from `data/Gap_input.gpkg`
2. Assign gages to catchment divides via point-in-polygon spatial join
3. Build a directed flow network and propagate gage coverage via BFS traversal
4. Propagate coverage: flowpaths → divides → HUC-12 → HUC-10
5. Apply state boundary filters to finalize ungauged HUC-10 candidates
6. Write all result layers to `Gap_output_final_v2.gpkg`

**Input layers (`data/Gap_input.gpkg`):**

| Layer | Description |
|---|---|
| `flowpaths` | Hydrofabric flowpath geometries with drainage area and hydrosequence |
| `divides` | Hydrofabric divide polygons |
| `gage` | Stream gage point locations |
| `huc12` | HUC-12 watershed polygons |
| `huc10` | HUC-10 watershed polygons |
| `state` | State boundary polygon |

**Output layers (`Gap_output_final_v2.gpkg`):**

| Layer | Description |
|---|---|
| `flowpaths_gagegap` | Flowpaths with `gage_status` and `well_gaged` flags |
| `divides_gagegap` | Divides with `has_gage` and `covered_divide` flags |
| `huc12_gagegap` | HUC-12s with `covered_huc` flag and area fractions |
| `huc10_gagegap` | HUC-10s with `covered_huc` flag |
| `huc10_enriched` | HUC-10s with state area fraction and outlet-in-state flag |
| `ungauged_huc10_final` | Final ungauged HUC-10 candidates |
| `gages_assigned` | Gages with their assigned divide ID |

---

### 2. `gauge_prioritization.ipynb` — Prioritization

Scores and ranks ungauged HUC-10 watersheds using four flood risk and infrastructure exposure indicators.

**Pipeline:**
1. Load layers from `data/prioritization.gpkg`
2. Compute four indicators per HUC-10 (disaster count, claim amount, road length, population)
3. Normalize indicators using IQR-clipped min-max scaling over ungauged candidates only
4. Compute equal-weight composite priority score and assign ranks
5. Export ranked GeoPackage, CSVs, and a choropleth map to `prioritization_out/`

**Input layers (`data/prioritization.gpkg`):**

| Layer | Description |
|---|---|
| `huc10` | All HUC-10 watershed polygons |
| `ungaugedhuc10` | Ungauged HUC-10 candidates (output of Notebook 1) |
| `disasters` | FEMA disaster declaration polygons with count field |
| `claim` | NFIP insurance claim points with total amount field |
| `roads` | Flood induced road closure lines |
| `population` | Census population polygons |

**Output files (`prioritization_out/`):**

| File | Description |
|---|---|
| `prioritization_outputs.gpkg` | GeoPackage with `huc10_prioritized_all` and `huc10_prioritized_ungauged` layers |
| `prioritization_outputs_all.csv` | Scores and ranks for all HUC-10s |
| `prioritization_outputs_ungauged.csv` | Scores and ranks for ungauged candidates only |
| `priority_map.png` | Prioritized map for study area |

---

## Requirements

Both notebooks require Python 3.9+ and the following libraries:

```
numpy
pandas
geopandas
shapely
networkx
matplotlib
```

Install all dependencies with:

```bash
pip install numpy pandas geopandas shapely networkx matplotlib
```

Or using conda:

```bash
conda install -c conda-forge numpy pandas geopandas shapely networkx matplotlib
```

---

## Configuration

Each notebook has a configuration block at the top (Section 2) where you can adjust:

**`gauge_gap_analysis.ipynb`**

| Parameter | Description |
|---|---|
| `INPUT_GPKG` | Path to input GeoPackage |
| `LAYER_NAMES` | Layer name mappings |
| `COLUMN_NAMES` | Column name mappings |
| `COVERAGE_THRESHOLD_KM2` | Drainage area window for BFS coverage (default: 2000 km²) |

**`gauge_prioritization.ipynb`**

| Parameter | Description |
|---|---|
| `INPUT_GPKG` | Path to input GeoPackage |
| `LAYER_NAMES` | Layer name mappings |
| `COLUMN_NAMES` | Column name mappings for indicator fields |
| `METRIC_CRS` | CRS for area/length computations (default: `EPSG:5070`) |
| `bin_size` | Rank bin size for choropleth map (default: `20`) |

---

## Usage

1. Clone the repository and install dependencies
2. Place input GeoPackages in the `data/` folder
3. Run `gauge_gap_analysis.ipynb` top to bottom to identify ungauged HUC-10s
4. Add the `ungauged_huc10_final` output layer to `data/prioritization.gpkg`
5. Run `gauge_prioritization.ipynb` top to bottom to score and rank candidates

---

## Coverage Logic

The coverage algorithm in `gauge_gap_analysis.ipynb` marks a flowpath as `well_gaged` based on its drainage area relative to the nearest upstream gage:

| Gage drainage area | Upstream bound | Downstream bound |
|---|---|---|
| ≥ threshold (2000 km²) | `area − threshold / 2` | `area + threshold / 2` |
| < threshold | `0.5 × area` | `1.5 × area` |
