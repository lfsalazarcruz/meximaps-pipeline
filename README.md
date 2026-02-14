# MexiMaps Data Science Pipeline

## Overview
Processes INEGI *Censo de Población y Vivienda 2020* urban AGEB data (from *Principales resultados por AGEB y manzana urbana, 3ra edición*) to compute the Socioeconomic Composite Index (SCI). This index blends 12+ socioeconomic proxies (e.g., education attainment via GRAPROES, employment via POCUPADA, housing quality via VPH_PISODT) into a 0–100 score per urban AGEB (~59k nationwide). Outputs feed into hex-grid mapping for the MexiMaps app, enabling neighborhood-level visualizations and filters.

**Data source:** INEGI's 32-state CSV bundles (~150MB raw), filtered to "Total AGEB urbana" aggregates (MZA=0 rows, 222 indicators per PDF pp. 4–39). CVEGEO keys constructed for seamless shapefile joins.

## Setup
**Environment:** Python 3.12+ with Jupyter.

**Install:**
```
pip install -r requirements.txt
# or
conda env create -f environment.yml
```

**Key libs:** pandas, geopandas, h3, matplotlib, seaborn, folium.

### Data Download
- **Raw CSVs:** Place the 32 unzipped state files in `data/raw/inegi_census/` (e.g., `ageb_mza_urbana_01_cpv2020/conjunto_de_datos/conjunto_de_datos_ageb_urbana_01_cpv2020.csv`).
- **AGEB Shapefiles:** Download urban 2020 bundle from INEGI Geo Portal to `data/external/shapes/`.

**gitignore note:** Large raws/processed (e.g., Parquets >50MB) ignored—use `.gitkeep` for empty dirs.

## Run the Pipeline
Execute notebooks sequentially in Jupyter:
```
jupyter notebook notebooks/
```

1. **01_data_ingestion.ipynb**: Loads 32 CSVs → Filters "Total AGEB urbana" rows → Concat to national DF (~59k rows) → Pads IDs (AGEB → 4 chars, constructs CVEGEO) → Exports `data/processed/national_ageb.parquet`.
	 - Output: Clean aggregates with vars like POBTOT, P15YM_AN (illiteracy), VPH_AGUADV (water access).
2. **02_merge_shapes.ipynb**: Loads Parquet → Joins with AGEB shapefiles on CVEGEO → Exports enriched GeoDataFrame to `data/processed/ageb_merged/national_ageb_geo.parquet` (or GeoJSON for small subsets).
3. **03_compute_index.ipynb**: Loads merged geo → Computes SCI (weights from `config/weights.yaml`: 30% education, 25% employment, etc.) → Normalizes 0–100 → Exports scores to `data/processed/indices/sci_national.parquet`.
4. **04_hex_aggregation.ipynb**: Generates H3 hex grids (coarse res=5 for national, fine res=9 for cities) → Aggregates SCI via area-weighted overlaps → Exports to `data/processed/hex_grids/mexico_coarse_hex.geojson`.
5. **05_validation_viz.ipynb**: Plots choropleths (Folium/Matplotlib), correlations (e.g., SCI vs. POBTOT), and popups mockups. Tests on GDL (ENT=14)/CDMX (ENT=09) for "real-life" accuracy.

**CLI option:**
```
python src/pipeline.py --step=3 --entity=14
```

## Data Flow
```
Raw CSVs (32 states, 222 vars)
	→ Filtered AGEB Totals (Parquet)
	→ Merged w/ Shapes (GeoParquet)
	→ SCI Computation (0–100 score)
	→ Hex Aggregation (Grids)
	→ App Exports (GeoJSON/MBTiles)
```

## Config & Customization
- **Weights:** Edit `config/weights.yaml` (e.g., education: 0.30, housing: 0.15). Re-run 03_ to update.
- **Proxies:** 12 core vars from PDF (e.g., GRAPROES for avg schooling, POCUPADA % employed, VPH_DRENAJ % drainage). Add via `src/index_calc.py`.
- **Updates:** For Censo 2025 (microdata Q2 2026), swap raw path and re-run ingestion.

## Outputs & App Integration
- **Processed artifacts:** Parquet/GeoJSON in subdirs (ignored if large; ~100MB national).
- **For MexiMaps:** Hex GeoJSONs load in Leaflet/Mapbox (e.g., color by SCI: green 70–100). Popups: "SCI: 82 | Pop: 1.2k | Education: 8.5 yrs".
- **Validation:** SCI correlates ~0.7–0.8 with crime proxies (test in 05_); aim 80%+ alignment to GDL/CDMX baselines.

## Notes
- **Ethics:** SCI proxies structural factors (PDF pp. 1–2: sociodemographics, urban housing)—disclaim correlations ≠ causation; anonymize for app.
- **License:** INEGI data CC-BY; cite "Censo 2020, INEGI" in app footer.
- **Troubleshooting:** Encoding errors? Use latin-1 fallback (script handles). Memory? Chunk CSVs in ingestion.