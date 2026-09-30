Data Details:
-------------

### Needed:

#### [MODIS Terra Surface Reflectance](https://developers.google.com/earth-engine/datasets/catalog/MODIS_061_MOD09GA)

**NDVI** = $\frac{NIR - RED}{NIR + RED}$ ; RED = sur\_refl\_b01, NIR =
sur\_refl\_b02

**VWC** =
$(1.9134* NDVI^2 -0.3215*NDVI) + stemfactor *\frac{NDVI_{max}-NDVI{min}}{1- NDVI_{min}}$

| Band name      | Pixel Size | Min-Max Value  | Description                           |
| -------------- | ---------- | -------------- | ------------------------------------- |
| sur\_refl\_b01 | 500m       | -100 to 16,000 | Surface Reflectance for band 1        |
| sur\_refl\_b02 | 500m       | -100 to 16,000 | Surface Reflectance for band 2        |
| QC\_500m       | 500m       | N/A            | Surface Reflectance quality Assurance |

#### [MODIS Land Cover Type](https://developers.google.com/earth-engine/datasets/catalog/MODIS_061_MCD12Q1)

| Band name | Pixel Size | Min-Max Value | Description                                                                                                           |
| --------- | ---------- | ------------- | --------------------------------------------------------------------------------------------------------------------- |
| LC\_Type1 | 500m       | N/A           | Land Cover Type 1: Annual International Geosphere-Biosphere Programme (IGBP) classification (used to get STEM factor) |
| QC        | 500m       | N/A           | Product Quality Flags                                                                                                 |

#### [ERA5 Hourly Climate Reanalysis](https://developers.google.com/earth-engine/datasets/catalog/ECMWF_ERA5_HOURLY)

| Band name | Pixel Size | Min-Max Value | Description |
| --------- | ---------- | ------------- | ----------- |
| TODO      | TODO       | TODO          | TODO        |

#### SMAP Soil Moisture

TODO: pick SMAP product and add link

| Band name | Pixel Size | Min-Max Value | Description |
| --------- | ---------- | ------------- | ----------- |
| TODO      | TODO       | TODO          | TODO        |

#### [SRTM Digital Elevation Model](https://developers.google.com/earth-engine/datasets/catalog/USGS_SRTMGL1_003)

| Band name | Pixel Size | Min-Max Value | Description |
| --------- | ---------- | ------------- | ----------- |
| TODO      | TODO       | TODO          | TODO        |

### Optional:

#### [MODIS Terra Vegetation Continous Fields](https://developers.google.com/earth-engine/datasets/catalog/MODIS_061_MOD44B#bands)

| Band name                   | Pixel Size | Min-Max Value | Description                                                                                                                                                                                       |
| --------------------------- | ---------- | ------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Percent\_Tree\_Cover        | 250m       | 0 - 100 (%)   | Percent of a pixel which is covered by tree canopy                                                                                                                                                |
| Percent\_NonTree\_Vegetated | 250m       | 0 - 100 (%)   | Percent of a pixel which is covered by non-tree vegetation                                                                                                                                        |
| Percent\_NonVegetated       | 250m       | 0 - 100 (%)   | Percent of a pixel which is not vegetated                                                                                                                                                         |
| Quality                     | 250m       | N/A           | Describes those inputs that had poor quality (cloudy, high aerosol, cloud shadow, or view zenith >45°). Each bit in the field represents 1 out of 8 input surface reflectance files to the model. |
