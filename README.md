# Road traffic crashes on the Lagos–Ibadan Expressway, 2013–2022: analysis code

This folder holds the Python code for the spatiotemporal, machine-learning and geoparsing analysis of Federal Road Safety Corps (FRSC) crash records on the Lagos–Ibadan Expressway. A single command runs the whole analysis, from the raw annual returns to the figures, tables, GIS layers and validation results.

**Reproducibility check (3 November 2026).** The full pipeline was re-run from the raw files in a clean folder. Every key output was byte-identical to the results used in the report and manuscript: the cleaned records, the hot-spot tables, the forecasts, the geoparsed dataset and both validation summaries.

## What this code does and does not reproduce

It reproduces:
- **The analytical report** (*Lagos_Ibadan_RTC_Spatiotemporal_ML_Report_v9*): all figures and tables.
- **The geoparsed GIS package:** CSV, shapefiles, GeoJSON, KML and maps.
- **The field-GPS validation and gazetteer verification.**
- **Manuscript v27:** Figure 24 (expected crashes per section in 2027) and Table S1 (field GPS positions).

It does **not** reproduce the tier A–D chainage geoparser described in Sections 2.3–2.4 of the manuscript, nor the 5,020-record figures that come from it. That pipeline was built separately and its code is not part of this package.

## Requirements

- Python 3 with the packages in `requirements.txt` (tested with Python 3.13 on Linux):
  ```
  pip install -r requirements.txt
  ```
- About 1 GB of free memory. A full run takes about 7 minutes on one CPU core.
- No internet connection is needed. The only exception is the optional web atlas (see below).

## Data: where to put the FRSC files

The FRSC data are not included. Place them as follows, keeping the original file names:

| Folder | Contents |
|---|---|
| `data/with_coordinates/` | The ten annual returns **with** latitude and longitude (`2013 MERGED ROUTES.csv` … `2022 MERGED ROUTES CSV.csv`), plus `2017_RTC.csv`. `2017_RTC.csv` holds the 2017 crash records with the GPS positions collected by FRSC field officers in 2026. |
| `data/without_coordinates/` | The same ten returns **without** coordinates (the files that are geoparsed). |

If the returns are still in RAR archives, extract them with any archive tool. `scripts/unrar.py` is a minimal extractor for Linux systems that have `libarchive` installed:
```
python3 scripts/unrar.py <archive.rar> <out_dir>
```

## Running

From this folder:
```
bash run_all.sh
```
All results are written to `outputs/`. On Windows, run the same commands from Git Bash or WSL, or type the lines of `run_all.sh` one by one. Every script can also be run on its own: its first lines give the usage.

## Pipeline

| Step | Script | What it does | Main outputs |
|---|---|---|---|
| A1 | `01_clean.py` | Harmonises the ten returns. Resolves ambiguous dates, swapped latitude/longitude, the 2019 header shift, casualty-total inconsistencies and duplicates. | `rtc_lagos_ibadan_clean_2013_2022.csv` (4,992 records), `rtc_all_rows_with_attributes.csv` (5,014 rows), `cleaning_log.txt` |
| A2 | `02_corridor.py` | Builds a data-derived centreline (Ojota → Iwo Road, about 119 km), then computes chainage and an on-corridor flag. | `centreline_derived.csv` |
| A3 | `03_eda.py` | Temporal and causal summaries. | Report Figs 2–3 |
| A4 | `04_hotspots.py` | Getis–Ord Gi* on 2-km, 5-km and inter-urban segments (Benjamini–Hochberg FDR), emerging hot spots (Mann–Kendall), DBSCAN clusters, kernel density. | `hotspots_*.csv`, Report Figs 6, 7 and 9, Tables 2–3 |
| A5 | `sankey_cause_vehicle.py` | Sankey diagram: cause → vehicle → outcome. | Report Fig 4 |
| A6 | `fig01_annual_panel.py` | Annual trends in crashes, casualties and severity. | Report Fig 1 |
| A7 | `report_maps_basemap.py` | Kernel-density and Gi*/DBSCAN maps on the Natural Earth basemap. | Report Figs 5 and 8 |
| B1 | `05_ml.py` | Monthly forecasting back-test (seasonal naïve, Poisson GLM, random forest, gradient boosting, ensemble); 2023 forecast for 2-km segments with conformal intervals; fatal-crash classifier with permutation importance. | Report Tables 4–6, Fig 11 |
| B2 | `fig12_severity_panel.py` | "What makes a crash fatal" (4 panels). | Report Fig 13 |
| B3 | `ml_2027.py`, `fig_2027.py` | Corridor forecast to December 2027, with plateau and trend scenarios. | Report Fig 10, `forecast_*_2023_2027.csv`, `fig_forecast_2027.png` |
| B4 | `ml_seg_2027.py`, `fig_seg_2027.py`, `fig_seg_2027_profile.py` | 5-year-ahead random forest for each 2-km section's share of crashes. Back-tested on 2020–2022; Monte-Carlo probability that each section is among the busiest 10% in 2027. | `hotspot_sections_2027.csv`, Report Fig 12, Manuscript Fig 24 |
| C1 | `geo10_prepare.py` | Extracts command and location text from the returns without coordinates. | `geoparsing/in/` |
| C2 | `geo_01_geocode.py` | Geoparser. For each year, the gazetteer is built only from the other nine years (leave-one-year-out). Matching runs from exact name to landmark token to fuzzy name, followed by spatial disambiguation by reporting command and a confidence class (High/Medium/Low). | `geoparsing/out/geo_YYYY.csv` |
| C3 | `geo10_combine.py`, `geo10_pass2.py` | Attaches crash attributes. A second pass handles names reported in only one year, using other records of the same year but never a record's own coordinate. | `geoparsing/combined_final.csv` |
| C4 | `geo10_export.py`, `geo10_maps.py` | GIS export (shapefiles written without GDAL) and static maps. | `geoparsing/GIS/`, Report Figs 14–18 |
| D1 | `geo_validate_2017.py` | Compares geoparsed positions with the 2026 field GPS positions, after screening out missing, placeholder, off-corridor and command-inconsistent GPS points. | `validation/`, Manuscript Section 3.1 |
| D2 | `gazetteer_verify.py` | Verifies each gazetteer entry (name × command zone) against the median of its field GPS fixes. | `gazetteer_verification/`, Manuscript Section 3.1 and Table S1 |

Optional: `geo10_atlas.py` builds an interactive HTML atlas (Leaflet with OpenStreetMap tiles). The page needs an internet connection when it is opened.
```
python3 scripts/geo10_atlas.py outputs/geoparsing/GIS <atlas_dir> atlas.html
```

## Role of the large language model

A large language model (Claude Opus 5.5) helped draft the location-name normalisation rules and alias table (`PHRASE` and related lists in `geo_01_geocode.py`). These were checked by the authors and stored as code. **No LLM is called when the pipeline runs**, so results are deterministic and fully reproducible.

## Notes

- **Chainage.** All distances are in km from Ojota along the data-derived centreline (step A2). This scale differs from the FRSC chainage used in the manuscript's tier A–D analysis, so section numbers will not line up exactly with the manuscript's Tables 7–8.
- **Randomness.** All random procedures (random forests, bootstraps, Monte-Carlo simulations) use a fixed seed (42).
- **File names containing "2017".** These refer to `2017_RTC.csv`, which contains 2017 crash records. The GPS positions in that file were collected by FRSC field officers in 2026.
- **Basemap.** The `basemap/` folder holds a crop of Natural Earth 1:10m data (raster NE1_HR_LC_SR_W and vectors). Natural Earth data are in the public domain.
- **Document builders.** The scripts that assembled the Word documents are not included, because they only format results that this pipeline produces.

## Licence

[To be added by the authors, e.g. MIT for the code. FRSC data are subject to FRSC's own terms of use.]
