# 🛣️ Step 5b — Distance to Roads, Railways, Waterways

<!-- AUDIT-UPDATE-2026-09-25 -->
> ### Audit update (2026-09-25)
> This repository's step was recalculated independently from the raw data in a full end-to-end audit.
> **Corrected results, reproduction checks and audit code: [`AUDIT_2026-09-25.md`](AUDIT_2026-09-25.md)** and `audit_2026-09-25/`.
> Earlier text below is kept for the record (it also remains in the git history). Statements superseded by the audit:
>
> - None of the specific numbers in this file were superseded; see `AUDIT_2026-09-25.md` for the audited results of this step.
>
> **Pipeline rerun (2026-09-25).** The notebook was corrected and re-executed end-to-end, and the result tables below now come from that rerun:
> - Fire points are rasterised to their *containing* pixel (`floor`, not `round`, which displaced 74.9% of points by one pixel).
> - Outputs go to this repository's own `Accessibility_Outputs/`.
> - Statistics use the final v2 NDVI-valid India mask (4,160,768 pixels, exactly the pixel set of Step 6's v2 table).
>
> The previous table used an earlier, wider mask (4,185,401 pixels); those values are in the git history.
>
> **Validation:**
> - The distance rasters are identical (max difference 0) to the `dist_*` columns of Step 6's v2 table.
> - On the audit's 4,161,009-pixel mask, the road mean is 5.6059 km, matching the independent audit recalculation (`audit_2026-09-25/results/R7_report.json`, r = 0.9999994).
<!-- AUDIT-UPDATE-2026-09-25 -->


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
three account for **10.8% of their model's total importance and 9.2% of its total
contribution**. **Corrected 2026-09-23**: an earlier version of this sentence claimed
the human-activity group's contribution (10.8%, actually the importance sum) was
"notably larger" than the topographic group's — that comparison mixed an importance
sum against a mislabeled figure from the sibling Step 5a README and does not hold up
once both are computed correctly on the same basis: by combined *importance*,
human-activity (10.8%) does modestly exceed topographic (9.7%); by combined
*contribution*, topographic (22.5%, driven by slope's own 16.7%) is more than double
human-activity's (9.2%). The genuinely interesting comparison is therefore the
opposite of what was previously stated: despite having *lower* combined importance
than the human-activity group, the topographic group's actual model contribution is
substantially larger — consistent with Biswas et al.'s own MaxEnt ranking slope as
their single second-highest contribution variable. Their data source:
**OpenStreetMap, 2022 vintage** (Biswas et al. Table 2). The other 3 missing variables
(elevation/slope/aspect) are the sibling repo's Step 5a. Together, Step 5a + 5b close
all 6 of the pipeline's remaining gaps, bringing the full pipeline (Steps 1–4 plus this
pair) to **15/15 predictor-group parity** with Biswas et al.'s Table 3 — wired into
Step 6's integrated stack on 2026-08-20.

## Why this step, and how (plain-language walkthrough)

Distance to roads, railways, and waterways are standard proxies for **human-caused
ignition likelihood**: most forest fires in India (and globally) are started by people —
agricultural burning, discarded cigarettes/matches, campfires, deliberate clearing — so
closer proximity to infrastructure that brings people into a forest is a reasonable proxy
for ignition-source density, independent of the fuel/climate conditions the rest of this
pipeline's other variables already capture. Waterways add a second, distinct mechanism on
top of accessibility: riparian corridors support denser vegetation, which is both more
fuel and (in dry-season India) more attractive to human activity such as fishing, grazing,
and settlement, making proximity to water a double signal rather than pure accessibility.
These three variables were the last human-activity gap in this pipeline — every other
step (NDVI, LST, FLDAS climatic variables, land cover, and the sibling Step 5a terrain
variables) was already built before this notebook was added 2026-08-18/19/20. The output
here (three GeoTIFFs: distance to roads/railways/waterways × native-1km and
0.25°-comparison) is consumed by Step 6, which stacks it alongside every other step's
rasters into the single `Integrated_FireRisk_Stack.tif` / `Integrated_FireRisk_
Pixels.parquet` that Step 7's Random Forest/MaxEnt models and Step 8's CDR-PINN both
train on directly.

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

## Results (India-masked, km; rerun 2026-09-25)

| Variable | Resolution | Min | Max | Mean | P95 |
|---|---|---:|---:|---:|---:|
| Distance to roads | native ~1km | 0.00 | 260.1 | 5.60 | 15.21 |
| Distance to roads | 0.25° comparison | 0.03 | 243.7 | 6.65 | — |
| Distance to railways | native ~1km | 0.00 | 1,610.0 | 37.44 | 157.50 |
| Distance to railways | 0.25° comparison | 0.22 | 1,603.7 | 51.01 | — |
| Distance to waterways | native ~1km | 0.00 | 385.6 | 6.68 | 25.86 |
| Distance to waterways | 0.25° comparison | 0.00 | 384.9 | 7.90 | — |

**The 1,610km railway maximum is real geography, not a bug** — traced directly: the
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

## Fire coincidence (Step 1 fire points; rerun 2026-09-25)

541,017 of the 541,545 Step 1 points fall on a valid in-India pixel of the final mask.

| Variable | National mean | Fire-point mean | Difference |
|---|---:|---:|---:|
| Distance to roads | 5.60 km | 3.42 km | **−38.9%** |
| Distance to railways | 37.44 km | 39.99 km | +6.8% |
| Distance to waterways | 6.68 km | 2.38 km | **−64.4%** |

Fires cluster far closer to roads and waterways than the national baseline — consistent
with human-accessibility ignition sources near roads, and denser fuel (vegetation)
accumulating near water. Railways show almost no effect, matching Biswas et al.'s own
model where distance-to-railway is their lowest-contribution human-activity factor
after waterways.

## Comparison against Biswas et al. (2025)

This project computes distances with a **full GPU Euclidean distance transform** over
Geofabrik OSM 2022 vector data, at the shared grid's native ~1km resolution (and
projected through a custom India-centred equidistant conic projection to avoid the
>19% latitude-dependent error a flat degree×111km conversion would introduce), before
producing a 0.25° comparison raster purely for benchmarking against Biswas et al.'s own
working resolution. Biswas et al.'s Table 2 states only "proximity...from OSM" and names
no distance algorithm — Euclidean, network, and cost-distance are all consistent with
that wording. Euclidean distance transform is the defensible, standard choice adopted
here (matching the field's mainstream practice, see the limitation note above), but it
is **not explicitly confirmed** as what Biswas et al. themselves used, and this project
does not claim otherwise. Their own distances also appear to have been rasterized
directly at 0.25° from an unspecified source resolution, whereas this project resolves
the underlying OSM vector geometry at native resolution before any aggregation — a
methodological improvement in precision, though not one directly comparable against
Biswas et al.'s numbers since they never published raw distance statistics, only
MaxEnt importance/contribution percentages.

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
