Specifications for RADD:
========================

### File Specifications:

keep all data until the 'scene' file (hdf5) is generated, then delete raw files
to save space.


**Potential** file path: ancillary/scene/AOI/sceneName\_dateRange

When running the script:

1. Check that the requested end date is at least `availability_lag_days`
   (default 8) days before today. MODIS is the slowest dataset to appear in GEE,
   so it sets this cushion. If not, exit with the "data not yet available" error
   (see [Error Handling](#error-handling)).
2. Check to see if the final scene file exists. If it does, compare its stored
   hashes against the current config (see [Scene File](#scene-file)):
   * `native_config_hash` differs: rebuild the whole scene (re-download).
   * only `resample_config_hash` differs: rebuild `/resampled` from `/native`
     offline, with no download.
   * both match: nothing to do, exit.
3. If other scenes for the same AOI overlap the requested dates and have a
   matching `native_config_hash`, copy those days from their `/native` group
   instead of re-downloading. Only the remaining days (plus the NDVI window) are
   downloaded. `/resampled` is always rebuilt from the new scene's own
   `/native`.
4. Before downloading each dataset, check if that specific file exists. This is
   importnant in the event of an error and it stops mid run, that way it won't
   have to re-download something.
5. It should not generate the final scene file unless all datasets that were
   requested were downloaded successfully.

### Scene File:

The scene file is NetCDF4 (HDF5 underneath) following CF conventions, so it
opens directly in xarray, MATLAB (`ncread` / `h5read`), GDAL, and QGIS.

**Path:** `ancillary/scene/AOI/AOI_YYYYMMDD_YYYYMMDD.nc`

The file stores derived variables only, at both native and resampled resolution.
Raw downloads are deleted once the scene file is written.

#### Layout:

```
/                     root attributes (see below)
/aoi                  vertices (lat, lon), bbox, crs
/time                 date (n_days), UTC calendar days
/native               one subgroup per config dataset key, each on its own
                      source grid with x, y, crs, resolution
    /mod09ga          ndvi, ndvi_max, ndvi_min, vwc            (date, y, x)
                      gapfill_flag, fallback_flag              (date, y, x)
    /mcd12q1          LC_Type1, stem_factor                    (y, x)
    /dem              elevation, slope, aspect_sin, aspect_cos (y, x)
    /era5             see ERA5 variables below
    /smap_l4          see SMAP variables below
    /mod44b           optional, only if requested
/resampled            every variable above on `target_grid`,
                      plus lc_fraction                         (igbp_class, y, x)
```

**Root attributes:** `radd_version`, `created_utc`, `aoi_name`, `start_date`,
`end_date`, `day_basis = "UTC"`, `native_config_hash`, `resample_config_hash`,
`dem_source`, `landcover_year`.

`/resampled` also stores the `target_grid` definition (crs, resolution, origin)
and the resampling method used for each variable as attributes.

#### ERA5 variables:

The `overpass` dimension holds the hours in `overpass_hours_utc`. If that list
is empty (e.g., CYGNSS), only the daily statistics are written.

* `temperature_2m`, `soil_temperature_level_1`: `_mean`, `_min`, `_max` (date,
  y, x) and `_overpass` (date, overpass, y, x)
* `total_precipitation`: `precip_sum` (date, y, x) and `precip_lookback`, the
  sum over `rain_lookback_hours` before each overpass (date, overpass, y, x)
* `dew_flag`: air temperature minus dewpoint at or below `dew_threshold_k`
  (date, overpass, y, x)
* `soil_type` (y, x)

The ERA5 download range includes the `rain_lookback_hours` before the first
overpass hour, which may reach into the previous UTC day.

#### SMAP variables:

* `sm_surface`, `sm_rootzone`, `sm_profile`: `_mean` (date, y, x) and
  `_overpass`, the 3-hourly step nearest each overpass hour (date, overpass, y,
  x)

#### Processing rules:

* Days are UTC calendar days.
* All derived variables are computed at native resolution, then resampled.
* Resampling to a coarser grid:
  * continuous variables: area-weighted mean
  * aspect: mean of `aspect_sin` and `aspect_cos`
  * categorical (`LC_Type1`, `soil_type`): mode, plus `lc_fraction` for land
    cover
  * flags: fraction of the pixel flagged (0 to 1)
* Resampling to a finer grid: `upsample_method` (`nearest` or `bilinear`) for
  continuous variables, nearest for everything else.
* `target_grid` is snapped to its global origin. The AOI bounding box is
  expanded outward to the nearest grid lines.

#### NDVI and VWC:

* Mask MOD09GA pixels flagged in `state_1km` (cloud, cloud shadow, high aerosol,
  snow) and pixels with bad band quality in `QC_500m`. Each `state_1km` flag
  applies to the 2x2 block of 500 m pixels it covers.
* Gap fill: hold the last valid NDVI for up to `max_gap_fill_days` (8). Longer
  gaps stay NaN. Filled pixels are marked in `gapfill_flag`.
* `ndvi_max` / `ndvi_min` for day d are the `ndvi_percentiles` (95th / 5th) of
  valid, non-gap-filled NDVI in the `ndvi_window_days` (16) days ending on d.
  The window looks backward only, so a day's VWC is identical in every scene
  that contains it.
* If a pixel has fewer than `ndvi_min_valid_obs` (5) valid days in the window,
  use the per-class values from `ndvi_fallback_table` and mark it in
  `fallback_flag`.
* VWC uses the formula in
  [data\_info](data_info.md#modis-terra-surface-reflectance) with the stem
  factor from `LC_Type1`.
* MODIS download range: start date minus (`ndvi_window_days` - 1) days, through
  the end date.

#### <mark> **TODO: Check the source and values of the per-class fallback NDVI table (`ndvi_fallback_table`).** </mark>

#### Terrain:

* If USGS 3DEP 1 m covers the entire AOI (no nodata pixels), use it. Otherwise
  use SRTM 30 m for the entire AOI. Record which one in `dem_source`.

#### <mark> **TODO: Decide which year of annual products (MCD12Q1, MOD44B) a recent date uses**</mark>

#### <mark> **TODO: Look into adding a finer soil texture dataset (e.g., OpenLandMap clay/sand content in GEE) for the dielectric models. ERA5 `soil_type` is ~28 km and describes the 30-100 cm root zone.** </mark>

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


#### Configuration Keys:

| Key                     | Example                                          | Purpose                                                |
| ----------------------- | ------------------------------------------------ | ------------------------------------------------------ |
| `target_grid`           | `crs=EPSG:6933, resolution=200, origin_x, origin_y` | Grid for `/resampled`, snapped to this origin       |
| `upsample_method`       | `nearest` or `bilinear`                          | Method when resampling to a finer grid (global)        |
| `overpass_hours_utc`    | `[12, 0]`                                        | Hours for overpass values; empty = daily stats only    |
| `rain_lookback_hours`   | TBD                                              | Hours of rain summed before each overpass              |
| `dew_threshold_k`       | TBD                                              | Max air temperature minus dewpoint that counts as dew  |
| `ndvi_window_days`      | `16`                                             | Length of the backward NDVI window                     |
| `ndvi_percentiles`      | `[5, 95]`                                        | Percentiles used as NDVI_min / NDVI_max                |
| `ndvi_min_valid_obs`    | `5`                                              | Valid days needed before using the per-class fallback  |
| `max_gap_fill_days`     | `8`                                              | Longest NDVI gap filled by holding the last value      |
| `availability_lag_days` | `8`                                              | Days after a date before it can be requested (MODIS)   |
| `ndvi_fallback_table`   | per IGBP class                                   | Fallback NDVI_min / NDVI_max                           |
| `stem_factor_table`     | per IGBP class                                   | Stem factor for VWC                                    |

Dataset collection IDs are pinned to a version (e.g., `MODIS/061/MOD09GA`).

`target_grid` and `upsample_method` make up `resample_config_hash`. Every other
key, the dataset list, and the band lists make up `native_config_hash`.

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
and not have to be re-downloaded. If the requested end date is within `availability_lag_days` of today, or a dataset does not have a requested date in GEE yet, do **not** use the 5 second
retry. Output to user: "<dataset> is only available through <latest date>,
try again after <date>." then exit. This is not a network error, and no scene
file is generated.

### Datasets:

#### Needed:

##### [MODIS Terra Surface Reflectance](data_info.md#modis-terra-surface-reflectance)

##### [MODIS Land Cover Type](data_info.md#modis-land-cover-type)

##### [ERA5 Hourly Climate Reanalysis](data_info.md#era5-hourly-climate-reanalysis)

##### [SMAP Soil Moisture](data_info.md#smap-surface-and-root-zone-soil-moisture)

##### [SRTM Digital Elevation Model](data_info.md#srtm-digital-elevation-model)

#### Optional:

##### [MODIS Terra Vegetation Continous Fields](data_info.md#modis-terra-vegetation-continous-fields)
