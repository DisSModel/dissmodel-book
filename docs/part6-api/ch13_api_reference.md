# Chapter 13: API Reference

*Part VI — API Reference*

Implemented by the [`dissmodel`](https://github.com/DisSModel/dissmodel) core package. This chapter
consolidates the module-level API documentation that previously lived only in `dissmodel`'s own
`docs/api/` — it is grounded directly in that source, not reconstructed from memory, so it can replace it
rather than duplicate it.

## Learning objectives

- Know the four-phase executor lifecycle (`validate → load → run → save`) and wire a new model into it
- Use the CLI (`run` / `validate` / `show`) and read a `model.toml` spec
- Read and build an `ExperimentRecord`, and understand what `DataSource`/`JobRequest` carry
- Validate an executor's contract with `ExecutorTestHarness` before opening a PR
- Choose between the vector and raster substrates, and use `fill()` to populate a grid from real data
- Load/save datasets through `dissmodel.io`, independent of local disk or S3
- Attach `Chart`, `Map`, and `RasterMap` to a running simulation
- Convert a shapefile straight into a `RasterBackend`, and know when that trade-off pays off

---

## 13.1 `dissmodel.core` — clock and lifecycle

Full walkthrough of `Model`/`Environment` and the `setup → pre_execute → execute → post_execute`
lifecycle is in [Chapter 2](../part1-core/ch02_core_concepts.md). Two details from the core API reference
are worth adding here because they aren't covered there:

**Instantiation order is enforced, not just conventional** — a `Model` (or visualization component)
created before its `Environment` fails to register:

```
Environment  →  Model  →  Visualization  →  env.run()
     ↑             ↑            ↑                ↑
  first         second        third           fourth
```

**Each model can define its own `start_time`/`end_time`**, independent of the environment's interval —
useful for composing models that come online or retire at different points in the same run (e.g. a
land-use driver that only becomes active after a policy year):

```python
from dissmodel.core import Model, Environment

class ModelA(Model):
    def execute(self):
        print(f"[A] t={self.env.now()}")

class ModelB(Model):
    def execute(self):
        print(f"[B] t={self.env.now()}")

env = Environment(start_time=2010, end_time=2016)

ModelA(start_time=2012)   # active from 2012 to end
ModelB(end_time=2013)     # active from start to 2013

env.run()
```

Models with no explicit `start_time`/`end_time` inherit the environment's interval. All active models
execute at each tick before the clock advances — this is what makes coupled CA + SysDyn runs inside a
single `env.run()` possible (see [§7.4](../part2-paradigms/ch06_abm.md)).

---

## 13.2 `dissmodel.executor` — the four-phase lifecycle

`dissmodel.executor` separates scientific logic (the `Model`) from execution infrastructure (I/O, CLI,
remote dispatch), so the same model code runs unchanged from a Jupyter cell to a platform worker.

| Component | Role |
|---|---|
| `ModelExecutor` | abstract base class — you implement `load`, `run`, `save` |
| `ExecutorRegistry` | maps a `name` string to an executor class, auto-populated via `__init_subclass__` |
| `execute_lifecycle` | the single orchestration function shared by the CLI and the platform's `job_runner.py` |

```
validate(record)            # stateless pre-flight checks — no I/O
data = load(record)         # resolve URI, apply column/band maps, return data
result = run(data, record)  # simulation — no I/O here
record = save(result, record)
```

Each phase is timed independently and written to `record.metrics` automatically (`time_validate_sec`,
`time_load_sec`, `time_run_sec`, `time_save_sec`, `time_total_sec`).

**Minimal implementation:**

```python
import geopandas as gpd
from dissmodel.executor import ModelExecutor, ExperimentRecord
from dissmodel.executor.cli import run_cli
from dissmodel.io import load_dataset, save_dataset


class MyExecutor(ModelExecutor):
    name = "my_model"

    def load(self, record: ExperimentRecord) -> gpd.GeoDataFrame:
        gdf, checksum = load_dataset(record.source.uri)
        record.source.checksum = checksum
        return gdf

    def run(self, data: gpd.GeoDataFrame, record: ExperimentRecord) -> gpd.GeoDataFrame:
        params = record.parameters
        # ... simulation logic — no I/O here ...
        return data

    def save(self, result, record: ExperimentRecord) -> ExperimentRecord:
        uri = record.output_path or "output.gpkg"
        checksum = save_dataset(result, uri)
        record.output_path   = uri
        record.output_sha256 = checksum
        record.status        = "completed"
        return record


if __name__ == "__main__":
    run_cli(MyExecutor)
```

A `name` class attribute is all that's needed for auto-registration — no boilerplate registration call:

```python
from dissmodel.executor import ExecutorRegistry

ExecutorRegistry.list()          # ["my_model", ...]
ExecutorRegistry.get("my_model") # → MyExecutor class
```

`name` is also the key referenced from the TOML model registry (see [Chapter 9](../part4-infra/ch09_platform.md)):

```toml
[model]
class   = "my_model"
package = "my_package"
```

---

## 13.3 CLI — `run`, `validate`, `show`

Adding a `__main__` block with `run_cli(MyExecutor)` exposes three subcommands.

### `run` — execute the simulation

```bash
python -m my_package.my_executor run \
    --toml   configs/model.toml \
    --input  data/grid.zip \
    --output results/output.tif \
    --param  end_time=50
```

After the run, three files are written next to the output:

| File | Content |
|---|---|
| `output_<id>.tif` | simulation result |
| `output_<id>.record.json` | full `ExperimentRecord` (provenance) |
| `profiling_<id>.md` | per-phase timing table |

The profiling table exposes `Load` as a separate phase from `Run`, so I/O time is never hidden inside
simulation time:

```
| Phase    | Time (s) | % of Total |
|----------|----------|------------|
| Validate | 0.000    | 0.0%       |
| Load     | 2.898    | 49.4%      |
| Run      | 2.972    | 50.6%      |
| Save     | 0.001    | 0.0%       |
| Total    | 5.871    | 100%       |
```

### `validate` — check the executor contract

Runs `ExecutorTestHarness` (§13.5) without loading data; add `--input` to also run a full cycle test.

```bash
python -m my_package.my_executor validate --input data/grid.zip
```

### `show` — inspect resolved parameters

Prints the merged parameters from `model.toml` plus any `--param` overrides, without running anything:

```bash
python -m my_package.my_executor show --toml configs/model.toml
```

### `model.toml` spec

```toml
[model]
class   = "my_model"
package = "my_package"

[model.parameters]
end_time   = 20
resolution = 100.0

[[model.potential]]
const = 0.74
```

CLI `--param` flags override TOML values.

### Output path intelligence

To prevent accidental overwrites, the CLI injects the short experiment ID automatically when `--output`
doesn't already carry one:

```bash
--output results/                 →  results/simulacao_ec17096d.tif
--output results/run.tif          →  results/run_ec17096d.tif
--output results/run_ec17096d.tif →  unchanged
```

---

## 13.4 Schemas — `ExperimentRecord`, `DataSource`, `JobRequest`

These are Pydantic models carrying provenance across the full lifecycle. `ExperimentRecord` is created
before `execute_lifecycle` runs, then mutated in place by `load`, `run`, and `save`.

| Field | Populated by | Description |
|---|---|---|
| `experiment_id` | automatically | UUID generated at creation |
| `created_at` | automatically | ISO 8601 timestamp |
| `source.checksum` | `load()` | SHA-256 of the input dataset |
| `parameters` | CLI / API | resolved model parameters |
| `output_path` | `save()` | URI of the output file |
| `output_sha256` | `save()` | SHA-256 of the output file |
| `metrics` | `execute_lifecycle` | per-phase timing + model-specific metrics |
| `artifacts` | `save()` | named checksums of extra artefacts |
| `status` | `save()` | `"completed"` or `"failed"` |

**Reading a saved record:**

```python
from dissmodel.executor.schemas import ExperimentRecord
from pathlib import Path

record = ExperimentRecord.model_validate_json(
    Path("output.record.json").read_text()
)
print(record.metrics["time_load_sec"])
```

**A complete `record.json`** looks like this — this is the artifact that makes a run reproducible years
later, referenced from [Chapter 11](../part5-reference/ch11_migration.md)'s validation methodology:

```json
{
  "experiment_id": "abc123",
  "model_commit": "a3f9c12",
  "code_version": "0.5.0",
  "resolved_spec": { "...TOML snapshot..." },
  "source": { "uri": "s3://...", "checksum": "e3b0c44..." },
  "artifacts": { "output": "sha256...", "profiling": "sha256..." },
  "metrics": { "time_run_sec": 2.15, "time_total_sec": 2.34 },
  "status": "completed"
}
```

**`DataSource`** describes the input; `checksum` is filled by `load()`, not before:

```python
DataSource(type="s3", uri="s3://bucket/input.tif", checksum="")
```

**`JobRequest`** is how the platform API (Chapter 9) accepts a job — it resolves the executor from
`model.class`, installs `model.package`, and builds the `ExperimentRecord` before dispatch:

```python
from dissmodel.executor.schemas import JobRequest

job = JobRequest(
    model={
        "class": "coastal_raster",
        "package": "git+https://github.com/DisSModel/coastal-dynamics@main",
        "parameters": {"end_time": 88},
    },
    source={"type": "s3", "uri": "s3://dissmodel-inputs/grid.zip"},
)
```

---

## 13.5 Testing — `ExecutorTestHarness`

Validates an executor's contract without needing real data — designed to run in a notebook before
opening a PR, using the same checks CI runs via pytest.

```python
from dissmodel.executor.testing import ExecutorTestHarness
from my_package.my_executor import MyExecutor

harness = ExecutorTestHarness(MyExecutor)
harness.run_contract_tests()
```

```
ExecutorTestHarness — MyExecutor
────────────────────────────────────────────────────
  ✅ name attribute exists
  ✅ load() is implemented
  ✅ run() is implemented
  ✅ save() is implemented
  ✅ run() signature is correct
  ✅ save() signature is correct
────────────────────────────────────────────────────
  All 8 checks passed ✅
```

The signature check catches the most common migration mistake — executors still on the pre-0.4.0
`run(self, record)` form fail with a direct message instead of a cryptic traceback:

```
❌ run() signature is correct: run() must accept exactly two parameters
   (data, record), got ['record']. Did you update to the 0.4.0 signature?
```

**Full lifecycle test** — runs `validate → load → run → save` against a real (or synthetic) record, for
smoke-testing after a migration:

```python
harness.run_with_sample_data()   # synthetic record if none given
```

**In pytest:**

```python
def test_contract():
    assert ExecutorTestHarness(MyExecutor).run_contract_tests() is True
```

---

## 13.6 `dissmodel.geo` — the dual substrate

|  | Vector | Raster |
|---|---|---|
| Module | `dissmodel.geo.vector` | `dissmodel.geo.raster` |
| Data structure | `GeoDataFrame` (GeoPandas) | `RasterBackend` (NumPy 2D arrays) |
| Grid factory | `vector_grid()` | `raster_grid()` |
| Neighbourhood | Queen / Rook (`libpysal`) | Moore / Von Neumann (`shift2d`) |
| Rule pattern | `rule(idx)` per cell | `rule(arrays) → dict`, vectorized |
| Best for | irregular grids, real-world GIS data | large grids, performance-critical models |

Both substrates share the same `Environment` and clock — a vector model and a raster model can run side
by side in one `env.run()`.

### Sync models — TerraME's `past` copy, formalized

`SyncSpatialModel` / `SyncRasterModel` manage `_past` snapshots automatically — this is the same
mechanism referenced in [Chapter 5](../part2-paradigms/ch05_ca.md)'s discussion of `Life.lua`'s
`cell.past.state`, now named explicitly instead of left implicit in `setup()`/`execute()` timing:

```python
from dissmodel.geo.vector.sync_model import SyncSpatialModel

class LUCCModel(SyncSpatialModel):
    def setup(self, gdf):
        self.gdf = gdf
        self.land_use_types = ["f", "d", "outros"]

    def execute(self):
        uso_past = self.gdf["f_past"]   # state at the beginning of this step
        # ... update self.gdf["f"] ...
```

`SyncRasterModel` provides the same semantics for `RasterBackend` arrays. Both expose `synchronize()`
publicly for cases where the automatic pre/post-step timing isn't sufficient.

### Filling grid attributes — `fill()`

`fill()` initialises columns from spatial data sources, replacing manual cell-by-cell loops:

```python
from dissmodel.geo import fill, FillStrategy

# zonal statistics from a raster
fill(FillStrategy.ZONAL_STATS, vectors=gdf, raster_data=raster, affine=affine,
     stats=["mean", "min", "max"], prefix="alt_")

# minimum distance to a feature layer
fill(FillStrategy.MIN_DISTANCE, from_gdf=gdf, to_gdf=rivers, attr_name="dist_river")

# random sampling by class proportion
fill(FillStrategy.RANDOM_SAMPLE, gdf=gdf, attr="land_use", data={0: 0.7, 1: 0.3}, seed=42)

# fixed pattern — useful for deterministic tests
fill(FillStrategy.PATTERN, gdf=gdf, attr="zone", pattern=[[1,0,0],[0,1,0],[0,0,1]])
```

This is the same vocabulary Chapter 10 maps against TerraME's `Layer:fill` strategies — `fill()` here is
the general-purpose entry point; `disscube`'s operators (Chapter 10) build on top of it for multi-source,
multi-resolution ingestion specifically.

Custom strategies register with a decorator:

```python
from dissmodel.geo.vector.fill import register_strategy

@register_strategy("my_strategy")
def fill_my_strategy(gdf, attr, **kwargs):
    ...
```

### Xarray interoperability (raster substrate)

`RasterBackend` converts to/from `xr.Dataset`, connecting to the Pangeo ecosystem (Zarr, Dask,
JupyterHub) — relevant if a model's output needs to feed a larger data-cube pipeline rather than stay a
standalone GeoTIFF:

```python
ds = backend.to_xarray(time=42)
ds.to_zarr("output.zarr")

backend2 = RasterBackend.from_xarray(ds)
```

CRS is preserved as a `spatial_ref` coordinate following the CF-1.8 convention; spatial coordinates come
from `backend.transform` (a rasterio `Affine`) when available.

---

## 13.7 `dissmodel.io` — storage-agnostic load/save

Format detection is automatic (by extension, or an explicit `fmt`); `s3://` URIs resolve transparently
through the configured MinIO/S3 client with no change to model code.

| Extension | `fmt` | Returns |
|---|---|---|
| `.gpkg`, `.shp`, `.geojson`, `.zip` (shapefile) | `"vector"` | `(GeoDataFrame, checksum)` |
| `.tif`, `.tiff`, `.zip` (GeoTIFF) | `"raster"` | `((backend, meta), checksum)` |

```python
from dissmodel.io import load_dataset, save_dataset

gdf, checksum = load_dataset("data/grid.gpkg")
gdf, checksum = load_dataset("s3://bucket/grid.gpkg")

checksum = save_dataset(gdf, "results/output.gpkg")
```

Both functions return a SHA-256 checksum — inside an executor, these are exactly the values that get
assigned to `record.source.checksum` and `record.output_sha256` (§13.4).

**`vector_to_raster_backend`** rasterizes a `GeoDataFrame` when a model needs the raster substrate but the
input is vector:

```python
from dissmodel.io.convert import vector_to_raster_backend

backend = vector_to_raster_backend(
    source=gdf, resolution=100.0,
    attrs={"uso": 0, "alt": 0.0, "solo": 1},
    crs="EPSG:31984", nodata=0,
)
```

The resulting backend includes a `"mask"` band marking valid (non-nodata) cells.

---

## 13.8 `dissmodel.visualization` — `Chart`, `Map`, `RasterMap`

All three components subclass `Model`, so they update automatically at each simulation step, and all
three support three output targets: a local `matplotlib` window, inline Jupyter display, or a Streamlit
`st.empty()` placeholder.

| Component | Substrate | Purpose |
|---|---|---|
| `Chart` | any | time-series plots of `@track_plot`-annotated variables |
| `Map` | vector (`GeoDataFrame`) | dynamic spatial map, redrawn each step |
| `RasterMap` | raster (NumPy) | categorical or continuous raster rendering |

`@track_plot` is the decorator used throughout this book's course chapters (e.g. `SIR`, `Lorenz`,
`Coffee`) — it marks which attributes `Chart` collects and how to label/colour them:

```python
from dissmodel.core import Model
from dissmodel.visualization import track_plot, Chart

@track_plot("Susceptible", "green")
@track_plot("Infected",    "red")
@track_plot("Recovered",   "blue")
class SIR(Model):
    ...
```

**`RasterMap`** is the one component with no prior mention elsewhere in this book — it supports both a
categorical mode (explicit value → colour/label mapping) and a continuous mode (colormap + colorbar,
with optional masking):

```python
from dissmodel.visualization.raster_map import RasterMap

# categorical
RasterMap(backend=b, band="uso", title="Land Use",
          color_map={1: "#006400", 3: "#00008b", 5: "#d2b48c"},
          labels={1: "Mangrove", 3: "Sea", 5: "Bare soil"})

# continuous, with masking
RasterMap(backend=b, band="alt", title="Altimetry", cmap="terrain",
          colorbar_label="Altitude (m)", mask_band="uso", mask_value=3)
```

When no display is available (e.g. a headless CI run or a platform worker), frames are written to
`raster_map_frames/<band>_step_NNN.png` instead of raising an error.

**`display_inputs`** generates Streamlit widgets from a model's type-annotated `setup()` parameters —
integers/floats become sliders, booleans become checkboxes:

```python
from dissmodel.visualization import display_inputs

sir = SIR()
display_inputs(sir, st.sidebar)
```

---

## 13.9 Shapefile → raster: when the substrate switch pays off

`shapefile_to_raster_backend` converts any vector file (Shapefile, GeoJSON, GeoPackage) straight into a
`RasterBackend`, so a raster model can run on real geographic data with no intermediate GIS step:

```
Shapefile → GeoDataFrame → rasterize (rasterio.features.rasterize) → NumPy arrays → RasterBackend
```

```python
from dissmodel.geo.raster.io import shapefile_to_raster_backend

b = shapefile_to_raster_backend(
    path="data/mangue_grid.shp", resolution=100,
    attrs={"uso": 5, "alt": 0.0, "solo": 1}, crs="EPSG:31984",
)
```

!!! note "Grid regularity"
    Most accurate when the input already contains a **regular grid** of equal-area polygons (the typical
    output of spatial homogenization — Chapter 10). For irregular polygons (municipalities, watersheds),
    cells are burned by centroid or by touch (`all_touched`); inspect the result before a long run.

### Where the "4,500× faster" figure comes from

This book's ecosystem-comparison discussions cite raster being dramatically faster than vector at scale.
The concrete source is a same-data comparison, run from the same shapefile through both substrates on a
94,000-cell grid:

```python
# vector — real geometries, Queen neighbourhood
gdf = gpd.read_file("data/mangue_grid.shp").to_crs("EPSG:31984")
# → ~2 min/step @ 94k cells

# raster — same data, rasterized once
b = shapefile_to_raster_backend("data/mangue_grid.shp", resolution=100,
                                 attrs=["uso", "alt", "solo"], crs="EPSG:31984")
# → ~8 ms/step @ 94k cells  (≈ 4,500× faster)
```

The gap is entirely the vectorized NumPy operations (`shift2d`, `focal_sum`) replacing per-geometry
GeoPandas/Shapely operations at every step — the one-time rasterization cost is paid once, not per tick.
This is the same trade-off Chapter 10 frames as "at rest vs. in motion": pay a conversion cost up front to
get vectorized speed for the run itself. The benchmark scripts behind this number
(`benchmark_flood_model.py`, `benchmark_game_of_life.py`) ship in the `dissmodel` repository's
`benchmarks/` directory for anyone who wants to reproduce it on their own hardware.

Saving results back to GeoTIFF closes the loop without leaving the raster substrate:

```python
from dissmodel.geo.raster.io import save_raster_backend

save_raster_backend(backend=b, path="output/flood_result.tif",
                     bands=["uso", "alt"], crs="EPSG:31984", transform=transform)
```
