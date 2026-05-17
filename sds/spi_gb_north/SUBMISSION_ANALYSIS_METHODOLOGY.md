# Final Submission Notebook: Statistical Analysis Methodology
## `final_spi_ai_submission.ipynb` — GB + North KP + AJK

---

## About This Document

This notebook has two sections:

- **Section 1 (Cells 2–12)**: SPI and AI raster generation — *fully documented in [SPI_AI_METHODOLOGY.md](SPI_AI_METHODOLOGY.md). All design choices, formulas, data sources, and step-by-step explanations are there. Do not repeat that reading here.*

- **Section 2 (Cells 14–44)**: Tehsil-level statistical analysis — **this document covers this section in full.**

---

## Table of Contents

1. [What Section 2 Is Trying to Do](#1-what-section-2-is-trying-to-do)
2. [Data Inputs for Statistical Analysis](#2-data-inputs-for-statistical-analysis)
3. [Step 1: Load and Join Tehsil Data](#3-step-1-load-and-join-tehsil-data)
4. [Step 2: Descriptive Statistics and the Gap Index](#4-step-2-descriptive-statistics-and-the-gap-index)
5. [Step 3: Spatial Weights — Queen Contiguity](#5-step-3-spatial-weights--queen-contiguity)
6. [Step 4: Global Spatial Autocorrelation — Moran's I](#6-step-4-global-spatial-autocorrelation--morans-i)
7. [Step 5: Local Clustering — LISA](#7-step-5-local-clustering--lisa)
8. [Step 6: OLS Regression — SPI ~ AI](#8-step-6-ols-regression--spi--ai)
9. [Step 7: Priority Region Identification](#9-step-7-priority-region-identification)
10. [Step 8: Component Contribution Analysis](#10-step-8-component-contribution-analysis)
11. [Step 9: Sensitivity Analysis for SPI Weights](#11-step-9-sensitivity-analysis-for-spi-weights)
12. [Step 10: Bivariate SPI-AI Classification](#12-step-10-bivariate-spi-ai-classification)
13. [Key Design Choices — Defended](#13-key-design-choices--defended)
14. [Fallback Implementation (No libpysal)](#14-fallback-implementation-no-libpysal)
15. [Output Files](#15-output-files)

---

## 1. What Section 2 Is Trying to Do

Section 1 produces pixel-level rasters and tehsil-level means. Section 2 asks the analytical questions that make those numbers meaningful for a proposal:

1. **Are high-SPI and high-AI tehsils geographically clustered, or randomly scattered?** (Moran's I, LISA)
2. **Does accessibility predict scenic potential — or are they independent?** (OLS regression)
3. **Which tehsils have high scenic value but poor access — the highest-priority targets for tourism investment?** (Gap index, priority flags)
4. **How much does the scenic ranking change if different weights had been used?** (Sensitivity analysis)
5. **What landscape components most strongly drive SPI variation across tehsils?** (Component correlations)

These are the questions a reviewer, policymaker, or examiner will ask after seeing the maps. This section provides evidence-based answers.

---

## 2. Data Inputs for Statistical Analysis

| Input | Source | Description |
|---|---|---|
| `tehsils_aoi_32643.gpkg` | Interim data | Tehsil polygons in EPSG:32643 |
| `tehsil_spi_ai_zonal_stats.csv` | Section 1 output | Per-tehsil SPI and AI means, medians, stds, counts |
| `tri_32643_100m.tif` | Interim data | TRI raster for component analysis |
| `forest_mask_32643_100m.tif` | Interim data | Binary forest raster |
| `water_mask_32643_100m.tif` | Interim data | Binary water raster |
| `snow_frequency_aligned_32643_100m.tif` | Interim data | Snow frequency raster |

The key link between sections is `tehsil_spi_ai_zonal_stats.csv` — the CSV written by Section 1 is read by Section 2. If Section 1 is rerun, Section 2 automatically picks up the updated values.

---

## 3. Step 1: Load and Join Tehsil Data

```python
gdf   = gpd.read_file(paths['tehsils'])   # polygon geometries
zonal = pd.read_csv(paths['zonal'])        # SPI/AI statistics

# Join on normalized tehsil names
analysis_gdf = gdf.merge(zonal, on='tehsil_key', how='left')

# Drop tehsils with no valid raster data
analysis_gdf = analysis_gdf.dropna(subset=['spi_mean', 'ai_mean'])
```

**Name normalization**: Both the geometry file and the CSV use different formatting conventions. The `norm_name()` function converts both to uppercase with only alphanumeric characters and spaces before joining. This handles differences like `"Ghizer"` vs `"GHIZER"` or `"Dera Ismail Khan"` vs `"DERA ISMAIL KHAN"`.

**Shorthand columns**: `analysis_gdf['spi']` and `analysis_gdf['ai']` are aliases for the mean values (`spi_mean`, `ai_mean`). All subsequent analysis uses these shorter names for readability.

**Area calculation**: `analysis_gdf['area_km2']` uses `geometry.area / 1e6`. Since the data is already in EPSG:32643 (a metric UTM projection), `area` returns values in m², dividing by 1e6 converts to km². No reprojection is needed — the `to_crs(analysis_gdf.crs)` call is a no-op that confirms the CRS is already correct.

---

## 4. Step 2: Descriptive Statistics and the Gap Index

### Z-scoring at the Tehsil Level

```python
analysis_gdf['spi_z'] = zscore(analysis_gdf['spi'])
analysis_gdf['ai_z']  = zscore(analysis_gdf['ai'])
```

These are **tehsil-level z-scores**, computed fresh from the 81 tehsil means — separate from the pixel-level z-scores used inside the SPI formula. They answer: "Is this tehsil above or below the average tehsil in scenic potential / accessibility?"

The `zscore()` helper uses population standard deviation (`ddof=0`) for consistency with the spatial statistics that follow. If the standard deviation is zero (all values identical), it returns zeros rather than NaN, which would break downstream calculations.

### The Gap Index

```python
analysis_gdf['gap_index'] = analysis_gdf['spi_z'] - analysis_gdf['ai_z']
```

The gap index is the core analytical construct. It measures the **mismatch** between scenic potential and accessibility:

- **High positive gap** (e.g., +1.5): the tehsil is much more scenic than average but much less accessible than average — a high-priority investment candidate.
- **Near zero gap**: scenic and accessibility scores are balanced relative to the AOI average.
- **Negative gap**: the tehsil is more accessible than it is scenic — lower priority for scenic tourism development.

**Why subtract z-scores and not raw values?** SPI and AI are on completely different scales (z-score vs [0,1] exponential decay). Subtracting z-scores compares each index to its own AOI distribution, making the gap a relative measure rather than an artifact of scale differences.

`gap_rank` ranks all tehsils from highest to lowest gap, giving a simple priority ordering (rank 1 = biggest scenic-access mismatch).

---

## 5. Step 3: Spatial Weights — Queen Contiguity

### What Are Spatial Weights?

In spatial statistics, a **weights matrix** defines which units are "neighbours" of each other. It is needed for every spatial statistic in this notebook. The weights matrix W is an 81×81 matrix where entry W[i,j] = 1 if tehsil i and tehsil j are neighbours, and 0 otherwise.

### Why Queen Contiguity?

**Queen contiguity** defines two polygons as neighbours if they share any boundary — even a single point (a corner). This is in contrast to Rook contiguity, which only counts shared edges.

In an irregular polygon dataset like tehsil boundaries, many adjacencies are corner-to-corner. Using Rook would miss these, producing fewer neighbours per tehsil and potentially creating isolated units (tehsils with no neighbours). Queen contiguity is the standard choice for irregular administrative polygons in regional analysis.

### Row Standardization

After building the contiguity matrix, it is **row-standardized** (`w.transform = 'r'`). This means each row of W is divided by the number of neighbours in that row, so all row sums equal 1.

**Why**: Without row standardization, tehsils with many neighbours contribute more to spatial statistics than tehsils with few neighbours — which is a function of the irregularity of the polygon dataset, not of any real spatial pattern. Row standardization gives equal weight to each neighbour relationship regardless of how many neighbours a tehsil has.

### Isolated Units (Islands)

If any tehsil has zero neighbours (an isolated polygon that doesn't touch any other tehsil), it would break the spatial statistics — Moran's I and LISA require every unit to have at least one neighbour. The code checks for islands and adds a nearest-neighbour link as a fallback.

---

## 6. Step 4: Global Spatial Autocorrelation — Moran's I

### What Is Moran's I?

Moran's I measures whether a variable is **spatially clustered** (similar values near each other), **dispersed** (dissimilar values near each other), or **random** (no spatial pattern).

The formula is:
```
I = (n / S₀) × (Σᵢ Σⱼ wᵢⱼ · zᵢ · zⱼ) / (Σᵢ zᵢ²)

Where:
  n   = number of units (81 tehsils)
  S₀  = sum of all weights
  wᵢⱼ = spatial weight between units i and j
  zᵢ  = deviation of unit i from the mean
```

**Interpretation**:
- I > 0: positive spatial autocorrelation — high values cluster near high values (and low near low)
- I = 0 (approximately): spatial randomness
- I < 0: negative autocorrelation — high values neighbour low values (checkerboard pattern)
- I < −1/(n−1): theoretically minimum expected value under randomness

### The Permutation Test

Rather than relying on a theoretical distribution (which assumes normality of the data), the code uses **999 random permutations** to build an empirical null distribution. In each permutation, the observed values are randomly shuffled across tehsils and Moran's I is recomputed. The p-value is the fraction of permutation I values that are as extreme or more extreme than the observed I.

**Why permutations?**: Tehsil SPI and AI values are not guaranteed to be normally distributed (they are means of skewed pixel distributions). The permutation test is distribution-free — it makes no assumptions about the shape of the data.

`two_tailed=True` means the test looks for any spatial pattern (clustering or dispersion), not just clustering. This is the conservative choice.

Moran's I is computed separately for **SPI**, **AI**, and the **gap index** — each may have different degrees of spatial autocorrelation, which is analytically meaningful.

---

## 7. Step 5: Local Clustering — LISA

### What Is LISA?

**Local Indicators of Spatial Association (LISA)** — specifically Local Moran's I — decompose the global Moran's I into a contribution from each individual unit. This reveals *where* in the study area the spatial clustering occurs, rather than just *whether* it exists overall.

For each tehsil i, Local Moran's I is:
```
Iᵢ = zᵢ × Σⱼ wᵢⱼ · zⱼ
```

Where zᵢ is the standardized value at tehsil i and Σⱼ wᵢⱼzⱼ is the weighted average of its neighbours' standardized values (the **spatial lag**).

### Quadrant Classification

Each tehsil is assigned to one of four quadrants based on the signs of its own z-score and its spatial lag:

| Quadrant | Own value | Neighbours' values | Meaning |
|---|---|---|---|
| **HH** (High-High) | above average | above average | scenic/accessible hotspot cluster |
| **LL** (Low-Low) | below average | below average | scenic/accessible coldspot cluster |
| **HL** (High-Low) | above average | below average | scenic outlier surrounded by low-scoring neighbours |
| **LH** (Low-High) | below average | above average | low-scoring outlier surrounded by high-scoring neighbours |

Only tehsils where the Local Moran's p-value ≤ 0.10 are labelled as a cluster type. The rest are "Not Significant."

**Why p < 0.10?** With only 81 tehsils, using the conventional p < 0.05 threshold is overly conservative — it would flag very few clusters even if real spatial patterns exist. p < 0.10 is the standard relaxed threshold used in LISA analysis with moderate sample sizes (Anselin 1995).

LISA is computed for SPI, AI, and the gap index. The gap index LISA is the most directly policy-relevant: **HH tehsils in the gap index** are locations where both the tehsil and its neighbours have high scenic potential relative to accessibility — a regionally concentrated investment opportunity.

---

## 8. Step 6: OLS Regression — SPI ~ AI

### What the Regression Tests

```python
y = SPI (tehsil mean)
X = AI  (tehsil mean)

model: SPI = β₀ + β₁·AI + ε
```

The regression asks: **does accessibility level predict scenic potential across tehsils?** If β₁ is positive and significant, accessible tehsils tend to be more scenic. If β₁ is negative, inaccessible tehsils are more scenic (the "hidden gems" pattern). If p > 0.05, accessibility does not meaningfully predict scenic variation.

### Diagnostics Reported

The notebook reports several regression diagnostics:

| Diagnostic | What it tests | Why it matters |
|---|---|---|
| **R²** | How much variance in SPI is explained by AI | Tells you how strong the relationship is |
| **p(AI)** | Whether the AI coefficient is statistically significant | Determines if the relationship is real or noise |
| **Residual Moran's I** | Whether the regression residuals are spatially autocorrelated | If significant, OLS is invalid — spatial regression (SLM/SEM) is needed |
| **Breusch-Pagan test** | Whether residual variance is constant (homoscedasticity) | Heteroscedasticity inflates standard errors — makes p-values unreliable |
| **Jarque-Bera test** | Whether residuals are normally distributed | OLS p-values assume normality; JB violation reduces confidence in inference |
| **Pearson r** | Linear correlation between AI and SPI | Quick effect size measure |
| **Spearman ρ** | Rank correlation between AI and SPI | More robust than Pearson if distributions are skewed |

### Why OLS Rather Than Spatial Regression?

OLS is fitted first because it is simpler and its residuals are tested for spatial autocorrelation (residual Moran's I). If the residual Moran's I is significant (p < 0.05), the OLS assumption of independent errors is violated and a spatial lag model (SLM) or spatial error model (SEM) would be required. The residual Moran's test result determines whether the OLS findings are trustworthy or whether they should be reported with a caveat about spatial dependence.

---

## 9. Step 7: Priority Region Identification

Three complementary priority definitions are used, each with a different threshold and justification:

### Strict Priority
```python
priority_strict = (spi_z >= Q75_spi) AND (ai_z <= Q25_ai)
```
A tehsil must be in the **top quartile for SPI** and the **bottom quartile for AI** simultaneously. This is a conservative definition — a tehsil truly stands out in both dimensions. Typically ~5 tehsils qualify (roughly Q75 × Q25 = 6.25% of 81 ≈ 5).

**When to cite it**: When making a high-confidence claim about the most unambiguous priority candidates. A reviewer cannot argue these tehsils don't qualify.

### Relaxed Priority
```python
priority_relaxed = (spi_z > 0) AND (ai_z < 0)
```
A tehsil simply needs to be **above-average in SPI** and **below-average in AI**. This uses z-score zero (the mean) as the threshold rather than quartile boundaries. Approximately 20–30% of tehsils typically qualify.

**When to cite it**: For a broader investment pipeline — tehsils that are reasonably scenic and reasonably inaccessible, even if not extreme on either dimension.

### LISA-Based Priority
```python
priority_lisa_gap = gap_cluster IN ('HH', 'HL') AND gap_local_p <= 0.10
```
A tehsil is prioritised if it is a **statistically significant High cluster in the gap index** according to LISA. This means the tehsil (and its neighbours) have persistently high scenic-access gaps — a geographically coherent priority zone, not an isolated outlier.

**When to cite it**: For spatially targeted investment — identifying contiguous priority zones rather than scattered individual tehsils. A single road improvement or tourism infrastructure project benefits a cluster, not an isolated tehsil.

### Why Three Definitions?

Different stakeholders apply different standards. Presenting all three and showing which tehsils appear in all three provides both rigor (strict) and comprehensiveness (relaxed), while the LISA definition adds spatial context that neither threshold-based method captures.

---

## 10. Step 8: Component Contribution Analysis

### What It Does

This section extracts tehsil-level means for each of the four SPI input components — directly from the raw raster files — and correlates each component with the tehsil SPI score.

```python
component_frames = [
    raster_zonal_mean(paths['tri'],    zones, 'tri'),
    raster_zonal_mean(paths['forest'], zones, 'forest'),
    raster_zonal_mean(paths['water'],  zones, 'water'),
    raster_zonal_mean(paths['snow'],   zones, 'snow'),
]
```

### Conversions Applied

| Component | Raw units | Stored as |
|---|---|---|
| TRI | metres | `tri_mean` (average ruggedness in metres) |
| Forest mask | binary 0/1 | `forest_pct` = mean × 100 (% of tehsil that is forest) |
| Water mask | binary 0/1 | `water_pct` = mean × 100 (% of tehsil that is water) |
| Snow frequency | 0–1 proportion | `snow_pct` = proportion × 100 (% frequency as percentage) |

Multiplying by 100 converts proportions to percentages for interpretability in the output tables and scatter plots. This does not affect the correlation or regression results — only the axis scale.

### Note on Raw vs Processed Components

These component zonal means are extracted from the **raw input rasters** (pre-z-score, pre-density-window). The SPI pixel formula uses neighbourhood-density versions of forest and water, and z-scored versions of all components. So the correlations here (e.g., "forest_pct vs SPI") are between a simple binary proportion and the weighted multi-component index — they will not be perfect. Their purpose is diagnostic: understanding which raw landscape characteristics are most associated with higher SPI tehsil scores.

### What the Correlations Tell You

For each component, both Pearson (linear) and Spearman (rank) correlation with SPI is reported, along with a simple OLS slope and R². A high R² for TRI means terrain ruggedness dominates SPI variation across tehsils. A low R² for water means water body presence is less differentiating at the tehsil scale (perhaps because all tehsils have some water, or water is spatially uniform).

---

## 11. Step 9: Sensitivity Analysis for SPI Weights

### What It Tests

The sensitivity analysis asks: **if different weights had been used in the SPI formula, would the tehsil ranking and priority identification change substantially?**

Four weight sets are tested:

| Weight Set | TRI | Forest | Water | Snow | Purpose |
|---|---|---|---|---|---|
| `proposal` | 0.35 | 0.25 | 0.20 | 0.20 | Matches the actual SPI formula |
| `equal` | 0.25 | 0.25 | 0.25 | 0.25 | No a priori preference between components |
| `terrain_emphasis` | 0.50 | 0.20 | 0.15 | 0.15 | Terrain dominates — maximally rugged |
| `vegetation_water_emphasis` | 0.30 | 0.30 | 0.25 | 0.15 | Ecological richness over raw terrain |

**Important note on the `proposal` weight set**: In the actual SPI formula, Relief (TPI + TRI combined) receives 0.35. Here, TRI alone receives 0.35 as a proxy because TPI zonal means are not extracted in this section. This is a known approximation — TRI captures ruggedness (the dominant component of Relief) but misses the topographic position dimension from TPI.

All weights within each set sum to 1.0 (validated by assertion before use).

### How Sensitivity Is Measured

For each alternative weight set, two things are computed:

1. **Pearson correlation with current SPI**: How similar is the alternative ranking to the actual SPI ranking? A correlation of 0.95+ means the ranking is highly stable — the weight choice barely matters. A correlation of 0.70 means the ranking changes substantially.

2. **Jaccard similarity with relaxed priority set**: Of the tehsils flagged as priority under the relaxed criterion using current weights, what fraction are also flagged under the alternative weights? Jaccard = |intersection| / |union|. A score of 1.0 means perfect agreement; 0.5 means half the priority tehsils change.

### Priority Stability Score

`priority_frequency` counts how many of the four weight sets flag each tehsil as a priority. A tehsil with `priority_frequency = 4` is flagged as high-scenic/low-access under every weighting scheme tested — it is robustly a priority regardless of which experts' weight preferences are used. A tehsil with `priority_frequency = 1` only qualifies under one specific weight set and should be treated as a marginal candidate.

**Why this matters for a proposal**: Any proposal recommending investment in specific tehsils will be scrutinized for sensitivity to methodological choices. Showing that the top-priority tehsils are stable across all weight sets is a strong defense.

---

## 12. Step 10: Bivariate SPI-AI Classification

```python
def tertile_label(series):
    q1, q2 = series.quantile([1/3, 2/3])
    return pd.cut(series, bins=[-inf, q1, q2, inf], labels=['Low', 'Medium', 'High'])

analysis_gdf['spi_class'] = tertile_label(analysis_gdf['spi'])
analysis_gdf['ai_class']  = tertile_label(analysis_gdf['ai'])
analysis_gdf['bivariate_class'] = analysis_gdf['spi_class'] + '-' + analysis_gdf['ai_class']
```

### What It Produces

Each tehsil is assigned one of nine categories (High-High, High-Medium, High-Low, Medium-High, ..., Low-Low). This 3×3 bivariate classification is displayed as a choropleth map.

### Why Tertiles (Not Quartiles or Quantiles)?

- **Tertiles** divide the 81 tehsils into three groups of 27 each. This ensures enough tehsils in each cell of the 3×3 matrix for the map to be readable — quartiles would give groups of ~20, which becomes sparse in the 4×4 = 16 cells.
- **Tertiles vs arbitrary thresholds**: Using quantile-based cuts means "High" always means "top third" regardless of the absolute SPI value. This is consistent and not sensitive to scale.

### Policy Interpretation

The most actionable cell is **High-SPI, Low-AI**: these tehsils have clear scenic value but are the hardest to reach. The **High-AI, Low-SPI** cell represents accessible but less scenic tehsils — possibly suitable for different tourism products (transit hubs, base camps). **High-High** tehsils are already well-served by both scenery and roads — lower investment priority but likely already attracting tourists.

---

## 13. Key Design Choices — Defended

### Why Separate z-scores for Tehsil Analysis?

The SPI formula uses z-scores computed from 12.5 million pixels. The statistical analysis uses z-scores computed from 81 tehsil means. These are different standardizations for different purposes:

- Pixel z-scores normalize within the landscape to give each component equal variance in the formula.
- Tehsil z-scores normalize across administrative units to make gap analysis and Moran's I comparable across variables with different absolute ranges.

Using pixel z-scores for the gap index would be meaningless because the gap would then reflect pixel-scale variance rather than tehsil-scale variation in planning-relevant scores.

### Why OLS Before Spatial Regression?

OLS is always the starting point in spatial econometrics (Anselin 1988). The residual Moran's I test determines whether spatial regression is necessary. If residuals are not spatially autocorrelated, OLS is sufficient. Jumping straight to a spatial model adds complexity without justification.

### Why 999 Permutations for Moran's I and LISA?

999 is the standard choice in spatial statistics literature (Anselin 1995, GeoDa documentation). It gives a minimum possible p-value of 1/1000 = 0.001 — sufficient precision for p < 0.05 and p < 0.10 thresholds. Using more permutations (e.g., 9999) would increase precision but not change conclusions at these thresholds.

### Why p < 0.10 for LISA?

With n = 81 tehsils, the power of individual Local Moran tests is limited. At p < 0.05, only clusters that are very extreme (or very consistent across many permutations) will be flagged. This would miss genuine spatial patterns in a small dataset. p < 0.10 is the recommended threshold for LISA with n < 100 (Anselin 1995). The trade-off is a slightly higher false positive rate, which is acceptable given the exploratory nature of cluster identification.

### Why Three Priority Definitions?

A single definition would either be too strict (missing relevant tehsils) or too permissive (including marginal ones). Presenting three definitions with different thresholds:

1. **Demonstrates methodological transparency** — the results are not cherry-picked at a convenient threshold.
2. **Provides a robustness check** — if the same 5–8 tehsils appear in all three, that is a strong finding.
3. **Serves different stakeholders** — a finance ministry needs strict thresholds; a planning department may act on relaxed ones.

### Why Jaccard Similarity for Sensitivity Analysis?

The Jaccard index measures set overlap: |intersection| / |union|. It is the right metric for comparing priority *sets* because it penalizes both false positives (tehsils included under one weighting but not another) and false negatives (tehsils excluded under one scheme but included under another). Pearson correlation alone would not capture changes in the identity of priority tehsils — two rankings can be highly correlated while flagging completely different tehsils as priorities if the quantile boundary shifts.

---

## 14. Fallback Implementation (No libpysal)

The notebook includes a full pure-Python fallback for environments where `libpysal` and `esda` are not installed. The fallback implements:

- **Queen contiguity**: via `shapely.touches()` and `shapely.intersects()` — O(n²) but correct for n = 81.
- **Spatial lag**: weighted mean of neighbour values using the row-standardized weights dict.
- **Moran's I**: direct computation of the formula with 999 random permutations using `numpy.random.default_rng(42)` (seeded for reproducibility).
- **Local Moran's I**: local index computation with permutation-based p-values and quadrant classification.

The fallback uses the same API as `libpysal`/`esda` (`.I`, `.p_sim`, `.EI`, `.z_sim`, `.Is`, `.q`, `.p_sim`), so all downstream code runs identically regardless of which backend is active. The backend in use is printed at startup.

**Limitations of the fallback**:
- The Queen contiguity uses `intersects()` which includes polygons that share interior area (overlapping polygons), not just shared boundaries. For clean non-overlapping tehsil data this is equivalent to touching.
- The LISA fallback computes quadrant classification using z-scored input values, matching the esda convention.

---

## 15. Output Files

All outputs are saved to the `outputs/` directory unless noted.

### Statistical Tables

| File | Description |
|---|---|
| `descriptive_statistics.csv` | Mean, std, min, max, percentiles for SPI, AI, gap index, area |
| `queen_neighbor_summary.csv` | Distribution of neighbour counts across tehsils |
| `global_moran_summary.csv` | Moran's I, expected I, z-score, p-value for SPI, AI, gap index |
| `lisa_cluster_counts.csv` | Count of HH/LL/HL/LH/NS tehsils for each variable |
| `ols_regression_summary.csv` | Full regression diagnostics (R², coefficients, Breusch-Pagan, Jarque-Bera, residual Moran) |
| `priority_regions_ranked.csv` | All 81 tehsils ranked by gap index with all three priority flags |
| `component_zonal_statistics.csv` | Tehsil-level means for TRI, forest %, water %, snow % |
| `component_spi_correlations.csv` | Pearson r, Spearman ρ, OLS slope and R² for each component vs SPI |
| `sensitivity_summary.csv` | Correlation and Jaccard similarity for each weight set |
| `priority_stability_by_weight_set.csv` | Per-tehsil priority flag under each weight set + stability count |

### Maps and Figures

| File | Description |
|---|---|
| `01_distributions.png` | Histograms of SPI, AI, gap index, area |
| `02_choropleth_maps.png` | Side-by-side tehsil maps: SPI, AI, gap index |
| `03_global_moran.png` | Bar chart of Moran's I values for SPI, AI, gap |
| `04_lisa_cluster_maps.png` | LISA cluster maps for SPI, AI, gap index |
| `05_regression_diagnostics.png` | Scatter plot + residual plots + Q-Q plot |
| `06_priority_maps.png` | Scatter plot and map of strict vs relaxed priority tehsils |
| `07_component_correlations.png` | Scatter plots of TRI/forest/water/snow vs SPI |
| `08_sensitivity_charts.png` | Bar charts of SPI correlation and Jaccard by weight set |
| `09_bivariate_map.png` | 3×3 bivariate SPI-AI choropleth |

### Final Dataset

| File | Location | Description |
|---|---|---|
| `tehsil_spi_ai_statistical_analysis.geojson` | `outputs/` | Full analysis GeoDataFrame with all columns |
| `tehsil_spi_ai_statistical_analysis.csv` | `outputs/` | Same data without geometry |
| `final_tehsil_spi_ai_analysis.geojson` | `data/processed/` | Same as above, in processed folder for downstream use |

### Reading the Priority Regions CSV

The most important output for a proposal. Key columns:

| Column | Meaning |
|---|---|
| `gap_index` | SPI_z − AI_z. Higher = bigger scenic-access mismatch |
| `gap_rank` | Rank by gap_index (1 = highest priority) |
| `priority_strict` | True if top-quartile SPI AND bottom-quartile AI |
| `priority_relaxed` | True if above-average SPI AND below-average AI |
| `priority_lisa_gap` | True if LISA-significant HH or HL cluster in gap index |
| `priority_frequency` | How many of the 4 sensitivity weight sets also flag this tehsil (0–4) |
| `spi_cluster` / `ai_cluster` / `gap_cluster` | LISA classification (HH/LL/HL/LH/NS) |
