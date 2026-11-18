RADD
====

R - Remote Sensing

A - Ancillary

D - Data

D - Downloader

Running RADD (test version)
---------------------------

One-time setup in WSL (Ubuntu):

```bash
sudo apt install -y python3-venv python3-pip
cd /mnt/c/Users/jgray/Documents/GitHub/RADD
python3 -m venv .venv
source .venv/bin/activate
pip install -r requirements.txt
earthengine authenticate
```

Then set `gee_project` in [config.toml](config.toml) to your Earth Engine
Cloud project (or `export EE_PROJECT=your-project`).

Run a scene for the predefined 2 km x 2 km Starkville, MS AOI:

```bash
python RADD.py Starkville 07/01/2023 07/06/2023
python RADD.py Starkville 07/01/2023 07/06/2023 optional   # also MOD44B
```

Define another AOI (3 or more vertices, up to 6 decimals):

```bash
python RADD.py --define-aoi MyField 33.459409,-88.829167 33.459409,-88.807633 33.441391,-88.807633 33.441391,-88.829167
```

The scene is written to `ancillary/scene/{AOI}/{AOI}_YYYYMMDD_YYYYMMDD.nc`
(NetCDF4, which is HDF5 underneath, so h5py can open it too). If a scene for the
same AOI, dates and config already exists, RADD prints its path and downloads
nothing. To rebuild a scene (e.g., after changing code or to pick up a newer
land cover year), delete it and run again.

### Opening a scene in QGIS

* Close QGIS before running RADD. QGIS locks a `.nc` file while it has it open,
  and other programs (including RADD) cannot read it until QGIS is closed.
* Drag the file from its Windows path
  (`C:\Users\...\RADD\ancillary\scene\{AOI}\`), not from a `\\wsl.localhost\...`
  path, or use Layer > Add Layer > Add Raster Layer.
* "Add all layers" is fine for scenes written since the `/time` group was
  removed. QGIS lists the `/native/...` rasters plus one mesh layer, which can
  be skipped. Scenes written earlier also show `/time` and `/native/mod09ga`
  vector layers: QGIS opens those for editing, which locks the file and makes
  every raster layer "unavailable". Delete and regenerate those scenes.
* Each layer carries its own CRS (MODIS sinusoidal, EPSG:4326, EPSG:4269), so
  QGIS reprojects them on the fly. Single-pixel layers (ERA5, SMAP over a small
  AOI) print a harmless "dfMax should be strictly greater than dfMin" warning.

### Not implemented yet

* `/resampled` (and `target_grid`, `upsample_method`)
* run order step 3 (copying overlapping days from other scenes)
* `precip_lookback` and `dew_flag` (need `rain_lookback_hours` and
  `dew_threshold_k`)
