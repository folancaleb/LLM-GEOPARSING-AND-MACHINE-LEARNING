# RTC Lagos–Ibadan analysis code

All source code for the analysis of FRSC road traffic crash records on the Lagos–Ibadan Expressway, 2013–2022, in one Markdown file. Each script is reproduced in full; to run the pipeline, save each block under the file name shown (scripts in a `scripts/` folder) or use the original ZIP package. The Natural Earth basemap files (`basemap/ne1_crop.png`, `ne1_crop.json`, `ne_vectors_clip.json`) are data, not code, and are only in the ZIP package.

## Contents

- [README](#readme)
- [run_all.sh](#run_allsh)
- [requirements.txt](#requirementstxt)
- A. Cleaning, corridor referencing, exploratory analysis and hot spots
  - [`01_clean.py`](#01_cleanpy)
  - [`02_corridor.py`](#02_corridorpy)
  - [`03_eda.py`](#03_edapy)
  - [`04_hotspots.py`](#04_hotspotspy)
  - [`sankey_cause_vehicle.py`](#sankey_cause_vehiclepy)
  - [`fig01_annual_panel.py`](#fig01_annual_panelpy)
  - [`report_maps_basemap.py`](#report_maps_basemappy)
- B. Machine learning and forecasts
  - [`05_ml.py`](#05_mlpy)
  - [`fig12_severity_panel.py`](#fig12_severity_panelpy)
  - [`ml_2027.py`](#ml_2027py)
  - [`fig_2027.py`](#fig_2027py)
  - [`ml_seg_2027.py`](#ml_seg_2027py)
  - [`fig_seg_2027.py`](#fig_seg_2027py)
  - [`fig_seg_2027_profile.py`](#fig_seg_2027_profilepy)
- C. LLM-assisted geoparsing
  - [`geo10_prepare.py`](#geo10_preparepy)
  - [`geo_01_geocode.py`](#geo_01_geocodepy)
  - [`geo10_combine.py`](#geo10_combinepy)
  - [`geo10_pass2.py`](#geo10_pass2py)
  - [`geo10_export.py`](#geo10_exportpy)
  - [`geo10_maps.py`](#geo10_mapspy)
  - [`geo10_atlas.py`](#geo10_atlaspy)
- D. Validation against the 2026 FRSC field GPS survey
  - [`geo_validate_2017.py`](#geo_validate_2017py)
  - [`gazetteer_verify.py`](#gazetteer_verifypy)
- E. Shared helpers and utilities
  - [`style.py`](#stylepy)
  - [`basemap_ne.py`](#basemap_nepy)
  - [`unrar.py`](#unrarpy)

---

## README


This folder holds the Python code for the spatiotemporal, machine-learning and geoparsing analysis of Federal Road Safety Corps (FRSC) crash records on the Lagos–Ibadan Expressway. A single command runs the whole analysis, from the raw annual returns to the figures, tables, GIS layers and validation results.

**Reproducibility check (3 November 2026).** The full pipeline was re-run from the raw files in a clean folder. Every key output was byte-identical to the results used in the report and manuscript: the cleaned records, the hot-spot tables, the forecasts, the geoparsed dataset and both validation summaries.

### What this code does and does not reproduce

It reproduces:
- **The analytical report** (*Lagos_Ibadan_RTC_Spatiotemporal_ML_Report_v9*): all figures and tables.
- **The geoparsed GIS package:** CSV, shapefiles, GeoJSON, KML and maps.
- **The field-GPS validation and gazetteer verification.**
- **Manuscript v27:** Figure 24 (expected crashes per section in 2027) and Table S1 (field GPS positions).

It does **not** reproduce the tier A–D chainage geoparser described in Sections 2.3–2.4 of the manuscript, nor the 5,020-record figures that come from it. That pipeline was built separately and its code is not part of this package.

### Requirements

- Python 3 with the packages in `requirements.txt` (tested with Python 3.13 on Linux):
  ```
  pip install -r requirements.txt
  ```
- About 1 GB of free memory. A full run takes about 7 minutes on one CPU core.
- No internet connection is needed. The only exception is the optional web atlas (see below).

### Data: where to put the FRSC files

The FRSC data are not included. Place them as follows, keeping the original file names:

| Folder | Contents |
|---|---|
| `data/with_coordinates/` | The ten annual returns **with** latitude and longitude (`2013 MERGED ROUTES.csv` … `2022 MERGED ROUTES CSV.csv`), plus `2017_RTC.csv`. `2017_RTC.csv` holds the 2017 crash records with the GPS positions collected by FRSC field officers in 2026. |
| `data/without_coordinates/` | The same ten returns **without** coordinates (the files that are geoparsed). |

If the returns are still in RAR archives, extract them with any archive tool. `scripts/unrar.py` is a minimal extractor for Linux systems that have `libarchive` installed:
```
python3 scripts/unrar.py <archive.rar> <out_dir>
```

### Running

From this folder:
```
bash run_all.sh
```
All results are written to `outputs/`. On Windows, run the same commands from Git Bash or WSL, or type the lines of `run_all.sh` one by one. Every script can also be run on its own: its first lines give the usage.

### Pipeline

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

### Role of the large language model

A large language model (Claude Opus 5.5) helped draft the location-name normalisation rules and alias table (`PHRASE` and related lists in `geo_01_geocode.py`). These were checked by the authors and stored as code. **No LLM is called when the pipeline runs**, so results are deterministic and fully reproducible.

### Notes

- **Chainage.** All distances are in km from Ojota along the data-derived centreline (step A2). This scale differs from the FRSC chainage used in the manuscript's tier A–D analysis, so section numbers will not line up exactly with the manuscript's Tables 7–8.
- **Randomness.** All random procedures (random forests, bootstraps, Monte-Carlo simulations) use a fixed seed (42).
- **File names containing "2017".** These refer to `2017_RTC.csv`, which contains 2017 crash records. The GPS positions in that file were collected by FRSC field officers in 2026.
- **Basemap.** The `basemap/` folder holds a crop of Natural Earth 1:10m data (raster NE1_HR_LC_SR_W and vectors). Natural Earth data are in the public domain.
- **Document builders.** The scripts that assembled the Word documents are not included, because they only format results that this pipeline produces.

### Licence

[To be added by the authors, e.g. MIT for the code. FRSC data are subject to FRSC's own terms of use.]


---

## run_all.sh

```bash
#!/usr/bin/env bash
# run_all.sh — Reproduce the full analysis of FRSC road traffic crashes on the Lagos–Ibadan Expressway, 2013–2022.
# Usage:  bash run_all.sh            (run from this folder)
# Inputs: data/with_coordinates/     ten annual FRSC returns WITH latitude/longitude + 2017_RTC.csv (field GPS file)
#         data/without_coordinates/  the same ten returns WITHOUT coordinates (the files to be geoparsed)
set -euo pipefail
cd "$(dirname "$0")"
S=scripts; RAW=data/with_coordinates; RAW_NC=data/without_coordinates; O=outputs; G=$O/geoparsing
mkdir -p "$O" "$G/in" "$G/out"
py() { echo ">> $*"; python3 -I "$@"; }

# ---- A. Cleaning, corridor referencing, exploratory analysis, hot spots (recorded coordinates) ----
py $S/01_clean.py "$RAW" "$O"                  # harmonised records: rtc_lagos_ibadan_clean_2013_2022.csv (+ all rows with attributes)
py $S/02_corridor.py "$O"                      # data-derived centreline (Ojota → Iwo Road), chainage, on-corridor flag
py $S/03_eda.py "$O"                           # temporal and causal summaries (Seaborn figures)
py $S/04_hotspots.py "$O"                      # Getis–Ord Gi* (2-km, 5-km, inter-urban), emerging hot spots, DBSCAN, KDE
py $S/sankey_cause_vehicle.py "$O"             # cause → vehicle → outcome Sankey
py $S/fig01_annual_panel.py "$O"               # annual trends (4 panels)
py $S/report_maps_basemap.py "$O" "$O/figures" # KDE and Gi*/DBSCAN maps on the Natural Earth basemap

# ---- B. Machine learning and forecasts ----
py $S/05_ml.py "$O"                            # monthly forecasting back-test, 2023 segment forecast, severity classifier
py $S/fig12_severity_panel.py "$O"             # what makes a crash fatal (4 panels; uses the 05_ml importance)
py $S/ml_2027.py "$O"                          # monthly/annual forecast to December 2027 (ensemble + scenarios)
py $S/fig_2027.py "$O"                         # 2027 forecast figure
py $S/ml_seg_2027.py "$O"                      # 2-km sections likely to be hot spots in 2027 (5-year-ahead model)
py $S/fig_seg_2027.py "$O"                     # map + profile of 2027 hot-spot sections
py $S/fig_seg_2027_profile.py "$O"             # expected crashes per section in 2027 (standalone chart)

# ---- C. LLM-assisted geoparsing of the returns without coordinates (leave-one-year-out) ----
py $S/geo10_prepare.py "$RAW_NC" "$G/in"
for y in $(seq 2013 2022); do                  # gazetteer built from the OTHER nine years only
  py $S/geo_01_geocode.py "$O/rtc_lagos_ibadan_clean_2013_2022.csv" "$O/centreline_derived.csv" "$G/in/in_$y.csv" "$y" "$G/out/geo_$y.csv"
done
py $S/geo10_combine.py "$G/out" "$O/rtc_all_rows_with_attributes.csv" "$G/combined_raw.csv"
py $S/geo10_pass2.py "$G/combined_raw.csv" "$O/rtc_all_rows_with_attributes.csv" "$O/centreline_derived.csv" "$G/combined_final.csv"
py $S/geo10_export.py "$G/combined_final.csv" "$O/centreline_derived.csv" "$G/GIS"   # CSV, shapefiles, GeoJSON, KML, segment hot spots
py $S/geo10_maps.py "$G/GIS" "$O/centreline_derived.csv"                              # static maps on the Natural Earth basemap

# ---- D. Validation against the 2026 FRSC field GPS survey ----
py $S/geo_validate_2017.py "$RAW/2017_RTC.csv" "$G/combined_final.csv" "$O/centreline_derived.csv" "$O/rtc_all_rows_with_attributes.csv" "$O/validation"
py $S/gazetteer_verify.py "$RAW/2017_RTC.csv" "$O/validation/validation_2017_records.csv" "$O/gazetteer_verification"
echo "Done. Results are in $O/"
```

## requirements.txt

```text
# Tested with Python 3.13.16 on Linux. Other recent versions are likely to work but were not tested.
pandas==3.0.5
numpy==2.5.3
scipy==1.18.1
scikit-learn==1.9.1
matplotlib==3.11.2
seaborn==0.13.2
pillow==12.3.0
```


---

## A. Cleaning, corridor referencing, exploratory analysis and hot spots


### 01_clean.py

```python
"""
01_clean.py — Harmonise FRSC road traffic crash (RTC) records, Lagos–Ibadan Expressway, 2013–2022.
Usage: python3 -I 01_clean.py <raw_dir> <out_dir>
"""
import sys, re, glob, os
import numpy as np, pandas as pd

RAW, OUT = sys.argv[1], sys.argv[2]
os.makedirs(OUT, exist_ok=True)

FILES = {2013: "2013 MERGED ROUTES.csv", 2014: "2014 MERGED ROUTES.csv", 2015: "2015 MERGED ROUTES.csv",
         2016: "2016 MERGED ROUTES.csv", 2017: "2017 MERGED ROUTES CSV.csv", 2018: "2018 MERGED ROUTES CSV.csv",
         2019: "2019 MERGED ROUTES.csv", 2020: "2020 MERGED ROUTES CSV.csv", 2021: "2021 MERGED ROUTES CSV.csv",
         2022: "2022 MERGED ROUTES CSV.csv"}

log = []
frames = []
for yr, fn in FILES.items():
    d = pd.read_csv(os.path.join(RAW, fn), encoding="latin1", dtype=str, low_memory=False)
    d.columns = [c.strip() for c in d.columns]
    d = d[d["DATE"].notna() & (d["DATE"].str.strip() != "")].copy()
    if yr == 2019:
        # header shift in 2019: extra 'UNKNOWN' column after LOCATION pushes later fields one column right
        d["LOCATION"] = d["UNKNOWN"].where(d["UNKNOWN"].notna(), d["LOCATION"])
        d["LONGITUDE"] = d["LONGITUDE CATEGORY"]
        d["VEHICLE CATEGORY"] = d["VEHICLE"]
        d["VEHICLE"] = d["NAME OF DRIVER"]
    if yr == 2017:
        alt = pd.read_csv(os.path.join(RAW, "2017_RTC.csv"), encoding="latin1", dtype=str)
        alt.columns = [c.strip() for c in alt.columns]
        d = d.reset_index(drop=True)
        assert (d["LOCATION"].str.strip().values == alt["LOCATION"].str.strip().values).all()
        d["DATE"] = alt["DATE"].values  # Excel serial dates (unambiguous)
    d["file_year"] = yr
    d["row_order"] = np.arange(len(d))
    frames.append(d)
    log.append(f"{yr}: {len(d)} dated records from '{fn}'")

df = pd.concat(frames, ignore_index=True)
df["COMMAND"] = df["COMMAND"].fillna("").str.strip().str.upper()
df["cmd_code"] = df["COMMAND"].str.extract(r"(RS\s*\d+\.\d+)")[0].str.replace(" ", "")

# ---------------- DATE PARSING ----------------
def parse_parts(s, fy=None):
    s = re.sub(r"^[^\d]+|[^\d]+$", "", str(s).strip())
    if re.fullmatch(r"\d{5}(\.0+)?", s):
        return ("serial", pd.Timestamp("1899-12-30") + pd.Timedelta(days=int(float(s))))
    m = re.fullmatch(r"(\d{1,2})[/\-\.](\d{1,2})[/\-\.](\d{2,4})", s)
    if not m:
        return ("bad", None)
    a, b, y = int(m.group(1)), int(m.group(2)), int(m.group(3))
    if y < 100: y += 2000
    if fy is not None and y != fy and y % 100 == fy % 100:  # typo years e.g. 1920/3019/2922
        y = fy
    return ("dm", (a, b, y))

def mk(y, mth, day):
    try: return pd.Timestamp(year=y, month=mth, day=day)
    except Exception: return None

dates, dflag = [], []
parsed = [parse_parts(s, fy) for s, fy in zip(df["DATE"], df["file_year"])]
# first pass: unambiguous
cand = []
for (kind, v), fy in zip(parsed, df["file_year"]):
    if kind == "serial":
        cand.append(("ok", v, "serial"))
    elif kind == "bad":
        cand.append(("bad", None, "unparseable"))
    else:
        a, b, y = v
        dm, md = mk(y, b, a), mk(y, a, b)   # dd/mm vs mm/dd
        if dm is not None and md is not None and a != b:
            cand.append(("amb", (dm, md), "ambiguous"))
        else:
            r = dm or md
            cand.append(("ok", r, "dd/mm" if dm is not None else "mm/dd") if r is not None else ("bad", None, "invalid"))
df["_kind"] = [c[0] for c in cand]
# second pass: resolve ambiguous using nearest unambiguous neighbours (same file & command, file order)
res = [c[1] if c[0] == "ok" else None for c in cand]
method = [c[2] for c in cand]
for (fy, cmd), g in df.groupby(["file_year", "cmd_code"], dropna=False):
    idx = g.index.tolist()
    for k, i in enumerate(idx):
        if cand[i][0] != "amb": continue
        neigh = []
        for j in idx[max(0, k-6):k] + idx[k+1:k+7]:
            if cand[j][0] == "ok" and res[j] is not None: neigh.append(res[j])
        dm, md = cand[i][1]
        if neigh:
            ref = pd.Series(neigh).median()
            pick = dm if abs((dm - ref).days) <= abs((md - ref).days) else md
            method[i] = "ambiguous→neighbour"
        else:
            pick = dm; method[i] = "ambiguous→dd/mm default"
        res[i] = pick
df["date"] = pd.to_datetime(pd.Series(res, index=df.index))
df["date_method"] = method
bad_year = df["date"].notna() & (df["date"].dt.year != df["file_year"])
log.append(f"Date parse: {df['date'].notna().sum()} parsed; methods: {pd.Series(method).value_counts().to_dict()}")
log.append(f"Records whose parsed year != file year (set to NaT): {int(bad_year.sum())}")
df.loc[bad_year, "date"] = pd.NaT

# ---------------- CRASH TIME ----------------
def parse_time(s):
    s = str(s).upper().replace("HRS", "").replace("HR", "").replace(":", "").replace(".", "").strip()
    if not re.fullmatch(r"\d{3,4}", s): return np.nan
    s = s.zfill(4); h, m = int(s[:2]), int(s[2:])
    return h + m/60 if h < 24 and m < 60 else np.nan
df["crash_hour_dec"] = df["CRASH TIME"].map(parse_time)
df["hour"] = np.floor(df["crash_hour_dec"])

# ---------------- COORDINATES ----------------
def num(col):
    return pd.to_numeric(df[col].astype(str).str.replace(r"[^\d\.\-]", "", regex=True), errors="coerce") if col in df else np.nan
lat, lon = num("LATITUDE"), num("LONGITUDE")
# Swap where clearly transposed (Nigeria SW: lat ~6.4–7.6, lon ~2.9–4.0)
sw = (lat.between(2.5, 4.5)) & (lon.between(6.0, 8.0))
lat2, lon2 = lat.where(~sw, lon), lon.where(~sw, lat)
inbox = lat2.between(6.40, 7.60) & lon2.between(2.90, 4.10)
df["lat"], df["lon"] = lat2.where(inbox), lon2.where(inbox)
df["coord_source"] = np.where(inbox, np.where(sw, "recorded (lat/lon swapped, corrected)", "recorded"), "missing/invalid")
log.append(f"Coordinates: swapped-corrected {int((sw & inbox).sum())}; valid {int(inbox.sum())}; missing/invalid {int((~inbox).sum())}")

# Impute missing coords from a gazetteer built from the data itself (median coord of same location name)
def norm_loc(s):
    s = str(s).upper()
    s = re.sub(r"[^A-Z0-9 ]", " ", s); s = re.sub(r"\b(AREA|BUS STOP|B/S|OPP|BY|AFTER|BEFORE|NEAR|INWARD|OUTWARD|LAGOS|IBADAN|LAG|IBD)\b", " ", s)
    return re.sub(r"\s+", " ", s).strip()
df["loc_norm"] = df["LOCATION"].map(norm_loc)
gaz = df[df["lat"].notna()].groupby("loc_norm").agg(glat=("lat", "median"), glon=("lon", "median"), gn=("lat", "size"))
gaz = gaz[(gaz.gn >= 2) & (gaz.index.str.len() >= 4)]
m = df["lat"].isna() & df["loc_norm"].isin(gaz.index)
df.loc[m, "lat"] = df.loc[m, "loc_norm"].map(gaz.glat); df.loc[m, "lon"] = df.loc[m, "loc_norm"].map(gaz.glon)
df.loc[m, "coord_source"] = "imputed from same-location records"
log.append(f"Coordinates imputed from in-data gazetteer: {int(m.sum())}; still missing: {int(df['lat'].isna().sum())}")

# ---------------- CASUALTIES ----------------
def tot(prefix, total_col):
    comps = [f"{prefix}/A/M", f"{prefix}/A/F", f"{prefix}/C/M", f"{prefix}/C/F"]
    c = df[comps].apply(lambda s: pd.to_numeric(s, errors="coerce")).fillna(0).sum(axis=1)
    t = pd.to_numeric(df[total_col], errors="coerce")
    return c, t
ni_c, ni_t = tot("NI", "TOTAL"); nk_c, nk_t = tot("NK", "TOTAL.1"); pi_c, pi_t = tot("PI", "TOTAL.2")
inv = pi_t.where(pi_t.notna() & (pi_t >= pi_c), pi_c)          # persons involved: larger of the two (comps often incomplete)
def reconcile(c, t, cap):
    # Prefer reported TOTAL when it is plausible (<= persons involved); else component sum if plausible; else the smaller
    t_ok = t.notna() & (t <= cap); c_ok = c <= cap
    return np.where(t_ok, t, np.where(c_ok, c, np.fmin(c, t.fillna(c))))
df["involved"] = inv
df["injured"] = reconcile(ni_c, ni_t, inv); df["killed"] = reconcile(nk_c, nk_t, inv)
for n, c, t in [("injured", ni_c, ni_t), ("killed", nk_c, nk_t), ("involved", pi_c, pi_t)]:
    log.append(f"{n}: component-sum vs reported TOTAL disagree in {int(((c - t.fillna(0)).abs() > 0.01).sum())} rows (reconciled against persons involved)")
df.loc[df["involved"] > 150, "involved"] = np.nan
df["fatal"] = (df["killed"] > 0).astype(int)
df["severity"] = np.select([df["killed"] > 0, df["injured"] > 0], ["Fatal", "Serious/Minor injury"], "No injury")

# ---------------- CAUSE ----------------
CODE = {"SPEED": "Speeding", "SPV": "Speeding", "SLV": "Speeding", "SOV": "Speeding", "LSV": "Speeding",
        "MECH DEFICIENT": "Mechanical deficiency", "BRAKE FAILURE": "Brake failure", "BREAK FAILURE": "Brake failure", "BFL": "Brake failure",
        "TYRE BURST": "Tyre burst", "TIRE BURST": "Tyre burst", "TBT": "Tyre burst",
        "LOSS OF CONTROL": "Loss of control", "LOC": "Loss of control",
        "WOV": "Dangerous driving/overtaking", "WOT": "Dangerous driving/overtaking", "DGD": "Dangerous driving/overtaking",
        "DAD": "Dangerous driving/overtaking", "DANGEROUS DRIV": "Dangerous driving/overtaking", "RTV": "Dangerous driving/overtaking",
        "ROUTE VIOLATION": "Dangerous driving/overtaking",
        "FATIQUE": "Fatigue", "FATIGUE": "Fatigue", "DUI": "DUI",
        "ROB": "Other", "ROBBERY": "Other", "HIT AND RUN": "Other", "OTHERS": "Other", "OTHES": "Other", "POOR WEATHER": "Other", "DOT": "Other"}
def cause(row):
    for col in ["PROBABLE CAUSE(S) OF RTC", "Unnamed: 16"]:
        v = row.get(col)
        if isinstance(v, str) and v.strip():
            first = re.split(r"[,/]", v.upper())[0].strip().strip(".")
            c = CODE.get(first, "Other")
            if c != "Other" or col == "Unnamed: 16":
                return c
    return "Other"
df["cause"] = df.apply(cause, axis=1)

# ---------------- VEHICLE ----------------
OWN = {"COMMERCIAL", "PRIVATE", "GOVERNMENT", "COMMECIAL", "COMMERCILA", "COMMERCILAL", "C0MMERCIAL", "PRIVATEV"}
def vehicle(row):
    a = str(row.get("VEHICLE CATEGORY", "")).upper().strip(" .")
    b = str(row.get("VEHICLE", "")).upper().strip(" .")
    own_src = a if any(o in a for o in OWN) else (b if any(o in b for o in OWN) else "")
    typ = b if own_src == a else a
    own = "Commercial" if "COMM" in own_src or "C0MM" in own_src else ("Private" if "PRIV" in own_src else ("Government" if "GOV" in own_src else "Unknown"))
    multi = int(bool(re.search(r"[/&]|AND", typ)))
    t0 = re.split(r"[/&,]", typ)[0].strip()
    T = "Other/unknown"
    for k, v in [("MOTORCYCLE", "Motorcycle"), ("TRICYCLE", "Tricycle"), ("TANKER", "Tanker"), ("TRAILER", "Trailer"),
                 ("TRUCK", "Truck"), ("BUS", "Bus"), ("SIENNA", "Bus"), ("JEEP", "Jeep/SUV"), ("PICK", "Pick-up"), ("P/UP", "Pick-up"), ("CAR", "Car")]:
        if k in t0: T = v; break
    return pd.Series({"ownership": own, "vehicle_type": T, "multi_vehicle": multi})
df = pd.concat([df, df.apply(vehicle, axis=1)], axis=1)
df["direction"] = np.where(df["ROUTES"].fillna("").str.upper().str.replace(" ", "").str.startswith("IBADAN"), "Ibadan→Lagos", "Lagos→Ibadan")

# ---------------- DE-DUPLICATION ----------------
key = ["date", "CRASH TIME", "loc_norm", "killed", "injured", "involved"]
dup = df.duplicated(subset=key, keep="first") & df["date"].notna()
log.append(f"Exact duplicates removed (same date, time, location, casualties): {int(dup.sum())}")
df.assign(is_duplicate=dup)[["file_year", "row_order", "COMMAND", "cmd_code", "DATE", "date", "date_method", "CRASH TIME", "hour", "ROUTES", "LOCATION",
    "lat", "lon", "coord_source", "ownership", "vehicle_type", "multi_vehicle", "cause", "injured", "killed", "involved", "fatal", "severity",
    "is_duplicate"]].rename(columns={"COMMAND": "command"}).to_csv(os.path.join(OUT, "rtc_all_rows_with_attributes.csv"), index=False)
df = df[~dup].copy()

keep = ["file_year", "COMMAND", "cmd_code", "DATE", "date", "date_method", "CRASH TIME", "crash_hour_dec", "hour",
        "ROUTES", "direction", "LOCATION", "loc_norm", "lat", "lon", "coord_source", "ownership", "vehicle_type",
        "multi_vehicle", "cause", "injured", "killed", "involved", "fatal", "severity"]
out = df[keep].rename(columns={"DATE": "date_raw", "CRASH TIME": "time_raw", "ROUTES": "route_raw", "LOCATION": "location_raw", "COMMAND": "command"})
out["year"] = out["date"].dt.year.fillna(out["file_year"]).astype(int)
out.to_csv(os.path.join(OUT, "rtc_lagos_ibadan_clean_2013_2022.csv"), index=False)
log.append(f"FINAL: {len(out)} records; with date {out['date'].notna().sum()}; with coords {out['lat'].notna().sum()}; with hour {out['hour'].notna().sum()}")
open(os.path.join(OUT, "cleaning_log.txt"), "w").write("\n".join(log))
print("\n".join(log))
```


### 02_corridor.py

```python
"""
02_corridor.py — Derive a data-driven expressway centreline, compute chainage (km from Lagos end),
flag on-corridor records, and validate chainage against 'KMxx' markers in location names.
Usage: python3 -I 02_corridor.py <out_dir>
"""
import sys, os, re, numpy as np, pandas as pd
OUT = sys.argv[1]
d = pd.read_csv(os.path.join(OUT, "rtc_lagos_ibadan_clean_2013_2022.csv"), parse_dates=["date"])

KX, KY = 111.32 * np.cos(np.deg2rad(6.9)), 110.57          # km per degree (local equirectangular)
def xy(lat, lon): return np.c_[np.asarray(lon) * KX, np.asarray(lat) * KY]

p = d.dropna(subset=["lat"])
# 1) robust centreline: 0.01° latitude bins, median longitude, keep dense bins
# Corridor end points are taken from the data: median coords of records labelled OJOTA (Lagos end, 6.599N)
# and IWO ROAD (Ibadan end, 7.401N).
LAT0, LAT1 = 6.595, 7.405
A0, A1 = xy([6.5994], [3.3783])[0], xy([7.4014], [3.9422])[0]     # Ojota, Iwo Road (data medians)
U = (A1 - A0) / np.linalg.norm(A1 - A0); N = np.array([-U[1], U[0]])
TOT = np.linalg.norm(A1 - A0)
def along(P): return (P - A0) @ U, (P - A0) @ N
P_all = xy(p.lat, p.lon); t_all, o_all = along(P_all)
keep = (t_all >= -1) & (t_all <= TOT + 1)
t_all, o_all = t_all[keep], o_all[keep]
edges = np.arange(0, TOT + 1.0, 1.0)            # 1-km bins along the Ojota→Iwo Road axis
def fit(t, o):
    idx = np.digitize(t, edges)
    g = pd.DataFrame({"i": idx, "t": t, "o": o}).groupby("i").agg(t=("t", "median"), o=("o", "median"), n=("o", "size"))
    g = g[g.n >= 2].sort_values("t").reset_index(drop=True)
    rm = g.o.rolling(5, center=True, min_periods=1).median()
    g = g[(g.o - rm).abs() <= 1.2].reset_index(drop=True)            # kink removal (km)
    g["o"] = g.o.rolling(3, center=True, min_periods=1).mean()
    return g
g = fit(t_all, o_all)
for _ in range(3):
    ref = np.interp(t_all, g.t, g.o)
    m = np.abs(o_all - ref) <= 1.0
    g = fit(t_all[m], o_all[m])
g = pd.concat([pd.DataFrame({"t": [0.0], "o": [0.0]}), g[["t", "o"]], pd.DataFrame({"t": [TOT], "o": [0.0]})]).sort_values("t").drop_duplicates("t").reset_index(drop=True)
line = A0 + np.outer(g.t, U) + np.outer(g.o, N)
g["lon"], g["b"] = line[:, 0] / KX, line[:, 1] / KY
seg_len = np.linalg.norm(np.diff(line, axis=0), axis=1)
cum = np.r_[0, np.cumsum(seg_len)]

def project(P):
    A, B = line[:-1], line[1:]
    AB = B - A; L2 = (AB ** 2).sum(1)
    t = np.clip(((P[:, None, :] - A[None]) * AB[None]).sum(2) / L2[None], 0, 1)
    proj = A[None] + t[..., None] * AB[None]
    dist = np.linalg.norm(P[:, None, :] - proj, axis=2)
    k = dist.argmin(1)
    return dist[np.arange(len(P)), k], cum[k] + t[np.arange(len(P)), k] * seg_len[k]

has = d.lat.notna()
dist, ch = project(xy(d.loc[has, "lat"], d.loc[has, "lon"]))
d.loc[has, "dist_to_axis_km"] = dist
d.loc[has, "chainage_km"] = ch
d["on_corridor"] = (d.dist_to_axis_km <= 1.0) & (d.chainage_km > 0.05) & (d.chainage_km < cum[-1] - 0.05) | ((d.dist_to_axis_km <= 1.0) & d.lat.between(LAT0, LAT1))

# 2) validation against KM markers in location text (e.g., 'KM19-RCC QUARRY AREA', 'KM47')
km = d.location_raw.astype(str).str.upper().str.extract(r"\bKM\s*(\d{1,3})")[0].astype(float)
v = d.assign(km=km).dropna(subset=["km", "chainage_km"])
v = v[v.on_corridor & (v.coord_source.str.startswith("recorded"))]
r_all = v[["km", "chainage_km"]].corr(method="spearman").iloc[0, 1]
by_cmd = v.groupby("cmd_code").apply(lambda s: pd.Series({"n": len(s), "spearman": s[["km", "chainage_km"]].corr(method="spearman").iloc[0, 1]}), include_groups=False)

pd.DataFrame({"lat": g.b, "lon": g.lon, "chainage_km": cum}).to_csv(os.path.join(OUT, "centreline_derived.csv"), index=False)
d.to_csv(os.path.join(OUT, "rtc_lagos_ibadan_clean_2013_2022.csv"), index=False)
msg = [f"Centreline length (data-derived): {cum[-1]:.1f} km from lat {g.b.min():.3f} to {g.b.max():.3f}",
       f"Records with coords: {has.sum()}; on corridor (<=1.0 km from axis): {int(d.on_corridor.sum())}",
       f"KM-marker validation: n={len(v)}, Spearman(KM label, chainage) overall={r_all:.2f}",
       by_cmd[by_cmd.n >= 5].round(2).to_string()]
open(os.path.join(OUT, "corridor_log.txt"), "w").write("\n".join(msg)); print("\n".join(msg))
```


### 03_eda.py

```python
"""03_eda.py — Seaborn exploratory spatiotemporal figures. Usage: python3 -I 03_eda.py <out_dir>"""
import sys, os, json, numpy as np, pandas as pd
sys.path.insert(0, os.path.dirname(__file__))
import style; style.setup()
import matplotlib.pyplot as plt, seaborn as sns
from style import CAT, SEQ, INK2

OUT = sys.argv[1]; FIG = os.path.join(OUT, "figures"); os.makedirs(FIG, exist_ok=True)
d = pd.read_csv(os.path.join(OUT, "rtc_lagos_ibadan_clean_2013_2022.csv"), parse_dates=["date"])
c = pd.read_csv(os.path.join(OUT, "centreline_derived.csv"))
S = {}

# ---- Fig 1: annual small multiples (one axis each, no dual axes)
y = d.groupby("year").agg(crashes=("year", "size"), killed=("killed", "sum"), injured=("injured", "sum"), fatal=("fatal", "sum")).reset_index()
y["deaths_per_100"] = 100 * y.killed / y.crashes
S["annual"] = y.round(2).to_dict("records")
fig, axs = plt.subplots(1, 3, figsize=(13, 3.6))
for ax, col, t, colr in zip(axs, ["crashes", "killed", "deaths_per_100"], ["Recorded crashes", "Persons killed", "Deaths per 100 crashes"], [CAT[0], CAT[7], CAT[1]]):
    sns.lineplot(data=y, x="year", y=col, ax=ax, color=colr, marker="o", markersize=7)
    ax.set_title(t); ax.set_xlabel(""); ax.set_ylabel(""); ax.set_xticks(y.year[::3]); ax.set_ylim(0, None)
    for _, r in y.iloc[[0, -1]].iterrows(): ax.annotate(f"{r[col]:.0f}", (r.year, r[col]), textcoords="offset points", xytext=(0, 8), ha="center", fontsize=9, color=INK2)
fig.tight_layout(); fig.savefig(os.path.join(FIG, "fig01_annual_trends.png")); plt.close()

# ---- Fig 2: year × month heatmap
dd = d.dropna(subset=["date"]).copy(); dd["month"] = dd.date.dt.month
ym = dd.pivot_table(index="year", columns="month", values="fatal", aggfunc="size", fill_value=0)
ym.columns = ["Jan", "Feb", "Mar", "Apr", "May", "Jun", "Jul", "Aug", "Sep", "Oct", "Nov", "Dec"]
fig, ax = plt.subplots(figsize=(10, 4.6))
sns.heatmap(ym, cmap=SEQ, annot=True, fmt="d", linewidths=1.5, linecolor="#fcfcfb", cbar_kws={"label": "Crashes"}, ax=ax, annot_kws={"size": 8})
ax.set_title("Monthly crash counts by year"); ax.set_xlabel(""); ax.set_ylabel("")
fig.tight_layout(); fig.savefig(os.path.join(FIG, "fig02_year_month_heatmap.png")); plt.close()
mon = dd.groupby("month").size(); S["month_share"] = (mon / mon.sum() * 100).round(1).to_dict()
S["peak_months"] = mon.sort_values(ascending=False).head(3).index.tolist()

# ---- Fig 3: hour × weekday heatmap
dd["dow"] = dd.date.dt.day_name().str[:3]
order = ["Mon", "Tue", "Wed", "Thu", "Fri", "Sat", "Sun"]
hw = dd.dropna(subset=["hour"]).pivot_table(index="dow", columns="hour", values="fatal", aggfunc="size", fill_value=0).reindex(order)
fig, ax = plt.subplots(figsize=(12, 3.8))
sns.heatmap(hw, cmap=SEQ, linewidths=1, linecolor="#fcfcfb", cbar_kws={"label": "Crashes"}, ax=ax)
ax.set_title("When crashes happen: hour of day × day of week (2013–2022)"); ax.set_xlabel("Hour of day"); ax.set_ylabel("")
ax.set_xticklabels([int(float(t.get_text())) for t in ax.get_xticklabels()], rotation=0)
fig.tight_layout(); fig.savefig(os.path.join(FIG, "fig03_hour_weekday_heatmap.png")); plt.close()
hr = dd.dropna(subset=["hour"]).groupby("hour").size()
S["hour_peak"] = int(hr.idxmax()); S["share_06_20"] = round(float(hr.loc[6:19].sum() / hr.sum() * 100), 1)
S["dow_counts"] = dd.groupby("dow").size().reindex(order).to_dict()
# night vs day fatality
dd["period"] = np.where(dd.hour.between(6, 18), "Day (06–18h)", "Night (19–05h)")
S["fatal_rate_by_period"] = (dd.dropna(subset=["hour"]).groupby("period").fatal.mean() * 100).round(1).to_dict()

# ---- Fig 4: causes & vehicle (counts + fatal share)
fig, axs = plt.subplots(1, 2, figsize=(13, 4.4))
for ax, col, t in [(axs[0], "cause", "Probable cause"), (axs[1], "vehicle_type", "Vehicle type (first-listed)")]:
    t2 = d.groupby(col).agg(n=("fatal", "size"), fatal_pct=("fatal", "mean")).sort_values("n", ascending=False)
    t2 = t2[t2.n >= 10]; t2["fatal_pct"] *= 100
    sns.barplot(x=t2.n, y=t2.index, ax=ax, color=CAT[0], width=0.6)
    for i, (n, fp) in enumerate(zip(t2.n, t2.fatal_pct)):
        ax.text(n + t2.n.max() * 0.01, i, f"{n:,}  ·  {fp:.0f}% fatal", va="center", fontsize=8.5, color=INK2)
    ax.set_title(t); ax.set_xlabel("Crashes"); ax.set_ylabel(""); ax.set_xlim(0, t2.n.max() * 1.35)
    S[f"{col}_table"] = t2.round(1).reset_index().to_dict("records")
fig.tight_layout(); fig.savefig(os.path.join(FIG, "fig04_cause_vehicle.png")); plt.close()

# ---- Fig 5: KDE hotspot map (corridor records)
p = d[d.on_corridor]
fig, axs = plt.subplots(1, 2, figsize=(11, 8), sharey=True)
for ax, (lab, sub) in zip(axs, [("2013–2017", p[p.year <= 2017]), ("2018–2022", p[p.year >= 2018])]):
    ax.plot(c.lon, c.lat, color="#c3c2b7", lw=3, zorder=1)
    sns.kdeplot(x=sub.lon, y=sub.lat, fill=True, cmap=SEQ, levels=12, thresh=0.04, bw_adjust=0.35, ax=ax, zorder=2, alpha=.9)
    ax.scatter(sub.lon, sub.lat, s=4, color="#0b0b0b", alpha=.25, zorder=3, linewidths=0)
    ax.set_aspect("equal"); ax.set_title(f"Crash density (KDE), {lab}  (n={len(sub):,})"); ax.set_xlabel("Longitude"); ax.set_ylabel("Latitude")
    for nm, la, lo in [("Ojota (Lagos)", 6.5994, 3.3783), ("Iwo Road (Ibadan)", 7.4014, 3.9422)]:
        ax.annotate(nm, (lo, la), xytext=(8, 0), textcoords="offset points", fontsize=8.5, color=INK2, va="center")
fig.tight_layout(); fig.savefig(os.path.join(FIG, "fig05_kde_hotspot_map.png")); plt.close()

# ---- Fig 6: chainage × year space–time heatmap (2-km)
p = p.assign(seg=(p.chainage_km // 2 * 2).astype(int))
st = p.pivot_table(index="seg", columns="year", values="fatal", aggfunc="size", fill_value=0)
st = st.reindex(range(0, int(p.chainage_km.max()) + 2, 2), fill_value=0)
fig, ax = plt.subplots(figsize=(8, 11))
sns.heatmap(st, cmap=SEQ, linewidths=.4, linecolor="#fcfcfb", cbar_kws={"label": "Crashes per 2-km segment"}, ax=ax)
ax.set_title("Space–time crash matrix (km from Ojota, Lagos)"); ax.set_ylabel("Chainage (km), Lagos → Ibadan"); ax.set_xlabel("")
ax.set_yticks(np.arange(0, len(st), 5) + .5); ax.set_yticklabels(st.index[::5])
fig.tight_layout(); fig.savefig(os.path.join(FIG, "fig06_space_time_matrix.png")); plt.close()

# ---- Fig 7: direction & ownership by year (small)
S["direction"] = d.direction.value_counts().to_dict(); S["ownership"] = d.ownership.value_counts().to_dict()
S["totals"] = {"records": int(len(d)), "killed": float(d.killed.sum()), "injured": float(d.injured.sum()),
               "fatal_crashes": int(d.fatal.sum()), "on_corridor": int(d.on_corridor.sum()),
               "with_date": int(d.date.notna().sum()), "with_hour": int(d.hour.notna().sum())}
json.dump(S, open(os.path.join(OUT, "eda_summary.json"), "w"), indent=1, default=str)
print(json.dumps(S, indent=1, default=str)[:3500])
```


### 04_hotspots.py

```python
"""04_hotspots.py — Getis-Ord Gi* on road segments, emerging-hotspot trend, DBSCAN clusters, sensitivity.
Usage: python3 -I 04_hotspots.py <out_dir>"""
import sys, os, json, numpy as np, pandas as pd
from scipy import stats
from sklearn.cluster import DBSCAN
sys.path.insert(0, os.path.dirname(__file__))
import style; style.setup()
import matplotlib.pyplot as plt, seaborn as sns
from matplotlib.patches import Patch
from style import CAT, INK2

OUT = sys.argv[1]; FIG = os.path.join(OUT, "figures")
d = pd.read_csv(os.path.join(OUT, "rtc_lagos_ibadan_clean_2013_2022.csv"), parse_dates=["date"])
c = pd.read_csv(os.path.join(OUT, "centreline_derived.csv"))
L = c.chainage_km.max()
p = d[d.on_corridor].copy()
# placeholder-coordinate flag: same coordinate pair reused for >=5 distinct location names
p["ck"] = p.lat.round(4).astype(str) + "," + p.lon.round(4).astype(str)
nn = d.assign(ck=d.lat.round(4).astype(str) + "," + d.lon.round(4).astype(str)).groupby("ck").loc_norm.nunique()
p["placeholder"] = p.ck.map(nn).fillna(0) >= 5

def gistar(x, band):
    """Getis-Ord Gi* for a 1-D chain of segments; binary weights within `band` segments (self included)."""
    x = np.asarray(x, float); n = len(x); xb = x.mean(); s = np.sqrt((x ** 2).mean() - xb ** 2)
    z = np.zeros(n)
    for i in range(n):
        j = np.arange(max(0, i - band), min(n, i + band + 1)); w = len(j)
        num = x[j].sum() - xb * w
        den = s * np.sqrt((n * w - w ** 2) / (n - 1))
        z[i] = num / den if den > 0 else 0
    return z

def bh(pv, q=0.05):
    pv = np.asarray(pv); o = np.argsort(pv); m = len(pv)
    thr = (np.arange(1, m + 1) / m) * q; ok = pv[o] <= thr
    k = np.where(ok)[0].max() + 1 if ok.any() else 0
    r = np.zeros(m, bool); r[o[:k]] = True; return r

def classify(z):
    return np.select([z >= 2.576, z >= 1.960, z >= 1.645, z <= -1.960], ["Hot 99%", "Hot 95%", "Hot 90%", "Cold 95%"], "Not significant")

def segment_table(df, size, weight=None):
    seg = (df.chainage_km // size).astype(int)
    nseg = int(np.ceil(L / size))
    cnt = (df.assign(seg=seg).groupby("seg")[weight].sum() if weight else seg.value_counts()).reindex(range(nseg), fill_value=0)
    t = pd.DataFrame({"seg": range(nseg), "from_km": np.arange(nseg) * size, "to_km": np.arange(1, nseg + 1) * size, "n": cnt.values})
    t["z"] = gistar(t.n, band=1); t["p"] = 2 * stats.norm.sf(np.abs(t.z)); t["class"] = classify(t.z)
    t["hot_fdr"] = bh(t.p) & (t.z > 0)
    return t

R = {}
# ---------- primary: 2-km, all crashes
t2 = segment_table(p, 2)
# Dominant place names per segment (for interpretation)
p["seg2"] = (p.chainage_km // 2).astype(int)
names = p.groupby("seg2").location_raw.agg(lambda s: ", ".join(s.astype(str).str.upper().str.strip().value_counts().head(3).index))
t2["top_location_names"] = t2.seg.map(names).fillna("")
fat = p.groupby("seg2").agg(killed=("killed", "sum"), fatal=("fatal", "sum"))
t2 = t2.join(fat, on="seg").fillna({"killed": 0, "fatal": 0})
t2["placeholder_share"] = t2.seg.map(p.groupby("seg2").placeholder.mean()).fillna(0).round(2)
t2.to_csv(os.path.join(OUT, "hotspots_gistar_2km.csv"), index=False)
R["gi_2km_hot"] = t2[t2.z >= 1.96].sort_values("z", ascending=False)[["from_km", "to_km", "n", "killed", "z", "class", "hot_fdr", "placeholder_share", "top_location_names"]].round(2).to_dict("records")

# ---------- inter-urban section (>= 20 km) Gi*, so the dense Lagos urban stretch does not mask rural hotspots
pr = p[p.chainage_km >= 20]
tr = segment_table(pr.assign(chainage_km=pr.chainage_km), 2)
tr = tr[tr.from_km >= 20].copy()
tr["z"] = gistar(tr.n.values, 1); tr["p"] = 2 * stats.norm.sf(np.abs(tr.z)); tr["class"] = classify(tr.z); tr["hot_fdr"] = bh(tr.p) & (tr.z > 0)
tr["top_location_names"] = tr.seg.map(names).fillna("")
tr["killed"] = tr.seg.map(fat.killed).fillna(0)
tr.to_csv(os.path.join(OUT, "hotspots_gistar_2km_interurban.csv"), index=False)
R["gi_interurban_hot"] = tr[tr.z >= 1.96].sort_values("z", ascending=False)[["from_km", "to_km", "n", "killed", "z", "class", "hot_fdr", "top_location_names"]].round(2).to_dict("records")

# ---------- sensitivity
t5 = segment_table(p, 5); t5.to_csv(os.path.join(OUT, "hotspots_gistar_5km.csv"), index=False)
R["gi_5km_hot"] = t5[t5.z >= 1.96][["from_km", "to_km", "n", "z", "class"]].round(2).to_dict("records")
tnp = segment_table(p[~p.placeholder], 2)
tk = segment_table(p, 2, weight="killed")
R["sens"] = {
    "placeholder_records_on_corridor": int(p.placeholder.sum()), "on_corridor": int(len(p)),
    "hot95_2km_all": t2.loc[t2.z >= 1.96, "from_km"].tolist(),
    "hot95_2km_no_placeholder": tnp.loc[tnp.z >= 1.96, "from_km"].tolist(),
    "hot95_2km_killed_weighted": tk.loc[tk.z >= 1.96, "from_km"].tolist(),
    "spearman_z_all_vs_noplaceholder": round(stats.spearmanr(t2.z, tnp.z)[0], 2),
    "spearman_z_count_vs_killed": round(stats.spearmanr(t2.z, tk.z)[0], 2)}

# ---------- emerging hotspot (simplified): yearly Gi* z + Mann–Kendall (Kendall tau vs year)
years = sorted(p.year.unique())
Z = pd.DataFrame({yr: segment_table(p[p.year == yr], 2).z.values for yr in years})
def emerge(row, hot_all):
    tau, pv = stats.kendalltau(years, row.values)
    nhot = int((row.values >= 1.96).sum()); last = row.values[-1] >= 1.96
    if not hot_all and nhot == 0: return "No pattern", tau, pv
    if nhot >= 8: lab = "Persistent hotspot"
    elif last and nhot <= 2 and not (row.values[:-2] >= 1.96).any(): lab = "New hotspot"
    elif nhot >= 5: lab = "Consecutive/frequent hotspot"
    elif nhot >= 1: lab = "Sporadic hotspot"
    else: lab = "No pattern"
    if lab != "No pattern" and pv < 0.05: lab = ("Intensifying " if tau > 0 else "Diminishing ") + lab.split(" ")[0].lower() + " hotspot"
    return lab, tau, pv
em = [emerge(Z.iloc[i], t2.z.iloc[i] >= 1.96) for i in range(len(Z))]
t2["emerging"] = [e[0] for e in em]; t2["mk_tau"] = [round(e[1], 2) if not np.isnan(e[1]) else 0 for e in em]; t2["mk_p"] = [round(e[2], 3) if not np.isnan(e[2]) else 1 for e in em]
t2["years_hot"] = (Z.values >= 1.96).sum(1)
t2.to_csv(os.path.join(OUT, "hotspots_gistar_2km.csv"), index=False)
R["emerging_counts"] = t2.emerging.value_counts().to_dict()

# ---------- DBSCAN point clusters (haversine, eps=400 m, min 25 crashes)
X = np.deg2rad(p[["lat", "lon"]].values)
lab = DBSCAN(eps=0.4 / 6371.0, min_samples=25, metric="haversine").fit_predict(X)
p["cluster"] = lab
cl = p[p.cluster >= 0].groupby("cluster").agg(n=("lat", "size"), killed=("killed", "sum"), lat=("lat", "median"), lon=("lon", "median"),
     km=("chainage_km", "median"), placeholder_share=("placeholder", "mean"),
     names=("location_raw", lambda s: ", ".join(s.astype(str).str.upper().str.strip().value_counts().head(3).index))).sort_values("n", ascending=False)
cl.round(3).to_csv(os.path.join(OUT, "hotspots_dbscan_clusters.csv"))
R["dbscan"] = {"n_clusters": int(cl.shape[0]), "share_in_clusters": round(float((p.cluster >= 0).mean() * 100), 1),
               "top": cl.head(10).round(3).reset_index().to_dict("records")}

# ---------- FIGURES
order = ["Hot 99%", "Hot 95%", "Hot 90%", "Not significant", "Cold 95%"]
pal = {"Hot 99%": "#d03b3b", "Hot 95%": "#ec835a", "Hot 90%": "#f4b393", "Not significant": "#d6d5cf", "Cold 95%": "#6da7ec"}
fig, axs = plt.subplots(2, 1, figsize=(13, 7.6))
for ax, tt, ttl in [(axs[0], t2, "a) Whole corridor, 2-km segments"), (axs[1], tr, "b) Inter-urban section only (≥ 20 km), Gi* recomputed")]:
    ax.bar(tt.from_km + 1, tt.n, width=1.7, color=tt["class"].map(pal), edgecolor="none")
    for _, r in tt[(tt.z >= 1.96) & (tt.n >= 50)].iterrows():
        nm = r.top_location_names.split(",")[0][:20]
        ax.annotate(nm, (r.from_km + 1, r.n), xytext=(0, 4), textcoords="offset points", ha="center", fontsize=7.5, color=INK2, rotation=90, va="bottom")
    ax.set_xlim(0, L); ax.set_ylim(0, tt.n.max() * 1.6); ax.set_ylabel("Crashes 2013–2022"); ax.set_title(ttl)
axs[1].set_xlabel("Chainage (km from Ojota, Lagos → Iwo Road, Ibadan)")
axs[0].legend(handles=[Patch(color=pal[k], label=k) for k in order], ncol=5, loc="upper right", fontsize=8.5)
fig.tight_layout(); fig.savefig(os.path.join(FIG, "fig07_gistar_profile.png")); plt.close()

# map of segments coloured by class + DBSCAN clusters
mid = (t2.from_km + 1).values
seg_lat = np.interp(mid, c.chainage_km, c.lat); seg_lon = np.interp(mid, c.chainage_km, c.lon)
fig, axs = plt.subplots(1, 2, figsize=(11, 8), sharey=True)
ax = axs[0]; ax.plot(c.lon, c.lat, color="#d6d5cf", lw=6, zorder=1, solid_capstyle="round")
for k in order[::-1]:
    m = t2["class"].values == k
    ax.scatter(seg_lon[m], seg_lat[m], s=np.where(k.startswith("Hot"), 90, 25), color=pal[k], label=k, zorder=3, edgecolor="#fcfcfb", linewidth=1)
ri = tr[tr.z >= 1.96]
ax.scatter(np.interp(ri.from_km + 1, c.chainage_km, c.lon), np.interp(ri.from_km + 1, c.chainage_km, c.lat), s=160, facecolor="none", edgecolor="#d03b3b", linewidth=1.8, zorder=4, label="Inter-urban hot (≥95%)")
ax.set_aspect("equal"); ax.set_title("Gi* class per 2-km segment"); ax.legend(loc="lower right", fontsize=8.5); ax.set_xlabel("Longitude"); ax.set_ylabel("Latitude")
ax = axs[1]; ax.plot(c.lon, c.lat, color="#d6d5cf", lw=6, zorder=1, solid_capstyle="round")
ax.scatter(p.lon, p.lat, s=3, color="#a8a79f", zorder=2, linewidths=0)
top = cl.head(12)
ax.scatter(top.lon, top.lat, s=top.n * 1.2, color=CAT[0], alpha=.55, edgecolor=CAT[0], zorder=3)
for i, (_, r) in enumerate(top.head(8).iterrows()):
    ax.annotate(f"{r.names.split(',')[0][:20]} ({r.n})", (r.lon, r.lat), xytext=(14, [-10, 4, -2, 8, 0, 6, -6, 2][i] * (1 if r.lat > 6.7 else 1.6)), textcoords="offset points", fontsize=7.5, color=INK2)
ax.set_aspect("equal"); ax.set_title("DBSCAN crash clusters (ε = 400 m, ≥25 crashes)"); ax.set_xlabel("Longitude")
fig.tight_layout(); fig.savefig(os.path.join(FIG, "fig08_hotspot_maps.png")); plt.close()

# emerging hotspot heat: yearly Gi* z
fig, ax = plt.subplots(figsize=(8, 10))
Zs = Z.copy(); Zs.index = t2.from_km
sns.heatmap(Zs, cmap=style.DIV, center=0, vmin=-3, vmax=6, linewidths=.3, linecolor="#fcfcfb", cbar_kws={"label": "Gi* z-score"}, ax=ax)
ax.set_yticks(np.arange(0, len(Zs), 5) + .5); ax.set_yticklabels(Zs.index[::5]); ax.set_ylabel("Chainage (km)")
ax.set_title("Year-by-year Gi* z-scores (emerging-hotspot view)")
fig.tight_layout(); fig.savefig(os.path.join(FIG, "fig09_emerging_gistar.png")); plt.close()

json.dump(R, open(os.path.join(OUT, "hotspot_summary.json"), "w"), indent=1, default=str)
print(json.dumps(R, indent=1, default=str)[:6000])
print(t2[t2.z >= 1.96][["from_km", "n", "z", "class", "emerging", "years_hot", "mk_tau", "mk_p"]].to_string())
```


### sankey_cause_vehicle.py

```python
"""sankey_cause_vehicle.py — Three-stage Sankey (alluvial) diagram: probable cause → vehicle type → crash outcome.
Built from the cleaned FRSC records (duplicates removed). Usage: python3 -I sankey_cause_vehicle.py <out_dir>"""
import sys, os, json, numpy as np, pandas as pd
sys.path.insert(0, os.path.dirname(__file__))
import style; style.setup()
import matplotlib.pyplot as plt
from matplotlib.path import Path
from matplotlib.patches import PathPatch, Rectangle
OUT = sys.argv[1]
d = pd.read_csv(os.path.join(OUT, "rtc_lagos_ibadan_clean_2013_2022.csv"), low_memory=False)

cause_map = {"Speeding": "Speeding", "Other": "Other / unspecified", "Brake failure": "Brake failure", "Tyre burst": "Tyre burst",
             "Mechanical deficiency": "Mechanical deficiency", "Dangerous driving/overtaking": "Dangerous driving / overtaking",
             "Fatigue": "Fatigue, loss of control, DUI", "Loss of control": "Fatigue, loss of control, DUI", "DUI": "Fatigue, loss of control, DUI"}
veh_map = {"Tricycle": "Other / tricycle", "Other/unknown": "Other / tricycle"}
d["C"] = d.cause.map(cause_map).fillna("Other / unspecified")
d["V"] = d.vehicle_type.replace(veh_map).fillna("Other / tricycle")
d["S"] = np.select([d.killed > 0, d.injured > 0], ["Fatal", "Injury (no deaths)"], "No injury")
N = len(d)
C_order = d.C.value_counts().index.tolist()
V_order = d.V.value_counts().index.tolist()
S_order = ["Fatal", "Injury (no deaths)", "No injury"]
CAUSE_COL = dict(zip(C_order, ["#2a78d6", "#a8a79f", "#eb6834", "#1baf7a", "#eda100", "#e87ba4", "#6250d6"]))
SEV_COL = {"Fatal": "#b3261e", "Injury (no deaths)": "#ec835a", "No injury": "#c9c8c1"}

GAP = 0.012 * N; W = 0.035
def stack(order, counts):
    y, pos = 0.0, {}
    for k in order:
        pos[k] = [y, y + counts[k]]; y += counts[k] + GAP
    return pos, y - GAP
cC, cV, cS = d.C.value_counts(), d.V.value_counts(), d.S.value_counts()
pC, HC = stack(C_order, cC); pV, HV = stack(V_order, cV); pS, HS = stack(S_order, cS)
Hmax = max(HC, HV, HS)
for pos, H in [(pC, HC), (pV, HV), (pS, HS)]:          # centre each column vertically
    off = (Hmax - H) / 2
    for k in pos: pos[k] = [pos[k][0] + off, pos[k][1] + off]
X = {"C": 0.0, "V": 1.0, "S": 2.0}

def band(ax, x0, y0a, y0b, x1, y1a, y1b, col, alpha=.55):
    xm = (x0 + x1) / 2
    verts = [(x0, y0a), (xm, y0a), (xm, y1a), (x1, y1a), (x1, y1b), (xm, y1b), (xm, y0b), (x0, y0b), (x0, y0a)]
    codes = [Path.MOVETO, Path.CURVE4, Path.CURVE4, Path.CURVE4, Path.LINETO, Path.CURVE4, Path.CURVE4, Path.CURVE4, Path.CLOSEPOLY]
    ax.add_patch(PathPatch(Path(verts, codes), fc=col, ec="none", alpha=alpha, zorder=1))

fig, ax = plt.subplots(figsize=(13.5, 9.2))
# flows cause → vehicle (coloured by cause)
f1 = d.groupby(["C", "V"]).size()
outC = {k: pC[k][0] for k in C_order}; inV = {k: pV[k][0] for k in V_order}
for v in V_order:                                      # fill vehicle nodes in cause order for tidy bands
    for c in C_order:
        n = f1.get((c, v), 0)
        if n == 0: continue
        band(ax, X["C"] + W, outC[c], outC[c] + n, X["V"] - W / 2, inV[v], inV[v] + n, CAUSE_COL[c]); outC[c] += n; inV[v] += n
# flows vehicle → outcome (coloured by outcome)
f2 = d.groupby(["V", "S"]).size()
outV = {k: pV[k][0] for k in V_order}; inS = {k: pS[k][0] for k in S_order}
for s in S_order:
    for v in V_order:
        n = f2.get((v, s), 0)
        if n == 0: continue
        band(ax, X["V"] + W / 2, outV[v], outV[v] + n, X["S"] - W, inS[s], inS[s] + n, SEV_COL[s], alpha=.5); outV[v] += n; inS[s] += n
# nodes + labels
def node(x, pos, k, col, side, text):
    y0, y1 = pos[k]; ax.add_patch(Rectangle((x - (W if side == "L" else 0 if side == "R" else W / 2), y0), W, y1 - y0, fc=col, ec="#fcfcfb", lw=.8, zorder=3))
    xt = x - W - 0.015 if side == "L" else (x + W + 0.015 if side == "R" else x + W / 2 + 0.012)
    ax.text(xt, (y0 + y1) / 2, text, ha="right" if side == "L" else "left", va="center", fontsize=9, zorder=4,
            bbox=dict(boxstyle="round,pad=0.15", fc="#fcfcfb", ec="none", alpha=.85) if side == "M" else None)
for c in C_order:
    node(X["C"] + W, pC, c, CAUSE_COL[c], "L", f"{c}  {cC[c]:,} ({100 * cC[c] / N:.1f}%)")
fatal_by_v = d.groupby("V").apply(lambda s: (s.S == "Fatal").mean(), include_groups=False)
for v in V_order:
    node(X["V"], pV, v, "#52514e", "M", f"{v}  {cV[v]:,} · {100 * fatal_by_v[v]:.0f}% fatal")
for s in S_order:
    node(X["S"] - W, pS, s, SEV_COL[s], "R", f"{s}  {cS[s]:,} ({100 * cS[s] / N:.1f}%)")
for x, t in [(X["C"], "Probable cause"), (X["V"], "Vehicle type (first-listed)"), (X["S"], "Crash outcome")]:
    ax.text(x, -0.025 * N, t, ha="center", va="bottom", fontsize=11, weight="bold", color="#0b0b0b")
ax.set_xlim(-0.75, 2.6); ax.set_ylim(Hmax + 0.05 * N, -0.05 * N); ax.axis("off")
ax.set_title(f"Probable cause → vehicle type → outcome, Lagos–Ibadan Expressway crashes 2013–2022 (n = {N:,})", loc="left", fontsize=12.5, weight="bold")
ax.text(-0.75, Hmax + 0.035 * N, "Band width ∝ number of crashes. Left bands coloured by cause; right bands by outcome. Source: FRSC crash returns (duplicates removed).",
        fontsize=8.5, color="#52514e")
fig.tight_layout(); fig.savefig(os.path.join(OUT, "figures", "fig04_sankey_cause_vehicle_outcome.png"), dpi=200); plt.close()

# summary numbers for the text
ct = pd.crosstab(d.C, d.V)
fat_c = d.groupby("C").apply(lambda s: (s.S == "Fatal").mean() * 100, include_groups=False).round(1)
share_bus_speed = 100 * ct.loc["Speeding", "Bus"] / cV["Bus"]
top_pairs = d.groupby(["C", "V"]).size().sort_values(ascending=False).head(6)
mech = d[d.C.isin(["Brake failure", "Tyre burst", "Mechanical deficiency"])].V.value_counts(normalize=True).mul(100).round(1)
S = {"N": N, "cause": cC.to_dict(), "veh": cV.to_dict(), "sev": cS.to_dict(), "fatal_by_cause": fat_c.to_dict(),
     "fatal_by_veh": (fatal_by_v * 100).round(1).to_dict(), "top_pairs": {f"{a} | {b}": int(n) for (a, b), n in top_pairs.items()},
     "speed_share_of_bus": round(share_bus_speed, 1), "mech_vehicle_mix": mech.head(4).to_dict(),
     "speed_share_by_veh": (100 * ct.loc["Speeding"] / cV).round(1).to_dict()}
json.dump(S, open(os.path.join(OUT, "sankey_summary.json"), "w"), indent=1, default=float); print(json.dumps(S, indent=1, default=float))
```


### fig01_annual_panel.py

```python
"""fig01_annual_panel.py — Redesigned Figure 1: annual crashes, casualties and severity, 2013–2022 (4 panels).
Usage: python3 -I fig01_annual_panel.py <out_dir>"""
import sys, os, json, numpy as np, pandas as pd
sys.path.insert(0, os.path.dirname(__file__))
import style; style.setup()
import matplotlib.pyplot as plt
from matplotlib.patches import Patch
from matplotlib.lines import Line2D
OUT = sys.argv[1]
d = pd.read_csv(os.path.join(OUT, "rtc_lagos_ibadan_clean_2013_2022.csv"), low_memory=False)
INK, INK2, GRID = "#0b0b0b", "#52514e", "#e4e3df"
FAT, INJ, NON = "#b3261e", "#ec835a", "#c9c8c1"
BLUE, RED = "#2a78d6", "#c0392b"
ct = pd.crosstab(d.year, d.severity).reindex(columns=["Fatal", "Serious/Minor injury", "No injury"])
yr = ct.index.values; n = ct.sum(1).values
g = d.groupby("year").agg(killed=("killed", "sum"), injured=("injured", "sum"))
dp100 = 100 * g.killed.values / n; fpct = 100 * ct["Fatal"].values / n; npct = 100 * ct["No injury"].values / n

fig = plt.figure(figsize=(14, 9.4))
gs = fig.add_gridspec(2, 2, hspace=0.42, wspace=0.22)

# (a) stacked crashes by outcome
ax = fig.add_subplot(gs[0, 0]); bot = np.zeros(len(yr))
for col, c, lab in [("Fatal", FAT, "Fatal"), ("Serious/Minor injury", INJ, "Injury only"), ("No injury", NON, "No injury")]:
    ax.bar(yr, ct[col].values, bottom=bot, color=c, width=.72, edgecolor="#fcfcfb", lw=.8, label=lab); bot += ct[col].values
for x, t in zip(yr, n): ax.text(x, t + 12, f"{t:,}", ha="center", va="bottom", fontsize=8.5, color=INK, weight="bold" if x in (2013, 2021, 2022) else "normal")
ax.set_ylim(0, n.max() * 1.15); ax.set_xticks(yr); ax.set_xticklabels([f"'{str(v)[2:]}" for v in yr]); ax.set_ylabel("Recorded crashes")
ax.legend(loc="upper left", fontsize=8.5, frameon=False); ax.grid(axis="x", visible=False)
ax.set_title("(a) Recorded crashes tripled to a 2021 peak")

# (b) persons killed and injured (two series, one axis)
ax = fig.add_subplot(gs[0, 1])
ax.fill_between(yr, g.injured.values, color=INJ, alpha=.18, lw=0); ax.plot(yr, g.injured.values, color=INJ, lw=2.4, marker="o", ms=6, label="Injured")
ax.fill_between(yr, g.killed.values, color=FAT, alpha=.25, lw=0); ax.plot(yr, g.killed.values, color=FAT, lw=2.4, marker="o", ms=6, label="Killed")
for s, c in [(g.injured, INJ), (g.killed, FAT)]:
    for x in (2013, 2021, 2022): ax.annotate(f"{int(s[x]):,}", (x, s[x]), xytext=(0, 8), textcoords="offset points", ha="center", fontsize=8.5, color=c, weight="bold")
ax.set_xticks(yr); ax.set_xticklabels([f"'{str(v)[2:]}" for v in yr]); ax.set_ylim(0, g.injured.max() * 1.15); ax.set_ylabel("Persons")
ax.legend(loc="upper left", fontsize=8.5, frameon=False); ax.set_title("(b) Casualties rose far more slowly than crashes")

# (c) index 2013 = 100
ax = fig.add_subplot(gs[1, 0])
ic, ik = 100 * n / n[0], 100 * g.killed.values / g.killed.values[0]
ax.fill_between(yr, ik, ic, where=ic >= ik, color=BLUE, alpha=.10, lw=0, interpolate=True)
ax.plot(yr, ic, color=BLUE, lw=2.6, marker="o", ms=6); ax.plot(yr, ik, color=FAT, lw=2.6, marker="o", ms=6)
ax.axhline(100, color=INK2, lw=.8, ls="--")
ax.text(2022.15, ic[-1], f"Crashes\n{ic[-1]:.0f}", color=BLUE, va="center", fontsize=9, weight="bold")
ax.text(2022.15, ik[-1], f"Deaths\n{ik[-1]:.0f}", color=FAT, va="center", fontsize=9, weight="bold")
ax.text(2020.35, (ic[7] + ik[7]) / 2 - 12, "gap = crashes\nwithout deaths", ha="center", fontsize=8.5, color=BLUE, style="italic")
ax.set_xlim(2012.6, 2023.3); ax.set_xticks(yr); ax.set_xticklabels([f"'{str(v)[2:]}" for v in yr]); ax.set_ylim(0, ic.max() * 1.12)
ax.set_ylabel("Index (2013 = 100)"); ax.set_title("(c) Crashes and deaths diverge after 2017")

# (d) severity ratios
ax = fig.add_subplot(gs[1, 1])
ax.plot(yr, dp100, color=FAT, lw=2.6, marker="o", ms=6); ax.plot(yr, fpct, color=INJ, lw=2.6, marker="s", ms=6); ax.plot(yr, npct, color="#8a8983", lw=2.2, marker="^", ms=6, ls="--")
for arr, c, dy in [(dp100, FAT, 0), (fpct, INJ, -7), (npct, "#6b6a66", 7)]:
    ax.annotate(f"{arr[0]:.1f}", (yr[0], arr[0]), xytext=(-6, 0), textcoords="offset points", ha="right", va="center", fontsize=8.5, color=c, weight="bold")
    ax.annotate(f"{arr[-1]:.1f}", (yr[-1], arr[-1]), xytext=(6, dy), textcoords="offset points", ha="left", va="center", fontsize=8.5, color=c, weight="bold")
ax.set_xlim(2012.2, 2022.9); ax.set_xticks(yr); ax.set_xticklabels([f"'{str(v)[2:]}" for v in yr]); ax.set_ylim(0, 85)
ax.legend(handles=[Line2D([], [], color=FAT, marker="o", lw=2.4, label="Deaths per 100 crashes"), Line2D([], [], color=INJ, marker="s", lw=2.4, label="% of crashes that were fatal"),
                   Line2D([], [], color="#8a8983", marker="^", lw=2.2, ls="--", label="% of crashes with no injury")], loc="upper right", fontsize=8.5, frameon=True, facecolor="#fcfcfb", edgecolor=GRID)
ax.set_ylabel("Rate"); ax.set_title("(d) Each crash became less deadly")

fig.suptitle("Annual crash trends, Lagos–Ibadan Expressway, 2013–2022", x=0.01, ha="left", fontsize=15, weight="bold", y=0.995)
fig.text(0.01, 0.955, f"{n.sum():,} recorded crashes (duplicates removed), {int(g.killed.sum()):,} persons killed, {int(g.injured.sum()):,} injured. Source: FRSC crash returns.", fontsize=10, color=INK2)
fig.savefig(os.path.join(OUT, "figures", "fig01_annual_panel.png"), dpi=200, bbox_inches="tight"); plt.close()
S = {"n": n.tolist(), "killed": g.killed.astype(int).tolist(), "injured": g.injured.astype(int).tolist(), "dp100": np.round(dp100, 1).tolist(), "fatal_pct": np.round(fpct, 1).tolist(),
     "noinj_pct": np.round(npct, 1).tolist(), "idx_crash_2022": round(ic[-1]), "idx_killed_2022": round(ik[-1]), "idx_crash_2021": round(ic[-2]), "idx_killed_2021": round(ik[-2])}
print(json.dumps(S))
```


### report_maps_basemap.py

```python
"""report_maps_basemap.py — Redraw the report's two maps (Fig 5 KDE, Fig 8 Gi*/DBSCAN) on the Natural Earth basemap.
Same data as the original report (recorded coordinates, out/). Usage: python3 -I report_maps_basemap.py <out_dir> <dest_dir>"""
import sys, os, numpy as np, pandas as pd
sys.path.insert(0, os.path.dirname(__file__))
import style; style.setup()
import basemap_ne
import matplotlib.pyplot as plt, seaborn as sns
from matplotlib.patches import Patch
from matplotlib.lines import Line2D
from matplotlib.colors import LinearSegmentedColormap
OUT, DEST = sys.argv[1:3]; os.makedirs(DEST, exist_ok=True)
d = pd.read_csv(os.path.join(OUT, "rtc_lagos_ibadan_clean_2013_2022.csv"), low_memory=False)
cl = pd.read_csv(os.path.join(OUT, "centreline_derived.csv"))
t2 = pd.read_csv(os.path.join(OUT, "hotspots_gistar_2km.csv")); tr = pd.read_csv(os.path.join(OUT, "hotspots_gistar_2km_interurban.csv"))
cls = pd.read_csv(os.path.join(OUT, "hotspots_dbscan_clusters.csv"))
XL, YL = (3.20, 4.05), (6.40, 7.50); ASP = 1 / np.cos(np.deg2rad(6.9)); INK2 = "#52514e"
KDECM = LinearSegmentedColormap.from_list("kde", [(1.0, 0.85, 0.30, 0.0), (0.99, 0.80, 0.30, 0.55), (0.93, 0.41, 0.20, 0.8), (0.82, 0.23, 0.23, 0.92), (0.45, 0.05, 0.10, 0.97)])

def furniture(ax, credit):
    ax.set_xlim(*XL); ax.set_ylim(*YL); ax.set_aspect(ASP); ax.set_xlabel("Longitude (°E)"); ax.set_ylabel("Latitude (°N)")
    dlon = 20 / (111.32 * np.cos(np.deg2rad(6.9))); x0, y0 = XL[1] - 0.04 - dlon, YL[0] + 0.06
    ax.plot([x0, x0 + dlon], [y0, y0], color="#0b0b0b", lw=2.5, solid_capstyle="butt", zorder=7); ax.plot([x0, x0 + dlon / 2], [y0, y0], color="#fcfcfb", lw=1.2, solid_capstyle="butt", zorder=7)
    ax.text(x0, y0 + 0.012, "0", fontsize=7, ha="center", zorder=7); ax.text(x0 + dlon, y0 + 0.012, "20 km", fontsize=7, ha="center", zorder=7)
    ax.annotate("N", xy=(XL[0] + 0.05, YL[1] - 0.04), xytext=(XL[0] + 0.05, YL[1] - 0.12), ha="center", fontsize=10, weight="bold", arrowprops=dict(arrowstyle="-|>", color="#0b0b0b", lw=1.5), zorder=7)
    ax.text(XL[1] - 0.01, YL[0] + 0.01, credit + " · " + basemap_ne.CREDIT, fontsize=6.3, color="#3a3936", ha="right", va="bottom",
            bbox=dict(boxstyle="round,pad=0.2", fc="#fcfcfb", ec="none", alpha=.85), zorder=7)

def road(ax):
    ax.plot(cl.lon, cl.lat, color="#fcfcfb", lw=7.5, solid_capstyle="round", zorder=1, alpha=.9)
    ax.plot(cl.lon, cl.lat, color="#8a8983", lw=3.2, solid_capstyle="round", zorder=1.1)

# ---------- Figure 5: KDE (same data/periods as the report) ----------
p = d[d.on_corridor == True]
fig, axs = plt.subplots(1, 2, figsize=(14, 9.6), sharey=True)
for ax, (lab, sub) in zip(axs, [("2013–2017", p[p.year <= 2017]), ("2018–2022", p[p.year >= 2018])]):
    basemap_ne.draw(ax, extent=XL + YL); road(ax)
    sns.kdeplot(x=sub.lon, y=sub.lat, fill=True, cmap=KDECM, levels=14, thresh=0.04, bw_adjust=0.35, ax=ax, zorder=2)
    ax.scatter(sub.lon, sub.lat, s=3, color="#0b0b0b", alpha=.35, zorder=3, linewidths=0)
    ax.set_title(f"Crash density (KDE), {lab}  (n = {len(sub):,})"); furniture(ax, "Recorded FRSC coordinates")
fig.tight_layout(); fig.savefig(os.path.join(DEST, "fig05_kde_hotspot_map_basemap.png"), dpi=200); plt.close()

# ---------- Figure 8: Gi* classes + DBSCAN (same data as the report) ----------
order = ["Hot 99%", "Hot 95%", "Hot 90%", "Not significant", "Cold 95%"]
pal = {"Hot 99%": "#d03b3b", "Hot 95%": "#ec835a", "Hot 90%": "#f4b393", "Not significant": "#c9c8c1", "Cold 95%": "#6da7ec"}
fig, axs = plt.subplots(1, 2, figsize=(14, 9.6), sharey=True)
ax = axs[0]; basemap_ne.draw(ax, extent=XL + YL)
for _, r in t2.iterrows():
    a, b = r.from_km, r.to_km; ts = np.r_[a, cl.chainage_km[(cl.chainage_km > a) & (cl.chainage_km < b)].values, min(b, cl.chainage_km.max())]
    hot = str(r["class"]).startswith("Hot")
    ax.plot(np.interp(ts, cl.chainage_km, cl.lon), np.interp(ts, cl.chainage_km, cl.lat), color=pal.get(r["class"], "#c9c8c1"), lw=9 if hot else 4.5, solid_capstyle="butt", zorder=2)
for _, r in tr[tr.z >= 1.96].iterrows():
    mid = (r.from_km + r.to_km) / 2
    ax.scatter(np.interp(mid, cl.chainage_km, cl.lon), np.interp(mid, cl.chainage_km, cl.lat), s=230, facecolor="none", edgecolor="#7a1418", linewidth=2, zorder=3)
h = [Patch(color=pal[k], label=k) for k in order[:4]] + [Line2D([], [], marker="o", ls="", markerfacecolor="none", markeredgecolor="#7a1418", markeredgewidth=2, markersize=11, label="Inter-urban hotspot (≥ 95%)")]
ax.legend(handles=h, loc="center right", bbox_to_anchor=(0.995, 0.62), fontsize=8.5, frameon=True, facecolor="#fcfcfb", edgecolor="#c9c8c1", framealpha=.95)
ax.set_title("Gi* class per 2-km segment"); furniture(ax, "Recorded FRSC coordinates")
ax = axs[1]; basemap_ne.draw(ax, extent=XL + YL); road(ax)
on = d[d.on_corridor == True]; ax.scatter(on.lon, on.lat, s=3, color="#52514e", alpha=.35, zorder=2, linewidths=0)
top = cls.sort_values("n", ascending=False).head(12)
ax.scatter(top.lon, top.lat, s=top.n * 1.2, color="#2a78d6", alpha=.55, edgecolor="#184f95", zorder=3)
lab = top.head(8).sort_values("lat")
for i, (_, r) in enumerate(lab.iterrows()):
    side = 1 if i % 2 == 0 else -1
    ax.annotate(f"{str(r.names).split(',')[0].title()} ({int(r.n)})", (r.lon, r.lat), xytext=(22 * side, 0), textcoords="offset points", ha="left" if side > 0 else "right", va="center",
                fontsize=7.8, arrowprops=dict(arrowstyle="-", color="#8a8983", lw=.6), bbox=dict(boxstyle="round,pad=0.15", fc="#fcfcfb", ec="none", alpha=.9), zorder=5)
ax.set_title("DBSCAN crash clusters (ε = 400 m, ≥ 25 crashes)"); furniture(ax, "Recorded FRSC coordinates")
fig.tight_layout(); fig.savefig(os.path.join(DEST, "fig08_hotspot_maps_basemap.png"), dpi=200); plt.close()
print("ok")
```


---

## B. Machine learning and forecasts


### 05_ml.py

```python
"""05_ml.py — Machine-learning prediction of future crash occurrence.
A) Monthly corridor-wide forecasting (2023–2024)
B) Segment-level (2-km) hotspot forecasting for 2023
C) Fatal-crash severity classification + permutation importance
Usage: python3 -I 05_ml.py <out_dir>"""
import sys, os, json, warnings, numpy as np, pandas as pd
from scipy import stats
from sklearn.linear_model import PoissonRegressor, LogisticRegression
from sklearn.ensemble import RandomForestRegressor, HistGradientBoostingRegressor, HistGradientBoostingClassifier
from sklearn.model_selection import StratifiedKFold, cross_val_predict
from sklearn.metrics import mean_absolute_error, mean_squared_error, roc_auc_score, average_precision_score, brier_score_loss
from sklearn.inspection import permutation_importance
from sklearn.preprocessing import OneHotEncoder, StandardScaler
from sklearn.compose import ColumnTransformer
from sklearn.pipeline import make_pipeline
warnings.filterwarnings("ignore")
sys.path.insert(0, os.path.dirname(__file__))
import style; style.setup()
import matplotlib.pyplot as plt, seaborn as sns
from style import CAT, INK2, SEQ

OUT = sys.argv[1]; FIG = os.path.join(OUT, "figures"); SEED = 42
d = pd.read_csv(os.path.join(OUT, "rtc_lagos_ibadan_clean_2013_2022.csv"), parse_dates=["date"])
R = {}

# ===================== A) MONTHLY FORECAST =====================
m = d.dropna(subset=["date"]).set_index("date").resample("MS").size().rename("n").to_frame()
m = m.reindex(pd.date_range("2013-01-01", "2022-12-01", freq="MS"), fill_value=0)
m["t"] = np.arange(len(m)); m["month"] = m.index.month

def design(df):
    X = pd.get_dummies(df["month"].astype(str), prefix="m").reindex(columns=[f"m_{i}" for i in range(1, 13)], fill_value=0).astype(float)
    X["t"] = df["t"] / 120.0
    return X

def lag_feats(hist, idx_t, month):
    h = np.asarray(hist, float)
    return [h[-1], h[-2], h[-3], h[-12], h[-3:].mean(), h[-12:].mean(), idx_t / 120.0, np.sin(2 * np.pi * month / 12), np.cos(2 * np.pi * month / 12)]

def fit_predict(train, horizon_index, model):
    """Return predictions for the months in horizon_index given training frame (recursive for ML)."""
    if model == "Ensemble (GLM + RF + GBM)":
        return np.mean([fit_predict(train, horizon_index, k) for k in ["Poisson GLM (trend + month)", "Random Forest (lags)", "Gradient Boosting (Poisson, lags)"]], axis=0)
    if model == "Seasonal naïve":
        last = train.n.values[-12:]; return np.array([last[i % 12] for i in range(len(horizon_index))])
    if model == "Poisson GLM (trend + month)":
        g = PoissonRegressor(alpha=1e-4, max_iter=3000).fit(design(train), train.n)
        fut = pd.DataFrame({"t": np.arange(len(train), len(train) + len(horizon_index)), "month": horizon_index.month}, index=horizon_index)
        return g.predict(design(fut))
    # ML recursive models
    h = list(train.n.values); X, y = [], []
    for i in range(12, len(h)):
        X.append(lag_feats(h[:i], i, train.month.iloc[i])); y.append(h[i])
    if model == "Random Forest (lags)":
        mdl = RandomForestRegressor(n_estimators=500, min_samples_leaf=3, random_state=SEED).fit(X, y)
    else:
        mdl = HistGradientBoostingRegressor(loss="poisson", max_iter=300, learning_rate=0.05, max_depth=3, min_samples_leaf=8, random_state=SEED).fit(X, y)
    out = []
    for k, ts in enumerate(horizon_index):
        f = lag_feats(h, len(h), ts.month); p = float(mdl.predict([f])[0]); out.append(p); h.append(p)
    return np.array(out)

MODELS = ["Seasonal naïve", "Poisson GLM (trend + month)", "Random Forest (lags)", "Gradient Boosting (Poisson, lags)", "Ensemble (GLM + RF + GBM)"]
bt = []
for test_year in [2020, 2021, 2022]:            # rolling-origin backtest, 12-month horizon
    tr = m[m.index.year < test_year]; te = m[m.index.year == test_year]
    for mod in MODELS:
        pr = fit_predict(tr, te.index, mod)
        bt.append({"test_year": test_year, "model": mod, "MAE": mean_absolute_error(te.n, pr),
                   "RMSE": mean_squared_error(te.n, pr) ** .5, "annual_error_pct": 100 * (pr.sum() - te.n.sum()) / te.n.sum(),
                   "actual_total": int(te.n.sum()), "pred_total": round(pr.sum(), 1)})
bt = pd.DataFrame(bt); bt.round(2).to_csv(os.path.join(OUT, "ml_monthly_backtest.csv"), index=False)
summ = bt.groupby("model")[["MAE", "RMSE"]].mean().sort_values("MAE")
R["monthly_backtest"] = bt.round(2).to_dict("records"); R["monthly_backtest_mean"] = summ.round(2).reset_index().to_dict("records")
# Selection rule: among models within 5% of the lowest mean MAE, pick the one with the smallest worst-case annual error
worst = bt.assign(ae=bt.annual_error_pct.abs()).groupby("model").ae.max()
cands = summ[summ.MAE <= summ.MAE.min() * 1.05].index
best = worst.loc[cands].idxmin(); R["monthly_best"] = best; R["monthly_worst_case_annual_error"] = worst.round(1).to_dict()

# Final forecast 2023–2024 with best model + GLM-based overdispersed interval width scaled by backtest RMSE
fut_idx = pd.date_range("2023-01-01", "2024-12-01", freq="MS")
fc = {mod: fit_predict(m, fut_idx, mod) for mod in MODELS}
bt_rmse = bt[bt.model == best].RMSE.mean()
lo = np.maximum(fc[best] - 1.645 * bt_rmse * np.sqrt(1 + np.arange(24) / 24), 0)
hi = fc[best] + 1.645 * bt_rmse * np.sqrt(1 + np.arange(24) / 24)
F = pd.DataFrame({"month": fut_idx.strftime("%Y-%m"), "forecast_best": fc[best].round(1), "pi90_low": lo.round(1), "pi90_high": hi.round(1),
                  **{f"fc_{k.split(' (')[0].replace(' ', '_').lower()}": v.round(1) for k, v in fc.items()}})
F.to_csv(os.path.join(OUT, "forecast_monthly_2023_2024.csv"), index=False)
R["forecast_annual"] = {"2023": round(fc[best][:12].sum()), "2024": round(fc[best][12:].sum()),
                        "2023_pi90": [round(lo[:12].sum()), round(hi[:12].sum())], "2024_pi90": [round(lo[12:].sum()), round(hi[12:].sum())],
                        "all_models_2023": {k: round(v[:12].sum()) for k, v in fc.items()}}

fig, ax = plt.subplots(figsize=(13, 4.4))
ax.plot(m.index, m.n, color=CAT[0], lw=1.6, label="Observed (2013–2022)")
ax.plot(m.index, m.n.rolling(12, center=True).mean(), color="#0d366b", lw=2.2, label="12-month moving average")
ax.fill_between(fut_idx, lo, hi, color=CAT[1], alpha=.18, linewidth=0, label="90% prediction band")
ax.plot(fut_idx, fc[best], color=CAT[1], lw=2.2, label=f"Forecast: {best}")
ax.axvline(pd.Timestamp("2023-01-01"), color="#a8a79f", lw=1, ls="--")
ax.set_title("Monthly recorded crashes and machine-learning forecast to December 2024"); ax.set_ylabel("Crashes per month"); ax.set_xlabel("")
ax.legend(loc="upper left", fontsize=9, ncol=2); ax.set_ylim(0, None)
fig.tight_layout(); fig.savefig(os.path.join(FIG, "fig10_monthly_forecast.png")); plt.close()

# ===================== B) SEGMENT-LEVEL HOTSPOT FORECAST =====================
p = d[d.on_corridor].copy(); SEG = 2
L = pd.read_csv(os.path.join(OUT, "centreline_derived.csv")).chainage_km.max()
nseg = int(np.ceil(L / SEG)); p["seg"] = (p.chainage_km // SEG).astype(int).clip(0, nseg - 1)
years = list(range(2013, 2023))
C = p.pivot_table(index="seg", columns="year", values="fatal", aggfunc="size", fill_value=0).reindex(index=range(nseg), columns=years, fill_value=0)
K = p.pivot_table(index="seg", columns="year", values="killed", aggfunc="sum", fill_value=0).reindex(index=range(nseg), columns=years, fill_value=0)

def feats(yr):
    """Features known at the end of year yr-1 for predicting counts in year yr."""
    rows = []
    for s in range(nseg):
        prev = [C.loc[s, y] if y in C.columns else 0 for y in (yr - 1, yr - 2, yr - 3)]
        nb = [C.loc[j, yr - 1] for j in (s - 1, s + 1) if 0 <= j < nseg]
        hist = C.loc[s, [y for y in years if y < yr]]
        rows.append({"seg": s, "year": yr, "lag1": prev[0], "lag2": prev[1], "lag3": prev[2], "hist_mean": hist.mean(),
                     "hist_max": hist.max(), "nb_lag1": np.mean(nb), "killed_lag1": K.loc[s, yr - 1],
                     "share_lag1": C.loc[s, yr - 1] / max(C[yr - 1].sum(), 1), "chainage": s * SEG + SEG / 2,
                     "trend3": (prev[0] - prev[2]) / 2, "y": C.loc[s, yr] if yr in C.columns else np.nan})
    return pd.DataFrame(rows)
FEATS = ["lag1", "lag2", "lag3", "hist_mean", "hist_max", "nb_lag1", "killed_lag1", "share_lag1", "chainage", "trend3"]
panel = pd.concat([feats(y) for y in range(2016, 2024)], ignore_index=True)

def seg_models():
    return {"Persistence (last year)": None, "Historical mean": None,
            "Poisson GLM": make_pipeline(StandardScaler(), PoissonRegressor(alpha=1.0, max_iter=3000)),
            "Random Forest": RandomForestRegressor(n_estimators=600, min_samples_leaf=3, random_state=SEED),
            "Gradient Boosting (Poisson)": HistGradientBoostingRegressor(loss="poisson", max_iter=250, learning_rate=0.05, max_depth=3, min_samples_leaf=10, random_state=SEED)}

def hit_rate(y, pred, top=0.10):
    k = max(1, int(round(top * len(y)))); idx = np.argsort(-pred)[:k]
    hr = y[idx].sum() / max(y.sum(), 1); return hr, hr / (k / len(y))

ev = []; preds_store = {}
for ty in [2020, 2021, 2022]:
    tr = panel[(panel.year < ty) & panel.y.notna()]; te = panel[panel.year == ty]
    for name, mdl in seg_models().items():
        if name.startswith("Persistence"): pr = te.lag1.values
        elif name.startswith("Historical"): pr = te.hist_mean.values
        else: pr = mdl.fit(tr[FEATS], tr.y).predict(te[FEATS])
        y = te.y.values; hr, pai = hit_rate(y, pr)
        ev.append({"test_year": ty, "model": name, "MAE": mean_absolute_error(y, pr), "Spearman": stats.spearmanr(y, pr)[0],
                   "hit_rate_top10pct": hr * 100, "PAI": pai})
        if ty == 2022: preds_store[name] = pr
ev = pd.DataFrame(ev); ev.round(3).to_csv(os.path.join(OUT, "ml_segment_backtest.csv"), index=False)
evm = ev.groupby("model")[["MAE", "Spearman", "hit_rate_top10pct", "PAI"]].mean().sort_values("MAE")
R["segment_backtest"] = ev.round(3).to_dict("records"); R["segment_backtest_mean"] = evm.round(3).reset_index().to_dict("records")
ml_only = evm.loc[["Poisson GLM", "Random Forest", "Gradient Boosting (Poisson)"]]
best_seg = ml_only.sort_values("MAE").index[0]; R["segment_best"] = best_seg

# Final: train on all target years 2016–2022, predict 2023
tr = panel[panel.y.notna()]; f23 = panel[panel.year == 2023].copy()
mdl = seg_models()[best_seg].fit(tr[FEATS], tr.y)
f23["pred_2023"] = mdl.predict(f23[FEATS])
# Split-conformal 90% intervals from backtest residuals of the chosen model (normalised by sqrt(pred+1))
res = []
for ty in [2020, 2021, 2022]:
    trb = panel[(panel.year < ty) & panel.y.notna()]; teb = panel[panel.year == ty]
    pb = seg_models()[best_seg].fit(trb[FEATS], trb.y).predict(teb[FEATS])
    res.extend(np.abs(teb.y.values - pb) / np.sqrt(pb + 1))
qc = np.quantile(res, 0.90); R["seg_conformal_q90"] = round(float(qc), 3)
# Monte-Carlo (Poisson around a model-error-perturbed mean) for probability of being a top-10% segment
rng = np.random.default_rng(SEED)
mu = np.clip(f23.pred_2023.values, 0, None)
mu_s = np.clip(mu[None] + rng.choice(np.r_[res, -np.array(res)], size=(5000, nseg)) * np.sqrt(mu + 1)[None], 0, None)
sims = rng.poisson(mu_s)
k10 = max(1, int(round(.1 * nseg)))
thr = np.sort(sims, axis=1)[:, -k10][:, None]
f23["p_top10"] = ((sims >= thr) & (sims > 0)).mean(0)
f23["pi90_low"] = np.clip(mu - qc * np.sqrt(mu + 1), 0, None); f23["pi90_high"] = mu + qc * np.sqrt(mu + 1)
f23["from_km"] = f23.seg * SEG; f23["to_km"] = f23.from_km + SEG
hs = pd.read_csv(os.path.join(OUT, "hotspots_gistar_2km.csv"))[["seg", "class", "emerging", "top_location_names"]]
f23 = f23.merge(hs, on="seg", how="left")
f23["risk_tier"] = pd.cut(f23.pred_2023.rank(pct=True), [0, .5, .8, .9, 1.0], labels=["Low", "Moderate", "High", "Very high"])
out23 = f23[["seg", "from_km", "to_km", "lag1", "hist_mean", "pred_2023", "pi90_low", "pi90_high", "p_top10", "risk_tier", "class", "emerging", "top_location_names"]]
out23 = out23.rename(columns={"lag1": "crashes_2022", "hist_mean": "mean_2013_2022", "class": "gistar_class_2013_2022"})
out23.round(3).sort_values("pred_2023", ascending=False).to_csv(os.path.join(OUT, "forecast_segments_2023.csv"), index=False)
R["seg_2023_top"] = out23.sort_values("pred_2023", ascending=False).head(10).round(2).to_dict("records")
R["seg_2023_total"] = round(float(out23.pred_2023.sum()), 1)
if hasattr(mdl, "feature_importances_"):
    R["seg_feature_importance"] = dict(sorted(zip(FEATS, mdl.feature_importances_.round(3)), key=lambda x: -x[1]))
else:
    te = panel[panel.year == 2022]; pi = permutation_importance(seg_models()[best_seg].fit(panel[(panel.year < 2022) & panel.y.notna()][FEATS], panel[(panel.year < 2022) & panel.y.notna()].y), te[FEATS], te.y, n_repeats=30, random_state=SEED, scoring="neg_mean_absolute_error")
    R["seg_feature_importance"] = dict(sorted(zip(FEATS, pi.importances_mean.round(3)), key=lambda x: -x[1]))

# Figures: 2022 test predicted vs observed; 2023 risk profile
te22 = panel[panel.year == 2022]
fig, axs = plt.subplots(1, 2, figsize=(13, 4.6), gridspec_kw={"width_ratios": [1, 1.9]})
ax = axs[0]; pr = preds_store[best_seg]
ax.scatter(te22.y, pr, s=26, color=CAT[0], edgecolor="#fcfcfb", linewidth=.8)
mx = max(te22.y.max(), pr.max()) * 1.05; ax.plot([0, mx], [0, mx], color="#a8a79f", ls="--", lw=1)
rho = stats.spearmanr(te22.y, pr)[0]
ax.set_title(f"Held-out 2022 test: {best_seg}"); ax.set_xlabel("Observed crashes per 2-km segment, 2022"); ax.set_ylabel("Predicted")
ax.text(.04, .92, f"Spearman ρ = {rho:.2f}", transform=ax.transAxes, color=INK2, fontsize=9)
ax = axs[1]
tiercol = {"Low": "#d6d5cf", "Moderate": "#9ec5f4", "High": "#3987e5", "Very high": "#0d366b"}
ax.bar(out23.from_km + 1, out23.pred_2023, width=1.7, color=out23.risk_tier.astype(str).map(tiercol))
ax.errorbar(out23.from_km + 1, out23.pred_2023, yerr=[(out23.pred_2023 - out23.pi90_low).clip(lower=0), (out23.pi90_high - out23.pred_2023).clip(lower=0)], fmt="none", ecolor="#a8a79f", elinewidth=.7)
for _, r in out23.sort_values("pred_2023", ascending=False).head(5).iterrows():
    ax.annotate(str(r.top_location_names).split(",")[0][:18], (r.from_km + 1, r.pi90_high), xytext=(0, 3), textcoords="offset points", ha="center", rotation=90, fontsize=7.5, color=INK2, va="bottom")
from matplotlib.patches import Patch
ax.legend(handles=[Patch(color=v, label=k) for k, v in tiercol.items()], title="2023 risk tier", fontsize=8, title_fontsize=8.5, ncol=4, loc="upper right")
ax.set_ylim(0, out23.pi90_high.max() * 1.55); ax.set_xlim(0, L)
ax.set_title("Predicted crashes per 2-km segment, 2023 (whiskers = 90% conformal interval)"); ax.set_xlabel("Chainage (km from Ojota, Lagos)"); ax.set_ylabel("Predicted crashes")
fig.tight_layout(); fig.savefig(os.path.join(FIG, "fig11_segment_forecast.png")); plt.close()

# ===================== C) SEVERITY CLASSIFIER =====================
s = d.copy(); s["month"] = s.date.dt.month; s["dow"] = s.date.dt.dayofweek
s["night"] = ((s.hour >= 19) | (s.hour <= 5)).astype(float).where(s.hour.notna())
num = ["hour", "month", "dow", "year", "involved", "multi_vehicle", "chainage_km"]
cat = ["vehicle_type", "ownership", "cause"]
s = s.dropna(subset=["involved"])
X = s[num + cat].copy(); y = s.fatal.values
for c_ in cat: X[c_] = X[c_].astype("category")
cv = StratifiedKFold(5, shuffle=True, random_state=SEED)
hgb = HistGradientBoostingClassifier(max_iter=300, learning_rate=0.04, max_depth=4, min_samples_leaf=30, l2_regularization=1.0,
                                     categorical_features=[X.columns.get_loc(c_) for c_ in cat], random_state=SEED)
p_h = cross_val_predict(hgb, X, y, cv=cv, method="predict_proba")[:, 1]
Xl = s[num + cat].copy(); Xl[num] = Xl[num].fillna(Xl[num].median())
lr = make_pipeline(ColumnTransformer([("n", StandardScaler(), num), ("c", OneHotEncoder(handle_unknown="ignore"), cat)]), LogisticRegression(max_iter=2000, C=0.5))
p_l = cross_val_predict(lr, Xl, y, cv=cv, method="predict_proba")[:, 1]
R["severity"] = {"n": int(len(y)), "fatal_share": round(float(y.mean()), 3),
                 "HGB": {"ROC_AUC": round(roc_auc_score(y, p_h), 3), "PR_AUC": round(average_precision_score(y, p_h), 3), "Brier": round(brier_score_loss(y, p_h), 3)},
                 "Logistic": {"ROC_AUC": round(roc_auc_score(y, p_l), 3), "PR_AUC": round(average_precision_score(y, p_l), 3), "Brier": round(brier_score_loss(y, p_l), 3)}}
# permutation importance on a hold-out split
from sklearn.model_selection import train_test_split
Xtr, Xte, ytr, yte = train_test_split(X, y, test_size=.3, stratify=y, random_state=SEED)
hgb.fit(Xtr, ytr)
pi = permutation_importance(hgb, Xte, yte, n_repeats=30, random_state=SEED, scoring="roc_auc")
imp = pd.DataFrame({"feature": X.columns, "importance": pi.importances_mean, "sd": pi.importances_std}).sort_values("importance", ascending=False)
imp.round(4).to_csv(os.path.join(OUT, "ml_severity_importance.csv"), index=False)
R["severity_importance"] = imp.round(4).to_dict("records")
fig, ax = plt.subplots(figsize=(7.5, 4))
nice = {"involved": "Persons involved", "chainage_km": "Location (chainage)", "year": "Year", "hour": "Hour of day", "month": "Month",
        "dow": "Day of week", "vehicle_type": "Vehicle type", "cause": "Probable cause", "ownership": "Ownership", "multi_vehicle": "Multi-vehicle"}
sns.barplot(x=imp.importance, y=imp.feature.map(nice), color=CAT[0], ax=ax, width=.6)
ax.errorbar(imp.importance, np.arange(len(imp)), xerr=imp.sd, fmt="none", ecolor="#a8a79f", elinewidth=1)
ax.set_title("What predicts a fatal crash? (permutation importance)"); ax.set_xlabel("Drop in ROC-AUC when feature is shuffled"); ax.set_ylabel("")
fig.tight_layout(); fig.savefig(os.path.join(FIG, "fig12_severity_importance.png")); plt.close()

json.dump(R, open(os.path.join(OUT, "ml_summary.json"), "w"), indent=1, default=str)
print(json.dumps({k: R[k] for k in ["monthly_backtest_mean", "monthly_best", "forecast_annual", "segment_backtest_mean", "segment_best", "seg_2023_total", "seg_feature_importance", "severity"]}, indent=1, default=str))
print(pd.DataFrame(R["seg_2023_top"])[["from_km", "crashes_2022", "pred_2023", "pi90_low", "pi90_high", "p_top10", "top_location_names"]].to_string())
print(imp.to_string())
```


### fig12_severity_panel.py

```python
"""fig12_severity_panel.py — Redesigned 'What makes a crash fatal?' figure (4 panels).
Usage: python3 -I fig12_severity_panel.py <out_dir>"""
import sys, os, json, numpy as np, pandas as pd
sys.path.insert(0, os.path.dirname(__file__))
import style; style.setup()
import matplotlib.pyplot as plt
from matplotlib.lines import Line2D
OUT = sys.argv[1]
d = pd.read_csv(os.path.join(OUT, "rtc_lagos_ibadan_clean_2013_2022.csv"), low_memory=False)
imp = pd.read_csv(os.path.join(OUT, "ml_severity_importance.csv"))
base = d.fatal.mean() * 100
INK2, GRID = "#52514e", "#e4e3df"
RED, BLUE, GREY = "#c0392b", "#2a78d6", "#a8a79f"

def wilson(k, n, z=1.96):
    p = k / n; den = 1 + z ** 2 / n; c = (p + z ** 2 / (2 * n)) / den; h = z * np.sqrt(p * (1 - p) / n + z ** 2 / (4 * n ** 2)) / den
    return 100 * p, 100 * (c - h), 100 * (c + h)

fig = plt.figure(figsize=(14, 10.2))
gs = fig.add_gridspec(2, 2, hspace=0.40, wspace=0.42)

# (a) importance lollipop, coloured by feature group
ax = fig.add_subplot(gs[0, 0])
nice = {"involved": "Persons involved", "chainage_km": "Location along corridor", "year": "Year", "hour": "Hour of day", "month": "Month",
        "dow": "Day of week", "vehicle_type": "Vehicle type", "cause": "Probable cause", "ownership": "Ownership", "multi_vehicle": "Multi-vehicle"}
grp = {"chainage_km": ("Where", "#1baf7a"), "year": ("When", BLUE), "hour": ("When", BLUE), "month": ("When", BLUE), "dow": ("When", BLUE),
       "cause": ("Why", "#eb6834"), "involved": ("Crash", "#6250d6"), "multi_vehicle": ("Crash", "#6250d6"),
       "vehicle_type": ("Vehicle", "#e87ba4"), "ownership": ("Vehicle", "#e87ba4")}
imp = imp.sort_values("importance")
y = np.arange(len(imp))
for i, r in enumerate(imp.itertuples()):
    col = grp[r.feature][1]
    ax.plot([0, r.importance], [i, i], color=col, lw=2.2, alpha=.85)
    ax.errorbar(r.importance, i, xerr=r.sd, fmt="o", color=col, ms=8, ecolor=GREY, elinewidth=1, capsize=0, markeredgecolor="#fcfcfb")
ax.axvline(0, color=INK2, lw=.8)
ax.set_yticks(y); ax.set_yticklabels([nice[f] for f in imp.feature]); ax.set_xlabel("Drop in ROC-AUC when the feature is shuffled")
ax.set_title("(a) Which factors carry predictive signal?")
ax.legend(handles=[Line2D([], [], marker="o", ls="-", color=c, label=g) for g, c in [("Where", "#1baf7a"), ("When", BLUE), ("Why (cause)", "#eb6834"), ("Crash", "#6250d6"), ("Vehicle", "#e87ba4")]],
          loc="lower right", fontsize=8.5, frameon=True, facecolor="#fcfcfb", edgecolor=GRID)

# (b) fatality rate by hour of day
ax = fig.add_subplot(gs[0, 1])
h = d.dropna(subset=["hour"]).groupby("hour").fatal.agg(["sum", "count"])
W = np.array([wilson(k, n) for k, n in zip(h["sum"], h["count"])])
hrs = h.index.values.astype(int)
ax.axvspan(-0.5, 5.5, color="#0d366b", alpha=.07, lw=0); ax.axvspan(18.5, 23.5, color="#0d366b", alpha=.07, lw=0)
ax.fill_between(hrs, W[:, 1], W[:, 2], color=RED, alpha=.15, lw=0)
ax.plot(hrs, W[:, 0], color=RED, lw=2.2, marker="o", ms=5)
ax.axhline(base, color=INK2, ls="--", lw=1); ax.text(23.4, base + 1, f"overall {base:.1f}%", ha="right", fontsize=8.5, color=INK2)
ax.text(2.5, ax.get_ylim()[1] * 0.97 if ax.get_ylim()[1] > 0 else 50, "night", ha="center", va="top", fontsize=9, color="#0d366b", style="italic")
ax.set_xlim(-0.5, 23.5); ax.set_xticks(range(0, 24, 3)); ax.set_xlabel("Hour of day"); ax.set_ylabel("% of crashes that were fatal")
ax.set_title("(b) Night-time crashes are more often fatal"); ax.set_ylim(0, None)
ax.text(21, ax.get_ylim()[1] * 0.97, "night", ha="center", va="top", fontsize=9, color="#0d366b", style="italic")

# (c) fatality rate along the corridor (5-km bins)
ax = fig.add_subplot(gs[1, 0])
p = d[d.on_corridor == True].copy(); p["b"] = (p.chainage_km // 5 * 5).astype(int)
g = p.groupby("b").fatal.agg(["sum", "count"]); g = g[g["count"] >= 25]
W = np.array([wilson(k, n) for k, n in zip(g["sum"], g["count"])])
x = g.index.values + 2.5
cols = [RED if lo > base else (BLUE if hi < base else GREY) for _, lo, hi in W]
ax.vlines(x, W[:, 1], W[:, 2], color=cols, lw=2, alpha=.6)
ax.scatter(x, W[:, 0], s=np.clip(g["count"].values / 4, 15, 160), c=cols, edgecolor="#fcfcfb", zorder=3)
ax.axhline(base, color=INK2, ls="--", lw=1)
for xi, (r, lo, hi), n in zip(x, W, g["count"].values):
    pass
LAB = {42.5: ("Danco filling stn", -9), 87.5: ("Dominion Univ.", -9), 92.5: ("", 0), 102.5: ("", 0)}
for xi, (r, lo, hi) in zip(x, W):
    if lo > base:
        t, dx = LAB.get(xi, ("", 0))
        ax.annotate(f"{r:.0f}%" + (f"\n{t}" if t else ""), (xi, r), xytext=(dx if t else 14, 0), textcoords="offset points", ha="right" if t else "left",
                    va="center", ma="right" if t else "left", fontsize=8, color=RED, weight="bold", bbox=dict(boxstyle="round,pad=0.12", fc="#fcfcfb", ec="none", alpha=.85))
ax.set_xlim(0, 120); ax.set_ylim(0, 66); ax.set_xlabel("Chainage (km from Ojota, Lagos → Ibadan; 5-km bins, n ≥ 25)"); ax.set_ylabel("% of crashes that were fatal")
ax.set_title("(c) Severity varies along the corridor")
ax.legend(handles=[Line2D([], [], marker="o", ls="", color=RED, label="Significantly above average"), Line2D([], [], marker="o", ls="", color=BLUE, label="Significantly below average"),
                   Line2D([], [], marker="o", ls="", color=GREY, label="Not different (95% CI)")], loc="upper left", fontsize=8, frameon=True, facecolor="#fcfcfb", edgecolor=GRID)

# (d) fatality rate by cause and by year (two dot strips)
ax = fig.add_subplot(gs[1, 1])
cm = {"Speeding": "Speeding", "Other": "Other / unspecified", "Brake failure": "Brake failure", "Tyre burst": "Tyre burst", "Mechanical deficiency": "Mechanical deficiency",
      "Dangerous driving/overtaking": "Dangerous driving", "Fatigue": "Fatigue / LOC / DUI", "Loss of control": "Fatigue / LOC / DUI", "DUI": "Fatigue / LOC / DUI"}
c = d.assign(C=d.cause.map(cm).fillna("Other / unspecified")).groupby("C").fatal.agg(["sum", "count"])
W = pd.DataFrame([wilson(k, n) for k, n in zip(c["sum"], c["count"])], index=c.index, columns=["r", "lo", "hi"]).join(c["count"]).sort_values("r")
yy = np.arange(len(W))
cols = [RED if lo > base else (BLUE if hi < base else GREY) for lo, hi in zip(W.lo, W.hi)]
ax.hlines(yy, W.lo, W.hi, color=cols, lw=2.2, alpha=.7); ax.scatter(W.r, yy, s=70, c=cols, edgecolor="#fcfcfb", zorder=3)
for yi, r in zip(yy, W.itertuples()): ax.text(r.hi + 1, yi, f"{r.r:.1f}%  (n = {r.count:,})", va="center", fontsize=8.5, color=INK2)
ax.axvline(base, color=INK2, ls="--", lw=1)
ax.set_yticks(yy); ax.set_yticklabels(W.index); ax.set_xlim(0, max(W.hi) + 18); ax.set_xlabel("% of crashes that were fatal (95% CI)")
ax.set_title("(d) Fatality rate by probable cause")

fig.suptitle("What makes a crash fatal? Lagos–Ibadan Expressway, 2013–2022", x=0.01, ha="left", fontsize=15, weight="bold", y=0.995)
fig.text(0.01, 0.955, f"Overall {base:.1f}% of {len(d):,} crashes were fatal (dashed lines). Intervals are 95% Wilson confidence intervals. Classifier ROC-AUC = 0.60, i.e. recorded attributes give only weak predictive signal.", fontsize=10, color=INK2)
fig.savefig(os.path.join(OUT, "figures", "fig12_what_makes_fatal_panel.png"), dpi=200, bbox_inches="tight"); plt.close()

# numbers for text
night = d[(d.hour >= 19) | (d.hour <= 5)].fatal.mean() * 100; day = d[d.hour.between(6, 18)].fatal.mean() * 100
peak = h.assign(r=h["sum"] / h["count"] * 100)["r"]
hi_bins = [(int(b), round(r, 1), int(n)) for b, (r, lo, hi_), n in zip(g.index, np.array([wilson(k, n) for k, n in zip(g["sum"], g["count"])]), g["count"]) if lo > base]
lo_bins = [(int(b), round(r, 1), int(n)) for b, (r, lo, hi_), n in zip(g.index, np.array([wilson(k, n) for k, n in zip(g["sum"], g["count"])]), g["count"]) if hi_ < base]
S = {"base": round(base, 1), "night": round(night, 1), "day": round(day, 1), "peak_hour": int(peak.idxmax()), "peak_rate": round(peak.max(), 1),
     "min_hour": int(peak.idxmin()), "min_rate": round(peak.min(), 1), "hi_bins": hi_bins, "lo_bins": lo_bins,
     "cause": W[["r", "lo", "hi", "count"]].round(1).to_dict("index")}
json.dump(S, open(os.path.join(OUT, "fig12_panel_summary.json"), "w"), indent=1, default=float); print(json.dumps(S, indent=1, default=float))
```


### ml_2027.py

```python
"""ml_2027.py — extend the corridor-level monthly crash forecast to December 2027 using the same models,
back-test and selection rule as 05_ml.py. Usage: python3 -I ml_2027.py <out_dir>"""
import sys, os, json
src = open(os.path.join(os.path.dirname(os.path.abspath(__file__)), "05_ml.py")).read()
head = src[:src.index("# Final forecast 2023–2024")]
sys.argv = [sys.argv[0], sys.argv[1]]
ns = {"__file__": os.path.join(os.path.dirname(os.path.abspath(__file__)), "05_ml.py")}
exec(compile(head, "05_ml_head", "exec"), ns)
np, pd, plt = ns["np"], ns["pd"], ns["plt"]; m, fit_predict, MODELS, bt, best, OUT, CAT = ns["m"], ns["fit_predict"], ns["MODELS"], ns["bt"], ns["best"], ns["OUT"], ns["CAT"]
H = 60
fut = pd.date_range("2023-01-01", "2027-12-01", freq="MS")
fc = {k: fit_predict(m, fut, k) for k in MODELS}
rmse = bt[bt.model == best].RMSE.mean()
w = 1.645 * rmse * np.sqrt(1 + np.arange(H) / 24)
lo, hi = np.maximum(fc[best] - w, 0), fc[best] + w
F = pd.DataFrame({"month": fut.strftime("%Y-%m"), "forecast_ensemble": fc[best].round(1), "pi90_low": lo.round(1), "pi90_high": hi.round(1),
                  **{f"fc_{k.split(' (')[0].replace(' ', '_').lower()}": v.round(1) for k, v in fc.items()}})
F.to_csv(os.path.join(OUT, "forecast_monthly_2023_2027.csv"), index=False)
yrs = list(range(2023, 2028)); A = []
for i, y in enumerate(yrs):
    s = slice(12 * i, 12 * i + 12)
    A.append({"year": y, "ensemble": round(fc[best][s].sum()), "pi90_low": round(lo[s].sum()), "pi90_high": round(hi[s].sum()),
              **{k: round(v[s].sum()) for k, v in fc.items()}})
A = pd.DataFrame(A); A.to_csv(os.path.join(OUT, "forecast_annual_2023_2027.csv"), index=False)
obs = m.n.groupby(m.index.year).sum()
json.dump({"best": best, "rmse": round(rmse, 2), "annual": A.to_dict("records"), "obs": {int(k): int(v) for k, v in obs.items()}}, open(os.path.join(OUT, "forecast_2027_summary.json"), "w"), indent=1)
print(A.to_string())

# ---------- figure: monthly series + annual totals ----------
fig, axs = plt.subplots(2, 1, figsize=(13, 8.2), gridspec_kw={"height_ratios": [1.15, 1]})
ax = axs[0]
ax.plot(m.index, m.n, color=CAT[0], lw=1.5, label="Observed (2013–2022)")
ax.plot(m.index, m.n.rolling(12, center=True).mean(), color="#0d366b", lw=2.2, label="12-month moving average")
ax.fill_between(fut, lo, hi, color=CAT[1], alpha=.18, linewidth=0, label="90% prediction band")
ax.plot(fut, fc[best], color=CAT[1], lw=2.2, label=f"Forecast: {best}")
ax.axvline(pd.Timestamp("2023-01-01"), color="#a8a79f", lw=1, ls="--")
ax.set_title("(a) Monthly recorded crashes and machine-learning forecast to December 2027"); ax.set_ylabel("Crashes per month"); ax.set_xlabel("")
ax.legend(loc="upper left", fontsize=9, ncol=2); ax.set_ylim(0, None)
ax = axs[1]
xo = obs.index.values; ax.bar(xo, obs.values, color=CAT[0], width=.65, label="Observed (dated records)")
ax.bar(A.year, A.ensemble, color=CAT[1], width=.65, label="Forecast (ensemble)")
ax.errorbar(A.year, A.ensemble, yerr=[A.ensemble - A.pi90_low, A.pi90_high - A.ensemble], fmt="none", ecolor="#52514e", elinewidth=1.2, capsize=4, label="90% interval")
ax.plot(A.year, A["Poisson GLM (trend + month)"], "^", color="#6250d6", ms=8, label="Scenario: trend continues (Poisson GLM)", zorder=5)
ax.plot(A.year, A["Random Forest (lags)"], "s", color="#1baf7a", ms=7, label="Scenario: plateau (Random Forest)", zorder=5)
for x, v in zip(xo, obs.values): ax.text(x, v + 25, f"{v:,}", ha="center", fontsize=8, color="#52514e")
for x, v in zip(A.year, A.ensemble): ax.text(x, 90, f"{v:,}", ha="center", fontsize=8.5, color="#fcfcfb", weight="bold")
ax.set_xticks(list(xo) + list(A.year)); ax.set_ylabel("Crashes per year"); ax.set_title("(b) Annual totals: observed 2013–2022 and forecast 2023–2027")
ax.legend(loc="upper left", fontsize=8.5, ncol=2); ax.set_ylim(0, max(A.pi90_high.max(), A["Poisson GLM (trend + month)"].max(), obs.max()) * 1.15)
fig.tight_layout(); fig.savefig(os.path.join(OUT, "figures", "fig10_monthly_forecast_2027.png"), dpi=200); plt.close()
```


### fig_2027.py

```python
"""fig_2027.py — Visualise the 2027 crash forecast (corridor total) from the ml_2027.py run.
Usage: python3 -I fig_2027.py <out_dir>"""
import sys, os, json, numpy as np, pandas as pd
sys.path.insert(0, os.path.dirname(__file__))
import style; style.setup()
import matplotlib.pyplot as plt
from matplotlib.lines import Line2D
from matplotlib.patches import Patch
OUT = sys.argv[1]
INK, INK2, GRID = "#0b0b0b", "#52514e", "#e4e3df"
ENS, GLM, RF, GBM, NAIVE, OBS = "#eb6834", "#c0392b", "#1baf7a", "#6250d6", "#8a8983", "#2a78d6"
A = pd.read_csv(os.path.join(OUT, "forecast_annual_2023_2027.csv"))
M = pd.read_csv(os.path.join(OUT, "forecast_monthly_2023_2027.csv"))
S = json.load(open(os.path.join(OUT, "forecast_2027_summary.json"))); obs = pd.Series({int(k): v for k, v in S["obs"].items()}).sort_index()
d = pd.read_csv(os.path.join(OUT, "rtc_lagos_ibadan_clean_2013_2022.csv"), parse_dates=["date"], low_memory=False)
o22 = d[d.date.dt.year == 2022].groupby(d.date.dt.month).size().reindex(range(1, 13), fill_value=0)
m27 = M[M.month.str.startswith("2027")].reset_index(drop=True); mo = np.arange(1, 13)
r27 = A[A.year == 2027].iloc[0]
cols = {"glm": "Poisson GLM (trend + month)", "rf": "Random Forest (lags)", "gbm": "Gradient Boosting (Poisson, lags)", "naive": "Seasonal naïve"}

fig = plt.figure(figsize=(14, 10.4)); gs = fig.add_gridspec(2, 2, height_ratios=[1, 1.05], hspace=0.55, wspace=0.45)

# (a) 2027 month by month
ax = fig.add_subplot(gs[0, 0])
ax.bar(mo, o22.values, color="#c9c8c1", width=.62, label="Observed 2022 (for reference)", zorder=1)
ax.fill_between(mo, m27.pi90_low, m27.pi90_high, color=ENS, alpha=.16, lw=0, label="90% prediction band", zorder=2)
ax.plot(mo, m27.fc_poisson_glm, color=GLM, lw=1.4, ls="--", zorder=3, label="Poisson GLM (trend)")
ax.plot(mo, m27.fc_random_forest, color=RF, lw=1.4, ls="--", zorder=3, label="Random forest (lags)")
ax.plot(mo, m27.fc_gradient_boosting, color=GBM, lw=1.4, ls=":", zorder=3, label="Gradient boosting (lags)")
ax.plot(mo, m27.forecast_ensemble, color=ENS, lw=2.8, marker="o", ms=5, zorder=4, label="Ensemble (selected model)")
ax.set_xticks(mo); ax.set_xticklabels(["J", "F", "M", "A", "M", "J", "J", "A", "S", "O", "N", "D"]); ax.set_xlim(.4, 12.6)
ax.set_ylim(0, max(m27.pi90_high.max(), m27.fc_poisson_glm.max()) * 1.08); ax.set_ylabel("Recorded crashes per month"); ax.grid(axis="x", visible=False)
pk = m27.forecast_ensemble.idxmax()
ax.annotate(f"Peak: {['Jan','Feb','Mar','Apr','May','Jun','Jul','Aug','Sep','Oct','Nov','Dec'][pk]} ≈ {m27.forecast_ensemble[pk]:.0f}", (pk + 1, m27.forecast_ensemble[pk]),
            xytext=(-60, 28), textcoords="offset points", fontsize=8.5, color=ENS, weight="bold", arrowprops=dict(arrowstyle="-", color=ENS, lw=.8))
ax.legend(loc="upper center", bbox_to_anchor=(0.5, -0.1), fontsize=7.8, ncol=3, frameon=False)
ax.set_title("(a) 2027 forecast, month by month")

# (b) 2027 total by model, against recent observations
ax = fig.add_subplot(gs[0, 1])
items = [("Observed 2021 (peak)", obs[2021], OBS, None), ("Observed 2022", obs[2022], OBS, None),
         ("Seasonal naïve", r27[cols["naive"]], NAIVE, None), ("Random forest (plateau)", r27[cols["rf"]], RF, None),
         ("Gradient boosting", r27[cols["gbm"]], GBM, None), ("Ensemble (selected)", r27.ensemble, ENS, (r27.pi90_low, r27.pi90_high)),
         ("Poisson GLM (trend)", r27[cols["glm"]], GLM, None)]
y = np.arange(len(items))[::-1]
ax.axvline(obs[2022], color=OBS, lw=1, ls=":", zorder=0)
for yi, (lab, v, c, ci) in zip(y, items):
    if ci: ax.hlines(yi, ci[0], ci[1], color=c, lw=7, alpha=.25, zorder=1)
    ax.hlines(yi, 0, v, color=c, lw=1.2, alpha=.35, zorder=1)
    ax.scatter(v, yi, s=110 if ci else 70, color=c, edgecolor="#fcfcfb", zorder=3, marker="s" if lab.startswith("Observed") else "o")
    ch = "" if lab.startswith("Observed") else f"  ({100 * (v / obs[2022] - 1):+.0f}% vs 2022)"
    ax.text(v + 30, yi, f"{v:,.0f}{ch}", va="center", fontsize=8.8, color=c if not lab.startswith("Observed") else INK2, weight="bold" if ci else "normal")
ax.text(r27.pi90_low, y[5] - .42, f"90% interval {r27.pi90_low:,.0f}–{r27.pi90_high:,.0f}", fontsize=8, color=ENS, va="top")
ax.set_yticks(y); ax.set_yticklabels([i[0] for i in items]); ax.set_xlim(0, max(r27[cols["glm"]], r27.pi90_high) * 1.33); ax.set_xlabel("Recorded crashes in the year")
ax.grid(axis="y", visible=False); ax.set_title("(b) 2027 total: how the models compare")

# (c) path from observed to 2027
ax = fig.add_subplot(gs[1, :])
ax.bar(obs.index, obs.values, color=OBS, width=.62, label="Observed (dated records)", zorder=2)
for x, v in obs.items():
    if x in (2013, 2021, 2022): ax.text(x - (0.18 if x == 2022 else 0), v + 25, f"{v:,}", ha="center", fontsize=8.5, color=INK2)
fy = A.year.values
ax.fill_between(fy, A.pi90_low, A.pi90_high, color=ENS, alpha=.14, lw=0, label="Ensemble 90% interval")
ax.plot(np.r_[2022, fy], np.r_[obs[2022], A[cols["glm"]]], color=GLM, lw=1.8, ls="--", marker="o", ms=4, label="Scenario: trend continues (Poisson GLM)")
ax.plot(np.r_[2022, fy], np.r_[obs[2022], A[cols["rf"]]], color=RF, lw=1.8, ls="--", marker="o", ms=4, label="Scenario: plateau (random forest)")
ax.plot(np.r_[2022, fy], np.r_[obs[2022], A.ensemble], color=ENS, lw=2.8, marker="o", ms=7, label="Ensemble forecast (selected)", zorder=4)
for x, v in zip(fy, A.ensemble): ax.text(x, v + 70, f"{v:,}", ha="center", fontsize=9, color=ENS, weight="bold")
ax.axvspan(2026.5, 2027.5, color=ENS, alpha=.06, lw=0, zorder=0)
ax.text(2027, ax.get_ylim()[1] if False else A[cols["glm"]].max() * 1.07, "2027", ha="center", fontsize=11, weight="bold", color=ENS)
ax.text(2027.42, A[cols["glm"]].iloc[-1], f"{A[cols['glm']].iloc[-1]:,}", va="center", fontsize=8.5, color=GLM)
ax.text(2027.42, A[cols["rf"]].iloc[-1], f"{A[cols['rf']].iloc[-1]:,}", va="center", fontsize=8.5, color=RF)
ax.axvline(2022.5, color="#a8a79f", lw=1, ls="--"); ax.text(2022.58, A[cols["glm"]].max() * 1.07, "forecast →", fontsize=9, color=INK2, va="center")
ax.set_xticks(range(2013, 2028)); ax.set_xlim(2012.4, 2027.9); ax.set_ylim(0, A[cols["glm"]].max() * 1.13)
ax.set_ylabel("Recorded crashes per year"); ax.grid(axis="x", visible=False)
ax.legend(loc="upper left", fontsize=8.5, frameon=True, facecolor="#fcfcfb", edgecolor=GRID)
ax.set_title("(c) From the 2013–2022 record to 2027: forecast and scenarios")

fig.suptitle("Predicted road traffic crashes on the Lagos–Ibadan Expressway in 2027", x=0.01, ha="left", fontsize=15, weight="bold", y=0.997)
fig.text(0.01, 0.962, f"Ensemble forecast for 2027: {r27.ensemble:,.0f} recorded crashes (90% interval {r27.pi90_low:,.0f}–{r27.pi90_high:,.0f}), {100 * (r27.ensemble / obs[2022] - 1):+.0f}% on 2022. "
         f"The models diverge after 2024, so 2027 is best read as a range between a plateau (~{r27[cols['rf']]:,.0f}) and a continued trend (~{r27[cols['glm']]:,.0f}).", fontsize=9.6, color=INK2)
fig.savefig(os.path.join(OUT, "figures", "fig_forecast_2027.png"), dpi=200, bbox_inches="tight"); plt.close()
print(json.dumps({"ens": int(r27.ensemble), "lo": int(r27.pi90_low), "hi": int(r27.pi90_high), "glm": int(r27[cols["glm"]]), "rf": int(r27[cols["rf"]]), "gbm": int(r27[cols["gbm"]]),
                  "obs22": int(obs[2022]), "peak_month": int(pk + 1), "peak": float(m27.forecast_ensemble[pk])}))
```


### ml_seg_2027.py

```python
"""ml_seg_2027.py — Predict which 2-km sections of the Lagos–Ibadan Expressway are likely to be crash hot spots in 2027.
Direct 5-year-ahead model: a section's share of corridor crashes in year t is predicted from information up to t−5.
Back-tested on 2020–2022 against persistence (share in t−5) and the historical mean share; 2027 is predicted from 2022.
Usage: python3 -I ml_seg_2027.py <out_dir>"""
import sys, os, json, numpy as np, pandas as pd
from scipy import stats
from sklearn.ensemble import RandomForestRegressor
OUT = sys.argv[1]; SEED = 42; H = 5; SEG = 2
d = pd.read_csv(os.path.join(OUT, "rtc_lagos_ibadan_clean_2013_2022.csv"), low_memory=False)
p = d[d.on_corridor == True].copy()
L = pd.read_csv(os.path.join(OUT, "centreline_derived.csv")).chainage_km.max(); nseg = int(np.ceil(L / SEG))
p["seg"] = (p.chainage_km // SEG).astype(int).clip(0, nseg - 1)
years = list(range(2013, 2023))
C = p.pivot_table(index="seg", columns="year", values="fatal", aggfunc="size", fill_value=0).reindex(index=range(nseg), columns=years, fill_value=0)
K = p.pivot_table(index="seg", columns="year", values="killed", aggfunc="sum", fill_value=0).reindex(index=range(nseg), columns=years, fill_value=0)
Sh = C / C.sum(0); KS = K / K.sum(0).replace(0, 1)

def feats(t):
    o = t - H; rows = []
    for s in range(nseg):
        lag = lambda k: Sh.loc[s, o - k] if (o - k) in Sh.columns else Sh.loc[s, o]
        hist = Sh.loc[s, [y for y in years if y <= o]]
        nb = [Sh.loc[j, o] for j in (s - 1, s + 1) if 0 <= j < nseg]
        rows.append({"seg": s, "year": t, "sh0": lag(0), "sh1": lag(1), "sh2": lag(2), "hist_mean": hist.mean(), "hist_max": hist.max(),
                     "nb0": np.mean(nb), "ksh0": KS.loc[s, o], "chainage": s * SEG + SEG / 2, "trend": (lag(0) - lag(2)) / 2,
                     "y": Sh.loc[s, t] if t in Sh.columns else np.nan, "n_t": C.loc[s, t] if t in C.columns else np.nan})
    return pd.DataFrame(rows)
FE = ["sh0", "sh1", "sh2", "hist_mean", "hist_max", "nb0", "ksh0", "chainage", "trend"]
panel = pd.concat([feats(t) for t in range(2013 + H, 2023)] + [feats(2027)], ignore_index=True)
rf = lambda: RandomForestRegressor(n_estimators=800, min_samples_leaf=3, random_state=SEED)
K10 = max(1, int(round(.1 * nseg)))
def score(te, pr):
    y = te.y.values; n = te.n_t.values; top = np.argsort(-pr)[:K10]; ttop = set(np.argsort(-y)[:K10])
    return {"Spearman": stats.spearmanr(y, pr)[0], "hit_rate_%": 100 * n[top].sum() / n.sum(), "top10_overlap": len(set(top) & ttop)}

ev, res = [], []
for t in [2020, 2021, 2022]:
    tr = panel[(panel.year < t) & panel.y.notna()]; te = panel[panel.year == t]
    m = rf().fit(tr[FE], tr.y); pr = m.predict(te[FE])
    res.extend((te.y.values - pr) / np.sqrt(pr + 1e-3))
    for name, q in [("Random forest (5-yr ahead)", pr), ("Persistence (share 5 yrs earlier)", te.sh0.values), ("Historical mean share", te.hist_mean.values)]:
        ev.append({"test_year": t, "origin": t - H, "model": name, **score(te, q)})
ev = pd.DataFrame(ev); ev.round(3).to_csv(os.path.join(OUT, "seg2027_backtest.csv"), index=False)
evm = ev.groupby("model")[["Spearman", "hit_rate_%", "top10_overlap"]].mean().round(3)
print(ev.round(3).to_string()); print(evm)

# final model: all target years 2018–2022, predict 2027 from 2022
tr = panel[panel.y.notna()]; f = panel[panel.year == 2027].copy()
m = rf().fit(tr[FE], tr.y); f["share"] = np.clip(m.predict(f[FE]), 0, None); f["share"] /= f.share.sum()
A = pd.read_csv(os.path.join(OUT, "forecast_annual_2023_2027.csv")); r27 = A[A.year == 2027].iloc[0]
oc = C[2022].sum() / d[d.year == 2022].shape[0]                          # share of 2022 records located on the corridor
TOT = {"ensemble": r27.ensemble * oc, "plateau": r27["Random Forest (lags)"] * oc, "trend": r27["Poisson GLM (trend + month)"] * oc}
f["exp_ensemble"] = f.share * TOT["ensemble"]; f["exp_plateau"] = f.share * TOT["plateau"]; f["exp_trend"] = f.share * TOT["trend"]

def gistar(x, band=1):
    x = np.asarray(x, float); n = len(x); xb = x.mean(); s = np.sqrt((x ** 2).mean() - xb ** 2); z = np.zeros(n)
    for i in range(n):
        j = np.arange(max(0, i - band), min(n, i + band + 1)); w = len(j)
        den = s * np.sqrt((n * w - w ** 2) / (n - 1)); z[i] = (x[j].sum() - xb * w) / den if den > 0 else 0
    return z
# Monte-Carlo: model error (back-test residuals, scaled) + Poisson noise, total drawn from the ensemble interval
rng = np.random.default_rng(SEED); NS = 4000; res = np.asarray(res)
sh = np.clip(f.share.values[None] + rng.choice(np.r_[res, -res], size=(NS, nseg)) * np.sqrt(f.share.values + 1e-3)[None], 0, None)
sh /= sh.sum(1, keepdims=True)
tot = np.clip(rng.normal(r27.ensemble, (r27.pi90_high - r27.pi90_low) / (2 * 1.645), NS), 100, None) * oc
sims = rng.poisson(sh * tot[:, None])
thr = np.sort(sims, 1)[:, -K10][:, None]
f["p_top10"] = ((sims >= thr) & (sims > 0)).mean(0)
Z = np.array([gistar(s) for s in sims]); f["p_hot95"] = (Z >= 1.96).mean(0)
f["gistar_z_expected"] = gistar(f.exp_ensemble.values)
f["lo90"], f["hi90"] = np.percentile(sims, 5, 0), np.percentile(sims, 95, 0)
f["from_km"], f["to_km"] = f.seg * SEG, f.seg * SEG + SEG
f["crashes_2022"] = C[2022].values; f["mean_2013_2022"] = C.mean(1).values
hs = pd.read_csv(os.path.join(OUT, "hotspots_gistar_2km.csv"))[["seg", "class", "top_location_names"]]
f = f.merge(hs, on="seg", how="left")
f["tier"] = np.select([f.p_hot95 >= .5, f.p_hot95 >= .2, f.p_top10 >= .2], ["Likely hot spot", "Possible hot spot", "Busy, unlikely hot spot"], "Low")
cols = ["seg", "from_km", "to_km", "top_location_names", "crashes_2022", "mean_2013_2022", "share", "exp_ensemble", "lo90", "hi90", "exp_plateau", "exp_trend", "p_top10", "p_hot95", "gistar_z_expected", "tier", "class"]
out = f[cols].rename(columns={"class": "gistar_class_2013_2022"}).sort_values("p_hot95", ascending=False)
out.round(4).to_csv(os.path.join(OUT, "forecast_segments_2027.csv"), index=False)
S = {"backtest_mean": evm.reset_index().to_dict("records"), "oncorr_share_2022": round(oc, 3), "totals_oncorridor": {k: round(v) for k, v in TOT.items()},
     "n_likely": int((f.tier == "Likely hot spot").sum()), "n_possible": int((f.tier == "Possible hot spot").sum()),
     "top": out.head(12)[["from_km", "to_km", "top_location_names", "crashes_2022", "exp_ensemble", "lo90", "hi90", "p_top10", "p_hot95", "tier"]].round(2).to_dict("records")}
json.dump(S, open(os.path.join(OUT, "seg2027_summary.json"), "w"), indent=1, default=float)
print(json.dumps(S["totals_oncorridor"]), S["n_likely"], S["n_possible"]); print(out.head(15)[["from_km", "top_location_names", "crashes_2022", "exp_ensemble", "lo90", "hi90", "p_top10", "p_hot95", "tier"]].round(2).to_string())
```


### fig_seg_2027.py

```python
"""fig_seg_2027.py — Map and corridor profile of 2-km sections likely to be crash hot spots in 2027.
Usage: python3 -I fig_seg_2027.py <out_dir>"""
import sys, os, json, numpy as np, pandas as pd
sys.path.insert(0, os.path.dirname(__file__))
import style; style.setup()
import basemap_ne
import matplotlib.pyplot as plt
from matplotlib.patches import Patch, Rectangle
from matplotlib.lines import Line2D
OUT = sys.argv[1]; INK2, GRID = "#52514e", "#e4e3df"
f = pd.read_csv(os.path.join(OUT, "forecast_segments_2027.csv")); cl = pd.read_csv(os.path.join(OUT, "centreline_derived.csv"))
S = json.load(open(os.path.join(OUT, "seg2027_summary.json"))); bt = {r["model"]: r for r in S["backtest_mean"]}
TIERS = [("Very likely (P ≥ 50%)", .5, "#b3261e"), ("Likely (30–50%)", .3, "#eb6834"), ("Possible (15–30%)", .15, "#f4b393"), ("Unlikely (< 15%)", 0, "#c9c8c1")]
def tier(pv):
    for lab, t, c in TIERS:
        if pv >= t: return lab, c
f["tier2"], f["col"] = zip(*f.p_top10.map(tier))
f = f.sort_values("from_km").reset_index(drop=True)
def label(r):
    n = str(r.top_location_names).split(",")[0].strip().title()
    return {"Paved": "Deeper Life / Lotto", "Km122-Guru Area": "km 10–12"}.get(n, n)
f["lab"] = f.apply(label, axis=1)
def seg_xy(a, b):
    ts = np.r_[a, cl.chainage_km[(cl.chainage_km > a) & (cl.chainage_km < b)].values, min(b, cl.chainage_km.max())]
    return np.interp(ts, cl.chainage_km, cl.lon), np.interp(ts, cl.chainage_km, cl.lat)
ASP = 1 / np.cos(np.deg2rad(6.9))
fig = plt.figure(figsize=(14, 12.2)); gs = fig.add_gridspec(2, 2, height_ratios=[1.45, 1], width_ratios=[1.05, 1], hspace=0.22, wspace=0.12)

def draw_map(ax, XL, YL, lw_hot, lw_low, nlab, credit=True, scale_km=20, exclude=None, sides=None):
    basemap_ne.draw(ax, extent=XL + YL)
    ax.plot(cl.lon, cl.lat, color="#fcfcfb", lw=lw_low + 4, solid_capstyle="round", zorder=1, alpha=.9)
    for r in f.itertuples():
        x, y = seg_xy(r.from_km, r.to_km); hot = r.p_top10 >= .15
        ax.plot(x, y, color=r.col, lw=lw_hot if hot else lw_low, solid_capstyle="butt", zorder=3 if hot else 2)
    top = f[(f.p_top10 >= .3)]
    vis = top[[XL[0] < np.interp(m, cl.chainage_km, cl.lon) < XL[1] and YL[0] < np.interp(m, cl.chainage_km, cl.lat) < YL[1] for m in (top.from_km + 1)]]
    if exclude is not None:
        ex = [(exclude[0][0] < np.interp(m, cl.chainage_km, cl.lon) < exclude[0][1]) and (exclude[1][0] < np.interp(m, cl.chainage_km, cl.lat) < exclude[1][1]) for m in (vis.from_km + 1)]
        vis = vis[~np.array(ex, bool)] if len(vis) else vis
    vis = vis.sort_values("p_top10", ascending=False).head(nlab).sort_values("from_km")
    for i, r in enumerate(vis.itertuples()):
        mx, my = np.interp(r.from_km + 1, cl.chainage_km, cl.lon), np.interp(r.from_km + 1, cl.chainage_km, cl.lat)
        side = (sides or {}).get(int(r.from_km), 1 if i % 2 == 0 else -1)
        ax.annotate(f"{r.lab}\nkm {r.from_km}–{r.to_km} · {100 * r.p_top10:.0f}%", (mx, my), xytext=(30 * side, 6 * (i % 3 - 1)), textcoords="offset points",
                    ha="left" if side > 0 else "right", va="center", fontsize=7.6, color="#0b0b0b", zorder=6,
                    bbox=dict(boxstyle="round,pad=0.2", fc="#fcfcfb", ec=r.col, lw=1, alpha=.95), arrowprops=dict(arrowstyle="-", color=INK2, lw=.6))
    ax.set_xlim(*XL); ax.set_ylim(*YL); ax.set_aspect(ASP); ax.set_xlabel("Longitude (°E)"); ax.set_ylabel("Latitude (°N)")
    dlon = scale_km / (111.32 * np.cos(np.deg2rad(6.9))); x0, y0 = XL[1] - (XL[1] - XL[0]) * .05 - dlon, YL[0] + (YL[1] - YL[0]) * .06
    ax.plot([x0, x0 + dlon], [y0, y0], color="#0b0b0b", lw=2.5, solid_capstyle="butt", zorder=7); ax.plot([x0, x0 + dlon / 2], [y0, y0], color="#fcfcfb", lw=1.2, solid_capstyle="butt", zorder=7)
    ax.text(x0 + dlon / 2, y0 + (YL[1] - YL[0]) * .015, f"{scale_km} km", fontsize=7, ha="center", zorder=7)
    if credit: ax.text(XL[1] - (XL[1] - XL[0]) * .01, YL[0] + (YL[1] - YL[0]) * .01, basemap_ne.CREDIT, fontsize=6.2, color="#3a3936", ha="right", va="bottom", bbox=dict(boxstyle="round,pad=0.2", fc="#fcfcfb", ec="none", alpha=.85), zorder=7)

ax = fig.add_subplot(gs[0, 0]); XL, YL = (3.20, 4.05), (6.45, 7.50)
zx, zy = (3.33, 3.47), (6.57, 6.76)
draw_map(ax, XL, YL, 8, 3.5, 4, exclude=(zx, zy), sides={26: 1, 54: 1}); ax.add_patch(Rectangle((zx[0], zy[0]), zx[1] - zx[0], zy[1] - zy[0], fill=False, ec="#0b0b0b", lw=1.2, ls="--", zorder=8))
ax.text(zx[1] + .01, zy[1], "(b)", fontsize=9, weight="bold", zorder=8)
ax.annotate("N", xy=(XL[0] + 0.05, YL[1] - 0.04), xytext=(XL[0] + 0.05, YL[1] - 0.12), ha="center", fontsize=10, weight="bold", arrowprops=dict(arrowstyle="-|>", color="#0b0b0b", lw=1.5), zorder=7)
ax.legend(handles=[Line2D([], [], color=c, lw=6, label=l) for l, _, c in TIERS], title="Probability of being among the\nbusiest 10% of sections in 2027", title_fontsize=8.5,
          loc="upper left", bbox_to_anchor=(0.09, 0.995), fontsize=8.3, frameon=True, facecolor="#fcfcfb", edgecolor=GRID, framealpha=.95)
ax.set_title("(a) Whole corridor, 2-km sections")
ax = fig.add_subplot(gs[0, 1]); draw_map(ax, zx, zy, 12, 5, 8, credit=False, scale_km=2, sides={2: 1, 4: 1, 8: 1, 12: -1, 14: 1, 18: -1}); ax.set_title("(b) Lagos end (km 0–30), enlarged"); ax.set_ylabel("")

ax = fig.add_subplot(gs[1, :])
x = f.from_km + 1
ax.bar(x, f.exp_ensemble, width=1.7, color=f.col, zorder=2)
ax.vlines(x, f.lo90, f.hi90, color=INK2, lw=.9, zorder=3)
ax.scatter(x, f.crashes_2022, marker="_", s=90, color="#0b0b0b", lw=1.6, zorder=4)
for r in f[f.p_top10 >= .3].itertuples():
    ax.text(r.from_km + 1, r.hi90 + 3, r.lab, rotation=90, ha="center", va="bottom", fontsize=7.6, color=INK2)
ax.set_xlim(-1, f.to_km.max() + 1); ax.set_ylim(0, f.hi90.max() * 1.42)
ax.set_xlabel("Chainage (km from Ojota, Lagos → Ibadan)"); ax.set_ylabel("Crashes per 2-km section"); ax.grid(axis="x", visible=False)
ax.legend(handles=[Patch(color=c, label=l) for l, _, c in TIERS] + [Line2D([], [], color=INK2, lw=1, label="90% interval"), Line2D([], [], marker="_", ls="", color="#0b0b0b", markersize=10, mew=1.6, label="Observed 2022")],
          loc="upper right", fontsize=8, ncol=3, frameon=True, facecolor="#fcfcfb", edgecolor=GRID)
ax.set_title(f"(c) Expected crashes per section in 2027 (ensemble total; plateau–trend range ×{S['totals_oncorridor']['plateau'] / S['totals_oncorridor']['ensemble']:.2f}–×{S['totals_oncorridor']['trend'] / S['totals_oncorridor']['ensemble']:.2f})")
rf_, hm = bt["Random forest (5-yr ahead)"], bt["Historical mean share"]
fig.suptitle("Sections of the Lagos–Ibadan Expressway likely to be crash hot spots in 2027", x=0.01, ha="left", fontsize=15, weight="bold", y=0.995)
fig.text(0.01, 0.962, f"Random-forest forecast made five years ahead from the 2013–2022 record. Back-test (2020–2022, five years ahead): rank correlation {rf_['Spearman']:.2f} (historical average {hm['Spearman']:.2f}); "
         f"on average {rf_['top10_overlap']:.0f} of the 6 busiest sections correctly identified.", fontsize=9.4, color=INK2)
fig.savefig(os.path.join(OUT, "figures", "fig_hotspots_2027.png"), dpi=200, bbox_inches="tight"); plt.close()
f[["from_km", "to_km", "lab", "top_location_names", "crashes_2022", "mean_2013_2022", "exp_ensemble", "lo90", "hi90", "exp_plateau", "exp_trend", "p_top10", "p_hot95", "tier2"]].sort_values("p_top10", ascending=False).round(3).to_csv(os.path.join(OUT, "hotspot_sections_2027.csv"), index=False)
print("ok")
```


### fig_seg_2027_profile.py

```python
"""fig_seg_2027_profile.py — Standalone chart: expected crashes per 2-km section in 2027.
Usage: python3 -I fig_seg_2027_profile.py <out_dir>"""
import sys, os, json, numpy as np, pandas as pd
sys.path.insert(0, os.path.dirname(__file__))
import style; style.setup()
import matplotlib.pyplot as plt
from matplotlib.patches import Patch
from matplotlib.lines import Line2D
OUT = sys.argv[1]; INK2, GRID = "#52514e", "#e4e3df"
f = pd.read_csv(os.path.join(OUT, "hotspot_sections_2027.csv")).sort_values("from_km").reset_index(drop=True)
S = json.load(open(os.path.join(OUT, "seg2027_summary.json"))); T = S["totals_oncorridor"]
TIERS = [("Very likely hot spot (P ≥ 50%)", .5, "#b3261e"), ("Likely (30–50%)", .3, "#eb6834"), ("Possible (15–30%)", .15, "#f4b393"), ("Unlikely (< 15%)", 0, "#c9c8c1")]
f["col"] = f.p_top10.map(lambda p: next(c for _, t, c in TIERS if p >= t))
fig, ax = plt.subplots(figsize=(14, 6.4))
x = f.from_km + 1
ax.bar(x, f.exp_ensemble, width=1.7, color=f.col, zorder=2)
ax.vlines(x, f.lo90, f.hi90, color=INK2, lw=.9, zorder=3)
ax.scatter(x, f.crashes_2022, marker="_", s=90, color="#0b0b0b", lw=1.6, zorder=4)
for r in f[f.p_top10 >= .3].itertuples():
    ax.text(r.from_km + 1, r.hi90 + 3, f"{r.lab} ({r.exp_ensemble:.0f})", rotation=90, ha="center", va="bottom", fontsize=8, color=INK2)
ax.set_xlim(-1, f.to_km.max() + 1); ax.set_ylim(0, f.hi90.max() * 1.45); ax.set_xticks(range(0, 121, 10))
ax.set_xlabel("Chainage (km from Ojota, Lagos → Ibadan)"); ax.set_ylabel("Expected crashes per 2-km section, 2027"); ax.grid(axis="x", visible=False)
ax.legend(handles=[Patch(color=c, label=l) for l, _, c in TIERS] + [Line2D([], [], color=INK2, lw=1, label="90% interval"), Line2D([], [], marker="_", ls="", color="#0b0b0b", markersize=10, mew=1.6, label="Observed 2022")],
          loc="upper right", fontsize=8.5, ncol=3, frameon=True, facecolor="#fcfcfb", edgecolor=GRID)
ax.set_title("Expected road traffic crashes per 2-km section of the Lagos–Ibadan Expressway in 2027", loc="left", fontsize=13, weight="bold", pad=26)
ax.text(0, 1.015, f"Bars: random-forest share of crashes × 2027 ensemble total (about {T['ensemble']:,} located crashes). Colour: probability of being among the busiest 10% of sections. "
        f"Plateau or trend scenarios scale all bars by ×{T['plateau'] / T['ensemble']:.2f} or ×{T['trend'] / T['ensemble']:.2f}.", transform=ax.transAxes, fontsize=8.6, color=INK2)
fig.savefig(os.path.join(OUT, "figures", "fig_expected_crashes_2027.png"), dpi=200, bbox_inches="tight"); plt.close(); print("ok")
```


---

## C. LLM-assisted geoparsing


### geo10_prepare.py

```python
"""geo10_prepare.py — Build the geoparser input for each year (file_year, row_order, SN, COMMAND, LOCATION) from the
FRSC returns WITHOUT coordinates (RTC_LAG_IBA). Rows are filtered exactly as in 01_clean.py (non-empty DATE), so
row_order matches the cleaned attribute table.
Usage: python3 -I geo10_prepare.py <raw_no_coords_dir> <out_dir>"""
import sys, os, pandas as pd
RAW, OUT = sys.argv[1], sys.argv[2]; os.makedirs(OUT, exist_ok=True)
FILES = {2013: "2013 MERGED ROUTES.csv", 2014: "2014 MERGED ROUTES.csv", 2015: "2015 MERGED ROUTES.csv", 2016: "2016 MERGED ROUTES.csv",
         2017: "2017 MERGED ROUTES CSV.csv", 2018: "2018 MERGED ROUTES CSV.csv", 2019: "2019 MERGED ROUTES.csv", 2020: "2020 MERGED ROUTES CSV.csv",
         2021: "2021 MERGED ROUTES CSV.csv", 2022: "2022 MERGED ROUTES CSV.csv"}
for yr, fn in FILES.items():
    d = pd.read_csv(os.path.join(RAW, fn), encoding="latin1", dtype=str, low_memory=False)
    d.columns = [c.strip() for c in d.columns]
    d = d[d["DATE"].notna() & (d["DATE"].str.strip() != "")].copy()
    if yr == 2019 and "UNKNOWN" in d.columns:            # 2019 header shift: place name sits in the extra column
        d["LOCATION"] = d["UNKNOWN"].where(d["UNKNOWN"].notna(), d["LOCATION"])
    o = pd.DataFrame({"file_year": yr, "row_order": range(len(d)), "SN": d["S/N"].values, "COMMAND": d["COMMAND"].values, "LOCATION": d["LOCATION"].fillna("(blank)").values})
    o.to_csv(os.path.join(OUT, f"in_{yr}.csv"), index=False); print(yr, len(o))
```


### geo_01_geocode.py

```python
"""
geo_01_geocode.py — Geocode FRSC crash records from free-text LOCATION names.

Method (hierarchical, every coordinate traceable to a source record):
  1. Name normalisation (alias table curated by the LLM: misspellings, abbreviations, qualifiers).
  2. Gazetteer = 2013–2022 records with recorded coordinates (target year excluded), weighted so that
     'placeholder' coordinates (one coordinate reused for >=5 different names) are used only as a fallback.
  3. Matching order: exact name + same command  ->  exact name (any command)  ->  landmark sub-token
     ->  fuzzy name (difflib >= 0.86)  ->  'KMnn'-only labels via command-specific KM→chainage calibration.
  4. Each output row carries method, matched key, number of supporting records, spread (km) and a confidence class.

Usage: python3 -I geo_01_geocode.py <clean_all_years.csv> <centreline.csv> <target.csv> <target_year> <out_csv>
"""
import sys, re, difflib, numpy as np, pandas as pd
from scipy import stats

CLEAN, CL, TARGET, TY, OUTCSV = sys.argv[1], sys.argv[2], sys.argv[3], int(sys.argv[4]), sys.argv[5]

# ---------------- 1. NORMALISATION (alias table) ----------------
PHRASE = [  # (regex, replacement) applied to upper-cased text, in order
    (r"\bF\s*/\s*S\b", " FILLING STATION "), (r"\bB\s*/\s*S(TOP)?\b", " BUS STOP "), (r"\bB4\b", " BEFORE "),
    (r"\bLAG\s*/\s*IBA?D?(AN)?\b", " "), (r"\bIBD\s*/\s*LAG\b", " "),
    (r"\bBRD\b|\bBRDG\b|\bBRIDGR\b|\bBIDGE\b|\bBRIGDE\b", " BRIDGE "),
    (r"LONG\s*-?\s*BRIDGE|LONGBRIDGE", " LONG BRIDGE "), (r"\bCON\s+OIL\b", " CONOIL "),
    (r"\bIHSARA\b|\bISHARA\b", " ISARA "), (r"\bKWAKY(I)?AMA?\b", " KWAKYAMA "),
    (r"\bSA?NDCRETE?\b|\bSADCRETE\b", " SANDCRETE "), (r"\bQUARY\b", " QUARRY "),
    (r"STRAIGHT\s*GATE", " STRAIGHT GATE "), (r"\bTOLLGATE\b", " TOLL GATE "),
    (r"\b4\s*SQUARE\b|\bFOUR\s*SQUARE\b", " FOURSQUARE "), (r"\bALAPERA\b", " ALAPERE "),
    (r"\bASEESE\b|\bASHESHE\b", " ASESE "), (r"\bRAMONA\b", " ROMONA "), (r"\bSEIKO\b", " SEKO "),
    (r"\bSONYO\b", " SANYO "), (r"GURU\s*-?\s*MARA[JG]I|GURUMARA[JG]I|GURU\s*MAHAJI|GURUMAHAJI", " GURU MARAJI "),
    (r"\bGURU\s+ARA\b", " GURU "), (r"\bQUARRY EXPRESS\b", " EXPRESS QUARRY "), (r"\bWHICHTEC\b|\bWICHTEC\b", " WICHTECH "),
    (r"\bMARKT\b", " MARKET "), (r"\bDEPER\b", " DEEPER "), (r"\bMOTO?R?WAYS?\b", " MOTORWAYS "),
    (r"\bMOBILE?\b", " MOBIL "), (r"\bFUEL STATION\b|\bPETROL STATION\b", " FILLING STATION "),
    (r"\bAGIRAH\b|\bALHAJIRA\b", " ALHAJIRAH "), (r"\bTABIEORA\b", " TABIEORA "), (r"\bOFF\b", " OFFICE "),
    (r"\bNIPOST\s+KARA\b", " KARA "), (r"\bMOGBORO\b", " MAGBORO "), (r"\bRAMODA\b", " ROMONA "), (r"\bTABIEORAR\b", " TABIEORA "), (r"\bJN\b|\bJUNC\b", " JUNCTION "), (r"\bOPP\.", " OPP "),
]
QUAL = r"\b(OPP|OPPOSITE|AFTER|BEFORE|NEAR|CLOSE|BY|AREA|INWARD|OUTWARD|LAGOS|IBADAN|ABOUT|TO|THE|AT)\b"
def norm(s):
    s = " " + str(s).upper() + " "
    for a, b in PHRASE: s = re.sub(a, b, s)
    s = re.sub(r"[^A-Z0-9\.\s,/()]", " ", s)
    return re.sub(r"\s+", " ", s).strip()
def km_value(s):
    m = re.search(r"\bKM\s*\.?\s*(\d{1,3}(?:\.\d+)?)|(\d{1,3}(?:\.\d+)?)\s*KM\b", s)
    return float(m.group(1) or m.group(2)) if m else np.nan
def core(s):
    s = re.sub(r"\bKM\s*\.?\s*\d{1,3}(\.\d+)?\s*-?|\b\d{1,3}(\.\d+)?\s*KM\b", " ", s)
    s = re.sub(r"[()/,\-\.]", " ", s); s = re.sub(QUAL, " ", s)
    s = re.sub(r"\b(BUS STOP|JUNCTION|TURNING)\b", " ", s)
    return re.sub(r"\s+", " ", s).strip()
GENERIC = {"BRIDGE", "FILLING STATION", "FILLING", "STATION", "QUARRY", "MARKET", "CHURCH", "COLLEGE", "PARK", "OFFICE", "HOSPITAL",
           "UNIVERSITY", "BUS STOP", "JUNCTION", "TURNING", "AREA", "GATE", "HOME", "BASE", "CAR PARK", "COMPANY", "OIL", "LIFE"}
def parts(s):
    """Landmark sub-tokens for compound names, e.g. 'AREPO (PUNCH)', 'OPIC/MFM', 'PAKURO B/S OPP DEEPER LIFE',
    'KM115 EXPRESS QUARRY OPP. MOTHERLESS BABY HOME'. Longest n-grams first; generic words never match alone."""
    s2 = re.sub(r"\bKM\s*\.?\s*\d{1,3}(\.\d+)?\s*-?|\b\d{1,3}(\.\d+)?\s*KM\b", " ", s)
    chunks = [core(x) for x in re.split(r"[()/,]| - |\bOPP\b|\bAFTER\b|\bBEFORE\b|\bTO\b|\bNEAR\b", s2)]
    out = []
    for ch in [c for c in chunks if c]:
        w = ch.split()
        for n in range(len(w), 0, -1):
            for i in range(len(w) - n + 1):
                gram = " ".join(w[i:i + n])
                if len(gram) >= 3 and gram not in GENERIC and gram not in out: out.append(gram)
    return out
def family(cmd):
    m = re.search(r"RS\s*(\d+)[\.,](\d+)", str(cmd).upper()); return f"RS{m.group(1)}.{m.group(2)}" if m else ""

# ---------------- 2. GAZETTEER ----------------
cl = pd.read_csv(CL)
KX, KY = 111.32 * np.cos(np.deg2rad(6.9)), 110.57
def km_dist(la1, lo1, la2, lo2): return np.hypot((np.asarray(lo1) - lo2) * KX, (np.asarray(la1) - la2) * KY)
def chain_to_ll(c): return np.interp(c, cl.chainage_km, cl.lat), np.interp(c, cl.chainage_km, cl.lon)

g = pd.read_csv(CLEAN, low_memory=False)
g = g[(g.file_year != TY) & g.coord_source.astype(str).str.startswith("recorded") & g.lat.notna()].copy()
g["n_norm"] = g.location_raw.map(norm); g["k"] = g.n_norm.map(core); g["fam"] = g.command.map(family)
ck = g.lat.round(4).astype(str) + "," + g.lon.round(4).astype(str)
g["placeholder"] = ck.map(g.assign(ck=ck).groupby("ck").k.nunique()) >= 5
g = g[g.k.str.len() >= 3]

def summarise(sub):
    """Dominant spatial cluster (points within 1 km of the densest point); non-placeholder coordinates preferred."""
    good = sub[~sub.placeholder]; use = good if len(good) else sub
    la, lo = use.lat.values, use.lon.values
    D = km_dist(la[:, None], lo[:, None], la[None], lo[None])
    centre = (D <= 1.0).sum(1).argmax(); inc = D[centre] <= 1.0
    cla, clo = np.median(la[inc]), np.median(lo[inc])
    spread = float(np.median(km_dist(la[inc], lo[inc], cla, clo))) if inc.sum() > 1 else np.nan
    return pd.Series({"lat": cla, "lon": clo, "n": int(inc.sum()), "n_total": len(use), "cluster_share": inc.mean(),
                      "n_years": len(set(use.file_year.values[inc].astype(int))),
                      "n_placeholder_only": int(len(good) == 0), "spread_km": spread,
                      "years": ",".join(map(str, sorted(set(use.file_year.values[inc].astype(int)))))})
gz_cmd = g.groupby(["k", "fam"]).apply(summarise, include_groups=False)
gz = g.groupby("k").apply(summarise, include_groups=False)
keys = list(gz.index)

# KM-only calibration per command: chainage ~ KM label using records whose label is a bare 'KMnn'
cl_line = np.c_[cl.lon * KX, cl.lat * KY]; seg = np.diff(cl_line, axis=0); cum = cl.chainage_km.values
def chainage(la, lo):
    P = np.c_[np.asarray(lo) * KX, np.asarray(la) * KY]; A = cl_line[:-1]
    L2 = (seg ** 2).sum(1); t = np.clip(((P[:, None] - A[None]) * seg[None]).sum(2) / L2, 0, 1)
    D = np.linalg.norm(P[:, None] - (A[None] + t[..., None] * seg[None]), axis=2); k = D.argmin(1)
    return cum[k] + t[np.arange(len(P)), k] * np.linalg.norm(seg, axis=1)[k], D.min(1)
# ---- command territories (chainage inter-quartile range ±10 km of each command's own recorded, non-placeholder points)
g["ch"], g["dax"] = chainage(g.lat, g.lon)
terr = g[(~g.placeholder) & (g.dax <= 1.0)].groupby("fam").ch.agg(lambda x: (np.percentile(x, 25), np.percentile(x, 75), len(x)))
def in_territory(fam, ch):
    if fam not in terr.index or terr[fam][2] < 15 or np.isnan(ch): return None
    lo, hi, _ = terr[fam]; return (lo - 10) <= ch <= (hi + 10)
PTS = {k: v for k, v in g.groupby("k")}
def clusters(sub):
    """Greedy 1-km clustering; returns list of dicts sorted by size."""
    good = sub[~sub.placeholder]; use = good if len(good) else sub
    la, lo, yr, fm = use.lat.values, use.lon.values, use.file_year.values.astype(int), use.fam.values
    D = km_dist(la[:, None], lo[:, None], la[None], lo[None]); left = np.ones(len(la), bool); out = []
    while left.any():
        cnt = ((D <= 1.0) & left[None] & left[:, None]).sum(1); cnt[~left] = -1; c = cnt.argmax()
        inc = (D[c] <= 1.0) & left; left &= ~inc
        cla, clo = np.median(la[inc]), np.median(lo[inc])
        out.append(dict(lat=cla, lon=clo, n=int(inc.sum()), n_years=len(set(yr[inc])), share=inc.sum() / len(la),
                        years=",".join(map(str, sorted(set(yr[inc])))), fams=set(fm[inc]), placeholder_only=len(good) == 0,
                        spread=float(np.median(km_dist(la[inc], lo[inc], cla, clo))) if inc.sum() > 1 else np.nan))
    return out
def pick(key, fam):
    """Choose among the spatial clusters for a gazetteer key: size × same-command bonus × territory plausibility."""
    cl_ = clusters(PTS[key]); best, bs = None, -1
    for c in cl_:
        chn, dax = chainage([c["lat"]], [c["lon"]]); dax = float(dax[0]); chn = chn[0] if dax <= 5.0 else np.nan
        terr_ok = in_territory(fam, chn)
        road = 1.0 if dax <= 5.0 else 0.2          # all records are on the Lagos–Ibadan route: far-off sites are implausible
        sc = c["n"] * (2.0 if fam in c["fams"] else 1.0) * (0.25 if terr_ok is False else 1.0) * road * (1 + 0.2 * c["n_years"])
        if sc > bs: best, bs = dict(c, territory_ok=terr_ok, n_clusters=len(cl_), dax=dax), sc
    rivals = [c for c in cl_ if (c["lat"], c["lon"]) != (best["lat"], best["lon"])]
    best["rival_n"] = max([c["n"] for c in rivals], default=0)
    best["rival_site"] = (f"{rivals[0]['lat']:.5f},{rivals[0]['lon']:.5f} (n={rivals[0]['n']})" if rivals and rivals[0]["n"] >= 2 else "")
    return best
g["kmv"] = g.n_norm.map(km_value)
bare = g[g.k.eq("") | g.n_norm.str.fullmatch(r"KM\s*\d{1,3}(\.\d+)?")] if False else g[g.n_norm.str.fullmatch(r"KM\s*\d{1,3}(\.\d+)?") & ~g.placeholder]
CAL = {}
if len(bare):
    ch, dd = chainage(bare.lat, bare.lon); bare = bare.assign(ch=ch, dd=dd)[dd < 1.5]
    for fam, s in bare.groupby("fam"):
        if len(s) >= 8 and s.kmv.nunique() >= 4:
            res = stats.theilslopes(s.ch, s.kmv); rho = stats.spearmanr(s.kmv, s.ch)[0]
            resid = s.ch - (res.intercept + res.slope * s.kmv)
            CAL[fam] = {"slope": res.slope, "intercept": res.intercept, "rho": rho, "mad_km": float(np.median(np.abs(resid))), "n": len(s)}
# also an exact-label lookup for bare KM labels per command
gz_km = bare.groupby(["fam", "kmv"]).apply(summarise, include_groups=False) if len(bare) else pd.DataFrame()

# ---------------- 3. MATCH ----------------
def conf(n, spread, placeholder_only, method, share=1.0, n_years=1):
    """High: same coordinate cluster confirmed in >=3 separate years, >=70% of the name's records agree, not placeholder-only.
    Medium: >=2 years or >=3 records agreeing (>=60%). Low: anything weaker, all fuzzy/landmark single-record matches."""
    fuzzy = method.startswith("fuzzy") or method.startswith("landmark")
    if share >= 0.7 and n_years >= 3 and not placeholder_only and not method.startswith("fuzzy"):
        return "High" if not fuzzy else "Medium"
    if share >= 0.6 and (n_years >= 2 or n >= 3): return "Medium"
    return "Low"

def _unused(): pass
def distinct(x):
    for gw in sorted(GENERIC, key=len, reverse=True): x = re.sub(rf"\b{gw}\b", " ", x)
    return re.sub(r"\s+", " ", x).strip()
def distinct_sim(a, b2):
    da, db = distinct(a), distinct(b2)
    return difflib.SequenceMatcher(None, da, db).ratio() if da and db else 0.0

def conf2(c, method):
    fuzzy = method.startswith("fuzzy") or method.startswith("landmark")
    if c["share"] >= 0.7 and c["n_years"] >= 3 and not c["placeholder_only"] and not method.startswith("fuzzy"): lvl = 1 if fuzzy else 0
    elif c["share"] >= 0.5 and (c["n_years"] >= 2 or c["n"] >= 3): lvl = 1
    else: lvl = 2
    if c["territory_ok"] is False: lvl += 1
    if c.get("rival_n", 0) >= 3: lvl = max(lvl, 1)          # competing site with >=3 records: cap at Medium
    if c.get("dax", 0) > 5.0: lvl = max(lvl, 2)              # > 5 km from the expressway: Low
    return ["High", "Medium", "Low", "Low"][min(lvl, 3)]

def geocode(loc, cmd):
    nm = norm(loc); k = core(nm); fam = family(cmd)
    def out(key, method):
        c = pick(key, fam)
        return dict(lat=c["lat"], lon=c["lon"], method=method, matched_key=key, n_support=c["n"], n_years=c["n_years"],
                    cluster_share=round(c["share"], 2), n_alt_sites=c["n_clusters"] - 1, spread_km=c["spread"], support_years=c["years"],
                    within_command_territory={True: "yes", False: "NO", None: "unknown"}[c["territory_ok"]], competing_site=c["rival_site"], confidence=conf2(c, method))
    if k:
        if k in PTS: return out(k, "exact name")
        for pk in parts(nm):
            if pk in PTS: return out(pk, "landmark token")
        best = [b2 for b2 in difflib.get_close_matches(k, keys, n=3, cutoff=0.86) if distinct_sim(k, b2) >= 0.8][:1]
        if best: return out(best[0], f"fuzzy name (similarity {difflib.SequenceMatcher(None, k, best[0]).ratio():.2f})")
        for pk in parts(nm):
            best = [b2 for b2 in difflib.get_close_matches(pk, keys, n=3, cutoff=0.86) if distinct_sim(pk, b2) >= 0.8][:1]
            if best: return out(best[0], f"fuzzy landmark token (similarity {difflib.SequenceMatcher(None, pk, best[0]).ratio():.2f})")
    kv = km_value(nm)
    base = dict(lat=np.nan, lon=np.nan, n_support=0, n_years=0, cluster_share=np.nan, n_alt_sites=0, spread_km=np.nan,
                support_years="", within_command_territory="unknown", competing_site="", confidence="None")
    if not np.isnan(kv):
        return dict(base, method="unresolved: KM label only (KM-post origin inconsistent in source data)", matched_key=f"KM{kv:g}")
    return dict(base, method="unresolved: name not in gazetteer", matched_key="")

t = pd.read_csv(TARGET, encoding="latin1", dtype=str, low_memory=False)
t.columns = [c.strip() for c in t.columns]
if "LONGITUDE CATEGORY" in t.columns: t = t.rename(columns={"LONGITUDE CATEGORY": "LONGITUDE"})
t = t[t["LOCATION"].notna()].copy()
res = pd.DataFrame([geocode(l, c) for l, c in zip(t["LOCATION"], t["COMMAND"])], index=t.index)
t = pd.concat([t, res.add_prefix("geo_")], axis=1)
t["geo_location_normalised"] = t["LOCATION"].map(lambda s: core(norm(s)))
ch, dd = chainage(t.geo_lat.fillna(0), t.geo_lon.fillna(0))
t["geo_chainage_km"] = np.where(t.geo_lat.notna(), ch, np.nan).round(2)
t["geo_dist_to_expressway_km"] = np.where(t.geo_lat.notna(), dd, np.nan).round(2)
t["geo_on_expressway"] = t.geo_dist_to_expressway_km <= 1.0
t.to_csv(OUTCSV, index=False)
print("calibrations:", {k: {kk: round(vv, 3) for kk, vv in v.items()} for k, v in CAL.items()})
print(t.geo_method.str.split(" \\(").str[0].value_counts().to_string()); print(t.geo_confidence.value_counts().to_string())
```


### geo10_combine.py

```python
"""geo10_combine.py — Stack the per-year geoparser outputs and attach the cleaned crash attributes.
Usage: python3 -I geo10_combine.py <geo_out_dir> <rtc_all_rows_with_attributes.csv> <combined_raw.csv>"""
import sys, glob, os, pandas as pd
GO, ATTR, OUTF = sys.argv[1:4]
g = pd.concat([pd.read_csv(f, low_memory=False) for f in sorted(glob.glob(os.path.join(GO, "geo_*.csv")))], ignore_index=True)
a = pd.read_csv(ATTR, low_memory=False)
m = g.merge(a, on=["file_year", "row_order"], how="left", suffixes=("", "_attr"))
m.to_csv(OUTF, index=False); print(m.shape)
```


### geo10_pass2.py

```python
"""geo10_pass2.py — second pass for names that occur in only one year: locate each record from OTHER records
with the same normalised name in the same year (leave-one-record-out). Never uses the record's own coordinate.
Usage: python3 -I geo10_pass2.py <combined_raw.csv> <attributes.csv> <centreline.csv> <out.csv>"""
import sys, runpy, numpy as np, pandas as pd
COMB, ATTR, CL, OUTF = sys.argv[1:5]
# import normalisation functions without running the geocoder main body
import os
src = open(os.path.join(os.path.dirname(os.path.abspath(__file__)), "geo_01_geocode.py")).read()
head = src[:src.index("# ---------------- 2. GAZETTEER")]
lib = {}; exec(compile(head.replace("CLEAN, CL, TARGET, TY, OUTCSV = sys.argv[1], sys.argv[2], sys.argv[3], int(sys.argv[4]), sys.argv[5]", ""), "norm", "exec"), lib)
norm, core, parts = lib["norm"], lib["core"], lib["parts"]
KX, KY = 111.32 * np.cos(np.deg2rad(6.9)), 110.57
cl = pd.read_csv(CL); line = np.c_[cl.lon * KX, cl.lat * KY]; seg = np.diff(line, axis=0); cum = cl.chainage_km.values
def chainage(la, lo):
    P = np.c_[np.asarray(lo) * KX, np.asarray(la) * KY]; A = line[:-1]; L2 = (seg ** 2).sum(1)
    t = np.clip(((P[:, None] - A[None]) * seg[None]).sum(2) / L2, 0, 1)
    D = np.linalg.norm(P[:, None] - (A[None] + t[..., None] * seg[None]), axis=2); k = D.argmin(1)
    return cum[k] + t[np.arange(len(P)), k] * np.linalg.norm(seg, axis=1)[k], D.min(1)

m = pd.read_csv(COMB, low_memory=False)
a = pd.read_csv(ATTR, low_memory=False)
a = a[a.coord_source.astype(str).str.startswith("recorded") & a.lat.notna()].copy()
a["k"] = a.LOCATION.map(lambda s: core(norm(s)))
ck = a.lat.round(4).astype(str) + "," + a.lon.round(4).astype(str)
a["ph"] = ck.map(a.assign(ck=ck).groupby("ck").k.nunique()) >= 5
todo = m.geo_method.str.startswith("unresolved: name")
n_fixed = 0
for i in m.index[todo]:
    y, ro, loc = m.at[i, "file_year"], m.at[i, "row_order"], m.at[i, "LOCATION"]
    k = core(norm(loc)); keys = [k] + parts(norm(loc))
    for kk in keys:
        if not kk: continue
        s = a[(a.file_year == y) & (a.k == kk) & ~((a.row_order == ro))]
        good = s[~s.ph]; s = good if len(good) else s
        if len(s) == 0: continue
        la, lo = s.lat.values, s.lon.values
        D = np.hypot((lo[:, None] - lo[None]) * KX, (la[:, None] - la[None]) * KY)
        c = (D <= 1.0).sum(1).argmax(); inc = D[c] <= 1.0
        cla, clo = np.median(la[inc]), np.median(lo[inc]); share = inc.mean()
        conf = "Medium" if (inc.sum() >= 3 and share >= 0.7 and len(good)) else "Low"
        ch_, dd = chainage([cla], [clo])
        if dd[0] > 5: conf = "Low"
        m.loc[i, ["geo_lat", "geo_lon", "geo_matched_key", "geo_n_support", "geo_n_years", "geo_cluster_share", "geo_support_years", "geo_confidence", "geo_competing_site", "geo_within_command_territory"]] = \
            [cla, clo, kk, int(inc.sum()), 1, round(share, 2), str(y), conf, "", "unknown"]
        m.loc[i, "geo_method"] = "same-year other records (name unique to this year)"
        m.loc[i, "geo_chainage_km"] = round(float(ch_[0]), 2); m.loc[i, "geo_dist_to_expressway_km"] = round(float(dd[0]), 2)
        m.loc[i, "geo_on_expressway"] = dd[0] <= 1.0
        n_fixed += 1; break
m.to_csv(OUTF, index=False); print("pass-2 resolved:", n_fixed, "of", int(todo.sum()))
```


### geo10_export.py

```python
"""geo10_export.py — Export geoparsed 2013–2022 crashes and segment hotspots.
Outputs: CSV (all records), point shapefile, 2-km segment polyline shapefile with Gi*, GeoJSON, KML, Leaflet HTML.
Usage: python3 -I geo10_export.py <combined_final.csv> <centreline.csv> <out_dir>"""
import sys, os, json, struct, numpy as np, pandas as pd
from scipy import stats
SRC, CL, OUT = sys.argv[1:4]; os.makedirs(os.path.join(OUT, "shapefiles"), exist_ok=True)
m = pd.read_csv(SRC, low_memory=False); cl = pd.read_csv(CL)
KX, KY = 111.32 * np.cos(np.deg2rad(6.9)), 110.57
num = lambda c: pd.to_numeric(m[c], errors="coerce")

d = pd.DataFrame({
    "YEAR": m.file_year.astype(int), "ROW": m.row_order.astype(int), "SN": m.SN.fillna("").astype(str).str.strip(),
    "COMMAND": m.COMMAND.fillna("").astype(str).str.strip(),
    "DATE": pd.to_datetime(m.date, errors="coerce").dt.strftime("%Y-%m-%d").fillna(""),
    "DATE_FLAG": np.where(m.date_method.astype(str).str.contains("ambiguous"), "day/month inferred", ""),
    "HOUR": num("hour"), "LOCATION": m.LOCATION.fillna("").astype(str).str.strip(),
    "MATCH_KEY": m.geo_matched_key.fillna(""), "METHOD": m.geo_method.fillna(""), "CONFIDENCE": m.geo_confidence.replace("None", "").fillna(""),
    "N_SUPPORT": num("geo_n_support").fillna(0).astype(int), "N_YEARS": num("geo_n_years").fillna(0).astype(int),
    "AGREE_SHR": num("geo_cluster_share").round(2), "ALT_SITE": m.geo_competing_site.fillna(""),
    "IN_TERR": m.geo_within_command_territory.fillna(""),
    "LAT": num("geo_lat").round(6), "LON": num("geo_lon").round(6),
    "CHAIN_KM": num("geo_chainage_km").round(2), "DIST_EXPWY": num("geo_dist_to_expressway_km").round(2),
    "VEH_TYPE": m.vehicle_type.fillna(""), "OWNERSHIP": m.ownership.fillna(""), "CAUSE": m.cause.fillna(""),
    "INJURED": num("injured").fillna(0).astype(int), "KILLED": num("killed").fillna(0).astype(int), "INVOLVED": num("involved").fillna(0).astype(int),
    "SEVERITY": m.severity.fillna(""), "DUPLICATE": np.where(m.is_duplicate == True, "yes", ""),
    "ORIG_LAT": num("lat").where(m.coord_source.astype(str).str.startswith("recorded")).round(6),
    "ORIG_LON": num("lon").where(m.coord_source.astype(str).str.startswith("recorded")).round(6),
})
d["ON_EXPWY"] = np.where(d.LAT.isna(), "", np.where(d.DIST_EXPWY <= 1.0, "yes", "no"))
d["ORIG_DIFF"] = np.hypot((d.LON - d.ORIG_LON) * KX, (d.LAT - d.ORIG_LAT) * KY).round(2)
d.to_csv(os.path.join(OUT, "RTC_Lagos_Ibadan_2013_2022_geoparsed.csv"), index=False)
pts = d[d.LAT.notna()].reset_index(drop=True)

# ---------------- generic shapefile writer (points = type 1, polylines = type 3) ----------------
def dbf_write(path, df):
    fields = []
    for c in df.columns:
        s = df[c]
        if pd.api.types.is_float_dtype(s): fields.append((c, "N", 14, 6 if c in ("LAT", "LON", "ORIG_LAT", "ORIG_LON") else 3))
        elif pd.api.types.is_integer_dtype(s): fields.append((c, "N", 10, 0))
        else: fields.append((c, "C", int(min(254, max(2, s.astype(str).map(lambda v: len(v.encode("latin1", "replace"))).max()))), 0))
    n = len(df); reclen = 1 + sum(f[2] for f in fields); hlen = 32 + 32 * len(fields) + 1
    b = bytearray(struct.pack("<BBBBIHH20x", 3, 126, 10, 16, n, hlen, reclen))
    for c, t, w, dec in fields: b += struct.pack("<11sc4xBB14x", c[:10].encode("ascii"), t.encode(), w, dec)
    b += b"\r"
    for i in range(n):
        b += b" "
        for c, t, w, dec in fields:
            v = df[c].iloc[i]
            if t == "N": s = "" if pd.isna(v) else (f"{float(v):.{dec}f}" if dec else str(int(v))); b += s.rjust(w)[:w].encode("ascii")
            else: b += str("" if pd.isna(v) else v).encode("latin1", "replace")[:w].ljust(w)
    b += b"\x1a"; open(path, "wb").write(b)
PRJ = 'GEOGCS["GCS_WGS_1984",DATUM["D_WGS_1984",SPHEROID["WGS_1984",6378137.0,298.257223563]],PRIMEM["Greenwich",0.0],UNIT["Degree",0.0174532925199433]]'
def shp_write(base, geoms, attrs, kind):
    allxy = np.vstack([np.atleast_2d(g) for g in geoms]); bbox = (allxy[:, 0].min(), allxy[:, 1].min(), allxy[:, 0].max(), allxy[:, 1].max())
    stype = 1 if kind == "point" else 3
    recs = []
    for g in geoms:
        if kind == "point": recs.append(struct.pack("<idd", 1, g[0], g[1]))
        else:
            g = np.asarray(g); bb = (g[:, 0].min(), g[:, 1].min(), g[:, 0].max(), g[:, 1].max())
            recs.append(struct.pack("<i4dii", 3, *bb, 1, len(g)) + struct.pack("<i", 0) + b"".join(struct.pack("<dd", x, y) for x, y in g))
    def hdr(words): return struct.pack(">i5ii", 9994, 0, 0, 0, 0, 0, words) + struct.pack("<ii4d4d", 1000, stype, *bbox, 0, 0, 0, 0)
    shp_len = 50 + sum(4 + len(r) // 2 for r in recs)
    shp = bytearray(hdr(shp_len)); shx = bytearray(hdr(50 + 4 * len(recs))); off = 50
    for i, r in enumerate(recs):
        shp += struct.pack(">ii", i + 1, len(r) // 2) + r; shx += struct.pack(">ii", off, len(r) // 2); off += 4 + len(r) // 2
    open(base + ".shp", "wb").write(shp); open(base + ".shx", "wb").write(shx)
    dbf_write(base + ".dbf", attrs); open(base + ".prj", "w").write(PRJ); open(base + ".cpg", "w").write("ISO-8859-1")
    # verify
    s = open(base + ".shp", "rb").read(); assert struct.unpack(">i", s[24:28])[0] * 2 == len(s)
    x = open(base + ".shx", "rb").read(); assert struct.unpack(">i", x[24:28])[0] * 2 == len(x)
    f = open(base + ".dbf", "rb").read(); nrec = struct.unpack("<I", f[4:8])[0]; hl, rl = struct.unpack("<HH", f[8:12]); assert nrec == len(geoms) and hl + rl * nrec + 1 == len(f)

pt_cols = ["YEAR", "SN", "COMMAND", "DATE", "HOUR", "LOCATION", "MATCH_KEY", "METHOD", "CONFIDENCE", "N_SUPPORT", "N_YEARS", "AGREE_SHR",
           "ALT_SITE", "IN_TERR", "LAT", "LON", "CHAIN_KM", "DIST_EXPWY", "ON_EXPWY", "VEH_TYPE", "OWNERSHIP", "CAUSE",
           "INJURED", "KILLED", "INVOLVED", "SEVERITY", "DUPLICATE"]
pt_attr = pts[pt_cols].copy(); pt_attr["METHOD"] = pt_attr.METHOD.str.slice(0, 60)
shp_write(os.path.join(OUT, "shapefiles", "RTC_crash_points_2013_2022"), list(zip(pts.LON, pts.LAT)), pt_attr, "point")

# ---------------- 2-km segments with Gi* from geoparsed points ----------------
L = cl.chainage_km.max(); nseg = int(np.ceil(L / 2))
def gistar(x, band=1):
    x = np.asarray(x, float); n = len(x); xb = x.mean(); s = np.sqrt((x ** 2).mean() - xb ** 2); z = np.zeros(n)
    for i in range(n):
        j = np.arange(max(0, i - band), min(n, i + band + 1)); w = len(j)
        den = s * np.sqrt((n * w - w ** 2) / (n - 1)); z[i] = (x[j].sum() - xb * w) / den if den > 0 else 0
    return z
cls = lambda z: np.select([z >= 2.576, z >= 1.960, z >= 1.645], ["Hot 99%", "Hot 95%", "Hot 90%"], "Not significant")
on = pts[(pts.ON_EXPWY == "yes") & (pts.DUPLICATE != "yes")].copy(); on["SEG"] = (on.CHAIN_KM // 2).astype(int).clip(0, nseg - 1)
hm = on[on.CONFIDENCE.isin(["High", "Medium"])]
cnt = np.bincount(on.SEG, minlength=nseg); cnt_hm = np.bincount(hm.SEG, minlength=nseg)
kil = on.groupby("SEG").KILLED.sum().reindex(range(nseg), fill_value=0).values
z_all, z_hm = gistar(cnt), gistar(cnt_hm)
inter = np.arange(nseg) * 2 >= 20; z_int = np.full(nseg, np.nan); z_int[inter] = gistar(cnt[inter])
top = on.groupby("SEG").MATCH_KEY.agg(lambda s: ", ".join(s.value_counts().index[:2])).reindex(range(nseg), fill_value="")
yrs = {y: np.bincount(on[on.YEAR == y].SEG, minlength=nseg) for y in range(2013, 2023)}
tau = []; pv = []
for i in range(nseg):
    zz = [gistar(yrs[y])[i] for y in range(2013, 2023)]; r = stats.kendalltau(range(10), zz); tau.append(0 if np.isnan(r[0]) else r[0]); pv.append(1 if np.isnan(r[1]) else r[1])
segs = pd.DataFrame({"SEG_ID": range(nseg), "FROM_KM": np.arange(nseg) * 2, "TO_KM": np.minimum(np.arange(1, nseg + 1) * 2, L).round(2),
                     "CRASHES": cnt, "CRASH_HM": cnt_hm, "KILLED": kil, "GI_Z": z_all.round(3), "GI_CLASS": cls(z_all),
                     "GI_Z_HM": z_hm.round(3), "GI_CLS_HM": cls(z_hm), "GI_Z_INTER": np.round(z_int, 3),
                     "GI_CLS_INT": np.where(inter, cls(np.nan_to_num(z_int)), "n/a (<20 km)"), "MK_TAU": np.round(tau, 3), "MK_P": np.round(pv, 4),
                     "TREND": np.where((np.array(pv) < 0.05) & (np.array(tau) > 0), "intensifying", np.where((np.array(pv) < 0.05) & (np.array(tau) < 0), "diminishing", "no trend")),
                     "TOP_NAMES": top.values})
geoms = []
for i in range(nseg):
    a, b = i * 2, min((i + 1) * 2, L); ts = np.r_[a, cl.chainage_km[(cl.chainage_km > a) & (cl.chainage_km < b)].values, b]
    geoms.append(np.c_[np.interp(ts, cl.chainage_km, cl.lon), np.interp(ts, cl.chainage_km, cl.lat)])
shp_write(os.path.join(OUT, "shapefiles", "RTC_segments_2km_hotspots"), geoms, segs, "line")
segs.to_csv(os.path.join(OUT, "RTC_segments_2km_hotspots.csv"), index=False)
shp_write(os.path.join(OUT, "shapefiles", "Expressway_centreline_derived"), [np.c_[cl.lon, cl.lat]], pd.DataFrame({"NAME": ["Lagos-Ibadan Expressway (data-derived)"], "LEN_KM": [round(L, 2)]}), "line")

# ---------------- GeoJSON / KML / Leaflet ----------------
def clean(v): return None if (isinstance(v, float) and np.isnan(v)) else (v.item() if hasattr(v, "item") else v)
gj = {"type": "FeatureCollection", "features": [{"type": "Feature", "geometry": {"type": "Point", "coordinates": [float(r.LON), float(r.LAT)]},
      "properties": {k: clean(r[k]) for k in pt_cols}} for _, r in pts.iterrows()]}
json.dump(gj, open(os.path.join(OUT, "RTC_crash_points_2013_2022.geojson"), "w"))
sj = {"type": "FeatureCollection", "features": [{"type": "Feature", "geometry": {"type": "LineString", "coordinates": g.round(6).tolist()},
      "properties": {k: clean(v) for k, v in segs.iloc[i].items()}} for i, g in enumerate(geoms)]}
json.dump(sj, open(os.path.join(OUT, "RTC_segments_2km_hotspots.geojson"), "w"))
esc = lambda s: str(s).replace("&", "&amp;").replace("<", "&lt;").replace(">", "&gt;")
col = {"High": "ff2a8a1b", "Medium": "ff00a5eb", "Low": "ff3b3bd0"}
k = ['<?xml version="1.0" encoding="UTF-8"?><kml xmlns="http://www.opengis.net/kml/2.2"><Document><name>Lagos–Ibadan RTC 2013–2022 (geoparsed)</name>']
for c, v in col.items(): k.append(f'<Style id="{c}"><IconStyle><color>{v}</color><scale>0.7</scale><Icon><href>http://maps.google.com/mapfiles/kml/shapes/placemark_circle.png</href></Icon></IconStyle></Style>')
for y in range(2013, 2023):
    k.append(f"<Folder><name>{y}</name>")
    for _, r in pts[pts.YEAR == y].iterrows():
        k.append(f"<Placemark><name>{esc(r.LOCATION)}</name><styleUrl>#{r.CONFIDENCE}</styleUrl><description>{esc(f'{r.DATE} | {r.COMMAND} | killed {r.KILLED}, injured {r.INJURED} | {r.CAUSE} | {r.CONFIDENCE} confidence ({r.MATCH_KEY})')}</description><Point><coordinates>{r.LON},{r.LAT},0</coordinates></Point></Placemark>")
    k.append("</Folder>")
k.append("</Document></kml>"); open(os.path.join(OUT, "RTC_crash_points_2013_2022.kml"), "w").write("".join(k))

S = {"records": int(len(d)), "geoparsed": int(len(pts)), "pct": round(100 * len(pts) / len(d), 1), "confidence": d.CONFIDENCE.replace("", "Unresolved").value_counts().to_dict(),
     "on_expressway": int((pts.ON_EXPWY == "yes").sum()), "segments_hot95": segs.loc[segs.GI_Z >= 1.96, "FROM_KM"].tolist(),
     "segments_hot95_hm": segs.loc[segs.GI_Z_HM >= 1.96, "FROM_KM"].tolist(), "segments_hot95_inter": segs.loc[segs.GI_Z_INTER >= 1.96, "FROM_KM"].tolist(),
     "intensifying": segs.loc[segs.TREND == "intensifying", "FROM_KM"].tolist(), "diminishing": segs.loc[segs.TREND == "diminishing", "FROM_KM"].tolist(),
     "per_year": pts.groupby("YEAR").size().to_dict(), "per_year_total": d.groupby("YEAR").size().to_dict()}
json.dump(S, open(os.path.join(OUT, "summary.json"), "w"), indent=1, default=int); print(json.dumps(S, indent=1, default=int))
print(segs[segs.GI_Z >= 1.96][["FROM_KM", "CRASHES", "KILLED", "GI_Z", "TOP_NAMES"]].to_string())
print(segs[(segs.GI_Z_INTER >= 1.96)][["FROM_KM", "CRASHES", "GI_Z_INTER", "TOP_NAMES"]].to_string())
```


### geo10_maps.py

```python
"""geo10_maps.py — Static maps of geoparsed 2013–2022 crashes. Usage: python3 -I geo10_maps.py <deliver_dir> <centreline.csv>"""
import sys, os, json, numpy as np, pandas as pd
sys.path.insert(0, os.path.dirname(__file__))
import style; style.setup()
import basemap_ne
import matplotlib.pyplot as plt, seaborn as sns
from matplotlib.lines import Line2D
from matplotlib.patches import Patch
from matplotlib.collections import LineCollection
DL, CLF = sys.argv[1], sys.argv[2]; FIG = os.path.join(DL, "maps"); os.makedirs(FIG, exist_ok=True)
d = pd.read_csv(os.path.join(DL, "RTC_Lagos_Ibadan_2013_2022_geoparsed.csv"), low_memory=False)
sg = pd.read_csv(os.path.join(DL, "RTC_segments_2km_hotspots.csv")); cl = pd.read_csv(CLF)
p = d[d.LAT.notna() & (d.DUPLICATE != "yes")].copy()
ASP = 1 / np.cos(np.deg2rad(6.9)); INK2 = "#52514e"; ROAD = "#d6d5cf"
CONF = {"High": "#1b8a2a", "Medium": "#eda100", "Low": "#d03b3b"}
XL, YL = (3.20, 4.05), (6.40, 7.50)

def frame(ax, title, legend_loc="lower right"):
    basemap_ne.draw(ax, extent=XL + YL)
    ax.plot(cl.lon, cl.lat, color="#fcfcfb", lw=7.5, solid_capstyle="round", zorder=1, alpha=.9)
    ax.plot(cl.lon, cl.lat, color="#8a8983", lw=3.2, solid_capstyle="round", zorder=1.1)
    ax.set_xlim(*XL); ax.set_ylim(*YL); ax.set_aspect(ASP); ax.set_title(title)
    ax.set_xlabel("Longitude (°E)"); ax.set_ylabel("Latitude (°N)")
    for nm, la, lo, dx in [("Ojota, Lagos", 6.5994, 3.3783, -30), ("Iwo Road, Ibadan", 7.4014, 3.9422, -10)]:
        ax.annotate(nm, (lo, la), xytext=(dx, -26 if nm.startswith("Ojota") else 8), textcoords="offset points", fontsize=8, color=INK2, ha="right",
                    arrowprops=dict(arrowstyle="-", color="#8a8983", lw=.5) if nm.startswith("Ojota") else None)
    # scale bar (20 km) and north arrow
    dlon = 20 / (111.32 * np.cos(np.deg2rad(6.9))); x0, y0 = XL[1] - 0.04 - dlon, YL[0] + 0.06
    ax.plot([x0, x0 + dlon], [y0, y0], color="#0b0b0b", lw=2.5, solid_capstyle="butt"); ax.plot([x0, x0 + dlon / 2], [y0, y0], color="#fcfcfb", lw=1.2, solid_capstyle="butt")
    ax.text(x0, y0 + 0.012, "0", fontsize=7, ha="center"); ax.text(x0 + dlon, y0 + 0.012, "20 km", fontsize=7, ha="center")
    ax.annotate("N", xy=(XL[0] + 0.05, YL[1] - 0.04), xytext=(XL[0] + 0.05, YL[1] - 0.12), ha="center", fontsize=10, weight="bold",
                arrowprops=dict(arrowstyle="-|>", color="#0b0b0b", lw=1.5))
    ax.text(XL[1] - 0.01, YL[0] + 0.01, "WGS 84 · crashes geoparsed from FRSC location names · " + basemap_ne.CREDIT, fontsize=6.5, color="#3a3936", ha="right", va="bottom",
            bbox=dict(boxstyle="round,pad=0.2", fc="#fcfcfb", ec="none", alpha=.8), zorder=6)

def label_sites(ax, sites, n=10, fs=7.5):
    s = sites.sort_values("n", ascending=False).head(n).sort_values("lat")
    for i, (_, r) in enumerate(s.iterrows()):
        side = 1 if (i % 2 == 0) else -1
        ax.annotate(f"{r.key.title()} ({r.n})", (r.lon, r.lat), xytext=(16 * side, 0), textcoords="offset points", ha="left" if side > 0 else "right",
                    va="center", fontsize=fs, color="#0b0b0b", arrowprops=dict(arrowstyle="-", color="#8a8983", lw=0.6),
                    bbox=dict(boxstyle="round,pad=0.15", fc="#fcfcfb", ec="none", alpha=0.85))

site = p.groupby(["LAT", "LON"]).agg(n=("YEAR", "size"), killed=("KILLED", "sum"), key=("MATCH_KEY", lambda s: s.value_counts().index[0]),
                                     conf=("CONFIDENCE", lambda s: s.value_counts().index[0])).reset_index().rename(columns={"LAT": "lat", "LON": "lon"})

# ---------- Map 1: overview by confidence ----------
fig, ax = plt.subplots(figsize=(10.4, 10))
frame(ax, f"Geoparsed road traffic crashes, Lagos–Ibadan Expressway, 2013–2022 (n = {len(p):,})")
for k in ["Low", "Medium", "High"]:
    s = site[site.conf == k]
    ax.scatter(s.lon, s.lat, s=8 + s.n * 2.2, color=CONF[k], alpha=.72, edgecolor="#fcfcfb", linewidth=.7, zorder=3)
label_sites(ax, site[site.lon.between(3.3, 4.0)], n=12)
h = [Line2D([], [], marker="o", ls="", color=CONF[k], markersize=8, label=f"{k} confidence ({(p.CONFIDENCE == k).sum():,} crashes)") for k in ["High", "Medium", "Low"]]
h += [Line2D([], [], marker="o", ls="", color="#a8a79f", markersize=np.sqrt(8 + n * 2.2), label=f"{n} crashes at site") for n in [10, 50, 150]]
ax.legend(handles=h, loc="center left", bbox_to_anchor=(1.01, 0.5), fontsize=8.5, labelspacing=1.3, borderpad=0.9, frameon=True, facecolor="#fcfcfb", edgecolor="#c9c8c1",
          title=f"Excludes {int((d.DUPLICATE == 'yes').sum())} duplicate\nrecords", title_fontsize=8.5)
fig.tight_layout(); fig.savefig(os.path.join(FIG, "Map1_geoparsed_crashes_by_confidence.png"), dpi=220); plt.close()

# ---------- Map 2: small multiples by year ----------
fig, axs = plt.subplots(2, 5, figsize=(15, 9.6), sharex=True, sharey=True)
for ax, y in zip(axs.ravel(), range(2013, 2023)):
    basemap_ne.draw(ax, light=True, extent=XL + YL)
    ax.plot(cl.lon, cl.lat, color="#8a8983", lw=2.2, solid_capstyle="round", zorder=1)
    s = p[p.YEAR == y].groupby(["LAT", "LON"]).size().reset_index(name="n")
    ax.scatter(s.LON, s.LAT, s=4 + s.n * 4, color="#2a78d6", alpha=.6, edgecolor="#fcfcfb", linewidth=.5, zorder=3)
    tot = int((d.YEAR == y).sum()); ax.set_title(f"{y}   ({(p.YEAR == y).sum()} of {tot} located)", fontsize=10)
    ax.set_xlim(*XL); ax.set_ylim(*YL); ax.set_aspect(ASP); ax.tick_params(labelsize=7)
fig.suptitle("Geoparsed crashes by year (symbol area ∝ crashes at site)", x=0.01, ha="left", fontsize=13, weight="bold")
fig.text(0.99, 0.005, basemap_ne.CREDIT, ha="right", fontsize=8, color="#52514e")
fig.tight_layout(); fig.savefig(os.path.join(FIG, "Map2_crashes_by_year.png"), dpi=200); plt.close()

# ---------- Map 3: kernel density (High + Medium) ----------
hm = p[p.CONFIDENCE.isin(["High", "Medium"]) & (p.ON_EXPWY == "yes")]
from matplotlib.colors import LinearSegmentedColormap
KDECM = LinearSegmentedColormap.from_list("kde", [(1.0, 0.85, 0.30, 0.0), (0.99, 0.80, 0.30, 0.55), (0.93, 0.41, 0.20, 0.8), (0.82, 0.23, 0.23, 0.92), (0.45, 0.05, 0.10, 0.97)])
fig, axs = plt.subplots(1, 2, figsize=(14, 9.5), sharey=True)
for ax, (lab, sub) in zip(axs, [("2013–2017", hm[hm.YEAR <= 2017]), ("2018–2022", hm[hm.YEAR >= 2018])]):
    frame(ax, f"Crash density (KDE), {lab}  (n = {len(sub):,}, High + Medium)")
    sns.kdeplot(x=sub.LON, y=sub.LAT, fill=True, cmap=KDECM, levels=14, thresh=0.04, bw_adjust=0.3, ax=ax, zorder=2)
    ax.scatter(sub.LON, sub.LAT, s=3, color="#0b0b0b", alpha=.35, zorder=3, linewidths=0)
fig.tight_layout(); fig.savefig(os.path.join(FIG, "Map3_crash_density_KDE.png"), dpi=200); plt.close()

# ---------- Map 4: fatalities ----------
fs = site[site.killed > 0].copy(); fs["dpc"] = fs.killed / fs.n
fig, ax = plt.subplots(figsize=(8.2, 10))
frame(ax, f"Persons killed at geoparsed crash sites, 2013–2022 (total {int(p.KILLED.sum()):,})")
sc = ax.scatter(fs.lon, fs.lat, s=6 + fs.killed * 3, c=fs.dpc.clip(0, 1.5), cmap=style.SEQ, vmin=0, vmax=1.5, alpha=.85, edgecolor="#52514e", linewidth=.4, zorder=3)
cb = fig.colorbar(sc, ax=ax, shrink=.45, pad=.02); cb.set_label("Deaths per crash at site")
lab = fs.rename(columns={"killed": "_k"}).assign(n=lambda x: x._k.astype(int))
label_sites(ax, lab[lab.lon.between(3.3, 4.0)], n=10)
h = [Line2D([], [], marker="o", ls="", color="#a8a79f", markeredgecolor="#52514e", markersize=np.sqrt(6 + k * 3), label=f"{k} killed") for k in [5, 25, 100]]
ax.legend(handles=h, loc="center right", bbox_to_anchor=(0.995, 0.56), fontsize=8, labelspacing=1.4, borderpad=0.9, title="Labels: persons killed", frameon=True, facecolor="#fcfcfb", edgecolor="#c9c8c1", framealpha=.95, title_fontsize=8.5)
fig.tight_layout(); fig.savefig(os.path.join(FIG, "Map4_fatalities.png"), dpi=220); plt.close()

# ---------- Map 5: Gi* hotspot segments ----------
pal = {"Hot 99%": "#d03b3b", "Hot 95%": "#ec835a", "Hot 90%": "#f4b393", "Not significant": "#c9c8c1"}
fig, axs = plt.subplots(1, 2, figsize=(14, 9.5), sharey=True)
for ax, col, title in [(axs[0], "GI_CLASS", "Gi* hotspots — whole corridor (2-km segments)"), (axs[1], "GI_CLS_INT", "Gi* hotspots — inter-urban section (≥ 20 km) + trends")]:
    basemap_ne.draw(ax, extent=XL + YL)
    ax.set_xlim(*XL); ax.set_ylim(*YL); ax.set_aspect(ASP); ax.set_title(title); ax.set_xlabel("Longitude (°E)"); ax.set_ylabel("Latitude (°N)")
    for _, r in sg.iterrows():
        ts = np.r_[r.FROM_KM, cl.chainage_km[(cl.chainage_km > r.FROM_KM) & (cl.chainage_km < r.TO_KM)].values, r.TO_KM]
        c = pal.get(r[col], "#e4e3df"); lw = 9 if str(r[col]).startswith("Hot") else 5
        ax.plot(np.interp(ts, cl.chainage_km, cl.lon), np.interp(ts, cl.chainage_km, cl.lat), color=c, lw=lw, solid_capstyle="butt", zorder=2)
    hot = sg[sg[col].astype(str).str.startswith("Hot") & (sg.CRASHES >= 20)].copy()
    hot["mid"] = (hot.FROM_KM + hot.TO_KM) / 2
    hot["la"] = np.interp(hot.mid, cl.chainage_km, cl.lat); hot["lo"] = np.interp(hot.mid, cl.chainage_km, cl.lon)
    hot = hot.sort_values("la")
    for grp in [hot[hot.la < 7.0], hot[hot.la >= 7.0]]:
        if not len(grp): continue
        y0 = max(grp.la.min() - 0.02, YL[0] + 0.12); ys = y0 + np.arange(len(grp)) * 0.045
        for (_, r), yy in zip(grp.iterrows(), ys):
            tx = r.lo + 0.13 if r.la < 7.0 else r.lo - 0.10
            ax.annotate(f"{r.TOP_NAMES.split(',')[0].title()} · {r.FROM_KM:.0f}–{r.TO_KM:.0f} km · {r.CRASHES} crashes", (r.lo, r.la), xytext=(tx, yy), textcoords="data",
                        ha="left" if r.la < 7.0 else "right", va="center", fontsize=7.8, arrowprops=dict(arrowstyle="-", color="#8a8983", lw=.6, shrinkA=0, shrinkB=2),
                        bbox=dict(boxstyle="round,pad=0.2", fc="#fcfcfb", ec="#e4e3df", lw=.5))
    ax.text(XL[1] - 0.01, YL[0] + 0.01, "WGS 84 · segments on data-derived centreline · " + basemap_ne.CREDIT, fontsize=6.5, color="#3a3936", ha="right",
            bbox=dict(boxstyle="round,pad=0.2", fc="#fcfcfb", ec="none", alpha=.8), zorder=6)
    ax.plot([cl.lon.iloc[0]], [cl.lat.iloc[0]], "o", color="#52514e", ms=4); ax.annotate("Ojota", (cl.lon.iloc[0], cl.lat.iloc[0]), xytext=(-8, -4), textcoords="offset points", ha="right", fontsize=8, color=INK2)
    ax.annotate("Iwo Road", (cl.lon.iloc[-1], cl.lat.iloc[-1]), xytext=(-8, 4), textcoords="offset points", ha="right", fontsize=8, color=INK2)
for _, r in sg[sg.TREND == "intensifying"].iterrows():
    mid = (r.FROM_KM + r.TO_KM) / 2
    axs[1].scatter(np.interp(mid, cl.chainage_km, cl.lon) - 0.03, np.interp(mid, cl.chainage_km, cl.lat), marker=">", s=55, color="#6250d6", zorder=4)
h = [Patch(color=pal[k], label=k) for k in pal] + [Line2D([], [], marker=">", ls="", color="#6250d6", markersize=8, label="Intensifying trend (Mann–Kendall p < 0.05)")]
for a_ in axs: a_.legend(handles=h if a_ is axs[1] else h[:-1], loc="center right", bbox_to_anchor=(0.995, 0.6), fontsize=8.5, frameon=True, facecolor="#fcfcfb", edgecolor="#c9c8c1", framealpha=.95)
fig.tight_layout(); fig.savefig(os.path.join(FIG, "Map5_hotspot_segments_GiStar.png"), dpi=200); plt.close()

# ---------- Figure: positional agreement ----------
KX, KY = 111.32 * np.cos(np.deg2rad(6.9)), 110.57
v = d[d.LAT.notna() & d.ORIG_LAT.notna()].copy(); v["err"] = v.ORIG_DIFF
fig, axs = plt.subplots(1, 2, figsize=(13, 4.3), gridspec_kw={"width_ratios": [1.2, 1]})
ax = axs[0]
for k in ["High", "Medium", "Low"]:
    e = np.sort(v[v.CONFIDENCE == k].err.values); ax.step(np.r_[0, e], np.r_[0, np.arange(1, len(e) + 1) / len(e)] * 100, where="post", color=CONF[k], lw=2, label=f"{k} (n = {len(e):,})")
ax.set_xscale("symlog", linthresh=1); ax.set_xlim(0, 120); ax.set_xticks([0, 0.5, 1, 2, 5, 10, 20, 50, 100]); ax.set_xticklabels(["0", "0.5", "1", "2", "5", "10", "20", "50", "100"])
ax.axvline(1, color="#a8a79f", ls="--", lw=1); ax.set_ylim(0, 100); ax.legend(title="Confidence", loc="lower right")
ax.set_xlabel("Distance from withheld original coordinate (km)"); ax.set_ylabel("Cumulative % of records"); ax.set_title("Agreement with withheld coordinates (leave-one-year-out)")
ax = axs[1]; w1 = v.groupby("YEAR").err.apply(lambda s: 100 * (s <= 1).mean())
sns.barplot(x=w1.index.astype(str), y=w1.values, color="#2a78d6", ax=ax, width=.6)
for i, val in enumerate(w1.values): ax.text(i, val + 1.5, f"{val:.0f}%", ha="center", fontsize=8, color=INK2)
ax.set_ylim(0, 100); ax.set_ylabel("% within 1 km"); ax.set_xlabel(""); ax.set_title("Share within 1 km, by year")
fig.tight_layout(); fig.savefig(os.path.join(FIG, "Fig_validation_agreement.png"), dpi=200); plt.close()
print("maps written:", sorted(os.listdir(FIG)))
```


### geo10_atlas.py

```python
"""geo10_atlas.py — Print-ready web atlas (Leaflet + OpenStreetMap basemap) of the geoparsed crash data.
Usage: python3 -I geo10_atlas.py <deliver_dir> <atlas_dir> <out_html>"""
import sys, os, json, base64, numpy as np, pandas as pd
DL, AT, OUT = sys.argv[1:4]
d = pd.read_csv(os.path.join(DL, "RTC_Lagos_Ibadan_2013_2022_geoparsed.csv"), low_memory=False)
p = d[d.LAT.notna() & (d.DUPLICATE != "yes")].copy()
segs = json.load(open(os.path.join(DL, "RTC_segments_2km_hotspots.geojson")))
for f in segs["features"]:
    f["geometry"]["coordinates"] = [[round(x, 5), round(y, 5)] for x, y in f["geometry"]["coordinates"]]
    f["properties"] = {k: f["properties"][k] for k in ["FROM_KM", "TO_KM", "CRASHES", "KILLED", "GI_Z", "GI_CLASS", "GI_CLS_INT", "TREND", "TOP_NAMES"]}
meta = json.load(open(os.path.join(AT, "kde_meta.json")))
img = {k: "data:image/png;base64," + base64.b64encode(open(os.path.join(AT, f"kde_{k}.png"), "rb").read()).decode() for k in ["2013_2017", "2018_2022"]}

def sites(df):
    g = df.groupby(["LAT", "LON"]).agg(n=("YEAR", "size"), k=("KILLED", "sum"), inj=("INJURED", "sum"),
                                      key=("MATCH_KEY", lambda s: s.value_counts().index[0]),
                                      conf=("CONFIDENCE", lambda s: s.value_counts().index[0])).reset_index()
    return [[round(r.LAT, 5), round(r.LON, 5), int(r.n), int(r.k), int(r.inj), str(r.key).title(), r.conf] for r in g.itertuples()]
S_all = sites(p)
S_year = {int(y): sites(p[p.YEAR == y]) for y in sorted(p.YEAR.unique())}
tot_year = {int(y): int((d.YEAR == y).sum()) for y in sorted(d.YEAR.unique())}
conf_n = p.CONFIDENCE.value_counts().to_dict()
data = {"all": S_all, "year": S_year, "tot": tot_year, "segs": segs, "kde": img, "kdeB": meta["bounds"], "kdeN": meta["n"],
        "conf": conf_n, "n": int(len(p)), "killed": int(p.KILLED.sum())}

HTML = r"""<!doctype html><html lang="en"><head><meta charset="utf-8"><meta name="viewport" content="width=device-width,initial-scale=1">
<title>Lagos–Ibadan Crash Atlas</title>
<link rel="stylesheet" href="https://cdnjs.cloudflare.com/ajax/libs/leaflet/1.9.4/leaflet.min.css">
<script src="https://cdnjs.cloudflare.com/ajax/libs/leaflet/1.9.4/leaflet.min.js"></script>
<style>
:root{--bg:#f9f9f7;--card:#fcfcfb;--ink:#0b0b0b;--ink2:#52514e;--line:#e4e3df;--acc:#1f4e79}
@media (prefers-color-scheme:dark){:root:not([data-theme="light"]){--bg:#0d0d0d;--card:#1a1a19;--ink:#f0efec;--ink2:#c3c2b7;--line:#383835;--acc:#86b6ef}}
:root[data-theme="dark"]{--bg:#0d0d0d;--card:#1a1a19;--ink:#f0efec;--ink2:#c3c2b7;--line:#383835;--acc:#86b6ef}
*{box-sizing:border-box}html,body{margin:0;background:var(--bg);color:var(--ink);font:14px/1.45 system-ui,-apple-system,"Segoe UI",sans-serif}
header{padding:18px 16px 6px;max-width:1180px;margin:auto}header h1{margin:0;font-size:22px;color:var(--acc)}header p{margin:6px 0;color:var(--ink2)}
nav{position:sticky;top:0;z-index:2000;background:var(--bg);border-bottom:1px solid var(--line)}
nav .in{max-width:1180px;margin:auto;padding:8px 16px;display:flex;gap:6px;flex-wrap:wrap}
nav button{font:inherit;border:1px solid var(--line);background:var(--card);color:var(--ink);padding:6px 11px;border-radius:999px;cursor:pointer}
nav button.on{background:var(--acc);color:var(--card);border-color:var(--acc)}
.sheet{max-width:1180px;margin:14px auto 28px;padding:0 16px}
.sheet h2{font-size:17px;margin:0 0 4px}.sheet .sub{color:var(--ink2);margin:0 0 8px;font-size:13px}
.map{height:78vh;min-height:520px;border:1px solid var(--line);border-radius:8px;background:#e9e8e3}
.legend{background:#fff;color:#0b0b0b;padding:8px 10px;border-radius:6px;box-shadow:0 1px 4px rgba(0,0,0,.25);font-size:12px;line-height:1.5;max-width:230px}
.legend b{display:block;margin-bottom:3px}.sw{display:inline-block;width:11px;height:11px;border-radius:50%;margin-right:6px;vertical-align:-1px;border:1px solid #fff}
.ln{display:inline-block;width:22px;height:6px;margin-right:6px;vertical-align:2px}
.north{background:#fff;color:#0b0b0b;border-radius:6px;box-shadow:0 1px 4px rgba(0,0,0,.25);padding:4px 8px;text-align:center;font:700 13px system-ui;line-height:1}
.north svg{display:block;margin:0 auto 1px}
.lbl{background:rgba(255,255,255,.88);border:0;box-shadow:none;color:#0b0b0b;font:600 11px system-ui;padding:1px 4px}
.lbl:before{display:none}
.yrs{display:flex;gap:5px;flex-wrap:wrap;margin:0 0 8px}.yrs button{font:inherit;font-size:12.5px;border:1px solid var(--line);background:var(--card);color:var(--ink);padding:4px 9px;border-radius:6px;cursor:pointer}
.yrs button.on{background:var(--acc);color:var(--card)}
.note{color:var(--ink2);font-size:12px;margin-top:6px}
.tb{float:right;font:inherit;font-size:12px;border:1px solid var(--line);background:var(--card);color:var(--ink);padding:4px 9px;border-radius:6px;cursor:pointer}
@media (max-width:640px){.map{height:70vh;min-height:420px}.legend{max-width:170px;font-size:11px}header h1{font-size:18px}}
@media print{nav,.yrs,.tb,header p.hint{display:none}.sheet{page-break-after:always;max-width:none}.map{height:180mm;border:none}body{background:#fff}}
</style></head><body>
<header><button class="tb" id="theme">Light / dark</button><h1>Road traffic crashes on the Lagos–Ibadan Expressway, 2013–2022</h1>
<p>FRSC crash records geoparsed from location names (LLM-assisted toponym resolution). Basemap © OpenStreetMap contributors.</p>
<p class="hint">Tip: choose a map, zoom/pan to the extent you want, then print (Ctrl/Cmd + P → Save as PDF, landscape) or take a screenshot. Each map prints on its own page.</p></header>
<nav><div class="in" id="tabs"></div></nav>
<div id="sheets"></div>
<script>
const D=__DATA__;
const CONF={High:"#1b8a2a",Medium:"#eda100",Low:"#d03b3b"};
const GI={"Hot 99%":"#d03b3b","Hot 95%":"#ec835a","Hot 90%":"#f4b393"};
const fmt=n=>n.toLocaleString("en-GB");
const SHEETS=[
 {id:"m1",t:"Map 1. Geoparsed crash sites by confidence class",s:`${fmt(D.n)} crashes (duplicates excluded). Symbol area ∝ crashes at site; colour = geoparsing confidence.`},
 {id:"m2",t:"Map 2. Crash sites by year",s:"Select a year. Symbol area ∝ crashes at site."},
 {id:"m3",t:"Map 3. Crash density (kernel density estimate)",s:`High + Medium confidence crashes within 1 km of the expressway. Switch periods below.`},
 {id:"m4",t:"Map 4. Persons killed",s:`${fmt(D.killed)} persons killed at geoparsed sites. Symbol area ∝ persons killed; colour = deaths per crash.`},
 {id:"m5",t:"Map 5. Getis-Ord Gi* hotspot segments (2 km)",s:"Whole-corridor Gi* classes; dashed outline = inter-urban hotspot (≥ 20 km analysis); ▶ = intensifying trend (Mann–Kendall p < 0.05)."}];
const tabs=document.getElementById("tabs"),wrap=document.getElementById("sheets"),maps={};
SHEETS.forEach((s,i)=>{const b=document.createElement("button");b.textContent=s.t.split(".")[0];b.onclick=()=>{document.getElementById(s.id).scrollIntoView({behavior:"smooth"})};tabs.appendChild(b);
 const el=document.createElement("section");el.className="sheet";el.id=s.id;
 el.innerHTML=`<h2>${s.t}</h2><p class="sub">${s.s}</p>${s.id==="m2"?'<div class="yrs" id="yrs"></div>':""}${s.id==="m3"?'<div class="yrs" id="kd"></div>':""}<div class="map" id="map_${s.id}"></div>
 <p class="note">WGS 84 · Points are landmark-level positions derived from FRSC location descriptions, not crash-scene GPS. Basemap © OpenStreetMap contributors.</p>`;wrap.appendChild(el)});
function base(id){const m=L.map("map_"+id,{zoomSnap:.25,preferCanvas:true}).fitBounds([[6.56,3.30],[7.44,3.98]]);
 L.tileLayer("https://tile.openstreetmap.org/{z}/{x}/{y}.png",{maxZoom:19,attribution:'&copy; <a href="https://www.openstreetmap.org/copyright">OpenStreetMap</a> contributors'}).addTo(m);
 L.control.scale({metric:true,imperial:false,position:"bottomleft",maxWidth:160}).addTo(m);
 const N=L.control({position:"topleft"});N.onAdd=()=>{const e=L.DomUtil.create("div","north");e.innerHTML='<svg width="18" height="22" viewBox="0 0 18 22"><path d="M9 1 L16 21 L9 16 L2 21 Z" fill="#0b0b0b"/><path d="M9 1 L9 16 L2 21 Z" fill="#8a8983"/></svg>N';return e};N.addTo(m);
 maps[id]=m;return m}
function legend(m,html,pos){const c=L.control({position:pos||"bottomright"});c.onAdd=()=>{const e=L.DomUtil.create("div","legend");e.innerHTML=html;return e};c.addTo(m);return c}
const esc=s=>String(s).replace(/[&<>]/g,c=>({"&":"&amp;","<":"&lt;",">":"&gt;"}[c]));
const rad=n=>2.5+Math.sqrt(n)*1.7;
function sizeLegend(vals,lab,fill){return vals.map(v=>`<div style="display:flex;align-items:center;gap:8px;margin:2px 0"><svg width="${2*rad(v)+2}" height="${2*rad(v)+2}"><circle cx="${rad(v)+1}" cy="${rad(v)+1}" r="${rad(v)}" fill="${fill}" stroke="#52514e" stroke-width=".6"/></svg>${v} ${lab}</div>`).join("")}
function labels(m,S,idx,n,fn){[...S].sort((a,b)=>b[idx]-a[idx]).slice(0,n).forEach((r,i)=>L.tooltip({permanent:true,direction:i%2?"left":"right",offset:[i%2?-rad(r[idx]):rad(r[idx]),0],className:"lbl"}).setLatLng([r[0],r[1]]).setContent(fn(r)).addTo(m))}
function pop(r){return `<b>${esc(r[5])}</b><br>${r[2]} crash(es) · ${r[3]} killed · ${r[4]} injured<br>Confidence: ${r[6]}`}
// ---------- Map 1
(()=>{const m=base("m1");D.all.sort((a,b)=>b[2]-a[2]).forEach(r=>L.circleMarker([r[0],r[1]],{radius:rad(r[2]),color:"#fff",weight:.8,fillColor:CONF[r[6]],fillOpacity:.82}).bindPopup(pop(r)).addTo(m));
 labels(m,D.all,2,10,r=>`${esc(r[5])} (${r[2]})`);
 legend(m,`<b>Geoparsing confidence</b>${["High","Medium","Low"].map(k=>`<div><span class="sw" style="background:${CONF[k]}"></span>${k} (${fmt(D.conf[k]||0)})</div>`).join("")}<b style="margin-top:6px">Crashes at site</b>${sizeLegend([10,50,150],"crashes","#bdbcb5")}`)})();
// ---------- Map 2
(()=>{const m=base("m2");let lay=L.layerGroup().addTo(m),lab=L.layerGroup().addTo(m);const box=document.getElementById("yrs");const yrs=Object.keys(D.year);
 const lg=legend(m,"");function draw(y){lay.clearLayers();lab.clearLayers();const S=D.year[y];
  S.forEach(r=>L.circleMarker([r[0],r[1]],{radius:rad(r[2])*1.25,color:"#fff",weight:.8,fillColor:"#2a78d6",fillOpacity:.75}).bindPopup(pop(r)).addTo(lay));
  [...S].sort((a,b)=>b[2]-a[2]).slice(0,6).forEach((r,i)=>L.tooltip({permanent:true,direction:i%2?"left":"right",offset:[i%2?-8:8,0],className:"lbl"}).setLatLng([r[0],r[1]]).setContent(`${esc(r[5])} (${r[2]})`).addTo(lab));
  const n=S.reduce((a,b)=>a+b[2],0);lg.getContainer().innerHTML=`<b>${y}</b>${fmt(n)} of ${fmt(D.tot[y])} records located<b style="margin-top:6px">Crashes at site</b>${sizeLegend([5,20,60],"crashes","#2a78d6")}`;
  [...box.children].forEach(b=>b.classList.toggle("on",b.textContent==y))}
 yrs.forEach(y=>{const b=document.createElement("button");b.textContent=y;b.onclick=()=>draw(y);box.appendChild(b)});draw(yrs[yrs.length-1])})();
// ---------- Map 3
(()=>{const m=base("m3");let ov=null;const box=document.getElementById("kd");const lg=legend(m,"");
 function draw(k){if(ov)m.removeLayer(ov);ov=L.imageOverlay(D.kde[k],D.kdeB,{opacity:.85}).addTo(m);
  lg.getContainer().innerHTML=`<b>Crash density, ${k.replace("_","–")}</b>n = ${fmt(D.kdeN[k])} crashes<div style="height:10px;width:170px;margin:5px 0 2px;border-radius:2px;background:linear-gradient(90deg,rgba(252,217,77,.55),rgba(237,105,51,.75),rgba(209,59,59,.9),rgba(115,13,26,.95))"></div><div style="display:flex;justify-content:space-between;width:170px"><span>lower</span><span>higher</span></div>`;
  [...box.children].forEach(b=>b.classList.toggle("on",b.dataset.k===k))}
 ["2013_2017","2018_2022"].forEach(k=>{const b=document.createElement("button");b.textContent=k.replace("_","–");b.dataset.k=k;b.onclick=()=>draw(k);box.appendChild(b)});draw("2018_2022")})();
// ---------- Map 4
(()=>{const m=base("m4");const ramp=["#cde2fb","#86b6ef","#3987e5","#1c5cab","#0d366b"];const cbin=v=>v<.2?0:v<.4?1:v<.7?2:v<1?3:4;
 const S=D.all.filter(r=>r[3]>0).sort((a,b)=>b[3]-a[3]);
 S.forEach(r=>L.circleMarker([r[0],r[1]],{radius:rad(r[3]),color:"#52514e",weight:.6,fillColor:ramp[cbin(r[3]/r[2])],fillOpacity:.85}).bindPopup(pop(r)+`<br>Deaths per crash: ${(r[3]/r[2]).toFixed(2)}`).addTo(m));
 labels(m,S,3,10,r=>`${esc(r[5])} (${r[3]})`);
 legend(m,`<b>Deaths per crash</b>${["< 0.2","0.2–0.4","0.4–0.7","0.7–1.0","≥ 1.0"].map((t,i)=>`<div><span class="sw" style="background:${ramp[i]};border-color:#52514e"></span>${t}</div>`).join("")}<b style="margin-top:6px">Persons killed</b>${sizeLegend([5,25,100],"killed","#bdbcb5")}`)})();
// ---------- Map 5
(()=>{const m=base("m5");
 L.geoJSON(D.segs,{style:f=>{const c=GI[f.properties.GI_CLASS];return c?{color:c,weight:10,opacity:.9}:{color:"#6b6a64",weight:4,opacity:.55}},
  onEachFeature:(f,l)=>{const p=f.properties;l.bindPopup(`<b>${p.FROM_KM}–${p.TO_KM} km</b> · ${esc(p.TOP_NAMES)}<br>${p.CRASHES} crashes · ${p.KILLED} killed<br>Gi* z = ${(+p.GI_Z).toFixed(2)} (${p.GI_CLASS})<br>Inter-urban class: ${p.GI_CLS_INT}<br>Trend: ${p.TREND}`)}}).addTo(m);
 L.geoJSON({type:"FeatureCollection",features:D.segs.features.filter(f=>String(f.properties.GI_CLS_INT).startsWith("Hot"))},{style:{color:"#7a1418",weight:16,opacity:.9,fill:false,dashArray:"4 5"}}).addTo(m).bringToBack();
 D.segs.features.filter(f=>f.properties.TREND==="intensifying").forEach(f=>{const c=f.geometry.coordinates[Math.floor(f.geometry.coordinates.length/2)];
  L.marker([c[1],c[0]],{icon:L.divIcon({className:"",html:'<div style="color:#6250d6;font:700 16px system-ui;transform:translate(-22px,-9px)">▶</div>'})}).addTo(m)});
 D.segs.features.filter(f=>(String(f.properties.GI_CLASS).startsWith("Hot")||String(f.properties.GI_CLS_INT).startsWith("Hot"))&&f.properties.CRASHES>=20).forEach((f,i)=>{const c=f.geometry.coordinates[Math.floor(f.geometry.coordinates.length/2)],p=f.properties;
  L.tooltip({permanent:true,direction:i%2?"left":"right",offset:[i%2?-10:10,0],className:"lbl"}).setLatLng([c[1],c[0]]).setContent(`${esc(String(p.TOP_NAMES).split(",")[0])} · ${p.FROM_KM}–${p.TO_KM} km (${p.CRASHES})`).addTo(m)});
 legend(m,`<b>Gi* hotspot (whole corridor)</b>${["Hot 99%","Hot 95%","Hot 90%"].map(k=>`<div><span class="ln" style="background:${GI[k]}"></span>${k}</div>`).join("")}<div><span class="ln" style="background:#6b6a64;height:3px"></span>Not significant</div><div><span class="ln" style="border:2px dashed #7a1418;height:9px;background:none"></span>Inter-urban hotspot (≥ 95%)</div><div><span style="color:#6250d6;font-weight:700;margin-right:6px">▶</span>Intensifying trend</div>`)})();
// theme toggle
document.getElementById("theme").onclick=()=>{const r=document.documentElement;const dark=r.dataset.theme?r.dataset.theme==="dark":matchMedia("(prefers-color-scheme: dark)").matches;r.dataset.theme=dark?"light":"dark"};
// highlight tab of visible sheet
const io=new IntersectionObserver(es=>es.forEach(e=>{if(e.isIntersecting){const i=SHEETS.findIndex(s=>s.id===e.target.id);[...tabs.children].forEach((b,j)=>b.classList.toggle("on",i===j))}}),{rootMargin:"-40% 0px -55% 0px"});
SHEETS.forEach(s=>io.observe(document.getElementById(s.id)));
</script></body></html>"""
open(OUT, "w").write(HTML.replace("__DATA__", json.dumps(data, separators=(",", ":"))))
print(OUT, os.path.getsize(OUT) // 1024, "KB")
```


---

## D. Validation against the 2026 FRSC field GPS survey


### geo_validate_2017.py

```python
"""geo_validate_2017.py — Validate the LLM-assisted geoparsing against the geotagged 2017 FRSC return.
The 2017 geoparse was produced with 2017 coordinates withheld (leave-one-year-out gazetteer); records located by the
same-year second pass are reported separately because that pass drew on other 2017 records.
Usage: python3 -I geo_validate_2017.py <raw_2017.csv> <combined_final.csv> <centreline.csv> <all_rows_attributes.csv> <out_dir>"""
import sys, os, re, json, numpy as np, pandas as pd
sys.path.insert(0, os.path.dirname(__file__))
import style; style.setup()
import basemap_ne
import matplotlib.pyplot as plt
from matplotlib.lines import Line2D
from scipy import stats
RAW, COMB, CL, ATTR, OUT = sys.argv[1:6]; os.makedirs(OUT, exist_ok=True)
INK2, GRID = "#52514e", "#e4e3df"
TC = {"High": "#1baf7a", "Medium": "#eda100", "Low": "#e34948"}

r = pd.read_csv(RAW, dtype=str); r.columns = [c.strip() for c in r.columns]
g = pd.read_csv(COMB, low_memory=False); g = g[g.file_year == 2017].sort_values("row_order").reset_index(drop=True)
assert len(r) == len(g) and (r.LOCATION.str.strip().values == g.LOCATION.str.strip().values).all()
la = pd.to_numeric(r.LATITUDE.str.strip(), errors="coerce"); lo = pd.to_numeric(r.LONGITUDE.str.strip(), errors="coerce")
sw = (la < 5) & (lo > 5); la, lo = la.where(~sw, lo), lo.where(~sw, la)
d = pd.DataFrame({"row": np.arange(len(r)), "SN": r["S/N"], "command": r.COMMAND.str.strip(), "location": r.LOCATION.str.strip(),
                  "ref_lat": la, "ref_lon": lo, "ref_swapped": sw, "geo_lat": g.geo_lat, "geo_lon": g.geo_lon,
                  "geo_conf": g.geo_confidence, "geo_method": g.geo_method, "geo_key": g.geo_matched_key, "geo_chainage": g.geo_chainage_km})

# centreline chainage for the reference points
KX, KY = 111.32 * np.cos(np.deg2rad(6.9)), 110.57
cl = pd.read_csv(CL); line = np.c_[cl.lon * KX, cl.lat * KY]; seg = np.diff(line, axis=0); cum = cl.chainage_km.values
def chain(lat, lon):
    P = np.c_[np.asarray(lon) * KX, np.asarray(lat) * KY]; A = line[:-1]; L2 = (seg ** 2).sum(1)
    t = np.clip(((P[:, None] - A[None]) * seg[None]).sum(2) / L2, 0, 1)
    D = np.linalg.norm(P[:, None] - (A[None] + t[..., None] * seg[None]), axis=2); k = D.argmin(1)
    return cum[k] + t[np.arange(len(P)), k] * np.linalg.norm(seg, axis=1)[k], D.min(1)
ok = d.ref_lat.notna() & d.ref_lon.notna()
c, dist = chain(d.ref_lat[ok], d.ref_lon[ok]); d.loc[ok, "ref_chainage"] = c; d.loc[ok, "ref_dist_axis_km"] = dist

# reference quality classes
key = d.ref_lat.round(5).astype(str) + "," + d.ref_lon.round(5).astype(str)
names_per = None
d["ref_names_sharing"] = 0
# all-years placeholder test and command-territory test, both built WITHOUT 2017 coordinates for territory
A = pd.read_csv(ATTR, low_memory=False); A = A[A.coord_source.astype(str).str.startswith("recorded") & A.lat.notna()].copy()
_src = open(os.path.join(os.path.dirname(os.path.abspath(__file__)), "geo_01_geocode.py")).read(); _lib = {}
exec(compile(_src[:_src.index("# ---------------- 2. GAZETTEER")].replace("CLEAN, CL, TARGET, TY, OUTCSV = sys.argv[1], sys.argv[2], sys.argv[3], int(sys.argv[4]), sys.argv[5]", ""), "norm", "exec"), _lib)
A["nm"] = A.LOCATION.map(lambda x: _lib["core"](_lib["norm"](x))); A["ck"] = A.lat.round(4).astype(str) + "," + A.lon.round(4).astype(str)
ph_all = set(A.groupby("ck").nm.nunique().loc[lambda x: x >= 5].index)
d["ref_placeholder_allyears"] = ok & (d.ref_lat.round(4).astype(str) + "," + d.ref_lon.round(4).astype(str)).isin(ph_all)
O = A[(A.file_year != 2017) & ~A.ck.isin(ph_all)].copy(); oc, od = chain(O.lat, O.lon); O["ch"] = oc; O = O[od <= 5]
T = O.groupby("cmd_code").ch.agg(q1=lambda x: x.quantile(.25), q3=lambda x: x.quantile(.75), n="size"); T = T[T.n >= 20]
T["lo"], T["hi"] = T.q1 - 15, T.q3 + 15   # interquartile range of the command's other-year geotags, widened by 15 km
d["cmd_code"] = g.cmd_code.values
d = d.join(T[["lo", "hi"]], on="cmd_code")
d["ref_in_cmd_territory"] = np.where(d.lo.isna() | d.ref_chainage.isna(), np.nan, ((d.ref_chainage >= d.lo) & (d.ref_chainage <= d.hi)).astype(float))
T.round(1).to_csv(os.path.join(OUT, "command_territories_excl2017.csv"))
d["ref_class"] = np.select([~ok, (d.ref_names_sharing >= 5) | d.ref_placeholder_allyears, d.ref_dist_axis_km > 5, d.ref_in_cmd_territory == 0],
                           ["missing", "placeholder coordinate", "off corridor (>5 km)", "outside reporting command's territory"], "usable")
def hav(a1, o1, a2, o2):
    a1, o1, a2, o2 = map(np.deg2rad, (a1, o1, a2, o2))
    return 2 * 6371 * np.arcsin(np.sqrt(np.sin((a2 - a1) / 2) ** 2 + np.cos(a1) * np.cos(a2) * np.sin((o2 - o1) / 2) ** 2))
d["err_km"] = hav(d.ref_lat, d.ref_lon, d.geo_lat, d.geo_lon)
d["err_chain_km"] = d.geo_chainage - d.ref_chainage
d["group"] = np.where(d.geo_method.astype(str).str.startswith("same-year"), "same-year pass (not independent)",
              np.where(d.geo_lat.isna(), "unresolved", "independent (2017 withheld)"))
d.to_csv(os.path.join(OUT, "validation_2017_records.csv"), index=False)

# ---------- summaries
def summ(x):
    e = x.err_km.dropna(); ae = x.err_chain_km.abs().dropna()
    return pd.Series({"n": len(e), "median_km": e.median(), "mean_km": e.mean(), "p90_km": e.quantile(.9),
                      "within_0.5km_%": 100 * (e <= .5).mean(), "within_1km_%": 100 * (e <= 1).mean(), "within_2km_%": 100 * (e <= 2).mean(),
                      "within_5km_%": 100 * (e <= 5).mean(), "median_abs_chainage_err_km": ae.median()})
V = d[(d.ref_class == "usable") & (d.group == "independent (2017 withheld)")]
V2 = d[d.ref_class.isin(["usable", "outside reporting command's territory"]) & (d.group == "independent (2017 withheld)")]
by_all = pd.concat([V2.groupby("geo_conf").apply(summ, include_groups=False).reindex(["High", "Medium", "Low"]), summ(V2).to_frame("All").T])
by_all.round(2).to_csv(os.path.join(OUT, "validation_2017_by_confidence_incl_suspect_refs.csv"))
by = pd.concat([V.groupby("geo_conf").apply(summ, include_groups=False).reindex(["High", "Medium", "Low"]), summ(V).to_frame("All").T])
by.round(2).to_csv(os.path.join(OUT, "validation_2017_by_confidence.csv"))
cls = pd.crosstab(d.ref_class, d.group, margins=True); cls.to_csv(os.path.join(OUT, "validation_2017_reference_classes.csv"))
# by location name (usable, independent)
loc = V.assign(L=V.geo_key.fillna(V.location)).groupby("L").agg(n=("err_km", "size"), median_err_km=("err_km", "median"),
        conf=("geo_conf", lambda s: s.mode().iat[0])).sort_values("n", ascending=False)
loc.round(2).to_csv(os.path.join(OUT, "validation_2017_by_location.csv"))
worst = V.sort_values("err_km", ascending=False).head(15)[["location", "command", "geo_conf", "geo_method", "geo_key", "ref_chainage", "geo_chainage", "err_km"]]
worst.round(2).to_csv(os.path.join(OUT, "validation_2017_largest_errors.csv"), index=False)
# segment-level agreement (5 km), 2017
V5 = V.dropna(subset=["ref_chainage", "geo_chainage"])
bins = np.arange(-10, 140, 5)
hr = np.histogram(V5.ref_chainage, bins)[0]; hg = np.histogram(V5.geo_chainage, bins)[0]
keep = (hr + hg) > 0; rho, prho = stats.spearmanr(hr[keep], hg[keep])
top = lambda h: set(np.argsort(-h)[:5])
bias = V5.err_chain_km
S = {"records": len(d), "reference_classes": d.ref_class.value_counts().to_dict(), "groups": d.group.value_counts().to_dict(),
     "usable_independent_n": len(V), "by_conf_incl_suspect": by_all.round(2).to_dict("index"), "suspect_n": int((V2.ref_class != "usable").sum()), "by_conf": by.round(2).to_dict("index"),
     "segment5_spearman": round(float(rho), 3), "segment5_p": float(prho), "segment5_n": int(keep.sum()),
     "top5_overlap": len(top(hr) & top(hg)), "chain_bias_median_km": round(float(bias.median()), 2),
     "chain_bias_wilcoxon_p": float(stats.wilcoxon(bias[bias != 0]).pvalue) if (bias != 0).sum() > 10 else None,
     "pass2_n": int((d.group == "same-year pass (not independent)").sum()), "swapped_fixed": int(d.ref_swapped.sum()),
     "placeholder_coords": d.loc[d.ref_class.str.startswith("placeholder"), ["ref_lat", "ref_lon"]].round(5).drop_duplicates().values.tolist(),
     "loc_n": len(loc), "loc_within1_share": round(float((loc.median_err_km <= 1).mean() * 100), 1)}
json.dump(S, open(os.path.join(OUT, "validation_2017_summary.json"), "w"), indent=1, default=float)
print(json.dumps({k: v for k, v in S.items() if k != "placeholder_coords"}, indent=1, default=float)); print(by.round(2)); print(cls); print(worst.round(2).to_string())

# ---------- figure
fig = plt.figure(figsize=(14, 10.5)); gs = fig.add_gridspec(2, 2, hspace=0.36, wspace=0.26)
ax = fig.add_subplot(gs[0, 0])
for t in ["High", "Medium", "Low"]:
    e = np.sort(V[V.geo_conf == t].err_km.values); ax.step(e, np.arange(1, len(e) + 1) / len(e) * 100, where="post", color=TC[t], lw=2.4, label=f"{t} (n = {len(e)})")
e = np.sort(V.err_km.values); ax.step(e, np.arange(1, len(e) + 1) / len(e) * 100, where="post", color="#0b0b0b", lw=1.6, ls="--", label=f"All (n = {len(e)})")
for x in (0.5, 1, 2, 5): ax.axvline(x, color=GRID, lw=1, zorder=0)
ax.set_xscale("log"); ax.set_xlim(0.005, 100); ax.text(0.006, 96, "errors < 5 m plotted at the left edge", fontsize=8, color=INK2, va="top"); ax.set_ylim(0, 100); ax.set_xticks([0.01, 0.1, 0.5, 1, 2, 5, 10, 50]); ax.set_xticklabels(["0.01", "0.1", "0.5", "1", "2", "5", "10", "50"])
ax.set_xlabel("Distance between geoparsed and recorded position (km, log scale)"); ax.set_ylabel("Cumulative % of records")
ax.legend(loc="lower right", fontsize=9, frameon=True, facecolor="#fcfcfb", edgecolor=GRID); ax.set_title("(a) Positional error by confidence tier")

ax = fig.add_subplot(gs[0, 1])
for t in ["Low", "Medium", "High"]:
    s = V5[V5.geo_conf == t]; ax.scatter(s.ref_chainage, s.geo_chainage, s=26, color=TC[t], alpha=.7, edgecolor="#fcfcfb", lw=.4, label=t)
ax.plot([-10, 135], [-10, 135], color=INK2, lw=1, ls="--"); ax.fill_between([-10, 135], [-12, 133], [-8, 137], color="#c9c8c1", alpha=.35, lw=0)
ax.set_xlim(-10, 135); ax.set_ylim(-10, 135); ax.set_aspect("equal"); ax.set_xlabel("Recorded position (chainage, km from Ojota)"); ax.set_ylabel("Geoparsed position (chainage, km)")
ax.text(-6, 126, "grey band = ±2 km", fontsize=8.5, color=INK2); ax.legend(loc="lower right", fontsize=9, frameon=True, facecolor="#fcfcfb", edgecolor=GRID)
ax.set_title("(b) Geoparsed vs recorded chainage")

ax = fig.add_subplot(gs[1, 0])
XL, YL = (3.25, 4.02), (6.45, 7.48); basemap_ne.draw(ax, extent=XL + YL)
ax.plot(cl.lon, cl.lat, color="#fcfcfb", lw=6, zorder=1, alpha=.9); ax.plot(cl.lon, cl.lat, color="#8a8983", lw=2.6, zorder=1.1)
for t in ["Low", "Medium", "High"]:
    s = V[V.geo_conf == t]
    for a in s.itertuples(): ax.plot([a.ref_lon, a.geo_lon], [a.ref_lat, a.geo_lat], color=TC[t], lw=.9, alpha=.75, zorder=2)
ax.scatter(V.ref_lon, V.ref_lat, s=16, facecolor="none", edgecolor="#0b0b0b", lw=.7, zorder=3)
ax.scatter(V.geo_lon, V.geo_lat, s=14, c=V.geo_conf.map(TC), edgecolor="#fcfcfb", lw=.4, zorder=4)
ax.set_xlim(*XL); ax.set_ylim(*YL); ax.set_aspect(1 / np.cos(np.deg2rad(6.9))); ax.set_xlabel("Longitude (°E)"); ax.set_ylabel("Latitude (°N)")
ax.legend(handles=[Line2D([], [], marker="o", ls="", markerfacecolor="none", markeredgecolor="#0b0b0b", label="Recorded (2017 geotag)")] +
          [Line2D([], [], marker="o", ls="-", color=TC[t], label=f"Geoparsed, {t}") for t in TC], loc="upper left", fontsize=8.3, frameon=True, facecolor="#fcfcfb", edgecolor=GRID, framealpha=.95)
ax.text(XL[0] + 0.01, YL[0] + 0.01, basemap_ne.CREDIT, fontsize=6.3, color="#3a3936", bbox=dict(boxstyle="round,pad=0.2", fc="#fcfcfb", ec="none", alpha=.85))
ax.set_title("(c) Recorded → geoparsed positions (lines = error)")

ax = fig.add_subplot(gs[1, 1])
mid = bins[:-1] + 2.5
ax.bar(mid - 1.1, hr, width=2.2, color="#52514e", label="Recorded geotags"); ax.bar(mid + 1.1, hg, width=2.2, color="#2a78d6", label="Geoparsed")
ax.set_xlim(-10, 135); ax.set_xlabel("5-km segment (chainage, km from Ojota)"); ax.set_ylabel("2017 crashes")
ax.legend(loc="upper right", fontsize=9, frameon=False); ax.grid(axis="x", visible=False)
ax.text(0.02, 0.95, f"Spearman ρ = {rho:.2f} ({int(keep.sum())} segments)\nTop-5 segments shared: {S['top5_overlap']} of 5", transform=ax.transAxes, va="top", fontsize=9, color=INK2)
ax.set_title("(d) Crashes per 5-km segment, 2017")
fig.suptitle("Validation of LLM-assisted geoparsing against geotagged 2017 FRSC records", x=0.01, ha="left", fontsize=15, weight="bold", y=0.995)
fig.text(0.01, 0.958, f"2017 coordinates were withheld when geoparsing. Evaluated: {len(V)} records with usable reference geotags (missing, placeholder, off-corridor and command-inconsistent geotags excluded).", fontsize=10, color=INK2)
fig.savefig(os.path.join(OUT, "fig_validation_2017.png"), dpi=200, bbox_inches="tight"); plt.close()
```


### gazetteer_verify.py

```python
"""gazetteer_verify.py — Build a gazetteer from the location strings of the 2017 FRSC return and verify each entry's
geoparsed position against the 2026 field GPS fixes (screened) collected by FRSC field officers.
Usage: python3 -I gazetteer_verify.py <gazetteer_names.csv> <validation_2017_records.csv> <out_dir>"""
import sys, os, re, json, numpy as np, pandas as pd
NAMES, VAL, OUT = sys.argv[1:4]; os.makedirs(OUT, exist_ok=True)
src = open(os.path.join(os.path.dirname(os.path.abspath(__file__)), "geo_01_geocode.py")).read(); lib = {}
exec(compile(src[:src.index("# ---------------- 2. GAZETTEER")].replace("CLEAN, CL, TARGET, TY, OUTCSV = sys.argv[1], sys.argv[2], sys.argv[3], int(sys.argv[4]), sys.argv[5]", ""), "n", "exec"), lib)
g = pd.read_csv(NAMES, dtype=str); g.columns = [c.strip() for c in g.columns]
v = pd.read_csv(VAL)
assert len(g) == len(v) and (g.LOCATION.str.strip().values == v.location.str.strip().values).all()
def zone(c):
    c = str(c).replace(",", ".").replace(" ", "")
    if c.startswith("RS11"): return "Oyo"
    if c in ("RS2.1", "RS2.17", "RS2.19", "RS2.11", "RS2.15"): return "Lagos"
    return "Ogun"
v["zone"] = g.COMMAND.map(zone).values
v["name"] = g.LOCATION.map(lambda s: lib["core"](lib["norm"](s))).values
v["km_only"] = g.LOCATION.str.upper().str.replace(r"\s", "", regex=True).str.fullmatch(r"KM\d+(\.\d+)?").values
v.loc[v.name.eq("") | v.name.isna(), "name"] = g.LOCATION.str.strip().str.upper()
v["entry"] = v.name + " | " + v.zone
def hav(a1, o1, a2, o2):
    a1, o1, a2, o2 = map(np.deg2rad, (a1, o1, a2, o2))
    return 2 * 6371 * np.arcsin(np.sqrt(np.sin((a2 - a1) / 2) ** 2 + np.cos(a1) * np.cos(a2) * np.sin((o2 - o1) / 2) ** 2))
rows = []
for e, s in v.groupby("entry"):
    gp = s[s.group == "independent (2017 withheld)"]; fx = s[s.ref_class == "usable"]
    r = dict(entry=s.name.iat[0], zone=s.zone.iat[0], variants=" / ".join(sorted(set(g.LOCATION[s.index].str.strip())))[:120], records=len(s), km_label=bool(s.km_only.all()),
             gaz_lat=gp.geo_lat.median() if len(gp) else np.nan, gaz_lon=gp.geo_lon.median() if len(gp) else np.nan,
             gps_n=len(fx), gps_lat=fx.ref_lat.median() if len(fx) else np.nan, gps_lon=fx.ref_lon.median() if len(fx) else np.nan,
             gps_excluded=int(s.ref_class.ne("usable").sum() - s.ref_class.eq("missing").sum()))
    rows.append(r)
G = pd.DataFrame(rows)
G["error_km"] = hav(G.gaz_lat, G.gaz_lon, G.gps_lat, G.gps_lon)
G["status"] = np.select([G.km_label, G.gaz_lat.isna(), G.gps_lat.isna()], ["KM label only (no landmark)", "no gazetteer position", "no usable GPS fix"], "verified")
G = G.sort_values(["status", "error_km"]); G.round(5).to_csv(os.path.join(OUT, "gazetteer_entries_verified.csv"), index=False)
V = G[G.status == "verified"]; e = V.error_km
def wil(k, n, z=1.96):
    p = k / n; den = 1 + z * z / n; c = (p + z * z / (2 * n)) / den; h = z * np.sqrt(p * (1 - p) / n + z * z / (4 * n * n)) / den; return [round(100 * (c - h), 1), round(100 * (c + h), 1)]
w = V.records
S = {"strings": int(g.LOCATION.str.strip().nunique()), "entries": len(G), "status": G.status.value_counts().to_dict(),
     "verified": len(V), "median_km": round(float(e.median()), 2), "mean_km": round(float(e.mean()), 2), "p90_km": round(float(e.quantile(.9)), 1),
     "within_0.5": round(float((e <= .5).mean() * 100), 1), "within_1": round(float((e <= 1).mean() * 100), 1), "within_1_ci": wil(int((e <= 1).sum()), len(e)),
     "within_2": round(float((e <= 2).mean() * 100), 1), "within_5": round(float((e <= 5).mean() * 100), 1),
     "records_in_verified": int(w.sum()), "records_within_1_share": round(float(w[e <= 1].sum() / w.sum() * 100), 1),
     "by_zone": V.groupby("zone").error_km.agg(["size", "median"]).round(2).to_dict("index"),
     "worst": V.sort_values("error_km", ascending=False).head(6)[["entry", "zone", "records", "error_km"]].round(1).values.tolist()}
json.dump(S, open(os.path.join(OUT, "gazetteer_verify_summary.json"), "w"), indent=1, default=float)
print(json.dumps(S, indent=1, default=float)); print(V[["entry", "zone", "records", "gps_n", "error_km"]].round(2).to_string())
```


---

## E. Shared helpers and utilities


### style.py

```python
import matplotlib
matplotlib.use("Agg")
import matplotlib.pyplot as plt
import seaborn as sns
from matplotlib.colors import LinearSegmentedColormap

CAT = ["#2a78d6", "#eb6834", "#1baf7a", "#eda100", "#e87ba4", "#008300", "#6250d6", "#e34948"]
SEQ_STEPS = ["#cde2fb", "#9ec5f4", "#6da7ec", "#3987e5", "#256abf", "#184f95", "#0d366b"]
SEQ = LinearSegmentedColormap.from_list("seqblue", ["#fcfcfb"] + SEQ_STEPS)
DIV = LinearSegmentedColormap.from_list("div", ["#184f95", "#6da7ec", "#f0efec", "#ec835a", "#d03b3b"])
INK, INK2, GRID = "#0b0b0b", "#52514e", "#e4e3df"

def setup():
    sns.set_theme(style="whitegrid", context="notebook", font="DejaVu Sans")
    plt.rcParams.update({
        "figure.facecolor": "#fcfcfb", "axes.facecolor": "#fcfcfb", "savefig.facecolor": "#fcfcfb",
        "axes.edgecolor": GRID, "grid.color": GRID, "grid.linewidth": 0.6,
        "axes.labelcolor": INK2, "xtick.color": INK2, "ytick.color": INK2, "text.color": INK,
        "axes.titleweight": "bold", "axes.titlesize": 12, "axes.titlelocation": "left",
        "axes.spines.top": False, "axes.spines.right": False, "lines.linewidth": 2,
        "legend.frameon": False, "figure.dpi": 110, "savefig.dpi": 200, "savefig.bbox": "tight",
    })
    sns.set_palette(CAT)
```


### basemap_ne.py

```python
"""basemap_ne.py — Public-domain Natural Earth basemap (1:10m raster NE1_HR_LC_SR_W + vectors) for matplotlib maps.
Natural Earth data are in the public domain (naturalearthdata.com/about/terms-of-use): no permission or attribution required."""
import json, os, numpy as np
from PIL import Image
from matplotlib.path import Path
from matplotlib.patches import PathPatch
from matplotlib import patheffects as pe
_D = os.path.join(os.path.dirname(os.path.abspath(__file__)), "..", "basemap")
_V = json.load(open(os.path.join(_D, "ne_vectors_clip.json")))
_R = np.asarray(Image.open(os.path.join(_D, "ne1_crop.png"))).astype(float) / 255
_E = json.load(open(os.path.join(_D, "ne1_crop.json")))["extent"]
_R = _R * 0.78 + 0.22          # lighten so data stay legible

def _polys(geom):
    t, c = geom["type"], geom["coordinates"]
    return [c] if t == "Polygon" else (c if t == "MultiPolygon" else [])
def _lines(geom):
    t, c = geom["type"], geom["coordinates"]
    return [c] if t == "LineString" else (c if t == "MultiLineString" else [])
def _patch(poly, **kw):
    verts, codes = [], []
    for ring in poly:
        ring = np.asarray(ring)[:, :2]
        if len(ring) < 3: continue
        verts += ring.tolist() + [ring[0].tolist()]; codes += [Path.MOVETO] + [Path.LINETO] * (len(ring) - 1) + [Path.CLOSEPOLY]
    return PathPatch(Path(verts, codes), **kw)

def draw(ax, light=False, labels=True, extent=None):
    ax.imshow(_R, extent=_E, origin="upper", interpolation="bicubic", zorder=0)
    for f in _V["urban_areas"]:
        for p in _polys(f["geometry"]): ax.add_patch(_patch(p, fc="#e9d8c4", ec="#d9c2a6", lw=.4, alpha=.75, zorder=0.2))
    for f in _V["ocean"]:
        for p in _polys(f["geometry"]): ax.add_patch(_patch(p, fc="#c7dceb", ec="#9ab8cf", lw=.6, zorder=0.3))
    for f in _V["rivers_lake_centerlines"]:
        for l in _lines(f["geometry"]): l = np.asarray(l); ax.plot(l[:, 0], l[:, 1], color="#7fa8c9", lw=1.0, zorder=0.4)
    for f in _V["admin_1_states_provinces"]:
        for p in _polys(f["geometry"]):
            for ring in p[:1]:
                r = np.asarray(ring); ax.plot(r[:, 0], r[:, 1], color="#9b8fae", lw=1.0, alpha=.9, zorder=0.5)
    if not light:
        for f in _V["railroads"]:
            for l in _lines(f["geometry"]): l = np.asarray(l); ax.plot(l[:, 0], l[:, 1], color="#6b6a64", lw=.8, ls=(0, (1, 1.5)), zorder=0.6)
    for f in _V["roads"]:
        major = f["properties"].get("type") == "Major Highway"
        for l in _lines(f["geometry"]):
            l = np.asarray(l); ax.plot(l[:, 0], l[:, 1], color="#fcfcfb", lw=(2.6 if major else 1.6), zorder=0.65, solid_capstyle="round")
            ax.plot(l[:, 0], l[:, 1], color=("#b9a27e" if major else "#c9c2b4"), lw=(1.4 if major else .8), zorder=0.7, solid_capstyle="round")
    if labels:
        ext = extent or (ax.get_xlim() + ax.get_ylim())
        halo = [pe.withStroke(linewidth=2.5, foreground="#fcfcfb")]
        for f in _V["admin_1_states_provinces"]:
            n = f["properties"]["name"]; pos = {"Lagos": (3.62, 6.52), "Ogun": (3.16, 7.05), "Oyo": (3.60, 7.42)}.get(n)
            if pos and ext[0] < pos[0] < ext[1] and ext[2] < pos[1] < ext[3]:
                ax.text(*pos, f"{n.upper()} STATE", fontsize=7 if light else 8.5, color="#8a8983", style="italic", ha="center", zorder=0.8, path_effects=halo)
        for f in _V["populated_places_simple"]:
            x, y = f["geometry"]["coordinates"][:2]; n = f["properties"]["name"]
            if ext[0] < x < ext[1] and ext[2] < y < ext[3]:
                ax.plot(x, y, "s", ms=4 if light else 5, color="#3a3936", zorder=5)
                ax.text(x + 0.012, y + 0.008, n, fontsize=7.5 if light else 9.5, weight="bold", color="#3a3936", zorder=5, path_effects=halo)
CREDIT = "Basemap: Natural Earth 1:10m (public domain)"
```


### unrar.py

```python
import ctypes, sys, os
la = ctypes.CDLL('libarchive.so.13')
la.archive_read_new.restype = ctypes.c_void_p
la.archive_entry_pathname.restype = ctypes.c_char_p
la.archive_entry_pathname.argtypes=[ctypes.c_void_p]
la.archive_error_string.restype = ctypes.c_char_p
for f in ['archive_read_support_filter_all','archive_read_support_format_all']:
    getattr(la,f).argtypes=[ctypes.c_void_p]
la.archive_read_open_filename.argtypes=[ctypes.c_void_p, ctypes.c_char_p, ctypes.c_size_t]
la.archive_read_next_header.argtypes=[ctypes.c_void_p, ctypes.POINTER(ctypes.c_void_p)]
la.archive_read_data.argtypes=[ctypes.c_void_p, ctypes.c_void_p, ctypes.c_size_t]
la.archive_read_data.restype=ctypes.c_ssize_t
la.archive_error_string.argtypes=[ctypes.c_void_p]
src, out = sys.argv[1], sys.argv[2]
a = la.archive_read_new(); la.archive_read_support_filter_all(a); la.archive_read_support_format_all(a)
assert la.archive_read_open_filename(a, src.encode(), 10240)==0, la.archive_error_string(a)
e = ctypes.c_void_p()
while True:
    r = la.archive_read_next_header(a, ctypes.byref(e))
    if r==1: break
    if r<0: print('ERR', la.archive_error_string(a)); break
    name = la.archive_entry_pathname(e).decode(errors='replace')
    buf = ctypes.create_string_buffer(65536); chunks=[]
    while True:
        n = la.archive_read_data(a, buf, 65536)
        if n<0: print('DATA ERR', la.archive_error_string(a)); break
        if n==0: break
        chunks.append(buf.raw[:n])
    safe = os.path.basename(name)
    if safe:
        open(os.path.join(out, safe),'wb').write(b''.join(chunks)); print(name, sum(map(len,chunks)))
```
