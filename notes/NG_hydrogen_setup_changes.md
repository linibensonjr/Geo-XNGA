# Nigeria (NG) green hydrogen with Geo-X: changes and open items

_Last updated: 2026-10-10_

## Why Geo-X and not GeoH2

The NG case runs in `external/Geo-X`, not `replication/GeoH2`. Geo-X already had the NG hexagons (`data/NG/hex_final_NG.geojson`) and the `parameters/NG/hydrogen/` layout. GeoH2 would have needed the parameter files moved, about 30 labels renamed, the `Global` sheet added and its hexagon `country` column handled. Geo-X needed only a few fixes, listed below.

**Choices made:**
- Generators are **solar and wind**. Hydro is dropped because there is no Nigerian dam or HydroBASINS data.
- The currency is **USD**.
- The solver is **GLPK**, because Gurobi is not installed.

## Status

| Stage | Status |
|---|---|
| `geox` env imports (geopandas, rasterio, atlite, pypsa) | ✅ Working |
| `optimize_transport` → `resources/hex_transport_NG_hydrogen.geojson` | ✅ Ran, 552 hexes, no NaNs |
| `calculate_water_costs` → `resources/hex_water_NG_hydrogen.geojson` | ✅ Ran |
| `get_weather_data` → `cutouts/NG_2023.nc` | ⏳ Not run yet (see open items) |
| `optimize_plant`, `calculate_total_costs`, costs by component, maps | ⏳ Blocked on the weather file |

Results from the transport stage:

| Demand centre | Pipeline transport + conversion (USD/kg) | Trucking (USD/kg) | Trucking state |
|---|---|---|---|
| Lagos_FTZ_Dangote_Refinery_H2 | 2.70 – 3.95 (median 3.26) | 2.79 – 6.25 | 500 bar |
| Lagos_FTZ_Dangote_Refinery_Fertiliser_H2 | 2.66 – 3.40 (median 3.00) | 2.77 – 6.23 | 500 bar |
| Lagos_FTZ_Dangote_Ammonia_Pathway_NH3 | 0.48 – 0.61 (median 0.54) | 0.52 – 1.58 | NH3 |

## What was changed

### 1. `geox` conda env (Windows)

Before the fix, any import of geopandas or atlite failed with `DLL load failed`. This is also why the 2026-10-09 `get_weather_data` run died after 1 second. There were two separate causes:

1. **`libtiff` 4.4.0 couldn't load.** `libdeflate` and `lerc` had been upgraded to versions it can't use. Fixed with:
   ```
   conda install -n geox -c conda-forge libdeflate=1.14 lerc=4.0
   ```
2. **rasterio loaded the wrong GDAL.** Base `miniforge3\Library\bin` is on the permanent user PATH and contains a newer `gdal.dll`. rasterio 1.3.4 (`rasterio/__init__.py` lines 21–24) registers every PATH folder that has a `gdal*.dll`, so it picked up base's copy instead of geox's GDAL 3.6.0.
   - Fixed by adding activation scripts to the env:
     `envs/geox/etc/conda/activate.d/zz-geox-drop-base-gdal.{bat,ps1,sh}` and matching `deactivate.d/` scripts.
   - They drop that one folder from PATH only while `geox` is active and restore it on deactivate. The global PATH and the base env are unchanged.
   - Verified through both `conda run -n geox` and `conda activate geox` in PowerShell.

> The `replication/GeoH2` env `geoh2` has the same `libtiff` failure and was **not** fixed. The rasterio GDAL clash is not checked there.

### 2. `config.yaml`

| Setting | Before | After | Reason |
|---|---|---|---|
| `generators_dict` | `{'solar': [], 'hydro': []}` | `{'solar': [], 'wind': []}` | Hydro needs data we don't have; wind added back |
| `solver` | `'gurobi'` | `'glpk'` | Gurobi not installed in `geox`; GLPK is |
| `currency` | `'dollars'` | `'USD'` | Geo-X builds lookup labels as `... ({currency}/kWh)`, and the NG workbooks say `USD` |

### 3. NG parameter workbooks (`parameters/NG/hydrogen/`)

Only labels changed. Every numeric cell was compared with the originals and none differ.

| File | Sheet(s) | Old label | New label |
|---|---|---|---|
| `technology_parameters.xlsx` | Water | Water transport cost (euros/100 km/m3) | Water transport cost (USD/100 km/m3) |
| `technology_parameters.xlsx` | Water | Water specific cost (euros/m3) | Water specific cost (USD/m3) |
| `transport_parameters.xlsx` | 500 bar, LH2, LOHC, NH3 | Net capacity (kg H2) | Net capacity (kg of commodity) |

> ⚠️ **Gotcha:** the road CAPEX/OPEX cells in the `technology_parameters.xlsx` `Infra` sheet are formulas (for example `=659451*0.95`).
> - If the file is saved with openpyxl, their cached values are dropped and pandas reads them as NaN.
> - This happened once, made every road construction cost NaN, and was fixed.
> - The label change was then made by editing the xlsx XML directly.
> - Edit this file in Excel, not with openpyxl.

### 4. `parameters/basic_h2_plant/generators.csv`

I removed the `Hydro`, `Geothermal` and `Nuclear` rows, leaving only `Wind` and `Solar`.
- The Geo-X README (line 186) says this file "must only include the generators that you wish to use".
- Unused rows stay in the PyPSA network with no capacity limit and no annualised cost, so the optimiser could build them.
- This file is shared by all hydrogen runs. Restore the rows if you run a case that uses those generators.

### 5. `src/main/plant_optimization.py`

The `lopf` calls hard-coded Gurobi-only solver options (`OutputFlag`, `LogToConsole`). PyPSA warns that when a solver gets options it doesn't recognise, "solving the problem will take forever". A test with GLPK did hang. The fix:

```python
solver = snakemake.config['solver']
# Silencing options are Gurobi-specific; other solvers (e.g. glpk) hang on unknown options
quiet_options = {'LogToConsole': 0, 'OutputFlag': 0} if solver == 'gurobi' else {}
```

All 5 `solver_options=` arguments now use `quiet_options`. Behaviour with Gurobi is unchanged. See `git diff src/main/plant_optimization.py`.

### Backups

The originals of every file changed in sections 2–4 are in `notes/backup_2026-10-10/`. For `plant_optimization.py`, use `git diff upstream/main -- src/main/plant_optimization.py`.

This work is committed on the `ng-hydrogen` branch of the fork `https://github.com/linibensonjr/Geo-XNGA` (`origin`). The original repo is the `upstream` remote. `data/NG/hex_final_NG.geojson` is force-added even though `data/` is gitignored, so a fresh clone can run.

Other modified files in `git status` were already changed before this work and were not touched here:
- `parameters/NA/ammonia/*`
- `parameters/ZM/copper/*`
- `parameters/basic_nh3_plant/*`

## Still open

### Needed before the full run

1. **Download the weather cutout.**
   ```
   conda activate geox
   cd external/Geo-X
   snakemake -j4 run_weather
   ```
   This downloads ERA5 2023 for Nigeria's bounding box (about 2.6–14.8°E, 4.3–13.9°N). Expect several GB and possibly hours. It needs the CDS key in `~/.cdsapirc`, which is present and points at the new CDS URL.

2. **Solver run time.** One full-year plant optimisation took **about 100 s with GLPK** in a test that used synthetic profiles and solved to optimal. `optimize_plant` solves 552 hexes × 3 demand centres, about 1,650 optimisations, so it would take **about 45 hours on GLPK**. Options:
   - Get a free **Gurobi academic licence**, install it in `geox`, and set `solver: 'gurobi'`. This is recommended because it is much faster.
   - Stay on GLPK and let it run (for example over a weekend).
   - Install HiGHS. This is not tested with PyPSA 0.26 here.

3. **Then run the rest.**
   ```
   snakemake -j4 optimise_all   # total costs
   snakemake -j4 map_all        # plots
   ```

### Model and input caveats (results will run, but check these)

4. **Mixed currencies.** The plant capex in `parameters/basic_h2_plant/*.csv` is still the Namibia **euro** values (wind 1.58M/MW, solar 1.47M/MW, electrolyser and storage too). Every NG workbook is in **USD**. Convert these, or document the assumption.

5. **Only the first finance row is used.** Geo-X reads only the **first row** of `country_parameters.xlsx` (`Nigeria`, 12% WACC). The `Nigeria_low_finance` (7%) and `Nigeria_high_finance` (20%) rows are ignored. To run those scenarios, move the wanted row to the top. Keep a separate results folder or copy, because the output filenames don't include the scenario.

6. **Demand point is outside the hexagon grid.** All three Lagos FTZ demand centres sit at (6.43°N, 3.65°E), about 0.15° (roughly 16 km) from the nearest hexagon. With `restrict_hexagons: False` this doesn't stop the run, but no hexagon is treated as the demand location itself. Check the coordinates.

7. **Two demand centres share a location.** The two `500 bar` H2 demands (refinery 379 kt/a and fertiliser 644 kt/a) are separate centres at the same point. Each is optimised on its own, with no shared infrastructure. Check this is what you intend.

8. **Wind capacity scaling.** `gen_capacity: wind: 4` multiplies the hexagon `wind` value (turbine count) by 4 MW. Confirm that the `wind` column from the data prep is a turbine count and `solar` is in MW.

9. **Leftover files.** `parameters/NG/hydrogen/conversion_parameters.csv` and `test.ipynb` are scratch files that the pipeline ignores.

### Not done / out of scope

- The `geoh2` env and `replication/GeoH2` are not fixed. Use Geo-X for NG.
- The pre-existing changes to `parameters/NA/ammonia/*`, `parameters/ZM/copper/*` and `parameters/basic_nh3_plant/*` are not committed.
- A first `run_weather` attempt on 2026-10-11 downloaded all 2023 data, but Claude Code stopped it because the machine was low on memory. It was stopped while writing `cutouts/NG_2023.nc`, so no cutout was produced.
