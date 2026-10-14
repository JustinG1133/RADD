Specifications for RADD:
========================

### File Specifications:

keep all data until the 'scene' file (hdf5) is generated, then delete raw files
to save space.

**Potential** file path: ancillary/scene/AOI/sceneName\_dateRange

When running the script:

1. Check to see if the final scene file exists
2. Before downloading each dataset, check if that specific file exists. This is
   importnant in the event of an error and it stops mid run, that way it won't
   have to re-download something. 
3. It should not generate the final scene file unless all datasets that were
   requested were downloaded successfully.

### User Inputs:

CLI call potential example:

`python RADD.py AOI MM/DD/YYYY MM/DD/YYYY`

A specific example would be:

`python RADD.py MSU_Forest 10/01/2026 10/06/2026 optional`

* `AOI` is a name for the area of interest. AOI should be saved in some way? If
  AOI is not recognized output an error to the user with instructions on how to
  make one. Like calling `python RADD.py --define-aoi (aoi_name) (lat1, lon1)
  (lat2, lon2) (lat3, lon3) (lat4, lon4)`.lat/lon should accept up to 6 decimals
  places, output an error message if too many.
* `start_date` / `end_date`'s format is MM/DD/YYYY
* `optional` argument should be added to the end of the call if the user wants
  to download the optional datasets as well.

### Customization:

TO-REVIEW The code should be designed in a way that makes adding additional
Google Earth Enigne hosted datasets **very easy**. Maybe by using a config file
that contains an array or something that contains all of the dataset specfic
name/tags. I guess this would require another array or some other setup that the
user can add the specified bands to request from their newly added dataset or
edit bands for existing datasets.

### Error Handling:

If AOI is not recognized, output to user "AOI not recognized, please give
lat/lon coordinates: (lat1, lon1) (lat2, lon2)...(lat#, lon#)

* If lat/lon provided is invalid, warn user, show the proper input format, then
  exit.

If the is a download error, wait 5 seconds and then retry the source that
failed. If it has failed 3 times warn the user with a network error message and
recommend the user try to run it again. 

**Note:** since the final scene file shouldnt have been generated because of the
network error, the other data sources files should still exist int the system
and not have to be re-downloaded.

### Datasets:

#### Needed:

##### [MODIS Terra Surface Reflectance](data_info.md#modis-terra-surface-reflectance)

##### [MODIS Land Cover Type](data_info.md#modis-land-cover-type)

##### [ERA5 Hourly Climate Reanalysis](data_info.md#era5-hourly-climate-reanalysis)

##### [SMAP Soil Moisture](data_info.md#smap-surface-and-root-zone-soil-moisture)

##### [SRTM Digital Elevation Model](data_info.md#srtm-digital-elevation-model)

#### Optional:

##### [MODIS Terra Vegetation Continous Fields](data_info.md#modis-terra-vegetation-continous-fields)
