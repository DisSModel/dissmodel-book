# Chapter 10: Spatial Data Cubes

*Part IV — Data & Infrastructure*

Implemented by the [`disscube`](https://github.com/DisSModel/disscube) package.

## Learning objectives

- Understand the `SpatialSource → Derivation → Variable → DerivedVariable` pipeline
- Register a grid and a spatial source
- Derive a variable and integrate it into a LUCC model

## 10.1 Core concept

DisSCube is the spatial data cube engine of the DisSModel ecosystem. It
converts raw geospatial sources (raster, vector) into derived variables
aligned to LUCC modeling grids, ready for Cellular Automata models and
spatio-temporal analysis.

```
SpatialSource  →  Derivation  →  Variable  →  DerivedVariable (Zarr)
```

A **source** goes through a **derivation** that applies an **operator** to
a **grid**, producing a **derived variable** registered in the SQLite
catalog and stored in Zarr.

## 10.2 Basic flow

```python
from disscube.client import CubeClient
from disscube.utils.grids import register_local_grid

cube = CubeClient(catalog="catalog.db", store="./data/")
grid = register_local_grid(cube, name="AC", bbox_geo=(-73.99, -11.15, -66.62, -7.11), resolution=5_000.0)
```

```python
from disscube.models import SpatialSource

cube.register_spatial_source(SpatialSource(
    id="mapbiomas_2020", name="MapBiomas Acre 2020", format="raster",
    asset_url="data/raw/mapbiomas_2020.tif", crs="EPSG:4326", time=2020,
))
```

```python
from disscube.derivation import Derivation

d = Derivation(target="forest_pct", source_id="mapbiomas_2020",
                operator="percentage", class_code=3, role="driver",
                valid_from="2020", valid_until="2020")
cube.derive_declarative(d, grid_id="AC/5km")
```

## 10.3 Integration with DisSModel

```python
backend = cube.to_lucc_data(["forest_pct", "dist_roads"], grid_id="AC/5km")
```

`CubeClient.to_lucc_data()` is, per its own docstring, "the standard
integration point for the DisSModel ecosystem": it returns a
`dissmodel.geo.raster.backend.RasterBackend` — the exact raster substrate
type from Chapter 2 — containing every requested variable as a named
band. Static variables load as plain `(y, x)` arrays; temporal variables
load as `(time, y, x)` arrays with an explicit time axis in
`backend.time_coords`, so a `RasterModel`/`RasterCellularAutomaton`
retrieves a single time slice via `backend.get(name, time=step)` with no
change to existing executor code. An optional `period=(start, end)` tuple
filters which time slices of a temporal variable get loaded — static
variables ignore it.

### TerraME `fillCellularSpace` correspondence

DisSCube's own `docs/terrame_fill_correspondence.md` documents this
directly: DisSCube's derivation layer is "the conceptual successor of
TerraME's `fillCellularSpace`" (the *Fill Cells* operation), reformulated
as a reproducible, catalogued data-cube layer. TerraME populates a
cellular space cell-by-cell in Lua through fill *strategies* (`area`,
`presence`, `count`, `distance`, `percentage`, `majority`, `average`,
...); DisSCube keeps the same semantic vocabulary but expresses each
strategy as a typed, auto-registered `Operator`:

| TerraME fill strategy | DisSCube operator | Status |
|---|---|---|
| `presence` | `presence` | implemented |
| `area` / `coverage` / `percentage` | `percentage` (window-based) | implemented, requires `class_code` |
| `majority` / `mode` | `majority` (window-based) | implemented; ties resolve to smallest class value |
| `minority` | `minority` (window-based) | implemented |
| `count` | `count` | implemented (proximity operator) |
| `distance` | `min_distance` | implemented (EDT × resolution) |
| `average` / `mean` | `mean` | implemented |
| `sum` | `sum` | implemented |
| `minimum` / `maximum` | `min` / `max` | implemented |
| `stdev` / `standardDeviation` | `std` (window-based) | implemented, true per-cell std over valid pixels |
| `attribute` (value copy) | `attribute` | implemented (vector) |

The document is explicit that "the advance is not the set of operations
— those are deliberately faithful to TerraME — but the engineering
around them": every derived product carries a deterministic `spec_hash`
for reproducibility, categorical/std operators aggregate over real
per-cell windows on a grid-origin-snapped fine array (so a 30m→1000m
resampling is well-defined rather than silently approximated), and
per-cell coverage/dominance purity is computed as first-class metadata —
"implicit in TerraME; here it is named and measurable." The same document
also names a known gap relative to TerraME: vector-source fractional
coverage is currently rasterized rather than area-weighted (matching the
"Known Limitations" caveat in 10's Summary below).

`to_lucc_data()`'s own docstring lists three explicitly **open contract
decisions** ("to be resolved before 1.0") worth knowing before relying on
it in a pipeline:

1. Whether year-only temporal format (`"2020"`) is validated at
   construction time, or left to fail silently later.
2. Whether a temporal variable with all slices outside `period` should be
   skipped with a warning (current behavior), raise, or return NaN — the
   caller currently cannot distinguish "static, so `period` was ignored"
   from "existed but fell outside the requested range".
3. Whether requesting variables that are *all* filtered out by `period`
   should raise, instead of silently returning a `RasterBackend` that
   holds no data (current behavior).

## 10.4 Relationship to Chapter 7 (DisSLUCC)

There is no direct Python import from `disslucc-continuous` or
`disslucc-discrete` into `disscube` today — the integration is a
*contract* match, not a dependency. `disslucc-continuous`'s
`ClueLikeRasterExecutor.load()` is documented to return "the `RasterBackend`
... expos[ing] bands named after `land_use_types`" — precisely the object
shape `to_lucc_data()` produces. In practice, the pipeline looks like:

```python
cube    = CubeClient(catalog="catalog.db", store="./data/")
backend = cube.to_lucc_data(
    ["forest_pct", "dist_roads", "assentamen", "uc_us"],
    grid_id="AC/5km",
)

# feed the backend as a driver source into a disslucc-continuous
# raster model / executor — same RasterBackend contract as Chapter 2
potential = PotentialLinearRegression(gdf=..., potential_data=[...])
```

In other words: DisSCube is the piece of the ecosystem responsible for
turning raw MapBiomas-style raster/vector sources into the named,
grid-aligned driver variables (`assentamen`, `dist_riobr`, `fertilidad`,
...) that show up as `betas` keys in a `disslucc-continuous`/
`disslucc-discrete` TOML config (Chapter 7, 9.5) — it is the data
preparation stage upstream of the LUCC potential/allocation components,
not a runtime dependency of them.

## Exercises

1. Register a grid and a raster `SpatialSource` following 10.2, then
   derive a `percentage` variable. Why does the `percentage` operator
   require `class_code` while `mean`/`sum`/`std` don't (see the operator
   table in the README)?
2. Call `cube.load("forest_pct", grid_id="AC/5km")` without a `tile_id`
   on a grid where the same variable was derived for multiple tiles.
   Per DisSCube's documented limitations, what does `load()` do instead
   of raising an error — and why is "always specify `tile_id` in
   multi-tile workloads" the recommended workaround rather than a fix?
3. `Derivation.purity_threshold` is included in the `spec_hash` cache key
   but is not applied to the output. What concrete problem could this
   cause if you set `purity_threshold=0.9` expecting it to filter noisy
   cells, then compared two runs with different thresholds?
4. Vector-source operators (`majority`, `percentage`, `attribute`,
   `presence`, `minority`) estimate cell coverage by rasterizing and
   counting pixels, not by exact geometric intersection area. Describe a
   scenario (grid resolution vs. source polygon size) where this
   approximation would meaningfully diverge from true area-weighted
   coverage — and what the README recommends as a workaround.
5. Using `to_lucc_data()`'s three open contract decisions (10.3) as a
   checklist, write a short test plan for a pipeline that combines a
   static variable (`dist_roads`) and a temporal one (`forest_pct` with
   a `period` filter) — what would you assert about the resulting
   `RasterBackend` in each case?

## Summary

DisSCube is the ecosystem's data-preparation layer: `SpatialSource →
Derivation → Variable → DerivedVariable` turns raw raster/vector sources
into named, grid-aligned variables cached in Zarr and cataloged in
SQLite, with `to_lucc_data()` as the single, documented handoff point
into the rest of DisSModel — it returns a plain `RasterBackend`, the same
type any `RasterModel`/`RasterCellularAutomaton` and any
`disslucc-continuous`/`disslucc-discrete` raster executor already
consumes (Chapters 2 and 7), with no disscube-specific type leaking into
model code. The package is explicitly **Alpha**: the README's own
"Known Limitations" section documents, as scope decisions rather than
bugs, that processing is in-memory/single-tile with no lazy or
distributed execution, that vector aggregation approximates coverage by
pixel counting rather than exact area, that `load()` silently picks the
first tile when disambiguation is needed, and that `purity_threshold`
and `SpatialRelation` are modeled in the schema but not yet wired into
the pipeline. None of this blocks the core declarative flow (register
grid → register source → derive → load → `to_lucc_data()`), but it does
mean production pipelines at continental scale (e.g. `BR/1km`) currently
need an explicit per-tile loop, not a single `derive()` call.
