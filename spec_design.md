Specifications for RADD:
========================

### File Specifications:

keep all data until the 'scene' file (hdf5) is generated, then delete raw
filesto save space. hdf5 file should be saved in where?

**Potential** file path: ancillary/scene/AOI/sceneName_dateRange

### User Inputs Required:

CLI call potential example:

<mark>I would do this more symbolically: `python RADD.py AOI start_date end_date` where:
- `AOI` is a name for the area of interest. TODO: how to create an AOI? Rather than ask for user inputs, I'd considered CLI options such as `--define-aoi`
- `start_date` is...

<mark>...etc.</mark>

```python RADD.py MSU_Forest start_date(MM/DD/YYYY) end_date(MM/DD/YYY)```

or

```python RADD.py MSU_Forest start_date(MM/DD/YYYY) end_date(MM/DD/YYY) optional```

AOI
if not one previously used, then request AOI name, lat lon, if lat lon, how many
decimal places accepted?


Time range, define behavior of what user should give, should output error if time range is invalid for any reason.

### Customization:

TO-REVIEW The code should be designed in a way that makes adding additional
Google Earth Enigne hosted datasets **very easy**. Maybe by using a config file that
contains an array or something that contains all of the dataset specfic
name/tags. I guess this would require another array or some other setup that the
user can add the specified bands to request from their newly added dataset or
edit bands for existing datasets.

### Datasets:

#### Needed:

##### [MODIS Terra Surface Reflectance](data_info.md#modis-terra-surface-reflectance)

Soil Textures, Terrain Classification NDVI derived from this. TODO: confirm soil
texture / terrain classification actually come from MOD09GA

##### [MODIS Land Cover Type](data_info.md#modis-land-cover-type)

VWC derived from this land cover classification and NDVI (from MOD09GA).

##### [ERA5 Hourly Climate Reanalysis](data_info.md#era5-hourly-climate-reanalysis)

Ambient Air Temperature, Soil Temp, Soil Type? TODO: soil type source

##### [SMAP Soil Moisture](data_info.md#smap-soil-moisture)

SMAP sets? TODO: which product(s)

##### [SRTM Digital Elevation Model](data_info.md#srtm-digital-elevation-model)

DEM (srtm)

#### Optional:

##### [MODIS Terra Vegetation Continous Fields](data_info.md#modis-terra-vegetation-continous-fields)
