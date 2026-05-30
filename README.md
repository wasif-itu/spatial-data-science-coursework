# Spatial Data Science Coursework

This repository is my learning archive for the Spatial Data Science course. It contains course notebooks, assignments, practice work, and supporting datasets used while learning geospatial analysis with Python.

The final project has been removed from this repository so this repo stays focused on course learning material. The project files have been preserved locally at:

```text
/home/wasif/spi-gb-north-project
```

## Repository Focus

- Spatial data handling and visualization
- Choropleth mapping
- Spatial weights and autocorrelation
- Local indicators of spatial association
- Point pattern analysis
- Spatial inequality analysis
- Clustering and regionalization
- Variography and geostatistical exploration
- Remote sensing exercises
- In-class assignments and practice notebooks

## Main Coursework Notebooks

| Notebook | Topic |
| --- | --- |
| `sds/05_choropleth.ipynb` | Choropleth mapping |
| `sds/06_spatial_autocorrelation.ipynb` | Spatial autocorrelation |
| `sds/07_local_autocorrelation.ipynb` | Local autocorrelation |
| `sds/08_point_pattern_analysis.ipynb` | Point pattern analysis |
| `sds/09_spatial_inequality.ipynb` | Spatial inequality |
| `sds/10_clustering_and_regionalization.ipynb` | Clustering and regionalization |
| `sds/variography.ipynb` | Variography |
| `sds/variography-revised.ipynb` | Revised variography work |
| `sds/EX1_DN2Radiance.ipynb` | Remote sensing exercise |

Additional notebooks in `sds/` include assignments, class practice, experiments, and self-work completed during the course.

## Environment

Create and activate a Python environment, then install the required packages:

```bash
pip install -r requirements.txt
```

Some notebooks may require geospatial system libraries such as GDAL, Fiona, or PROJ depending on the local Python setup.

## Data

Course datasets are kept in `Course-Datasets/` and `data/`. Large generated files, caches, local environments, and temporary notebook artifacts are ignored where possible.

## Final Project

The final project is being moved into a separate focused repository. This keeps the course repo readable as a record of learning, while the project repo can be structured around a single research question, methodology, outputs, and final submission.
