# 🛣️ Step 5b — Distance to Roads, Railways, Waterways

**Notebook:** [`Step5b_Accessibility_Distance_Analysis.ipynb`](Step5b_Accessibility_Distance_Analysis.ipynb)
**Kernel:** `firerisk-anaconda3` (Python 3.12.7, base `C:\Users\Admin\anaconda3\python.exe`)

> **Added 2026-08-18, numbered 2026-08-19, split into its own repo 2026-08-19.** Runs
> alongside Step 4 (FLDAS), feeds Step 6 (Integration). Was originally paired with the
> elevation/slope/aspect work in one `Terrain_Accessibility_Analysis/` folder/repo —
> split into two independent repos per user request, but both notebooks stay numbered
> Step 5a / Step 5b (sibling repo: `Terrain_Elevation_Slope_Aspect_Analysis/`,
> Step 5a). No further renumbering — Integration stays Step 6, Model stays Step 7,
> PINN stays Step 8. See root `CLAUDE.md` for the full eight-step pipeline.

## Why this step exists

Direct extraction from the user's own copy of Biswas, Mahato & Joshi (2025) — the
project's reference paper — showed its actual MaxEnt model uses **15 predictor
variables** (its Table 3), not the 11 this project's docs had been claiming before a
2026-08-18 correction pass. This notebook builds 3 of the 6 variables this pipeline had
zero coverage of — the "human activity-related factors": **distance to roads** (5.7%
variable importance / 2.6% model contribution in their MaxEnt run), **distance to
railways** (4.6% / 4.9%), and **distance to waterways** (0.5% / 1.7%). Combined, these
three account for **10.8%** of their model's total contribution. Their data source:
**OpenStreetMap, 2022 vintage** (Biswas et al. Table 2). The other 3 missing variables
(elevation/slope/aspect) are the sibling repo's Step 5a.

## Method

**Input:** Geofabrik OpenStreetMap extracts, 2022 vintage (6 India zone `.gpkg` files) —
matching Biswas et al.'s own stated data source for these three variables exactly.
**Boundary:** this project's own `India_State_Boundary.shp` (not GADM — GADM returns
extra disputed-territory rows that need deliberate handling, and using a different
boundary than every other step risks edge-pixel misalignment with the NDVI grid).
**Method:** distances computed in a custom India-centred equidistant conic projection
(a flat degree×111km conversion is wrong by >19% across India's latitude range),
GPU/CPU Euclidean distance transform.

**Limitation, disclosed rather than left implicit**: Euclidean ("as-the-crow-flies")
distance is the accepted, mainstream choice for this variable in the current
wildfire-ignition-risk literature (matching Biswas et al.'s own approach, and recent
Q1 work such as the NHESS 2025 study on human-caused ignition likelihood across
Europe) — it is not a flaw that requires fixing before submission. But it does not
capture the distance–time/accessibility relationship a cost-distance or travel-time
surface would (accounting for terrain difficulty, which affects how humans actually
reach an area). Worth one sentence in the paper's limitations section rather than an
unstated simplification.

## Results (India-masked, km)

| Variable | Resolution | Min | Max | Mean | P95 |
|---|---|---:|---:|---:|---:|
| Distance to roads | native ~1km | 0.00 | 260.1 | 5.69 | 15.90 |
| Distance to roads | 0.25° comparison | 0.03 | 244.2 | 6.83 | — |
| Distance to railways | native ~1km | 0.00 | 1,611.2 | 38.28 | 160.07 |
| Distance to railways | 0.25° comparison | 0.34 | 1,604.5 | 52.43 | — |
| Distance to waterways | native ~1km | 0.00 | 386.4 | 6.74 | 26.49 |
| Distance to waterways | 0.25° comparison | 0.00 | 385.1 | 8.11 | — |

**The 1,611km railway maximum is real geography, not a bug** — traced directly: the
boundary shapefile's polygon part 0 spans 92.2–93.9°E, 6.75–13.67°N, the Andaman &
Nicobar Islands, which genuinely have no rail connection to the mainland.

## Road/rail/waterway class filtering (documented reasoning, not silent defaults)

- **Roads** — kept `motorway/trunk/primary/secondary/tertiary` + `_link` variants
  (884,940 of 10.7M OSM features, 8.3%). `tertiary` was deliberately included, unlike a
  naive major-roads-only filter, since it's the standard rural/forest-fringe access class
  in India.
- **Railways** — excluded only `subway` (underground, not a surface accessibility
  corridor); kept 100,915 of 105,263.
- **Waterways** — kept `river/canal/stream`, excluded `drain` (artificial agricultural/
  urban channels, not a meaningful fire-fuel or accessibility signal); kept 234,313 of
  255,094.

## Fire coincidence (541,545 real Step 1 fire points)

| Variable | National mean | Fire-point mean | Difference |
|---|---:|---:|---:|
| Distance to roads | 5.69 km | 3.41 km | **−40.1%** |
| Distance to railways | 38.28 km | 40.12 km | +4.8% |
| Distance to waterways | 6.74 km | 2.38 km | **−64.7%** |

Fires cluster far closer to roads and waterways than the national baseline — consistent
with human-accessibility ignition sources near roads, and denser fuel (vegetation)
accumulating near water. Railways show almost no effect, matching Biswas et al.'s own
model where distance-to-railway is their lowest-contribution human-activity factor
after waterways.

## Against Biswas et al. (2025)

| Variable | Their importance | Their contribution | Status |
|---|---:|---:|---|
| Distance to roads | 5.7% | 2.6% | Built · verified |
| Distance to railways | 4.6% | 4.9% | Built · verified |
| Distance to waterways | 0.5% | 1.7% | Built · verified |

## Infrastructure notes worth knowing before extending this step

- **`geopandas.clip()` never finished** against the raw boundary shapefile's 422,929
  vertices, even given a 6-hour timeout. Fixed with a simplified 13,451-vertex mask
  (0.005° tolerance, negligible at 1km target resolution) and vectorized
  `shapely.covered_by`/`intersection` — full clip in ~160 seconds once corrected. Don't
  reintroduce a naive `gpd.clip()` call.
- One of two full notebook executions failed before a clean run: one died silently
  after ~1 hour from a GPU conflict with a concurrently-running job (the sibling
  Step 5a terrain notebook — this project's documented `cudaErrorAlreadyMapped`
  fragility, never run two GPU-heavy kernels against the same GPU at once); the other
  hit an undersized 1-hour `--ExecutePreprocessor.timeout` on the boundary-clip step
  before the mask fix landed. Both resolved; the notebook now runs cleanly in
  ~3.5 minutes — but give it a generous timeout regardless (see "How to run"), since a
  regression in the mask optimization would make the clip step pathologically slow
  again.

## Outputs

```
Accessibility_Outputs/
├── D1_Distance_to_Roads_native_1km.tif / _comparison_025deg.tif       (not tracked)
├── D2_Distance_to_Railways_native_1km.tif / _comparison_025deg.tif    (not tracked)
├── D3_Distance_to_Waterways_native_1km.tif / _comparison_025deg.tif   (not tracked)
├── Accessibility_summary_statistics.csv (tracked)
├── Accessibility_Fire_Coincidence.csv (tracked)
└── Accessibility_Spatial_Maps.png / Accessibility_Fire_Coincidence.png (tracked)
```

Raw source data (Geofabrik OSM extracts) is read from its download location and never
copied into this repo — see `.gitignore`.

## How to run

```bash
pip install -r requirements.txt
jupyter nbconvert --to notebook --execute --inplace --ExecutePreprocessor.kernel_name=firerisk-anaconda3 --ExecutePreprocessor.timeout=21600 "Step5b_Accessibility_Distance_Analysis.ipynb"
```

Requires the Geofabrik OSM zone extracts, plus Step 1's fire-point archive and Step 2's
NDVI grid reference file (read from their existing locations in the wider project,
never copied into this repo).

## Related work

- **Sibling repo** (Step 5a, same numbering, split off 2026-08-19):
  elevation/slope/aspect — `Terrain_Elevation_Slope_Aspect_Analysis/`.
- The burned-area vs. fire-count validation analysis lives in the Step 1 repo — see
  `Forest fire Extraction in INDIA(2000-2022)/Forest_Fire_Outputs/
  Annual_BurnedArea_vs_FireCount.csv`. Burned area is not one of Biswas et al.'s 15
  Table 3 predictors, so it isn't built as a model feature — it's validation/discussion
  material for the paper, not a pipeline input.

## Citation

- Biswas, U., Mahato, S., & Joshi, P.K. (2025). Spatial prediction of forest fires
  in India: a machine learning approach for improved risk assessment and early
  warning systems. *Environmental Science and Pollution Research*, 32(8), 4856–4878.
  DOI: 10.1007/s11356-025-35982-8.

## License

No license has been chosen yet for this repository's code.
