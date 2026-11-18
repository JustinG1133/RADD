Table of Contents
=================
1. [RADD](README.md#radd)
   1. [Running RADD (test version)](README.md#running-radd-test-version)

2. [Problem Statement](Problem_Statement_LP_.md#problem-statement)

3. [Specifications for RADD](spec_design.md#specifications-for-radd)

   1. [Datasets](spec_design.md#datasets)
      1. [Needed](spec_design.md#needed)<br>
         1. [MODIS Terra Surface Reflectance](spec_design.md#modis-terra-surface-reflectance)

         2. [MODIS Land Cover Type](spec_design.md#modis-land-cover-type)

         3. [ERA5 Hourly Climate Reanalysis](spec_design.md#era5-hourly-climate-reanalysis)

         4. [SMAP Soil Moisture](spec_design.md#smap-soil-moisture)

         5. [USGS 3DEP 10 m Digital Elevation Model](spec_design.md#usgs-3dep-10-m-digital-elevation-model)

         6. [SRTM Digital Elevation Model](spec_design.md#srtm-digital-elevation-model)

         
      2. [Optional](spec_design.md#optional)
         1. [MODIS Terra Vegetation Continuous Fields](spec_design.md#modis-terra-vegetation-continuous-fields)
         2. More to follow
   2. [File Specifications](spec_design.md#file-specifications)
   3. [Raw File Cache](spec_design.md#raw-file-cache)
   4. [Scene File](spec_design.md#scene-file)
      1. [Layout](spec_design.md#layout)
      2. [ERA5 variables](spec_design.md#era5-variables)
      3. [SMAP variables](spec_design.md#smap-variables)
      4. [Processing rules](spec_design.md#processing-rules)
      5. [NDVI and VWC](spec_design.md#ndvi-and-vwc)
      6. [Land Cover Year](spec_design.md#land-cover-year)
      7. [Terrain](spec_design.md#terrain)
   5. [User Inputs](spec_design.md#user-inputs)
   6. [Customization](spec_design.md#customization)
      1. [Configuration Keys](spec_design.md#configuration-keys)
   7. [Error Handling](spec_design.md#error-handling)
4. [Data Details](data_info.md#data-details)

   1. [Needed](data_info.md#needed)

      1. [MODIS Terra Surface Reflectance](data_info.md#modis-terra-surface-reflectance)
      2. [MODIS Land Cover Type](data_info.md#modis-land-cover-type)
      3. [ERA5 Hourly Climate Reanalysis](data_info.md#era5-hourly-climate-reanalysis)
      4. [SMAP Soil Moisture](data_info.md#smap-surface-and-root-zone-soil-moisture)
      5. [USGS 3DEP 10 m Digital Elevation Model](data_info.md#usgs-3dep-10-m-digital-elevation-model)
      6. [SRTM Digital Elevation Model](data_info.md#srtm-digital-elevation-model)
      
   2. [Optional](data_info.md#optional)

      1. [MODIS Terra Vegetation Continuous Fields](data_info.md#modis-terra-vegetation-continuous-fields)
