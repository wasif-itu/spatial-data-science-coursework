# Spatial Data Science Coursework

This repository is a structured archive of my Spatial Data Science coursework. It contains lecture notebooks, assignments, practice work, remote sensing exercises, geostatistics work, and supporting datasets.

The final project has been separated from this coursework repository. Its files are preserved locally at:

```text
/home/wasif/spi-gb-north-project
```

## Repository Structure

```text
.
├── assignments/                    # Submitted and in-class assignments
├── datasets/                       # Course and example geospatial datasets
│   ├── course/source-materials/     # Course-provided datasets and reference material
│   ├── examples/data/               # Smaller example datasets used in notebooks
│   └── national-constituency-boundary/
├── docs/                           # Local notes and helper commands
├── exercises/
│   └── remote-sensing/             # Landsat and raster processing exercises
├── notebooks/
│   ├── core/                       # Main course topic notebooks
│   ├── foundations/                # Introductory spatial data notebooks
│   ├── geostatistics/              # Variography and geostatistical work
│   └── practice/                   # Short practice and scratch notebooks
├── requirements.txt
└── spatial-data-science.code-workspace
```

## Main Course Notebooks

| Notebook | Topic |
| --- | --- |
| `notebooks/core/05_choropleth.ipynb` | Choropleth mapping |
| `notebooks/core/06_spatial_autocorrelation.ipynb` | Spatial autocorrelation |
| `notebooks/core/07_local_autocorrelation.ipynb` | Local spatial autocorrelation |
| `notebooks/core/08_point_pattern_analysis.ipynb` | Point pattern analysis |
| `notebooks/core/09_spatial_inequality.ipynb` | Spatial inequality |
| `notebooks/core/10_clustering_and_regionalization.ipynb` | Clustering and regionalization |

## Other Coursework

- `notebooks/foundations/` contains introductory spatial data notebooks.
- `notebooks/geostatistics/` contains variography and related self-work.
- `notebooks/practice/` contains smaller exploratory notebooks.
- `assignments/` contains assignment notebooks.
- `exercises/remote-sensing/` contains raster and Landsat exercises.
- `datasets/` contains the data used across the notebooks.

## Environment

Create and activate a Python environment, then install the course dependencies:

```bash
pip install -r requirements.txt
```

Some geospatial packages may require system libraries such as GDAL, Fiona, GEOS, and PROJ depending on the local Python setup.

## Notes

Generated caches, virtual environments, notebook checkpoints, editor settings, temporary files, and large local outputs are ignored. The repository is intended to stay focused on readable coursework and reusable learning material.
