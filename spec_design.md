Specifications for RADD:
========================

### File Specifications:

Keep all data until the scene file (NetCDF4, see [Scene File](#scene-file)) is
generated, then delete raw files for daily and hourly datasets. Keep data from
static / annual datasets like MODIS' MCD12Q1, which is a yearly Land Cover
classification. The kept datasets are stored in the
[Raw File Cache](#raw-file-cache) under the AOI so they can be easily
referenced.

**File path:** `ancillary/scene/{AOI}/{AOI}_YYYYMMDD_YYYYMMDD.nc`

When running the script:

1. Check the request before anything is downloaded:
   * The dates parse and the start date is not after the end date.
   * The requested end date is at least `availability_lag_days` (default 8)
     days before today. "Today" is the UTC date. If not, exit with the "data not
     yet available" error (see [Error Handling](#error-handling)).
   * Each daily / hourly dataset is queried for its latest and earliest image.
     If GEE does not yet cover the dates a dataset needs (including the NDVI
     window and the extra ERA5 hour and SMAP day), or the dates are before the
     start of its record, exit with the "data not yet available" error.
     `availability_lag_days` is only a quick pre-check.
 2. Check whether a matching scene already exists. Look at every scene in
    `ancillary/scene/{AOI}/` with the same start and end dates. A scene matches
    if its `config` attribute and `/aoi` vertices are identical to the current
    ones (see [Scene File](#scene-file)). `config` is compared as parsed JSON,
    not as text. If one matches, do not download anything. Tell the user the
    scene already exists, print its path, and exit. If none match, continue and
    build a new scene.
 3. If other scenes for the same AOI overlap the requested dates and have an
    identical `config` attribute and `/aoi` vertices, copy those days from their
    `/native` group instead of re-downloading. Only copy days whose
    `landcover_year` (and `vcf_year`) match what this run would use. Only the
    remaining days (plus the NDVI window) are downloaded. `/resampled` is always
    rebuilt from the new scene's own `/native`. *(Not implemented yet.)*
 4. Before downloading each dataset, check if that specific file exists. This is
    important in the event of an error and it stops mid run, that way it won't
    have to re-download something.
    * Daily and hourly raw files are stored one file per dataset per UTC day:
      `ancillary/raw/{AOI}_{geom_hash}/{dataset}/YYYYMMDD.npz`.
    * Each file records its band list and native grid. It is reused only if
      both match the current request; otherwise it is downloaded again.
    * Files are downloaded to a temporary name (`.part`) and renamed when the
      download finishes, so a crash never leaves a partial file that looks
      complete.
    * Static and annual datasets are checked in the
      [Raw File Cache](#raw-file-cache) first.

5. It should not generate the final scene file unless all datasets that were
   requested were downloaded successfully. The scene is also written to a
   temporary name and renamed when complete.

### Raw File Cache:

Raw files for static and annual datasets are small (MCD12Q1 is under 1 KB per
year for a 10 x 10 km AOI), do not change once published, and are reused by
every scene for the same AOI and year. They are kept instead of deleted.

| Kind           | Datasets                           | Raw files                                                       |
| -------------- | ---------------------------------- | --------------------------------------------------------------- |
| Static         | SRTM, 3DEP, ERA5 `soil_type`       | Kept in the cache (3DEP: see [Terrain](#terrain))               |
| Annual         | MCD12Q1, MOD44B                    | Kept in the cache                                               |
| Daily / hourly | MOD09GA, ERA5 hourly, SMAP         | Deleted after the scene is written; reused via run order step 3 |

**Path:** `ancillary/cache/{dataset}/{AOI}/{version}_{year}_{geom_hash}.npz`
(static datasets omit `{year}`)

* Raw files are NumPy `.npz` archives holding each band as an array, plus the
  band list and the native grid (CRS, affine transform, shape).
* `{version}` is the pinned collection version (e.g., `061`), or `na` for
  collections without one. `{geom_hash}` is a hash of the AOI vertices, so a
  redefined AOI never reuses an old file.
* DEM cache folders are named after the collection
  (e.g., `dem_usgs_3dep_10m_collection`), so changing the DEM source never
  reuses the old file.
* Download to a temporary name and rename it when the download finishes, so a
  file in the cache is always complete.
* Never overwrite a cached year. A newly published year is a new file.
* Scene files still store their variables in `/native`, so each scene is
  self-contained. The cache is only where the data comes from.
* The cache can be deleted at any time. Files are re-downloaded when needed.

### Scene File:

The scene file is NetCDF4 (HDF5 underneath) following CF conventions. It opens
in xarray (`xr.open_dataset(file, group="native/mod09ga")`), h5py, MATLAB
(`ncread` / `h5read`), GDAL and QGIS. GDAL and QGIS list each variable as a
subdataset, e.g. `NETCDF:"file.nc":/native/mod09ga/vwc`.

**Path:** `ancillary/scene/{AOI}/{AOI}_YYYYMMDD_YYYYMMDD.nc`

If that name is already used by a scene with a different config, add a number
to the end (`AOI_YYYYMMDD_YYYYMMDD_2.nc`, `_3`, ...). Existing scenes are never
overwritten. Scenes are matched by their `config` attribute, not by file name.

The file stores derived variables only, at both native and resampled resolution.
Raw downloads are deleted once the scene file is written.

#### Layout:

```
/                     root attributes (see below)
/aoi                  lat, lon (vertex); attributes bbox, crs
/native               one subgroup per config dataset key, each on its own
                      source grid
    /mod09ga          ndvi, ndvi_max, ndvi_min, vwc            (date, y, x)
                      gapfill_flag, fallback_flag              (date, y, x)
    /mcd12q1          LC_Type1, stem_factor                    (lc_year, y, x)
                      landcover_year, landcover_provisional    (date)
    /dem              elevation, slope, aspect_sin, aspect_cos (y, x)
    /era5             see ERA5 variables below
    /smap_l4          see SMAP variables below
    /mod44b           optional, only if requested; same year rule as
                      mcd12q1 (vcf_year, vcf_provisional)
/resampled            every variable above on `target_grid`,
                      plus lc_fraction                (lc_year, igbp_class, y, x)
                      (not implemented yet)
```

Each `/native` subgroup is self-contained, so it opens on its own:

* its own `date`, `y` and `x` dimensions and coordinate variables (`x` / `y`
  are pixel centers in the dataset's CRS; `date` is days since 1970-01-01),
  plus `lc_year`, `vcf_year` or `overpass` where used
* a `spatial_ref` grid-mapping variable with `crs_wkt`, `spatial_ref` (WKT) and
  a GDAL `GeoTransform`. Every map variable has `grid_mapping = "spatial_ref"`.
  The `GeoTransform` lets GIS tools place grids too small to have coordinate
  spacing (e.g., a single ERA5 pixel).
* group attributes `collection`, `crs`, `transform`, `resolution`
* every variable lists its non-x/y coordinates in a CF `coordinates`
  attribute (e.g., `"date overpass"`)

There is no separate `/time` group: each `/native` group carries its own
`date` coordinate. GDAL/OGR reads any group whose only extra variable is an
unreferenced 1-D list (such as a bare `date` list) as a vector table, and QGIS
opens vector layers for editing. That locks the whole file and makes every
raster layer "unavailable". The `coordinates` attribute and the missing
`/time` group keep the scene raster-only for GDAL/QGIS.

**Root attributes:** `radd_version`, `created_utc`, `aoi_name`, `start_date`,
`end_date`, `day_basis = "UTC"`, `config`, `dem_source`. The land cover year is
stored per day in `/native/mcd12q1` (see [Land Cover Year](#land-cover-year)).

`config` is a JSON string (sorted keys) recording the configuration the scene
was built with:

* every key in [Configuration Keys](#configuration-keys) except
  `availability_lag_days` and `gee_project`, which do not affect the output
* the requested dataset list (including optional datasets, if requested)
* each dataset's pinned collection ID and band list

It is the record of how the scene was made, and run order step 2 compares it to
find existing scenes. `radd_version` is stored but is not compared. To view it,
use `ncdump -h {file}` or `xr.open_dataset(file).attrs["config"]`.

`/resampled` also stores the `target_grid` definition (crs, resolution, origin)
and the resampling method used for each variable as attributes.

#### ERA5 variables:

The `overpass` dimension holds the hours in `overpass_hours_utc`, and the
`overpass` coordinate variable is stored in `/native/era5`. If that list is
empty (e.g., CYGNSS), only the daily statistics are written.

* `temperature_2m`, `soil_temperature_level_1`: `_mean`, `_min`, `_max` over
  the 24 hourly values 00:00-23:00 of the UTC day (date, y, x) and `_overpass`,
  the value at each overpass hour (date, overpass, y, x)
* `total_precipitation`: `precip_sum` (date, y, x), the sum of the values
  stamped 01:00 of the day through 00:00 of the next day. Each value is the
  accumulation over the hour ending at its time stamp, so this covers the UTC
  day exactly. Missing hours make `precip_sum` NaN.
* `precip_lookback`: the sum over `rain_lookback_hours` before each overpass
  (date, overpass, y, x), using the same convention (the values stamped after
  `overpass - rain_lookback_hours`, through the overpass hour). Written only
  when `rain_lookback_hours` is set. *(Not implemented yet.)*
* `dew_flag`: air temperature minus dewpoint at or below `dew_threshold_k`
  (date, overpass, y, x). Written only when `dew_threshold_k` is set. *(Not
  implemented yet.)*
* `soil_type` (y, x)

The ERA5 download for each day runs from 00:00 of that day through 00:00 of
the next day (25 hourly images). When `precip_lookback` is added, the download
must also include the `rain_lookback_hours` before the first overpass hour,
which may reach into the previous UTC day.

#### SMAP variables:

* `sm_surface`, `sm_rootzone`, `sm_profile`: `_mean` (date, y, x), the mean of
  the 3-hourly steps whose centers fall in the UTC day, and `_overpass` (date,
  overpass, y, x), the step whose center is nearest each overpass hour
* Step centers are at :30 (01:30, 04:30, ... UTC). If GEE stamps an image on
  the hour, it is treated as the start of the step and its center is 90 minutes
  later.
* **Ties:** when two steps are equally close (e.g., 10:30 and 13:30 for 12 UTC),
  `_overpass` is the mean of both.
* The SMAP download starts one day before the start date, because the nearest
  step to 0 UTC can be 22:30 of the previous day.

#### Processing rules:

* Days are UTC calendar days.
* All derived variables are computed at native resolution, then resampled.
* **Native grids:** every download passes the dataset's own CRS and affine
  transform (`crsTransform`) to GEE (`ee.data.computePixels`), so GEE never
  reprojects the data. Bands of one dataset at a coarser resolution (e.g.,
  MOD09GA `state_1km`) are read onto the dataset's main grid with nearest
  neighbour.
* **Extent:** the native window is the smallest block of whole native pixels
  covering the AOI bounding box. Pixels inside the bounding box but outside the
  AOI polygon are kept (not masked).
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

* Mask MOD09GA pixels flagged in `state_1km` and pixels with bad band quality
  in `QC_500m`. Each `state_1km` flag applies to the 2x2 block of 500 m pixels
  it covers. Masked when:
  * `state_1km` bits 0-1 (cloud state) are 01 (cloudy) or 10 (mixed)
  * bit 2 (cloud shadow) is set
  * bits 6-7 (aerosol quantity) are 11 (high)
  * bit 12 (MOD35 snow/ice) or bit 15 (internal snow mask) is set
  * `QC_500m` bits 2-5 (band 1 quality) or bits 6-9 (band 2 quality) are not
    0000 (highest quality)
  * either reflectance is negative, or their sum is not positive
* Gap fill: hold the last valid NDVI for up to `max_gap_fill_days` (8). Longer
  gaps stay NaN. Filled pixels are marked in `gapfill_flag`.
* `ndvi_max` / `ndvi_min` for day d are the `ndvi_percentiles` (95th / 5th) of
  valid, non-gap-filled NDVI in the `ndvi_window_days` (16) days ending on d.
  The window looks backward only, so a day's VWC is identical in every scene
  that contains it (for the same config and `landcover_year`).
* If a pixel has fewer than `ndvi_min_valid_obs` (5) valid days in the window,
  use `ndvi_fallback_table` (NDVI\_min = 0.1, NDVI\_max = that day's NDVI; the
  same for every class) and mark it in `fallback_flag`.
* VWC uses the formula in
  [data\_info](data_info.md#modis-terra-surface-reflectance) with the stem
  factor from `stem_factor_table` for that pixel's `LC_Type1` in that day's
  `landcover_year` (see [Land Cover Year](#land-cover-year)). The stem factor
  is set only by the land cover class.
* MODIS download range: start date minus (`ndvi_window_days` - 1) days, through
  the end date. A day with no MOD09GA image in GEE is left empty (NaN) and
  reported.

#### Land Cover Year:

MCD12Q1 and MOD44B are annual and are published a year or more late.

* Each day uses `lc_year = min(year of the day, latest year in GEE)`. The
  latest year is queried from the collection at run time, not assumed.
* A scene that crosses Jan 1 stores every year it needs along the `lc_year`
  dimension (usually one, at most two for short scenes). `landcover_year`
  (date) records the year each day used.
* Days whose year is older than the day's own year are marked in
  `landcover_provisional`. Existing scenes are not rebuilt when GEE publishes a
  newer year. To pick up the new year, delete the scene and run it again.
* MOD44B follows the same rule with `vcf_year` and `vcf_provisional`.
* Select a year with `filterDate('{Y}-01-01', '{Y+1}-01-01')`. This works no
  matter which day of the year an image is stamped.
* Expected artifact: MCD12Q1 has some class flipping from year to year that is
  classification noise, not real change. In a scene that crosses Jan 1,
  `stem_factor` and VWC can jump on Jan 1.

#### Terrain:

* If USGS 3DEP 10 m (`USGS/3DEP/10m_collection`) covers the entire AOI (no
  nodata pixels), use it. Otherwise use SRTM 30 m for the entire AOI. Record
  which one in `dem_source`.
* 3DEP is a tiled collection: the tiles that intersect the AOI are mosaicked on
  the first tile's native grid (EPSG:4269, 1/3 arc-second). If no tile
  intersects the AOI, SRTM is used.
* `slope` (degrees) and aspect come from central differences on the native
  grid. Pixel spacing in metres for geographic grids is
  `111320 * cos(latitude)` per degree east-west and `110574` per degree
  north-south. Aspect is the downslope direction, clockwise from north, stored
  as `aspect_sin` / `aspect_cos` (NaN where the slope is 0).
* Both DEMs are kept in the [Raw File Cache](#raw-file-cache).


#### <mark> **TODO: Look into adding a finer soil texture dataset (e.g., OpenLandMap clay/sand content in GEE) for the dielectric models. ERA5 `soil_type` is ~28 km and describes the 30-100 cm root zone.** </mark>

### User Inputs:

CLI call:

`python RADD.py AOI MM/DD/YYYY MM/DD/YYYY [optional]`

A specific example would be:

`python RADD.py Starkville 07/01/2023 07/06/2023 optional`

* `AOI` is the name of an area of interest saved in `aois.json`. If the AOI is
  not recognized, RADD prints the known AOIs and how to define one, then exits.
* Define an AOI with
  `python RADD.py --define-aoi NAME lat,lon lat,lon lat,lon [lat,lon ...]`.
  * At least 3 vertices; lat/lon accept up to 6 decimal places. An error
    message is shown if a vertex has too many decimals or is out of range.
  * `NAME` may use letters, digits, `_` and `-`.
  * The AOI is saved to `aois.json`. Redefining a name replaces it.
  * `aois.json` ships with `Starkville`, a 2 km x 2 km test AOI centered on
    Starkville, MS (33.4504, -88.8184).
* `start_date` / `end_date`'s format is MM/DD/YYYY (YYYY-MM-DD is also
  accepted).
* `optional` argument should be added to the end of the call if the user wants
  to download the optional datasets as well.

### Customization:

TO-REVIEW The code should be designed in a way that makes adding additional
Google Earth Engine hosted datasets **very easy**. The configuration is in
`config.toml` (TOML). Each dataset is a `[datasets.{key}]` table with its
pinned `collection` ID and the `bands` to request, so bands can be edited for
existing datasets. A new dataset also needs code that says how its bands are
processed (see open issues).

#### Configuration Keys:

| Key                     | Example                                             | Purpose                                               |
| ----------------------- | --------------------------------------------------- | ----------------------------------------------------- |
| `gee_project`           | `"my-ee-project"`                                   | Earth Engine Cloud project (or `EE_PROJECT` env var)  |
| `target_grid`           | `crs=EPSG:6933, resolution=200, origin_x, origin_y` | Grid for `/resampled`, snapped to this origin         |
| `upsample_method`       | `nearest` or `bilinear`                             | Method when resampling to a finer grid (global)       |
| `overpass_hours_utc`    | `[12, 0]`                                           | Hours for overpass values; empty = daily stats only   |
| `rain_lookback_hours`   | TBD                                                 | Hours of rain summed before each overpass             |
| `dew_threshold_k`       | TBD                                                 | Max air temperature minus dewpoint that counts as dew |
| `ndvi_window_days`      | `16`                                                | Length of the backward NDVI window                    |
| `ndvi_percentiles`      | `[5, 95]`                                           | Percentiles used as NDVI\_min / NDVI\_max             |
| `ndvi_min_valid_obs`    | `5`                                                 | Valid days needed before using the per-class fallback |
| `max_gap_fill_days`     | `8`                                                 | Longest NDVI gap filled by holding the last value     |
| `availability_lag_days` | `8`                                                 | Days after a date before it can be requested (MODIS)  |
| `ndvi_fallback_table`   | `ndvi_min = 0.1`, `ndvi_max = "current"`            | Fallback NDVI\_min / NDVI\_max (Chan et al. 2013)     |
| `stem_factor_table`     | per IGBP class                                      | Stem factor for VWC (Chan et al. 2013, Table 1)       |

* `target_grid` and `upsample_method` are not used until `/resampled` is
  implemented.
* `rain_lookback_hours` and `dew_threshold_k` are left out of `config.toml`
  until values are chosen; while they are unset, `precip_lookback` and
  `dew_flag` are not written.
* `stem_factor_table` values are listed in
  [data\_info](data_info.md#modis-terra-surface-reflectance). Class 17 (water)
  is not in the source table and is set to 0.

Dataset collection IDs are pinned to a version (e.g., `MODIS/061/MOD09GA`).

Every key except `availability_lag_days` and `gee_project`, plus the dataset
list, collection IDs, and band lists, is saved in the scene's `config`
attribute (see [Scene File](#scene-file)).

### Error Handling:

If AOI is not recognized, output to user "AOI '{name}' not recognized", the
known AOIs, and the `--define-aoi` instructions, then exit. RADD never prompts
for input, so it can run unattended.

* If lat/lon provided is invalid, warn user, show the proper input format, then
  exit.

If there is a download error, wait 5 seconds and then retry the source that
failed. If it has failed 3 times warn the user with a network error message and
recommend the user try to run it again. Not every GEE error is retried:

| Error                                           | Handling                                         |
| ----------------------------------------------- | ------------------------------------------------ |
| Network / timeout                               | Retry 3 times, 5 seconds apart                   |
| Rate limit / 429 / "Too many concurrent"        | Retry 3 times with backoff (5, 10 seconds)       |
| "User memory limit exceeded", payload too large | Fail immediately (splitting into tiles: open)    |
| Authentication / project not set                | Fail immediately with setup instructions         |
| Collection ID or band not found                 | Fail immediately (bad config)                    |

**Note:** since the final scene file shouldn't have been generated because of
the network error, the other data sources files should still exist in the
system and not have to be re-downloaded. If the requested end date is within
`availability_lag_days` of today, or a dataset does not have a requested date
in GEE yet, do **not** use the 5 second retry. Output to user: "{dataset} is
only available through {latest_date}, try again after {date}." then exit. This
is not a network error, and no scene file is generated.

Exit codes: 0 success (or the scene already exists), 2 input / config error, 3
data not yet available, 4 network error.

### Datasets:

#### Needed:

##### [MODIS Terra Surface Reflectance](data_info.md#modis-terra-surface-reflectance)

##### [MODIS Land Cover Type](data_info.md#modis-land-cover-type)

##### [ERA5 Hourly Climate Reanalysis](data_info.md#era5-hourly-climate-reanalysis)

##### [SMAP Soil Moisture](data_info.md#smap-surface-and-root-zone-soil-moisture)

##### [USGS 3DEP 10 m Digital Elevation Model](data_info.md#usgs-3dep-10-m-digital-elevation-model)

##### [SRTM Digital Elevation Model](data_info.md#srtm-digital-elevation-model)

#### Optional:

##### [MODIS Terra Vegetation Continuous Fields](data_info.md#modis-terra-vegetation-continuous-fields)
