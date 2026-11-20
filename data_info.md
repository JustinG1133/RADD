Data Details
============

Example python code for pulling from a dataset that is stored as an **Image**
is:

```python
import ee
import geemap

ee.Initialize(project='your-gee-project')

dataset = ee.Image('USGS/SRTMGL1_003')
elevation = dataset.select('elevation')
slope = ee.Terrain.slope(elevation)

m = geemap.Map()
m.set_center(-112.8598, 36.2841, 10)
m.add_layer(slope, {'min': 0, 'max': 60}, 'slope')
```

Example python code when the dataset is stored as an **Image Collection** is:

```python
import ee
import geemap

ee.Initialize(project='your-gee-project')

# Load the collection, filter by date and region, select a band
collection = (
    ee.ImageCollection('NASA/SMAP/SPL4SMGP/008')
    .filterDate('2023-06-01', '2023-08-31')
    .filterBounds(ee.Geometry.Point(-88.8, 33.45))
    .select('sm_surface')
)

# Reduce the collection to a single image (temporal mean)
sm_mean = collection.mean()

m = geemap.Map()
m.set_center(-88.8, 33.45, 6)
m.add_layer(
    sm_mean,
    {'min': 0.0, 'max': 0.5, 'palette': ['brown', 'yellow', 'green', 'blue']},
    'SMAP mean surface soil moisture',
)
m
```

These snippets are for exploring a dataset. RADD itself downloads pixels with
`ee.data.computePixels` on each dataset's own grid (see
[Processing rules](spec_design.md#processing-rules)).

## Needed

### [MODIS Terra Surface Reflectance](https://developers.google.com/earth-engine/datasets/catalog/MODIS_061_MOD09GA)

**NDVI** = $\frac{NIR - RED}{NIR + RED}$ ; RED = sur\_refl\_b01, NIR =
sur\_refl\_b02

**VWC** =
$(1.9134* NDVI^2 -0.3215*NDVI) + stemfactor *\frac{NDVI_{max}-NDVI_{min}}{1- NDVI_{min}}$

RADD clamps VWC at 0 and sets it to 0 for classes 15-17 (see
[NDVI and VWC](spec_design.md#ndvi-and-vwc)).

The formula and the stem factors are from the SMAP Ancillary Data Report:
Vegetation Water Content (Chan et al., 2013), Equation 1 and Table 1. The stem
factor is a look-up value set only by the land cover class (MCD12Q1
`LC_Type1`):

| IGBP class | Land cover                         | Stem factor |
| ---------- | ---------------------------------- | ----------- |
| 1          | Evergreen needleleaf forest        | 15.96       |
| 2          | Evergreen broadleaf forest         | 19.15       |
| 3          | Deciduous needleleaf forest        | 7.98        |
| 4          | Deciduous broadleaf forest         | 12.77       |
| 5          | Mixed forest                       | 12.77       |
| 6          | Closed shrublands                  | 3.00        |
| 7          | Open shrublands                    | 1.50        |
| 8          | Woody savannas                     | 4.00        |
| 9          | Savannas                           | 3.00        |
| 10         | Grasslands                         | 1.50        |
| 11         | Permanent wetlands                 | 4.00        |
| 12         | Croplands                          | 3.50        |
| 13         | Urban and built-up                 | 6.49        |
| 14         | Cropland / natural vegetation mosaic | 3.25      |
| 15         | Snow and ice                       | 0.00        |
| 16         | Barren or sparsely vegetated       | 0.00        |
| 17         | Water bodies (not in Table 1)      | 0.00        |

The same report suggests a global NDVI\_min of 0.1. RADD uses that, with the
current day's NDVI as NDVI\_max, as the fallback when a pixel has too few valid
days (see [NDVI and VWC](spec_design.md#ndvi-and-vwc)).

| Band name      | Pixel Size | Min-Max Value  | Description                                                     |
| -------------- | ---------- | -------------- | --------------------------------------------------------------- |
| sur\_refl\_b01 | 500m       | -100 to 16,000 | Surface Reflectance for band 1                                  |
| sur\_refl\_b02 | 500m       | -100 to 16,000 | Surface Reflectance for band 2                                  |
| QC\_500m       | 500m       | N/A            | Surface Reflectance quality assurance                           |
| state\_1km     | 1000m      | N/A            | Reflectance state QA flags (cloud, cloud shadow, aerosol, snow) |

### [MODIS Land Cover Type](https://developers.google.com/earth-engine/datasets/catalog/MODIS_061_MCD12Q1)

Annual product: one image per year from 2001, dated Jan 1. Published a year or
more late, so recent dates use the latest available year (see
[Land Cover Year](spec_design.md#land-cover-year)). Raw files are cached. Its
500 m grid is the same as MOD09GA's, so each MOD09GA pixel has one class.
`LC_Type1` also sets the stem factor (above) and the surface roughness
(`roughness_table`, see [Surface Roughness](spec_design.md#surface-roughness)).

| Band name | Pixel Size | Min-Max Value | Description                                                                                                           |
| --------- | ---------- | ------------- | --------------------------------------------------------------------------------------------------------------------- |
| LC\_Type1 | 500m       | N/A           | Land Cover Type 1: Annual International Geosphere-Biosphere Programme (IGBP) classification (used to get STEM factor) |
| QC        | 500m       | N/A           | Product Quality Flags                                                                                                 |

### [ERA5 Hourly Climate Reanalysis](https://developers.google.com/earth-engine/datasets/catalog/ECMWF_ERA5_HOURLY)

Each `total_precipitation` value is the accumulation over the hour ending at
the image's time stamp ("accumulated over the 1 hour ending at the validity
date and time", GEE catalog). `dewpoint_temperature_2m` is only needed for
`dew_flag`, which is not downloaded until `dew_threshold_k` is set.

| Band name                   | Pixel Size | Min-Max Value | Description                                                                                                                                                                                                                                                                           |
| --------------------------- | ---------- | ------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| temperature\_2m             | 27830m     | N/A (K)       | Temperature of Air 2m above the surface of land, sea or inland water.                                                                                                                                                                                                                 |
| dewpoint\_temperature\_2m   | 27830m     | N/A (K)       | The temperature to which the air, at 2 metres above the surface of the Earth, would have to be cooled for saturation to occur. It is a measure of the humidity of the air. Combined with temperature, it can be used to calculate the relative humidity                               |
| total\_precipitation        | 27830m     | N/A (m)       | Accumulated liquid and frozen water (rain and snow) reaching the surface, from both large-scale and convective precipitation. Used instead of large\_scale\_precipitation, which leaves out convective rain.                                                                          |
| soil\_temperature\_level\_1 | 27830m     | N/A (K)       | The temperature of the soil at level 1 (in the middle of layer 1). The ECMWF Integrated Forecasting System (IFS) has a four-layer representation of soil, where the surface is at 0cm: Layer 1: 0 - 7cm                                                                               |

### [SMAP Surface and Root Zone Soil Moisture](https://developers.google.com/earth-engine/datasets/catalog/NASA_SMAP_SPL4SMGP_008)

3-hourly steps. In GEE the native grid is geographic (EPSG:4326, about 0.095
degrees).

| Band name    | Pixel Size | Min-Max Value | Description                                    |
| ------------ | ---------- | ------------- | ---------------------------------------------- |
| sm\_surface  | 11km       | 0 to 0.9      | Top layer soil moisture (0-5 cm)               |
| sm\_rootzone | 11km       | 0 to 0.9      | Root zone soil moisture (0 - 100 cm)           |
| sm\_profile  | 11km       | 0 to 0.9      | Total profile soil moisture (0cm to bedrock depth) |

### [USGS 3DEP 10 m Digital Elevation Model](https://developers.google.com/earth-engine/datasets/catalog/USGS_3DEP_10m_collection)

`USGS/3DEP/10m_collection` (1/3 arc-second, EPSG:4269). It replaces the
deprecated single-image asset `USGS/3DEP/10m`. The collection is tiled, so
RADD mosaics the tiles that touch the AOI on the first tile's grid. It covers
CONUS, Hawaii and the U.S. territories; outside that, or where it has gaps,
RADD uses SRTM (see [Terrain](spec_design.md#terrain)).

| Band name | Pixel Size | Min-Max Value | Description |
| --------- | ---------- | ------------- | ----------- |
| elevation | 10.2m      | N/A (m)       | Elevation   |

### [SRTM Digital Elevation Model](https://developers.google.com/earth-engine/datasets/catalog/USGS_SRTMGL1_003)

Fallback DEM, used when 3DEP 10 m does not cover the whole AOI.

| Band name | Pixel Size | Min-Max Value | Description |
| --------- | ---------- | ------------- | ----------- |
| elevation | 30m        | -10 to 6500   | Elevation   |

### [OpenLandMap Clay Content](https://developers.google.com/earth-engine/datasets/catalog/OpenLandMap_SOL_SOL_CLAY-WFRACTION_USDA-3A1A1A_M_v02)

`OpenLandMap/SOL/SOL_CLAY-WFRACTION_USDA-3A1A1A_M/v02`. Static, global, about
250 m (EPSG:4326). Clay fraction for the Mironov dielectric model (see
[Soil Texture](spec_design.md#soil-texture)). Each band is one depth; RADD
requests `b0` (surface) by default. Raw files are cached. Check the band names
and units in the GEE catalog before pinning.

| Band name | Pixel Size | Min-Max Value | Description                       |
| --------- | ---------- | ------------- | --------------------------------- |
| b0        | 250m       | 0 - 100 (%)   | Clay content (mass fraction), 0 cm  |
| b10       | 250m       | 0 - 100 (%)   | Clay content (mass fraction), 10 cm |

## Optional

### [MODIS Terra Vegetation Continuous Fields](https://developers.google.com/earth-engine/datasets/catalog/MODIS_061_MOD44B#bands)

Annual product, published a year or more late. Uses the same year rule as
MCD12Q1 (see [Land Cover Year](spec_design.md#land-cover-year)). Raw files are
cached.

| Band name                    | Pixel Size | Min-Max Value | Description                                                                                                                                                                                       |
| ---------------------------- | ---------- | ------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Percent\_Tree\_Cover         | 250m       | 0 - 100 (%)   | Percent of a pixel which is covered by tree canopy                                                                                                                                                |
| Percent\_NonTree\_Vegetation | 250m       | 0 - 100 (%)   | Percent of a pixel which is covered by non-tree vegetation                                                                                                                                        |
| Percent\_NonVegetated        | 250m       | 0 - 100 (%)   | Percent of a pixel which is not vegetated                                                                                                                                                         |
| Quality                      | 250m       | N/A           | Describes those inputs that had poor quality (cloudy, high aerosol, cloud shadow, or view zenith >45°). Each bit in the field represents 1 out of 8 input surface reflectance files to the model. |
