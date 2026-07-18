# Chapter 10: Spatial Data Cubes

*Part IV — Data & Infrastructure*

Implemented by the [`disscube`](https://github.com/DisSModel/disscube) package.

## Learning objectives

- Know what a data cube is, and why it's a semantic guarantee (alignment
  plus indexability), not a file format
- Understand why the ecosystem needs *two* cube implementations — one at
  rest, one in motion — and what each is built for
- Register a grid and a spatial source, and derive a variable
- Bridge a derived cube into a running DisSModel simulation

## 10.1 What is a data cube

In Earth-observation literature (the datacube manifesto tradition, the
Open Data Cube project, the Brazil Data Cube), a **data cube** is a
regular multidimensional array — typically `(variable, time, y, x)` —
with three properties that distinguish it from an arbitrary collection
of files:

1. **Spatial alignment.** Every variable and every date shares the same
   grid — same CRS, same resolution, same origin. Cell `(i, j)` is the
   same physical location in any variable, at any time.
2. **An explicit, regular time axis.** Time is an indexable dimension,
   not a filename suffix or an accident of capture dates.
3. **Analysis-ready.** The cost of preparation — reprojecting,
   resampling, aligning — is paid once, at ingestion, not repeated on
   every query.

A common misconception is that a data cube must be one giant file. It
doesn't have to be — the reference implementation in this space, the
[Open Data Cube](https://www.opendatacube.org/), is an index (Postgres)
plus individual files on disk; the "cube" materializes on demand when a
query assembles the aligned array from the index. **A cube is a semantic
guarantee — alignment plus indexability — not a storage format.** That
distinction is what the rest of this chapter is built on.

**Materialized vs. virtual.** A cube can be *materialized* (pixels
already computed and written to disk as a new product) or *virtual*
(nothing computed yet — the array is a promise, resolved only when
queried). Both are legitimate; which one to use is an architectural
choice, not a universal rule, and Section 10.9 comes back to exactly
this choice when contrasting `disscube` with the Brazil Data Cube and
Google Earth Engine.

## 10.2 Two regimes: the cube at rest, the cube in motion

The DisSModel ecosystem's data-cube story is not one implementation —
it's two, deliberately, because a data cube is used for genuinely
different things depending on where it sits in the pipeline:

| | `disscube` (rest) | `RasterBackend` (motion) |
|---|---|---|
| Role | Cube infrastructure | One cube instance |
| Persistence | Yes — Zarr + SQLite/JSON catalog | No — lives in process memory |
| Scope | A *family* of cubes, multi-grid | One cube, one grid, one run |
| Mutation | New `DerivedVariable` per derivation | In-place array writes, every tick |
| Typical consumer | Analysis, cataloging, publication | A running `Environment` (Chapter 2) |

Both satisfy the definition in 10.1 — `RasterBackend` enforces spatial
alignment by construction (one `shape`/`transform`/`crs` for every
variable it holds), exposes an indexable time axis (`time_coords`, with
ceiling lookup), and assumes alignment cost was already paid before data
reaches it. It is a cube. It just isn't the ecosystem's cube
*infrastructure* — that's `disscube`'s job. The split mirrors the Open
Data Cube itself: an index (`disscube`'s catalog) is a different thing
from the `xarray.Dataset` a query returns (`RasterBackend`, once a
`Environment` starts mutating it tick by tick).

**Two species of in-memory instance.** There's a finer distinction worth
naming: the `xarray.Dataset` that tools like `odc-stac`'s `load()` return
is typically **lazy** — backed by Dask, a graph of pending operations,
nothing in RAM until `.compute()`. That's ideal for analysis: define the
whole chain (clip, compute an index, reduce over time), and only then do
bytes move, once, optimized. `RasterBackend` is **materialized** — a
concrete `np.ndarray`, available immediately. That's mandatory for
simulation: a cellular automaton needs the *concrete* value at step `t`
to decide the transition to `t+1`; laziness is fundamentally incompatible
with that sequential dependency (a lazy cube run through a CA would
accumulate one new Dask graph layer per tick, recomputing the whole
history on every evaluation). Neither is a better substrate in general —
each fits its own regime. The boundary between them is `.compute()`
(implicit inside `RasterBackend.from_xarray`), the exact moment a cube
crosses from rest into motion.

## 10.3 `disscube`: the cube at rest

`disscube` converts raw geospatial sources (raster, vector) into derived
variables aligned to a modeling grid, cataloged for reuse:

```
SpatialSource  →  Derivation  →  Variable  →  DerivedVariable (Zarr)
```

- **`GridSpec`** is the alignment contract: `id`, CRS, resolution,
  bbox — plus a stable `cell_id(row, col)` (e.g. `"AC/5km:R0991C0047"`,
  literally `f"{self.id}:R{row:04d}C{col:04d}"`), so any variable derived
  on that grid is comparable cell-by-cell by construction.
- **`SpatialSource`** is a raw source registered in the catalog — local,
  HTTP, or S3 — with `asset_url`, `crs`, `bbox`, `time`.
- **`Derivation`** is a declarative recipe: which operator, applied to
  which source, produces which target variable. It's grid-independent —
  the same `Derivation` can run against `AC/5km` or `AC/1km`, producing
  two distinct cubes.
- **`DerivedVariable`** is the materialized product of a recipe on a
  specific grid: a Zarr `asset_url`, a `spec_hash` (deterministic
  SHA-256 of the recipe + grid — uniquely identifies the variable) and a
  `content_hash` (hash of the produced bytes).
- **`CatalogStore`** is a `Protocol` (with SQLite and JSON
  implementations today — a Postgres implementation could replace either
  without changing any caller) exposing both discovery
  (`search_derived_variables(grid_id, role, tile_id)`) and an execution
  cache (`get_derived_by_hash(spec_hash)` — a lookup that guarantees an
  identical recipe on an identical grid is never recomputed).

**One catalog, many cubes.** The grid *is* the cube's identity — `AC/5km`
and `AC/1km` are different cubes; cell `(i, j)` in one is not the same
place as `(i, j)` in the other. `disscube` doesn't manage a cube; it
manages a *family* of cubes over one shared catalog, where `Derivation`s
are reusable recipes and `spec_hash` (which folds in `grid_id`) prevents
collisions between products of different grids.

**Reproducibility as architecture, not discipline.** `spec_hash` ties
*what was asked for* (the recipe) to *what was produced* (the array) —
this is a property of the catalog's design, not something that depends
on a modeler's notes or external documentation. Every derived variable
carries its own proof of provenance.

## 10.4 Basic flow

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

## 10.5 The bridge: from the cube at rest to the cube in motion

The integration between the two regimes already exists as code, not as a
proposal — but it's worth being precise about the actual API, since it's
easy to assume a shape that isn't there.

`CubeClient.load(variable_id, tile_id=None, grid_id=None)` loads **one**
named variable as a plain `xr.DataArray` — it does not accept a list of
variables, and there is no module-level `disscube.load(...)` function.
The multi-variable handoff into DisSModel is a different, dedicated
method:

```python
backend = cube.to_lucc_data(["forest_pct", "dist_roads"], grid_id="AC/5km")
```

`CubeClient.to_lucc_data()` is, per its own docstring, "the standard
integration point for the DisSModel ecosystem": it calls `load()` once
per requested variable internally and assembles the result directly into
a `dissmodel.geo.raster.backend.RasterBackend` — the exact raster
substrate type from Chapter 2 — with every variable as a named band.
There is no separate `from_xarray()` step in this path; `to_lucc_data()`
builds the `RasterBackend` itself. Static variables load as plain `(y,
x)` arrays; temporal variables load as `(time, y, x)` arrays with an
explicit time axis in `backend.time_coords`, so a
`RasterModel`/`RasterCellularAutomaton` retrieves a single time slice via
`backend.get(name, time=step)` with no change to existing executor code.
An optional `period=(start, end)` tuple filters which time slices of a
temporal variable get loaded — static variables ignore it.

**`RasterBackend.from_xarray()` is the second, independent entry point**
into the motion regime — for any `xr.Dataset` with `(y, x)` or `(time, y,
x)` dims and named variables, regardless of who produced it. This
matters because it means DisSCube is not the *only* way to feed a
simulation: an `xr.Dataset` assembled by `odc-stac`'s `load()` from a
STAC catalog (the Brazil Data Cube, or a catalog `disscube` might publish
per 10.9) crosses into the motion regime through the exact same door:

```python
# Path A — disscube's own catalog (spec_hash-tracked provenance)
backend = cube.to_lucc_data(["uso", "alt", "solo"], grid_id="maranhao/30m")

# Path B — any STAC catalog, via odc-stac
from pystac_client import Client
from odc.stac import load

items = Client.open("https://some-catalog/stac").search(
    collections=["brmangue-maranhao"], bbox=[...],
).items()
ds = load(items, bands=["uso", "alt", "solo"], crs="EPSG:31983", resolution=30).compute()
backend = RasterBackend.from_xarray(ds)

# from here on, both paths are identical — the model never knows which door was used
```

`brmangue-dissmodel`'s executor demonstrates the real, shipped version of
this bridge:

```python
# brmangue/executors/raster_executor.py
@staticmethod
def from_cube(backend: RasterBackend) -> tuple:
    """Adapts a RasterBackend from DisSCube to the internal format
    expected by BrmangueRasterExecutor (backend, meta, start_time)."""
    meta = {"crs": backend.crs, "transform": backend.transform, "tags": {}}
    start_time = 1
    return backend, meta, start_time
```

And the actual synchronization mechanism that lets `FloodModel` and
`MangroveModel` share one backend safely within the same tick is
`dissmodel.geo.raster.sync_model.SyncRasterModel` — both models subclass
it and declare `self.land_use_types = ["uso", "alt", "solo"]` in
`setup()`; `SyncRasterModel.pre_execute()`/`post_execute()` then freezes
each into a `<name>_past` array in the shared `RasterBackend` before and
after every step, so both models read the same frozen state during a
tick — the Python equivalent of TerraME's `cell.past[attr]`. This is the
strongest real-world validation of the "motion" regime design: two
models, one grid, one clock, no risk of one seeing the other's partial
write.

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
"Known Limitations" caveat in the Summary below).

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

## 10.6 Relationship to Chapter 7 (DisSLUCC)

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

## 10.7 Multiple cubes and coupling

Three design questions come up whenever more than one grid is in play,
each with a concrete answer in the current codebase:

**Does changing the grid create a new cube?** Yes — the grid is the
cube's identity (10.3). The catalog is inherently multi-cube; a
`Derivation` is reusable across grids, but each instantiation on a
distinct grid is a distinct cube, with its own `spec_hash`-tracked
provenance.

**Does a model use only one cube?** A model *instance*, yes — a cellular
automaton's state lives on one grid, because its transition rule assumes
cell-by-cell alignment between state and drivers. `Environment` (Chapter
2), on the other hand, already coordinates multiple *models* — the
one-cube constraint is per model, not per simulation (`FloodModel` and
`MangroveModel` share one `RasterBackend`, one grid, 10.5).

**Do two cubes in play require coupling?** It depends on when information
crosses the boundary:

- **Static crossing** (before the simulation starts) is regridding — a
  `disscube` job. Resampling a variable from one grid to another is just
  another `Derivation`, with its own `spec_hash`. The model itself never
  sees two grids.
- **Dynamic crossing** (during the run, with tick-by-tick feedback
  between two models on different grids) is real coupling, and needs
  runtime translation. The vocabulary for it already exists in the
  catalog — `SpatialRelation` (`source_grid_id`, `target_grid_id`,
  `strategy: "simple" | "chooseone" | "keepinboth"`, confirmed directly
  in `disscube/models/grid.py`) — but it has no consumer yet: no pipeline
  stage reads it during computation today (which is exactly why it's
  deliberately excluded from `spec_hash` — including it would make the
  cache key sensitive to metadata that doesn't affect the result). The
  missing piece is an `Exchange`-style component registered in
  `Environment` like any other model, applying `SpatialRelation` each
  tick to translate state between two `RasterBackend`s.

## 10.8 Positioning: `disscube`, the Brazil Data Cube, and INPE's eCube draft

It's worth situating `disscube` against the wider Brazilian Earth
observation stack, because the comparison sharpens exactly what this
chapter has been building toward — a data cube used for *simulation*,
not only *observation*.

**The NDVI contrast.** The canonical thing a data cube does well is
something like NDVI: `(B08 - B04) / (B08 + B04)`, computed independently
per cell and per date, with no dependency on any neighbor or any earlier
time step. That's precisely why a *lazy* cube (`xarray` + Dask) handles
it well — the whole computation parallelizes trivially, and whether to
materialize is a free choice made once, at the end
(`.median(dim="time").compute()`). If, instead of NDVI, the computation
were "spread fire cell-by-cell, one year after another," no step of that
lazy pipeline would work — it needs `RasterBackend`, the tick loop, the
`_past` snapshot, arrays materialized from tick zero. NDVI is exactly the
case where a lazy cube shines *because it lacks* the dimension the motion
regime exists to resolve. Observation (NDVI, classification, change
detection) is point-to-point and lives at rest; simulation (LUCC, coastal
dynamics, fire spread) is sequential-with-state and lives in motion.

**Brazil Data Cube (BDC).** The BDC is production infrastructure for
*observation* cubes — analysis-ready satellite mosaics, indexed via STAC.
Everything typically computed over it (NDVI, time-series classification)
is point-to-point, living entirely in the rest regime — DisSModel doesn't
compete with that; it *consumes* that kind of data as an input. The
complexity DisSModel adds isn't one of volume (the BDC processes
terabytes of national-scale imagery, and is objectively larger on that
axis) — it's one of *kind*: state that depends on the previous step,
something an observation cube never needed to solve because observation
has no "next step," only "next capture."

**eCube.** INPE has circulated a draft specification, internally called
eCube, for a cube of *derived variables for modeling* — one step past
BDC's observation focus. Its fill-cell operator list (majority, presence,
minimum distance, percentage, sum, mean, std, count) converges almost
one-to-one with `disscube`'s own `OPERATOR_REGISTRY` (10.5), and both
trace back to the same TerraME `fillCellularSpace` ancestor. This is
independent convergence, not coordination — two efforts starting from the
same TerraME lineage arriving at the same design is a signal that the
design is the natural shape of the problem, not an idiosyncrasy of either
implementation. It's also worth being precise about what that
convergence does *not* cover: eCube's own scope is the data layer only —
it names external "consumer applications" (LUCC-style models) as clients,
the same relationship `disscube` has to `disslucc-continuous`/
`disslucc-discrete` (10.6). Nothing in a data-cube specification, eCube
included, plays the role `RasterBackend`/`Environment` play here — the
motion regime is the half of this ecosystem's design that a data-cube
spec, by definition, doesn't build.

**A note on sourcing.** This section reflects positioning discussion
about external systems (BDC, eCube) that don't live in this book's
sibling-repo tree — eCube in particular is an unpublished, in-progress
INPE specification, not something verifiable against
`../../dissmodel/*`. Treat it as context for where `disscube` sits in
the wider landscape, not as a claim about DisSCube's own tested
behavior — everything about `disscube` itself in 10.3–10.7 is grounded
in its source and its own `docs/`.

## Exercises

1. Register a grid and a raster `SpatialSource` following 10.4, then
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
5. Using `to_lucc_data()`'s three open contract decisions (10.5) as a
   checklist, write a short test plan for a pipeline that combines a
   static variable (`dist_roads`) and a temporal one (`forest_pct` with
   a `period` filter) — what would you assert about the resulting
   `RasterBackend` in each case?
6. `SyncRasterModel` (10.5) requires every subclass to declare
   `self.land_use_types` in `setup()` and never write `<name>_past`
   arrays manually. What would go wrong — concretely, in `FloodModel` or
   `MangroveModel` — if a subclass wrote to a `_past` array itself
   instead of leaving it to `pre_execute`/`post_execute`?
7. `SpatialRelation` is persisted in the catalog and deliberately
   excluded from `spec_hash` (10.7) because no pipeline stage reads it
   yet. If an `Exchange` component were added tomorrow that *did* read
   it during a run, would `SpatialRelation` then need to join
   `spec_hash`? Justify your answer using the same reasoning the
   `Derivation.spec_hash()` docstring gives for excluding it today.
8. Using the NDVI-vs-fire-spread contrast in 10.8, classify each of the
   following as "rest" or "motion": (a) computing NDVI for one Sentinel-2
   scene, (b) simulating mangrove migration under sea-level rise for 88
   annual steps, (c) resampling a MapBiomas raster onto a new grid before
   a simulation starts, (d) two coupled models exchanging state every
   tick across different grids via an `Exchange` component.

## Summary

The DisSModel ecosystem's cube story rests on one deliberate split: a
data cube is used for genuinely different things at rest (catalog,
analyze, publish) and in motion (mutate every tick, couple models on a
shared clock), so the ecosystem builds one implementation for each rather
than forcing one substrate to do both. `disscube` is the cube at rest:
`SpatialSource → Derivation → Variable → DerivedVariable` turns raw
raster/vector sources into named, grid-aligned variables cached in Zarr
and cataloged in SQLite/JSON, with `spec_hash` making reproducibility a
property of the architecture rather than of documentation discipline.
`RasterBackend` is the cube in motion: a materialized, mutable array
store that `Environment` (Chapter 2) drives tick by tick, validated
against two genuinely different domains — `disslucc-continuous`'s
regression-driven allocation and `brmangue-dissmodel`'s two-model,
`SyncRasterModel`-coupled coastal automaton. `to_lucc_data()` and
`RasterBackend.from_xarray()` are the two doors between the regimes —
one tied to `disscube`'s own catalog, one open to any STAC-sourced
`xarray.Dataset` — and neither leaks a `disscube`-specific type into
model code (Chapters 2 and 7). The package is explicitly **Alpha**: its
own "Known Limitations" document in-memory/single-tile processing with
no lazy or distributed execution, pixel-counted (not area-weighted)
vector aggregation, silent first-tile disambiguation in `load()`, and a
`SpatialRelation`/`purity_threshold` vocabulary that's modeled but not
yet wired into the pipeline — none of which blocks the core declarative
flow, but all of which matter before trusting a continental-scale
(`BR/1km`) production run.
