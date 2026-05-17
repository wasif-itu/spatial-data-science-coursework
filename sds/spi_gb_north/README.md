# Scenic Potential and Accessibility Analysis — GB, North KP, and AJK

## Overview

This project computes a Scenic Potential Index (SPI) and Accessibility Index (AI) for tehsils across Gilgit-Baltistan, northern Khyber Pakhtunkhwa, and Azad Jammu & Kashmir, then identifies priority areas where high scenic value meets poor road access.

## Main Submission File

**`human_submission.ipynb`** — the primary notebook. Run all cells top to bottom; no manual edits are required.

## How to Run

1. Open `human_submission.ipynb` from the `spi_gb_north/` folder (or from any parent directory — the notebook resolves its own root path automatically).
2. Run all cells from top to bottom using **Kernel → Restart & Run All**.
3. Outputs are written to `outputs/` and `data/processed/`.

The notebook has two self-contained halves:
- **Sections 1–6**: compute SPI and AI rasters from the preprocessed inputs in `data/interim/` and aggregate to tehsil level.
- **Section 7 onward**: statistical analysis (descriptive stats, Moran's I, LISA, OLS regression, priority ranking, sensitivity). This half re-reads the zonal CSV from disk and can be re-run independently.

## Required Folder Structure

The preprocessed inputs must be present under `data/interim/`. The notebook checks all paths at startup and raises a clear error if anything is missing.

```
spi_gb_north/
├── data/
│   ├── interim/          ← preprocessed, grid-aligned inputs (read-only)
│   ├── processed/        ← generated outputs: rasters, zonal CSV, final dataset
│   └── raw/
│       └── admin_boundaries/
├── outputs/              ← figures and summary CSVs
└── human_submission.ipynb
```

## Required Libraries

Standard scientific Python stack plus:

| Library | Purpose |
|---------|---------|
| `geopandas` | vector spatial data (tehsil polygons, road network) |
| `rasterio` | raster I/O and metadata |
| `numpy`, `pandas` | array and table operations |
| `scipy` | `uniform_filter` for neighborhood density, `distance_transform_edt` for travel time |
| `matplotlib` | all plots and maps |
| `libpysal`, `esda` | Queen contiguity weights, Moran's I, LISA |
| `statsmodels` | OLS regression and diagnostics (Breusch-Pagan, Jarque-Bera) |

Install any missing packages with:
```bash
pip install geopandas rasterio scipy libpysal esda statsmodels
```

## Key Outputs

| File | Description |
|------|-------------|
| `data/processed/spi_index.tif` | SPI raster |
| `data/processed/ai_index.tif` | AI raster |
| `data/processed/tehsil_spi_ai_zonal_stats.csv` | Per-tehsil SPI and AI means |
| `data/processed/final_tehsil_spi_ai_analysis.csv` | Final analysis dataset with all statistics |
| `data/processed/final_tehsil_spi_ai_analysis.geojson` | Spatial version of the above |
| `outputs/*.png` | Figures (raster maps, choropleths, LISA, regression, priority, sensitivity, bivariate) |
| `outputs/analysis_summary.json` | Summary report (Moran's I, priority counts, etc.) |

## SPI Formula

```
SPI = 0.35·Relief + 0.25·Forest + 0.20·Water + 0.20·Snow
Relief = Z(0.5·TPI_z + 0.5·TRI_z)
```

All components are z-scored and capped at ±3.5σ before weighting. Forest and water are computed as 3 km neighborhood density (30-pixel window at 100 m resolution). Glacier pixels (snow frequency > 0.5) are removed from the water mask.

## AI Formula

```
AI = exp(-λ · t)    where λ = ln(2)/90  (half-life = 90 minutes)
```

`t` is travel time to the nearest road pixel. Road speed comes from OSM highway tags and is reduced by a slope modifier (slope > 5°: ×0.8, > 15°: ×0.6, > 25°: ×0.4).
