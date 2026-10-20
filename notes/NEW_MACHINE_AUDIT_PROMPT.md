# Prompt for Claude Code on the new machine

Paste everything inside the box into Claude Code on the new machine, started in an empty parent folder (or in an existing clone of the repo).

````text
I am continuing a Nigeria (NG) green-hydrogen run with Geo-X on this machine. The work was set up on another machine (Windows) and committed to my fork. Before running anything, AUDIT what is here, then give me the exact runs to do. Do not start any long run, download, or install until I approve your plan.

## Project facts (from the previous machine)

- Repo: https://github.com/linibensonjr/Geo-XNGA, branch `ng-hydrogen` (commit f7bca08 "Set up Nigeria (NG) green hydrogen case"). Remote `origin` = my fork; the original is ClimateCompatibleGrowth/Geo-X, which should be remote `upstream`.
- Case: country NG, weather year 2023, plant_type hydrogen (see `config.yaml`). Generators are solar + wind only (hydro dropped, no Nigerian dam data). Currency label `USD`. Solver was `glpk` because Gurobi was not installed.
- Full change log and open items: `notes/NG_hydrogen_setup_changes.md` in the repo. Read it first.
- `data/NG/hex_final_NG.geojson` (552 hexagons) is force-added to git even though `data/` is gitignored.
- Pipeline (Snakemake, conda env from `environment.yaml`, env name `geox`):
  `run_weather` -> `get_weather_data` (cutouts/NG_2023.nc) ; `optimize_transport` ; `calculate_water_costs` ; `optimize_plant` ; `calculate_total_costs` ; `calculate_cost_components` ; `calculate_map_costs`. Bulk targets: `optimise_all`, `map_all`.
- Already done on the old machine: transport and water stages ran clean (~2 min, 552 hexes, no NaNs). `resources/` is gitignored, so they are NOT in the clone and must be re-run (cheap).
- NOT done: the ERA5 weather cutout. The old run was killed by a low-memory reaper during the final cutout merge, so the cutout never completed. A partial `cutouts/tmp*NG_2023.nc` there is junk; do not rely on any cutout copied from it.

## Known gotchas to check for

1. Weather download needs a CDS key in `~/.cdsapirc` (new CDS URL `https://cds.climate.copernicus.eu/api`, key only, no `uid:`) and accepted ERA5 licence terms. Check that it exists and works with a tiny request before the big one.
2. `run_weather` downloads ERA5 2023 for NG's bounding box (about 2.6-14.8E, 4.3-13.9N): several GB, possibly hours, and the merge step is memory-hungry. Check free RAM and disk first and tell me if it is tight.
3. Solver time: one full-year plant LP takes ~100 s on GLPK; `optimize_plant` solves 552 hexes x 3 demand centres (~1,650 LPs), so ~45 h on GLPK single-threaded. Check whether Gurobi (academic licence) or HiGHS is available/installable here, and how many CPU cores there are. `src/main/plant_optimization.py` only passes Gurobi-only options when `solver == 'gurobi'`, so switching `solver` in `config.yaml` is safe. HiGHS has not been tested with the PyPSA version in `environment.yaml`.
4. `basic_h2_plant` capex files are still the Namibia euro values while NG parameters are USD. Flag this as a validity caveat for results; do not silently change numbers.
5. Geo-X reads only the FIRST row of `parameters/NG/hydrogen/country_parameters.xlsx` (Nigeria, 12% WACC). Low/high finance scenarios need row reordering.
6. Do NOT open/save `parameters/NG/hydrogen/technology_parameters.xlsx` with openpyxl: the Infra road CAPEX/OPEX cells are formulas and their cached values get dropped (pandas then reads NaN).
7. If this machine is Windows, `import rasterio, atlite` may fail with "DLL load failed" (libtiff/libdeflate/lerc mismatch, and a newer base-env GDAL on PATH). Fixes used before: `conda install -n geox -c conda-forge libdeflate=1.14 lerc=4.0`, and an `activate.d` script that removes base `Library\bin` from PATH while `geox` is active. On Linux/macOS this should not apply.
8. `git status` on the old machine also showed uncommitted edits to `parameters/NA/ammonia/*`, `parameters/ZM/copper/*`, `parameters/basic_nh3_plant/*`. They are not part of the NG case and are not in the fork; ignore them.

## Audit checklist (read-only; report each as OK / PROBLEM / UNKNOWN with the evidence)

1. Machine: OS, CPU cores, RAM free, disk free in the project drive, network access to github.com and cds.climate.copernicus.eu.
2. Repo: is it cloned; current branch is `ng-hydrogen`; HEAD is f7bca08 or a descendant; `git remote -v` shows `origin` = linibensonjr/Geo-XNGA and `upstream` = ClimateCompatibleGrowth/Geo-X (add upstream if missing); `git status` clean; `data/NG/hex_final_NG.geojson` present with 552 features.
3. Config matches the plan: `config.yaml` has country NG, weather_year 2023, plant_type hydrogen, generators solar+wind, currency USD, and report the current solver. `parameters/basic_h2_plant/generators.csv` has only Wind and Solar rows.
4. Environment: conda/mamba available; env `geox` exists (create from `environment.yaml` if not, but ask me first); `python -c "import geopandas, rasterio, atlite, pypsa"` works via `conda run -n geox`; `snakemake --version`; glpk (`glpsol --version`) present; Gurobi/HiGHS presence and licence status.
5. Weather credentials: `~/.cdsapirc` present and well-formed (do not print the key); report whether a small test request works.
6. Existing outputs: list anything already in `cutouts/`, `resources/`, `results/`, `plots/`; for any `*.nc` cutout check it opens and covers the full NG bounding box and all of 2023 hourly (8,760 steps). Mark partial/corrupt files for deletion but ask before deleting.
7. Dry run: `snakemake -n optimise_all` from the repo root and summarise which jobs would run and why.
8. Sanity of inputs: the files `parameters/NG/hydrogen/*.xlsx` exist and load in pandas with no NaN in the Infra road CAPEX/OPEX cells.

## Then provide the runs

After the audit, give me an ordered run plan as copy-paste commands for THIS machine's shell, each with: what it does, expected duration and memory/disk, which output file proves it finished, and how to resume if it dies. Cover at minimum:

1. Any setup fixes needed (env, cdsapirc, upstream remote), as a short list for me to approve.
2. `snakemake -j4 run_weather` (or lower `-j` if RAM is tight), plus a validity check of `cutouts/NG_2023.nc` afterwards.
3. `snakemake -j1 resources/hex_transport_NG_hydrogen.geojson resources/hex_water_NG_hydrogen.geojson` (cheap re-run).
4. A SHORT smoke test of `optimize_plant` before the full run (for example a handful of hexes, or time a single LP) so we can extrapolate total runtime, then your recommendation: stay on GLPK, move to HiGHS, or get Gurobi, and the matching `solver:` value for `config.yaml`.
5. The full run: `snakemake -j<N> optimise_all` using a number of jobs that fits cores and RAM (each job is a separate LP), run detached with a log file (`nohup`/`tmux` on Linux, `Start-Process` on Windows) so a closed terminal does not kill it. Use `--keep-going` and `--rerun-incomplete` so one failed hex batch does not lose everything.
6. `snakemake -j1 calculate_cost_components` and `snakemake -j1 map_all`, and where the final results and plots land (`results/`, `plots/NG_2023_hydrogen`).
7. A monitoring one-liner I can paste to see progress, and what to send back to the other machine / to me when done (results/ and plots/ are gitignored, so say how to bring them back, for example zip).

Rules for this session: stay read-only until I approve; never commit or push without asking; never delete files without asking; if something in my facts above contradicts what you find on disk, tell me and trust the disk.
````

## Notes for you (not part of the prompt)

- This file is in the repo working tree but not committed. If the new machine will clone from the fork, commit and push it to `ng-hydrogen` first, or just paste the prompt above.
- The prompt asks Claude to trust the disk over its facts, since the memory/notes are from 2026-10-10 to 2026-10-20.
