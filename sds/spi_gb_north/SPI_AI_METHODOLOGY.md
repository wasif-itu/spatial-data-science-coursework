# Scenic Potential Index (SPI) & Accessibility Index (AI)
## Complete Methodology Guide — GB + North KP + AJK Study Area

---

## Table of Contents

1. [What Are We Trying to Do?](#1-what-are-we-trying-to-do)
2. [The Study Area](#2-the-study-area)
3. [What Is the Scenic Potential Index (SPI)?](#3-what-is-the-scenic-potential-index-spi)
4. [What Is the Accessibility Index (AI)?](#4-what-is-the-accessibility-index-ai)
5. [Data Sources and Why Each Was Chosen](#5-data-sources-and-why-each-was-chosen)
6. [Step-by-Step: SPI Calculation](#6-step-by-step-spi-calculation)
7. [Step-by-Step: AI Calculation](#7-step-by-step-ai-calculation)
8. [Combining SPI and AI](#8-combining-spi-and-ai)
9. [Zonal Statistics by Tehsil](#9-zonal-statistics-by-tehsil)
10. [Key Design Choices — Defended](#10-key-design-choices--defended)
11. [Limitations and Caveats](#11-limitations-and-caveats)
12. [Output Files](#12-output-files)

---

## 1. What Are We Trying to Do?

This notebook answers a practical geographic question:

> **Which tehsils in Gilgit-Baltistan, North KP, and AJK have the highest scenic beauty AND are accessible enough for tourism?**

Tourism potential is not just about whether a place is beautiful. A stunning valley that takes three days of off-road driving to reach has very different tourism potential than an equally beautiful valley one hour from a paved road. We need two separate scores:

- **SPI** — how scenically attractive is a location (terrain drama, forests, water, snow)?
- **AI** — how accessible is that location from the road network?

Combining them gives a realistic picture of where tourism investment would be most impactful.

---

## 2. The Study Area

**Coverage**: 81 tehsils across Gilgit-Baltistan, northern Khyber Pakhtunkhwa, and Azad Jammu & Kashmir.

**Coordinate system**: EPSG:32643 (UTM Zone 43N). This is a metric projection — distances and areas are in metres and square metres, which is essential for calculations like slope, distance to roads, and neighbourhood density.

**Resolution**: 100m × 100m pixels. Each pixel represents a 1-hectare ground area. At this scale we can capture meaningful landscape features (river corridors, treelines, glaciers) without the computational cost of finer resolutions across a ~126,000 km² study area.

**Bounding box coverage**: Only about 43% of the rectangular bounding box is within the actual study area (the rest is nodata — mountain ranges, international borders, etc.). All calculations explicitly mask outside-AOI pixels.

---

## 3. What Is the Scenic Potential Index (SPI)?

### The Core Idea

Scenic beauty in mountain landscapes is driven by a combination of landform drama, vegetation, water presence, and snow/ice. The SPI quantifies this by combining four measurable components into a single continuous score.

### The Formula

```
SPI = 0.35 × Z(Relief) + 0.25 × Z(Forest) + 0.20 × Z(Water) + 0.20 × Z(Snow)

Where:
  Relief = Z(0.5 × TPI_z + 0.5 × TRI_z)  [re-z-scored after combining]
  Z(x) = z-score normalization of variable x
```

### Why These Four Components?

| Component | What It Measures | Why It Matters Scenically |
|---|---|---|
| **Relief** | Terrain ruggedness — how dramatic the landscape is | High peaks, deep valleys, and sharp ridgelines are the core visual draw of mountain tourism |
| **Forest** | Density of tree and shrub cover around each pixel | Forests add texture, colour, and ecological richness to a landscape |
| **Water** | Density of rivers, lakes, and wetlands nearby | Water features (lakes, rivers, waterfalls) are consistently rated as high scenic value |
| **Snow/Ice** | How frequently a pixel is snow-covered | Permanent snowfields and glaciers are iconic scenic features of the Karakoram/Hindu Kush/Himalaya |

### Why These Weights?

The weights sum to 1.0 and were assigned based on the relative scenic importance of each component in a high-altitude mountain context:

- **Relief (35%)** gets the highest weight because terrain drama — towering peaks, plunging gorges — is the primary reason tourists visit this region. It subsumes two terrain metrics (TPI and TRI) for a comprehensive ruggedness measure.
- **Forest (25%)** gets the second-highest weight. Forests are a major visual and ecological asset but are secondary to terrain in a region defined by its mountains.
- **Water (20%)** and **Snow (20%)** share equal weight. Both are strong scenic attractors and are roughly equivalent in scenic contribution across this landscape.

These weights are defensible as expert-derived judgements. They can be adjusted if stakeholder input suggests different priorities.

### What Does the Final SPI Score Mean?

The SPI is a **z-score-based index** with a mean near zero and a standard deviation around 0.5. It is not a percentage or a rating out of 10.

- A **positive SPI** means the location is more scenic than the average AOI pixel.
- A **negative SPI** means it is less scenic than average.
- The range is approximately −2 to +4 (after outlier clipping).

Higher is always better for scenic potential.

---

## 4. What Is the Accessibility Index (AI)?

### The Core Idea

Accessibility measures how long it would take a traveller to reach a location from the nearest road. Places close to roads score high (AI near 1). Places deep in roadless terrain score low (AI near 0).

### The Formula

```
AI = exp(−λ × travel_time_minutes)
λ = ln(2) / 90  ≈  0.00770

→ AI = 1.000 at the roadside (0 min)
→ AI = 0.500 at 90 min travel time  (the half-life)
→ AI = 0.250 at 180 min travel time
```

### Why Exponential Decay?

An exponential decay function was chosen over a simple linear or step function for three reasons:

1. **No arbitrary boundary**: A step function (e.g., "accessible if within 2 hours, inaccessible otherwise") creates a hard cliff in the index. Reality is continuous — a place 91 minutes away is not categorically different from a place 89 minutes away.

2. **Diminishing returns**: The difference between 0 minutes and 30 minutes from a road matters enormously for tourist access. The difference between 400 and 430 minutes matters very little — both are effectively unreachable for day-trippers. Exponential decay captures this correctly; linear decay does not.

3. **Methodological precedent**: Exponential decay is used in the Weiss et al. (2018) global travel time mapping framework (published in *Nature*) and in the World Bank's Rural Access Index.

### Why a 90-Minute Half-Life?

The 90-minute half-life means AI drops to 0.5 (half its maximum) when travel time is 90 minutes. This was chosen because:

- In the GB/North KP/AJK terrain, off-road movement is extremely slow (the model uses 3 km/h off-road, dropping to 1.2 km/h on slopes >25°). A pixel 10 km from a road in flat terrain might take 3–4 hours off-road in practice.
- The mean distance to roads in the AOI is ~4,500 m and the mean travel time is ~187 minutes — this is a remote, difficult landscape. A 60-minute half-life would flag most of the AOI as effectively inaccessible, which is not useful for distinguishing between locations. A 90-minute half-life gives meaningful spread.
- 90 minutes corresponds approximately to the practical threshold for a guided hiking excursion from a roadhead — a common format for adventure tourism in the region.

### How Travel Time Is Calculated

1. **Slope is computed** from the DEM using the actual pixel spacing (100m × 100m) to get degrees of incline. Steeper terrain = slower movement.

2. **Speed is assigned** to each pixel:
   - On a road: speed from the road type (motorway = 80 km/h, track = 10 km/h, etc.)
   - Off-road: 3 km/h (walking pace)

3. **Terrain impedance** is applied — a slope modifier reduces speed:
   - Flat (≤5°): no reduction (modifier 1.0)
   - Gentle (5–15°): 20% reduction (modifier 0.8)
   - Moderate (15–25°): 40% reduction (modifier 0.6)
   - Steep (>25°): 60% reduction (modifier 0.4)

4. **Distance to nearest road** is computed for every pixel using a Euclidean distance transform (EDT). This efficiently gives the straight-line distance to the nearest road pixel for all 12 million pixels simultaneously.

5. **Travel time** = distance ÷ terrain-adjusted speed.

6. **AI** = exp(−λ × travel time).

> **Note on method**: This is a proximity-based approximation, not full network routing. It assumes straight-line travel (no path-finding around obstacles). This is the standard approach for large-area raster analysis at this scale — full routing for 12 million pixels is computationally infeasible. See Weiss et al. (2018) for the academic basis.

---

## 5. Data Sources and Why Each Was Chosen

### DEM (Digital Elevation Model)
- **File**: `dem_32643_100m.tif`
- **Purpose**: Source for slope calculation (AI), and defines the valid AOI pixels (nodata mask)
- **Why**: The DEM is the geometric backbone of all terrain-derived products. All other rasters are aligned to match it exactly (same shape, resolution, CRS).

### TPI — Topographic Position Index
- **File**: `tpi_zscore_radius2_32643_100m.tif`
- **What it measures**: Whether a pixel is a ridge (positive TPI), valley (negative TPI), or flat terrain (near zero), relative to its neighbourhood.
- **Why pre-processed**: The raw TPI in GB/Karakoram has extreme outliers (cliff faces at ±1200m deviation). The upstream `tpi_new` notebook clipped at ±300m and then z-scored. We use that output directly rather than re-processing.
- **In SPI**: Captures the "peak vs valley" dimension of terrain drama.

### TRI — Terrain Ruggedness Index
- **File**: `tri_32643_100m.tif`
- **What it measures**: The mean absolute elevation difference between a pixel and its 8 neighbours — a measure of local roughness. TRI is always ≥ 0.
- **Why complement TPI**: TPI captures position (ridge/valley), TRI captures texture (smooth vs jagged). A flat plateau has low TPI but also low TRI; a jagged ridge has high positive TPI and high TRI. Together they give a fuller picture of terrain drama.
- **Processing**: Clipped at P99.9 to remove extreme cliff-face pixels before z-scoring.

### Forest Mask
- **File**: `forest_mask_32643_100m.tif`
- **What it measures**: Binary (0/1) — whether a pixel is classified as trees or shrubs in the 2024 land cover data.
- **Processing**: Converted to neighbourhood density (3km window) before z-scoring, because a binary mask z-scores poorly (two-spike distribution).

### Water Mask
- **File**: `water_mask_32643_100m.tif`
- **What it measures**: Binary (0/1) — whether a pixel is classified as a water body.
- **Glacier removal**: Pixels where snow frequency > 0.5 are removed from the water mask before processing. This prevents glaciers from being counted in both the Water and Snow components (double-counting).
- **Processing**: Same neighbourhood density approach as forest mask.

### Snow Frequency
- **File**: `snow_frequency_aligned_32643_100m.tif`
- **What it measures**: The proportion of observation days when a pixel was covered by snow (0 = never, 1 = always).
- **In SPI**: Higher snow frequency = more consistently snowy = higher scenic value in a mountain context.

### Distance to Roads
- **File**: `dist_to_roads_all_32643_100m.tif`
- **Used in**: AI calculation (as a cross-check; the notebook recomputes travel time from the vector road network for speed type attribution).

### Road Network (Vector)
- **File**: `roads_all_32643.gpkg`
- **What it contains**: All road types from the highway field (motorway, primary, secondary, tertiary, track, etc.) — 65,462 road segments.
- **Used in**: AI — rasterized to assign speed values to road pixels.

### Tehsil Boundaries
- **Source**: `geoBoundaries-PAK-ADM3.geojson` (admin level 3 — tehsil)
- **Filtered to**: 81 tehsils in the study area using the `aoi_tehsils_list.txt` names plus hardcoded AJK tehsils.
- **Used in**: Zonal statistics — computing mean SPI and AI per tehsil.

---

## 6. Step-by-Step: SPI Calculation

### Step 0: Load Rasters

All rasters are loaded and verified to share the same shape `(4801, 6122)` and CRS `EPSG:32643`. This is essential — if any raster is misaligned by even one pixel, the weighted sum would be adding values from different geographic locations.

### Step 0.5: Glacier Removal from Water Mask

Before any processing, pixels where `snow_frequency > 0.5` are set to 0 in the water mask.

**Why**: Glaciers are classified as "water" in many land cover datasets but they are permanent ice — not rivers or lakes. The Snow component already captures glaciated terrain. Leaving glaciers in the Water mask would give high-altitude glaciated zones credit from two components simultaneously, inflating SPI in the Karakoram where glaciers are abundant. After removal, the Water component reflects only actual liquid water: rivers, lakes, seasonal wetlands.

### Step 1: Build the SPI Nodata Mask

A pixel is only considered valid if **all five** input layers have valid data at that location:
- TPI z-score is not NaN
- TRI is not NaN
- Forest mask is not NaN
- Water mask is not NaN (after glacier removal)
- Snow frequency is not NaN

This gives ~12.5 million valid pixels (42.6% of the bounding box). This mask is created once and used consistently across both SPI and AI — ensuring both indices cover exactly the same spatial extent.

### Step 2: Convert Binary Masks to Neighbourhood Density

Both the forest mask and water mask are binary (0 = absent, 1 = present). Z-scoring a binary variable produces a bimodal distribution (two spikes at −0.31 and 3.16 for something that is true 9% of the time). This is not a useful continuous signal.

Instead, for each pixel, we compute the **proportion of pixels within a 3km × 3km window that are forested (or watered)**. This produces a continuous density surface: a pixel in the middle of a dense forest gets a high value, a pixel at the edge of a forest gets an intermediate value, and a pixel in open terrain gets zero.

**Why 3km?**: 3km (30 pixels at 100m resolution) is a landscape-scale context window. It captures what a visitor standing at a location would see around them — not just their immediate pixel, but the surrounding landscape character. This matches how scenic quality is actually perceived.

The density computation uses a **masked neighbourhood filter** — pixels outside the AOI are excluded from the denominator, so edge pixels are not penalised for having nodata neighbours.

### Step 3: Clip and Z-Score TRI

TRI is always positive (ruggedness cannot be negative). Its raw range in the AOI is 0 to 1091m. Before z-scoring, extreme outliers are clipped at the 99.9th percentile (approximately 158m in this dataset). This removes 0.1% of the most extreme cliff-face pixels — the same principle used in the upstream TPI processing.

**Why clip before z-scoring?**: If even a handful of pixels have values 5–10× higher than the rest, the standard deviation is inflated, and all other pixels get compressed toward zero in the z-score. Clipping first ensures the z-score spread reflects the actual terrain variation across the 99.9% of normal pixels.

### Step 4: Compute Relief Index

```
relief_combined = 0.5 × TPI_z + 0.5 × TRI_z
```

TPI captures **where** in the topography a pixel sits (ridge/valley position). TRI captures **how rough** the local terrain is. Averaging them equally combines both dimensions into a single relief signal.

**Why re-z-score after combining?**

When you average two z-scored variables (each with mean=0, std=1), the result also has mean=0, but its standard deviation is not 1 — it's approximately 0.71 (because `Var(0.5a + 0.5b) = 0.25 + 0.25 + 0.5·Cov(a,b)`, and the covariance between TPI and TRI is positive but less than 1). If the Relief variable has std=0.71 while Forest, Water, and Snow each have std=1, then Relief's effective contribution to the weighted sum is `0.35 × 0.71 = 0.25`, not the intended 35%. Re-z-scoring corrects this so each component contributes exactly its assigned weight.

### Step 5: Z-Score All Components

Each component (Relief, Forest density, Water density, Snow frequency) is z-scored across all valid AOI pixels:

```
z = (value − mean) / standard_deviation
```

After z-scoring: mean = 0, std = 1 for every component. This puts all four on the same scale so the weights are meaningful — each unit of z-score difference has the same significance regardless of which component it came from.

### Step 6: Clip Z-Scores at ±3.5σ

After z-scoring, all four components are clipped at ±3.5 standard deviations.

**Why**: In a normal distribution, only 0.046% of values lie beyond ±3.5σ. Pixels at ±10σ (as seen in the original TPI output) are data artifacts — isolated cliff-edge pixels, single-pixel anomalies, boundary effects. Without clipping, these extreme pixels dominate the weighted SPI sum in their local area, making the weights meaningless. A pixel at TPI z = +10.7 contributes `0.35 × 10.7 = 3.75` to the SPI sum; a pixel at TPI z = +3.5 contributes `0.35 × 3.5 = 1.23`. Clipping at ±3.5 caps each component's maximum possible contribution to `weight × 3.5`, preserving the intended weight design for 99.95% of the AOI.

### Step 7: Compute SPI as Weighted Sum

```python
SPI = 0.35 × relief_z + 0.25 × forest_z + 0.20 × water_z + 0.20 × snow_z
```

The result is a continuous raster. Pixels outside the AOI (nodata mask) are set to NaN. The final SPI has:
- Mean ≈ 0 (because all inputs are mean-zero z-scores)
- Range approximately −2 to +4 (asymmetric because forest, water, and snow density are right-skewed — most pixels have low density, a few have very high density)

---

## 7. Step-by-Step: AI Calculation

### Step 0: Adopt the SPI Nodata Mask

The AI cell uses `spi_nodata_mask` as its spatial mask rather than rebuilding a mask from the DEM alone. This is critical: SPI and AI must share exactly the same valid pixel footprint. The SPI mask is stricter (requires all 5 input layers to be valid), so any pixel excluded from SPI is also excluded from AI. This ensures the combined index has a consistent spatial domain.

### Step 1: Compute Slope from DEM

Slope is calculated using `numpy.gradient`, which computes the rate of elevation change in both x and y directions using the actual pixel spacing (100m):

```python
dy, dx = np.gradient(dem, pixel_size_y, pixel_size_x)
slope_degrees = arctan(sqrt(dx² + dy²))
```

Passing the actual pixel size (100m) to `np.gradient` is essential. Without it, `np.gradient` assumes unit spacing (1 pixel = 1 unit), which would produce slopes in "elevation per pixel" rather than "elevation per metre" — giving slopes roughly 100× too large.

In this AOI, 58.5% of pixels have slopes >25° — this is a genuinely steep landscape. This matters for the AI because most off-road movement is heavily penalised by terrain.

### Step 2: Assign Road Speeds

Each road segment is assigned a travel speed based on its highway classification:

| Road Type | Speed |
|---|---|
| Motorway | 80 km/h |
| Trunk | 60 km/h |
| Primary | 45 km/h |
| Secondary | 35 km/h |
| Tertiary | 25 km/h |
| Unclassified/Residential | 15–20 km/h |
| Track | 10 km/h |
| Off-road (default) | 3 km/h |

These speeds represent realistic travel speeds in mountain terrain, not ideal highway speeds. Off-road is 3 km/h — a conservative walking pace reflecting pathless mountain movement.

### Step 3: Rasterize Roads

The vector road network (65,462 segments) is burned into a raster matching the DEM grid. Each road pixel gets the speed value of the road type it contains. Where multiple road types overlap, the faster one wins. Road pixels cover 4.4% of the AOI.

`all_touched=True` is used in the rasterization — for linear features (roads), this ensures every pixel the road line passes through is captured, not just pixels whose centre the line crosses.

### Step 4: Apply Terrain Impedance

Speed is multiplied by a slope modifier for every pixel:

```
modifier = 1.0  if slope ≤ 5°
modifier = 0.8  if 5° < slope ≤ 15°
modifier = 0.6  if 15° < slope ≤ 25°
modifier = 0.4  if slope > 25°
```

This applies to all pixels — roads and off-road alike. A motorway on a steep hillside is slower than the same motorway on flat terrain. Off-road movement on slopes >25° becomes `3 × 0.4 = 1.2 km/h` — slower than a casual walk, which is realistic for steep pathless terrain.

A minimum speed floor of 1 km/h is enforced — no pixel is treated as completely impassable, since people can in principle move anywhere at some speed.

### Step 5: Compute Travel Time Surface

For every pixel, we need the time to reach the nearest road:

1. **Euclidean Distance Transform (EDT)**: `scipy.ndimage.distance_transform_edt` efficiently computes the straight-line distance from every non-road pixel to the nearest road pixel, for all 12.5 million pixels in one pass. The `sampling=(100, 100)` parameter tells it that pixels are 100m apart, so output distances are in metres.

2. **Convert to time**: `time_min = distance_m / (speed_km_h × 1000/60)`

3. **Apply mask**: Pixels outside the AOI get NaN.

The maximum travel time in the AOI is ~2,567 minutes (about 43 hours) — the most remote pixels are extremely difficult to reach.

### Step 6: Convert to AI via Exponential Decay

```python
λ = ln(2) / 90 ≈ 0.00770
AI = exp(−λ × travel_time_minutes)
```

Key values for interpretation:
```
  0 min → AI = 1.000   (on the road)
 15 min → AI = 0.891
 30 min → AI = 0.794
 60 min → AI = 0.630
 90 min → AI = 0.500   ← half-life
120 min → AI = 0.397
240 min → AI = 0.157
480 min → AI = 0.025
```

The mean AI across the AOI is approximately 0.46, reflecting that a significant portion of this landscape is more than 90 minutes from a road.

---

## 8. Combining SPI and AI

### Why They Need to Be on the Same Scale First

SPI is a z-score index (range ≈ −2 to +4, mean ≈ 0). AI is a probability-like score (range 0 to 1). Averaging them directly would be meaningless — a location with SPI = 2.0 and AI = 0.5 would get a combined score of `0.5 × 2.0 + 0.5 × 0.5 = 1.25`, which is outside the AI's [0,1] range and dominated by the SPI's larger numerical scale.

### The Normalization: P2–P98 Robust Rescaling

```python
spi_lo = np.nanpercentile(spi_data[spi_valid], 2)
spi_hi = np.nanpercentile(spi_data[spi_valid], 98)
spi_01 = clip((spi_data − spi_lo) / (spi_hi − spi_lo), 0, 1)
```

**Why P2/P98 and not absolute min/max?**

The absolute SPI minimum and maximum are extreme outlier pixels — even after ±3.5σ clipping on individual components, the weighted sum can still produce occasional extreme values at bounding box edges or in unusual pixel configurations. Using the absolute extremes as anchors compresses the meaningful scenic range for 98% of pixels into a narrow band. P2/P98 robustly represents the realistic range of scenic variation: 2% of pixels are floor-clamped to 0, 2% are ceiling-clamped to 1, and the other 96% span the full [0,1] range. This is the same approach used for SPI visualization, so the saved combined raster is consistent with what the maps show.

### The Combined Index

```
Combined = 0.5 × SPI_normalized + 0.5 × AI
```

Equal weighting between scenic potential and accessibility was chosen because:
- A location is only useful for tourism if it scores reasonably on both dimensions.
- Equal weights ensure neither dimension dominates — a spectacular but completely inaccessible location should not score higher than a moderately scenic but easily accessible one.
- This weight can be adjusted by stakeholders (e.g., 0.6 SPI + 0.4 AI to prioritize scenic quality, or 0.4 SPI + 0.6 AI to prioritize accessibility).

---

## 9. Zonal Statistics by Tehsil

After computing pixel-level SPI and AI rasters, we aggregate to the tehsil level. For each of the 81 tehsils, we:

1. Create a pixel mask from the tehsil polygon geometry (using `rasterio.features.geometry_mask`).
2. Extract all valid SPI and AI values within the polygon.
3. Compute: mean, median, standard deviation, min, max, P25, P75, count, and range.

**Why multiple statistics?**

- **Mean**: Best for comparing tehsils to each other.
- **Median**: More robust than mean for skewed distributions — useful if a tehsil has a small high-value hotspot.
- **Std / Range**: Tells you how internally diverse a tehsil is — a high-std tehsil might have spectacular spots alongside unremarkable terrain.
- **Count**: Validates that enough pixels fell within the polygon for the statistics to be reliable (a tehsil with only 10 valid pixels should be treated cautiously).

The tehsil names are matched using a normalized string comparison (uppercase, alphanumeric only) to handle variations in spelling and special characters across different data sources.

---

## 10. Key Design Choices — Defended

### Why Z-Score Normalization for SPI Components?

Z-scoring is the standard approach for combining heterogeneous variables into a composite index. It achieves two things:
1. **Centering**: Subtracting the mean ensures each component's zero point is the AOI average, so a pixel at average forest density contributes zero to the SPI, not a positive or negative bias.
2. **Scaling**: Dividing by the standard deviation ensures each unit of deviation means the same thing across all components — one standard deviation of forest density difference has the same numerical impact as one standard deviation of relief difference.

Without z-scoring, the weights would interact with the original measurement scales (TRI in metres, snow in proportions, forest in binary), making the nominal weights meaningless.

### Why Neighbourhood Density for Forest and Water?

Scenic quality is perceived at the landscape scale, not the pixel scale. A visitor standing in a single-pixel forest at the edge of a clearing does not experience the same scenic value as a visitor surrounded by dense forest in all directions. Neighbourhood density captures this: it answers "how forested is the landscape around this pixel?" rather than "is this exact pixel a tree?". The 3km window (30 pixels) represents the approximate horizon of landscape perception in mountain terrain.

### Why Separate TPI and TRI?

- **TPI (Topographic Position Index)** is directional: positive = ridge, negative = valley, zero = slope or flat. It captures the positional drama of the landscape.
- **TRI (Terrain Ruggedness Index)** is always positive: it captures jaggedness regardless of position. A rugged plateau has high TRI but near-zero TPI.

Using only TPI would miss the ruggedness of high plateaus and penalise valleys (negative TPI) even when they are dramatically scenic. Using only TRI would miss the distinction between ridge positions (with views in all directions) and valley positions. Their equal combination captures both dimensions of terrain character.

### Why Remove Glaciers from the Water Mask?

In the 2024 land cover data, glaciers are frequently classified as water bodies because their spectral signature overlaps with open water in satellite imagery. However, glaciers are not rivers or lakes — they are permanent ice. The Snow/Ice component already represents them through the snow frequency data (glaciers have snow_frequency near 1.0). Including them in the Water component as well would count the same physical feature twice, biasing the SPI upward in heavily glaciated tehsils (primarily in upper Gilgit-Baltistan). The threshold of 0.5 (snow present >50% of days) is a reasonable operational definition of "permanent ice" versus seasonal snow.

### Why Clip Z-Scores at ±3.5σ?

In any real-world dataset, a small number of pixels will have extreme values due to data artifacts: pixels at the exact edge of a sensor swath, cliff faces where the DEM is uncertain, single-pixel water bodies that are actually cloud shadows, etc. These extreme values, when z-scored, can reach ±10σ or beyond. In the weighted sum, a single +10σ pixel in the Relief component contributes `0.35 × 10 = 3.5` to the SPI — dominating the contributions from all other components combined. Clipping at ±3.5σ bounds the maximum contribution of any one component to `weight × 3.5 = 1.225` for Relief. This ensures the weight design (0.35/0.25/0.20/0.20) actually controls relative influence across 99.95% of the AOI.

### Why the Euclidean Distance Transform for AI?

The Euclidean Distance Transform is the standard raster-based approach for computing distance-to-nearest-feature at scale. Computing road network distance (via actual routing) for 12.5 million pixels is computationally infeasible. The Euclidean approach is a well-established approximation used in the academic literature (Weiss et al. 2018, *Nature*). Its main limitation is that it assumes straight-line travel, which overestimates accessibility in areas where cliffs or rivers block direct movement. This is an accepted limitation at this scale.

---

## 11. Limitations and Caveats

### What SPI Does Not Capture

- **Cultural and heritage value**: Forts, ancient trade routes, shrines — not in any input layer.
- **Seasonal variation**: SPI is computed from multi-year averages (snow) and a single year of land cover (2024). A location covered by snow 60% of the year may be inaccessible for much of that period.
- **Micro-scale features**: At 100m resolution, individual waterfalls, single large trees, or narrow gorges may not be captured.
- **Viewshed**: SPI measures what is *at* a location, not what can be *seen from* it. A ridge with panoramic views would need a viewshed analysis to capture that quality.

### What AI Does Not Capture

- **Actual road condition**: A "primary" road in the dataset may be a rough jeep track in practice. Road quality data is not available in OSM at this level of detail for the study area.
- **Seasonal road closures**: Many mountain roads are blocked by snow 4–6 months per year. The AI represents year-round average accessibility.
- **River crossings**: The distance transform passes straight through rivers. In reality, a river without a bridge is an absolute barrier.
- **Elevation gain**: Two pixels at the same Euclidean distance from a road may have very different actual travel times depending on whether travel involves ascending 1000m of vertical relief.

### The Combined Index Assumption

Equal weighting of SPI and AI (0.5/0.5) is a defensible starting point but is a value judgement. Different users may have different priorities. The individual SPI and AI rasters are saved separately for users who want to apply custom weights.

---

## 12. Output Files

| File | Location | Description |
|---|---|---|
| `spi_index.tif` | `data/processed/` | Raw SPI raster (z-score scale, ~−2 to +4) |
| `ai_index.tif` | `data/processed/` | AI raster (0 to 1 scale) |
| `spi_ai_combined.tif` | `data/processed/` | Combined index (0 to 1, P2/P98 normalized SPI + AI) |
| `tehsil_spi_ai_zonal_stats.csv` | `data/processed/` | Per-tehsil statistics for SPI and AI |
| `spi_visualization.png` | `outputs/` | SPI map + distribution histogram |
| `ai_visualization_v2.png` | `outputs/` | AI map, slope map, travel time map, AI histogram |
| `spi_ai_combined_visualization.png` | `outputs/` | Side-by-side SPI and AI maps with distributions |
| `spi_ai_overlay_map.png` | `outputs/` | Combined index map |
| `spi_ai_analysis_summary.txt` | `outputs/` | Plain-text summary report with top/bottom tehsils |

### Reading the Zonal Statistics CSV

Each row is one tehsil. Key columns:

| Column | Meaning |
|---|---|
| `tehsil` | Tehsil name (from shapeName field) |
| `spi_mean` | Average SPI score — use this to rank tehsils |
| `spi_median` | Median SPI — more robust if distribution is skewed |
| `spi_std` | Internal variation — high std = diverse landscape |
| `spi_count` | Number of valid pixels in this tehsil |
| `ai_mean` | Average accessibility — higher = more accessible |
| `ai_median` | Median AI |

Tehsils with `spi_count = 0` or `ai_count = 0` had no valid pixels within their boundary — this can happen for very small tehsils or for boundaries that lie entirely outside the AOI.
