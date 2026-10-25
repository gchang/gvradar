# Changes from upstream

Upstream: https://github.com/GPM-GV/GVradar, branch `main` at commit 834dc4a32adb5fbd80db0b329cfe503ed8094edc (2026-10-13), by Jason Pippitt (NASA GSFC / SSAI, GPM Ground Validation). The latest upstream tag, `V1.5` (9522a10, 2024-02-13), is 309 commits older; `setup.py`, `__init__.py` and the start-up banner on `main` still say 1.5.

License: upstream's `LICENSE.txt` reads only `NASA - GPM-GV`. It grants no right to redistribute or modify the code, so publishing this copy depends on the authors' permission. `LICENSE.txt`, the author and contact details in `setup.py`, `PKG-INFO` and the README, and the NASA and GPM logos are kept unchanged.

This copy starts from upstream's file tree at 834dc4a without its git history. Nothing was left out (upstream has no `.github/` folder). It is unchanged except for the items below.

## 2026-10-25: QC adds the computed Kdp when the radar has none

`calculate_kdp` in `gvradar/dpqc.py` always computes Kdp with the CSU Bringi method, but it added the `KD` field only when `get_Bringi_kdp` was True, and the default is False. NEXRAD Level II and other radars that do not record Kdp therefore came out of QC with no `KD`: with the default settings every NEXRAD volume stopped at the KD plot with `KeyError: 'KD'`, and with plotting off `KD` was silently missing from the QC output. `scripts/PRODUCTS.py` avoided it only because it sets `get_Bringi_kdp` to True.

The field is now added also when the radar has no `KD` yet, the same rule `GVradar.py` already uses in the product step before it calls `get_kdp`. Radars that record their own Kdp keep it unless `get_Bringi_kdp` is set, as before, so their results do not change. For NEXRAD the QC output now has the Bringi Kdp it was meant to have.

## 2026-10-25: the package installs its logos and shapefiles, and `pip install -e .` works

The plots read `nasa.png` and `gpm.png` (`add_logos` is on by default) and the reef and island shapefiles in `shape_files/` from the installed `gvradar` folder, but `setup.py` did not list them, so an installed copy stopped at the first plot with `FileNotFoundError` for `nasa.png`. They are now declared as `package_data`.

`setup.py` also listed `''` (the repository root) as a package. That copied the root `__init__.py` straight into `site-packages`, and it made `pip install -e .`, the README's "Installing from source" route, map only the root folder, so `import gvradar` failed with `ModuleNotFoundError`. `packages` now lists only `gvradar`; the root `__init__.py` stays in the repository. No computed value changes.

## 2026-10-25: both example dictionaries parse

`GVradar.py` reads `--thresh_dict` and `--product_dict` with `ast.literal_eval`, and both examples in `Input_Dictionaries/` stopped it with a `SyntaxError`: `QC_dict.txt` was missing the comma after `'cal_dir'`, and `DP_product_dict.txt` closed the dictionary twice. The comma is added and the extra brace removed; no values change. `DP_product_dict.txt` keeps its CRLF line endings. Note that `QC_dict.txt` is a site example (for instance `plot_dir` is `/data/plots/` and `sounding_type` is `get_ruc`), not the code's defaults.

## 2026-10-25: environment.yml lists the dependencies

`setup.py` declares no dependencies and the README names only some of them, so a new user had to work out the environment from import errors. `environment.yml` at the repository root now lists the conda-forge packages this copy was checked with: Python 3.11, Py-ART (`arm_pyart`), `numpy<2` (SkewT needs `numpy.trapz`), SciPy, matplotlib, netCDF4, cartopy, pandas, xarray, cftime, Pillow, and Cython plus a C compiler for building CSU_RadarTools. CSU_RadarTools and SkewT stay as the README's tarball installs because they are not on conda-forge. On 2026-10-25 the file resolved to the same versions as the Check below (Python 3.11.17, arm_pyart 2.2.4, numpy 1.26.4, cartopy 0.26.0, matplotlib 3.11.2, scipy 1.17.1, pandas 3.0.6, xarray 2026.9.0). No code changes.

## Not changed: special values in the rain and DSD fields

`RC`, `RP`, `RT`, `DM` and `NW` are written with `_FillValue` 0.0 (`set_missing` in `gvradar/dp_products.py`), but they also hold coded values: -32767 beyond 150 km (`mask_beyond_150`, with `do_150_mask` on), -999 where the hydrometeor class is ice (`remove_ice`, with `de_ice` on), and rain rates above 300 mm/h stored as negative numbers (the "max rain rate test" in `add_csu_blended_rain`, `add_csu_blended_rain_tropical` and `add_polZR_rr`). Zero rain is therefore read as missing by CF readers, while the coded values are read as data unless the reader knows the convention. This looks like a deliberate GPM-GV convention rather than a bug, and any change (a different `_FillValue`, masking the codes, or un-negating the heavy rain) would change what downstream tools read, so it is left as it is. When reading these fields, treat -32767 as outside 150 km, -999 as ice (no rain), other negative values as flagged rain above 300 mm/h, and 0 as no rain.

## Notes for running it

- Dependencies: Py-ART, NumPy, SciPy, matplotlib, netCDF4, cartopy, pandas, xarray, cftime and Pillow from conda-forge; CSU_RadarTools and SkewT from the tarballs linked in the README (`https://pmm-gv.gsfc.nasa.gov/pub/NPOL/temp/GVradar/`). CSU_RadarTools builds Cython code, so it needs a C compiler and Cython. SkewT imports `numpy.trapz`, which NumPy 2 removed, so the environment needs `numpy<2`. `setup.py` declares no dependencies.
- `setup.py` imports `distutils.core`; on Python 3.12 or newer that works only through setuptools, which pip provides when it builds the package.
- The product step needs a temperature profile. The default `sounding_type='ruc_archive'` reads `./sounding/YYYY/MMDD/SITE/SITE_YYYY_MMDD_HHUTC.txt` (radar time rounded to the nearest hour), a whitespace table with the columns `PRES HGHT TEMP DWPT RELH MIXR DRCT SKNT THTA THTE THTV`. `get_ruc` fetches model soundings from www.twisterdata.com instead.
- Output folders (`./cf`, `./plots`, `./sounding`) are relative to the working folder. With `--do_qc --dp_products` the QC and product cfRadial files get the same name in `./cf`, so the product file replaces the QC file (it carries the QC fields too).
- The first plot downloads Natural Earth coastlines through cartopy; set `plot_images` to False, or seed cartopy's data folder, to run without network. Plotting is most of the run time.

## Check

On 2026-10-25, in a new conda env (conda-forge: Python 3.11.17, arm_pyart 2.2.4, numpy 1.26.4, cartopy 0.26.0, matplotlib 3.11.2, scipy 1.17.1, pandas 3.0.6, xarray 2026.9.0) with CSU_RadarTools 1.3.0.dev0 and SkewT 1.2.0 from the README's tarballs, a local clone of this repository was installed with the README's command:

```sh
pip install -e .
```

`from gvradar import GVradar` worked from outside the clone, and a regular wheel built from the clone contains `nasa.png`, `gpm.png` and all 22 shapefile parts. There are no tests in the repository. The run used NEXRAD Level II volume `KMHX20220930_234819_V06` (13.6 MB, Newport/Morehead City NC, 2022-09-30 23:48 UTC, rain bands of Hurricane Ian, from `https://unidata-nexrad-level2.s3.amazonaws.com/2022/09/30/KMHX/`), the Newport radiosonde (WMO 72305) of 2022-10-01 00 UTC from the University of Wyoming written as `sounding/2022/1001/KMHX/KMHX_2022_1001_00UTC.txt`, and the code's own default QC and product dictionaries with only `output_cf` switched on:

```sh
python gvradar/GVradar.py --do_qc --dp_products --thresh_dict qc_dict.txt --product_dict product_dict.txt KMHX20220930_234819_V06
```

It ran to the end with the default plots: 3 min 9 s on one core, 3.1 GB peak memory, one cfRadial file of 13.0 MB (15 fields, 9 sweeps) and 11 PNG images. Every field, every attribute and all 11 images are identical to a run of upstream's code at 834dc4a with only the Kdp and `setup.py` changes applied by hand; only the processing time stamps differ. Against the NWS Level III instantaneous dual-pol rain rate (DPR) for the same volume, within 150 km, the lowest-sweep areal mean is 0.164 mm/h for `RC` and 0.199 mm/h for `RP`, against 0.238 mm/h for DPR, and 5 degree by 5 km block means correlate at r = 0.69 (`RC`) and 0.68 (`RP`). These are different algorithms, so this is a plausibility check, not a reproduction.
