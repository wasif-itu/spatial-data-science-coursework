# BSCS23020 EX2: DN to Reflectance (Simple, Step-by-Step)

This note explains the notebook in simple words. It follows the same order as the notebook and covers both the idea and the code at each step.

## 1) What reflectance means (concept)
Reflectance is how much light a surface reflects compared to how much light hits it. It is unitless and usually between 0 and 1 (0% to 100%). We prefer reflectance over raw sensor values because it is more related to the material and less to the sensor or lighting conditions.

## 2) Activate the environment (code)
You need the conda environment and the library `rioxarray` to read raster images.

- Concept: Make sure the right tools are loaded before running the code.
- Code action:
  - Activate conda: `conda activate rsdm`
  - Install `rioxarray` if missing.

## 3) Set your working folders (concept + code)
- Concept: Tell Python where your data lives so it can find the files.
- Code action:
  - Set `rootDirectory` to the project folder.
  - Change to that folder using `os.chdir`.
  - Build `dataDirectory` for the Landsat product.

## 4) Choose the band file (concept + code)
- Concept: Landsat data is split by bands. Here we use Band 4 (red).
- Code action:
  - Set `band = 4`.
  - Build the filename: `LC09_L1TP_..._B4.TIF`.

## 5) Read metadata (concept + code)
- Concept: The MTL metadata file has the scaling factors needed to convert DN to reflectance.
- Code action:
  - Read the MTL text file line-by-line.
  - Store key/value pairs in `Landsat9_mtt_dict`.

## 6) Get scaling factors (concept + code)
- Concept: The reflectance conversion needs:
  - `REFLECTANCE_MULT_BAND_4` (multiplicative factor)
  - `REFLECTANCE_ADD_BAND_4` (additive factor)
  - `SUN_ELEVATION` (sun elevation angle)
- Code action:
  - Read these three values from the dictionary and print them.

## 7) Read the band image (concept + code)
- Concept: The DN image is the raw data in 16-bit integers.
- Code action:
  - Use `rioxarray.open_rasterio(image_L1_filename)`.

## 8) Convert strings to numbers (concept + code)
- Concept: Metadata values come as strings; math requires floats.
- Code action:
  - Convert `mult`, `add`, `sun_elev` to floats.
  - Compute solar zenith angle in radians: `90 - sun_elev`, then `np.radians`.

## 9) Check no-data values (concept + code)
- Concept: Some pixels are invalid and must be masked.
- Code action:
  - Use `dn_image.rio.nodata` to read the no-data value.

## 10) Convert DN to reflectance (concept + code)
- Concept: First compute uncorrected TOA reflectance, then adjust for sun angle.
- Formula:
  - Uncorrected: rho' = M * DN + A
  - Corrected: rho = rho' / cos(theta_sun_zenith)
- Code action:
  - `reflectance_b4 = (m_lambda * dn_image + a_lambda)`
  - `reflectance_b4 = reflectance_b4 / np.cos(sun_zenith_angle_rad)`

## 11) Plot DN vs reflectance (concept + code)
- Concept: DN values can be 0 to 65535; reflectance is typically 0 to 1.
- Code action:
  - Create a no-data mask.
  - Plot both images side by side with colorbars.

## 12) Why reflectance can exceed 1 (concept + code)
- Concept: Reflectance can go above 1 if the sun angle correction amplifies values or if very bright surfaces are present.
- Code action:
  - Create a mask of values > 1.
  - Plot the mask to see where it happens.
  - Optional fix: clip to [0, 1] for visualization.

## 13) Pixel-wise sun angle correction (concept + code)
- Concept: Sun angles change across the scene. Using per-pixel angles gives a better correction than one scene-wide value.
- Code action:
  - Move into `l8_angles` directory.
  - Load the `*_solar_B04.img` file.
  - Note: band 1 is azimuth, band 2 is zenith.

## 14) Scale solar angles (concept + code)
- Concept: The angle file stores angles in hundredths of a degree.
- Code action:
  - `solar_zenith_angle = solar_angles.sel(band=2) / 100`
  - Replace no-data values with `NaN`.

## 15) Visual checks (concept + code)
- Concept: Plot the solar zenith angle image and the DN image to compare.
- Code action:
  - Use `.plot()` for quick visualization.
  - Plot both in a 1x2 figure.

## 16) Recompute reflectance using pixel-wise angles (concept + code)
- Concept: Use the local angle for each pixel in the correction.
- Code action:
  - `reflectance_b4_pw_cor = (m_lambda * dn_image + a_lambda) / np.cos(np.radians(solar_zenith_angle))`
  - Extract band 1 as `reflectance_b4_image`.

## 17) Inspect results (concept + code)
- Concept: The corrected reflectance image should look similar but values are more accurate.
- Code action:
  - Plot values greater than 1.
  - Inspect the output array.

## 18) Final takeaway (concept)
- DN values are raw sensor numbers.
- Reflectance is physically meaningful and comparable between scenes.
- Using pixel-wise sun angles improves correction, even if the visual change is subtle.
