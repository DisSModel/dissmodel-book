# Chapter 11: Migrating from TerraME/LUCCME to DisSModel

*Part V — Migration & Reference*

## Learning objectives

- Translate TerraME/LUCCME concepts to their DisSModel equivalents
- Know what numerical equivalence guarantees already exist
- Plan an incremental migration of an existing model

## 11.1 Why migrate

TerraME is C++/Lua, with its latest release (2.0.1) from August 2020 and
installation via OS-specific installers. DisSModel is Python-native
(`pip install`), runs from notebook to distributed cluster with the same
code, and is numerically validated against TerraME reference outputs
where applicable (e.g. `brmangue-dissmodel`).

## 11.2 General equivalence table

| TerraME/LUCCME | DisSModel |
|---|---|
| TerraME (framework) | `dissmodel` |
| LUCCME | `disslucc-continuous` |
| CLUE-S-like (discrete) | `disslucc-discrete` |
| TerraLib | `geopandas` / `rasterio` |
| FillCell | `disscube` / `disslucc_continuous.io` |
| `Agent`/`Society` | `dissmodel-abm` — see Ch. 6 (Agent-Based Modeling) |
| `init(self)` | `setup()` |
| `execute(self)` | `execute()` |

## 11.3 Step-by-step incremental migration

1. **Identify the original model's paradigm.** A TerraME `CellularSpace`
   with a per-cell rule maps to Chapter 5 (`dissmodel-ca`,
   `CellularAutomaton`); a stock/flow model maps to Chapter 4
   (`dissmodel-sysdyn`, plain `Model` + `@track_plot`); a TerraME `Agent`/
   `Society` model maps to Chapter 6 (`dissmodel-abm`, `AgentModel`/
   `Society`); a LUCCME-style continuous allocation maps to
   `disslucc-continuous`, and a CLUE-S-like discrete allocation to
   `disslucc-discrete` (Chapter 7); a coupled raster/vector domain model
   in the style of BR-MANGUE maps to the `SpatialModel`/`RasterModel`
   pair directly (Chapter 8 as a worked example).

2. **Choose the corresponding satellite package** (or, if none fits,
   build directly on `dissmodel.core`/`dissmodel.geo` as `dissmodel-ca`
   and `dissmodel-sysdyn` themselves do) — install it per Chapter 3
   (`pip install "git+https://github.com/DisSModel/<repo>.git"`, since no
   extension package is on PyPI).

3. **Map input data.** Where TerraME used TerraLib to read
   `CellularSpace` data, DisSModel uses `geopandas`/`rasterio` directly:
   a `.shp`/`.gpkg` becomes a `GeoDataFrame` for the vector substrate, a
   GeoTIFF becomes a `RasterBackend` for the raster substrate (Chapter
   2). If the original data needs alignment to a modeling grid first
   (resampling, zonal aggregation, deriving driver variables from a raw
   source), that step is `disscube`'s job (Chapter 10), not something
   `disslucc`/`dissmodel-ca` executors do themselves.

4. **Validate outputs against the original model** before trusting a
   migrated model in production. The ecosystem has three real precedents
   for how to do this, in increasing order of rigor:
   - `disslucc-continuous`'s `benchmark/` directory and
     `LUCCBenchmarkExecutor` compare Vector vs. Raster vs. a TerraME/LUCCME
     reference dataset, asserting MAE/RMSE below a configurable
     `tolerance` (Chapter 7, 7.3).
   - `brmangue-dissmodel`'s `main_benchmark.py` runs the same tolerance-based
     comparison for a continuous coastal model (Chapter 8).
   - `disslucc-discrete`'s `benchmark/validate_lab6.py` and
     `tests/test_validation_lab6.py` go further, asserting **exact**
     100% cell-level parity (accuracy, Cohen's κ, F1 all equal to 1.0)
     against the TerraME/LuccME Lab6 reference on the Moju dataset
     (Chapter 7, 7.4) — the bar to aim for whenever the migrated
     variable is categorical rather than continuous.

   Pick the validation style that matches your model's output type
   (continuous → tolerance-based; categorical → exact parity), not the
   other way around.

## 11.4 Existing validation cases in the ecosystem

- **`brmangue-dissmodel`** — the raster substrate is validated against
  TerraME golden-output CSVs (`validation_executor.py`,
  `tests/fixtures/golden/step_NN.csv`); raster and vector substrates are
  additionally cross-checked against each other via `main_benchmark.py`
  (match%/MAE/RMSE per band). See Chapter 8.
- **`disslucc-continuous`** — `LUCCBenchmarkExecutor` runs Vector vs.
  Raster vs. a TerraME/LUCCME reference in one pass; the automated test
  suite asserts MAE/RMSE below `tolerance=0.01` for both
  `Vector_vs_TerraME` and `Raster_vs_TerraME`. See Chapter 7, 7.3.
- **`disslucc-discrete`** — **Lab6 validation**: a cell-by-cell parity
  check against the original TerraME/LuccME reference implementation,
  run on the Moju dataset (1999–2004). The integration test
  (`tests/test_validation_lab6.py`) instantiates `LuccValidationExecutor`
  and asserts **100.00% accuracy, Cohen's κ = 1.0000, F1 = 1.0000, 0
  false positives/negatives** — exact numerical parity, not a tolerance
  band, because discrete land-use allocation is categorical. This test
  runs automatically on every CI build. See Chapter 7, 7.4.

Across all three, the pattern is the same: never trust a migrated model
in production without first running it against the original TerraME/
LUCCME output on a known dataset, using whichever comparison granularity
(tolerance vs. exact) matches the variable's type.

## Exercises

1. Take a hypothetical TerraME model with a `CellularSpace` where each
   cell's `execute(self)` looks only at its 4 cardinal neighbors. Using
   11.2's equivalence table and Chapter 5's `FireModel` as a reference,
   write the class skeleton (imports, base class, neighborhood strategy)
   you'd start from in DisSModel — you don't need to implement `rule()`,
   just get the setup right.
2. A colleague migrating a LUCCME model asks whether to validate it with
   `disslucc-continuous`'s benchmark tolerance approach or
   `disslucc-discrete`'s exact Lab6-style parity check. What single
   question about their model's output (from 11.4) determines the
   answer?
3. `disscube` has no direct code dependency from `disslucc-continuous`/
   `disslucc-discrete` (Chapter 10, 10.6) — the connection is a shared
   `RasterBackend` contract. If you were migrating a TerraME model whose
   driver variables came from a raw MapBiomas raster, at which step in
   11.3's roadmap would `disscube` enter the picture, and why not
   earlier or later?
4. Chapter 1 states TerraME's last release was August 2020. If you
   inherited a TerraME model today, what does that fact alone tell you
   about the urgency of migration versus, say, a model built on an
   actively maintained but merely unfamiliar framework?

## Summary

Migrating from TerraME/LUCCME to DisSModel is a mapping exercise more
than a rewrite: `init(self)`/`execute(self)` become `setup()`/`execute()`
on the same four-hook `Model` lifecycle (Chapter 2), `CellularSpace`
becomes `SpatialModel`/`RasterModel`, and LUCCME's Demand/Potential/
Allocation triad becomes `disslucc-continuous` or `disslucc-discrete`
depending on whether the migrated variable is fractional or categorical
(Chapter 7). The one step that must never be skipped is validation —
the ecosystem's own precedents (`brmangue-dissmodel`'s golden-CSV and
raster/vector checks, `disslucc-continuous`'s tolerance-based benchmark,
`disslucc-discrete`'s exact Lab6 parity) all treat "runs without errors"
as necessary but nowhere near sufficient; the actual bar is numerical
agreement with the original TerraME/LUCCME output on a known dataset,
automated in CI wherever possible. Even where a direct satellite package
exists (`dissmodel-ca`, `dissmodel-sysdyn`, `dissmodel-abm`, `disslucc-*`),
it may not cover every TerraME feature yet — Chapter 6, 6.6 documents
`dissmodel-abm`'s own gaps (no raster substrate, no social networks, no
state machines) as a concrete example of checking a package's stated
scope before assuming a migration is complete.
