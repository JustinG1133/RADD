Data Details:
-------------

Example python code for pulling from a dataset that is stored as an **Image**
is:

```dataset
elevation = dataset.select('elevation')
slope = ee.Terrain.slope(elevation)

m = geemap.Map()
m.set_center(-112.8598, 36.2841, 10)
m.add_layer(slope, {'min': 0, 'max': 60}, 'slope')
```

Example python code when the dataset is stored as an **Image Collection** is:

```import
import geemap

ee.Initialize()

# Load the collection, filter by date and region, select a band
collection = (
    ee.ImageCollection('NASA/SMAP/SPL3SMP_E/006')
    .filterDate('2023-06-01', '2023-08-31')
    .filterBounds(ee.Geometry.Point(-88.8, 33.45))
    .select('soil_moisture_am')
)

# Reduce the collection to a single image (temporal mean)
sm_mean = collection.mean()

m = geemap.Map()
m.set_center(-88.8, 33.45, 6)
m.add_layer(
    sm_mean,
    {'min': 0.0, 'max': 0.5, 'palette': ['brown', 'yellow', 'green', 'blue']},
    'SMAP mean soil moisture',
)
m
```

### Needed:

#### [MODIS Terra Surface Reflectance](https://developers.google.com/earth-engine/datasets/catalog/MODIS_061_MOD09GA)

**NDVI** = $\frac{NIR - RED}{NIR + RED}$ ; RED = sur\_refl\_b01, NIR =
sur\_refl\_b02

**VWC** =
$(1.9134* NDVI^2 -0.3215*NDVI) + stemfactor *\frac{NDVI_{max}-NDVI{min}}{1- NDVI_{min}}$

| Band name      | Pixel Size | Min-Max Value  | Description                                                     |
| -------------- | ---------- | -------------- | --------------------------------------------------------------- |
| sur\_refl\_b01 | 500m       | -100 to 16,000 | Surface Reflectance for band 1                                  |
| sur\_refl\_b02 | 500m       | -100 to 16,000 | Surface Reflectance for band 2                                  |
| QC\_500m       | 500m       | N/A            | Surface Reflectance quality ssurance                            |
| state\_1km     | 1000m      | N/A            | Reflectance state QA flags (cloud, cloud shadow, aerosol, snow) |

#### [MODIS Land Cover Type](https://developers.google.com/earth-engine/datasets/catalog/MODIS_061_MCD12Q1)

| Band name | Pixel Size | Min-Max Value | Description                                                                                                           |
| --------- | ---------- | ------------- | --------------------------------------------------------------------------------------------------------------------- |
| LC\_Type1 | 500m       | N/A           | Land Cover Type 1: Annual International Geosphere-Biosphere Programme (IGBP) classification (used to get STEM factor) |
| QC        | 500m       | N/A           | Product Quality Flags                                                                                                 |

#### [ERA5 Hourly Climate Reanalysis](https://developers.google.com/earth-engine/datasets/catalog/ECMWF_ERA5_HOURLY)

| Band name                   | Pixel Size | Min-Max Value | Description                                                                                                                                                                                                                                                                           |
| --------------------------- | ---------- | ------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| temperature\_2m             | 27830m     | N/A (K)       | Temperature of Air 2m above the surface of land, sea or inland water.                                                                                                                                                                                                                 |
| dewpoint\_temperature\_2m   | 27830m     | N/A (K)       | The temperature to which the air, at 2 metres above the surface of the Earth, would have to be cooled for saturation to occur. It is a measure of the humidity of the air. Combined with temperature, it can be used to calculate the relative humidity                               |
| total\_precipitation        | 27830m     | N/A (m)       | Accumulated liquid and frozen water (rain and snow) reaching the surface, from both large-scale and convective precipitation. Used instead of large\_scale\_precipitation, which leaves out convective rain.                                                                          |
| soil\_temperature\_level\_1 | 27380m     | N/A (K)       | The temperature of the soil at level 1 (in the middle of layer 1). The ECMWF Integrated Forecasting System (IFS) has a four-layer representation of soil, where the surface is at 0cm: Layer 1: 0 - 7cm                                                                               |
| soil\_type                  | 27830m     | N/A           | The texture (or classification) of soil used by the land surface scheme of the ECMWF Integrated Forecasting System (IFS) to predict the water holding capacity of soil in soil moisture and runoff calculations. It is derived from the root zone data (30-100 cm below the surface). |

#### [SMAP Surface and Root Zone Soil Moisture](https://developers.google.com/earth-engine/datasets/catalog/NASA_SMAP_SPL4SMGP_008)

| Band name    | Pixel Size | Min-Max Value | Description                                               |
| ------------ | ---------- | ------------- | --------------------------------------------------------- |
| sm\_surface  | 11km       | 0 to 0.9      | Top layer soil moisture (0-5 cm)                          |
| sm\_rootzone | 11km       | 0 to 0.9      | <p>Root zone soil moisture (0 - 100 cm)</p>               |
| sm\_profile  | 11km       | 0 to 0.9      | <p>Total profile soil moisture (0cm to bedrock depth)</p> |

#### <mark> **TODO: After recieving feedback, need to add the USGS 1 meter DEM. BUT still keep the SRTM 30 meter DEM because the 1 meter is not quite available everywhere in CONUS.** </mark>

#### [SRTM Digital Elevation Model](https://developers.google.com/earth-engine/datasets/catalog/USGS_SRTMGL1_003)

| Band name | Pixel Size | Min-Max Value | Description      |
| --------- | ---------- | ------------- | ---------------- |
| elevation | 30m        | -10 to 6500   | <p>Elevation</p> |

### Optional:

#### [MODIS Terra Vegetation Continous Fields](https://developers.google.com/earth-engine/datasets/catalog/MODIS_061_MOD44B#bands)

| Band name                   | Pixel Size | Min-Max Value | Description                                                                                                                                                                                       |
| --------------------------- | ---------- | ------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Percent\_Tree\_Cover        | 250m       | 0 - 100 (%)   | Percent of a pixel which is covered by tree canopy                                                                                                                                                |
| Percent\_NonTree\_Vegetated | 250m       | 0 - 100 (%)   | Percent of a pixel which is covered by non-tree vegetation                                                                                                                                        |
| Percent\_NonVegetated       | 250m       | 0 - 100 (%)   | Percent of a pixel which is not vegetated                                                                                                                                                         |
| Quality                     | 250m       | N/A           | Describes those inputs that had poor quality (cloudy, high aerosol, cloud shadow, or view zenith >45°). Each bit in the field represents 1 out of 8 input surface reflectance files to the model. |
