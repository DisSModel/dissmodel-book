# Chapter 2: Core concepts

*Part I — DisSModel Core*

## Learning objectives

- Understand the `SpatialModel` / `ModelExecutor` pattern
- Grasp the dual-substrate architecture (vector and raster)
- Know when to use each substrate

## 2.1 Dual-substrate architecture

The DisSModel core operates over two interchangeable substrates:

- **Vector** (`GeoDataFrame`) — geometric precision, real polygon geometry.
- **Raster** (`NumPy`/`RasterBackend`) — vectorized performance, ideal for regular grids.

Both substrates expose the same model logic — a subclass only differs in
which base class it extends and which shared object it reads/writes
(`self.gdf` vs. `self.backend`):

```python
# vector substrate
class MangroveVectorModel(SpatialModel):
    def execute(self):
        flooded = self.gdf["elevation"] < self.env.now() * self.slr_rate
        self.gdf.loc[flooded, "state"] = "flooded"

# raster substrate
class MangroveRasterModel(RasterModel):
    def execute(self):
        elevation = self.backend.get("elevation")
        flooded = elevation < self.env.now() * self.slr_rate
        self.backend.arrays["state"][flooded] = 1
```

`brmangue-dissmodel` demonstrates this in production: its raster and
vector mangrove/flood models implement "identical equations, thresholds,
parameter names, and update ordering" (README), and the project ships a
dedicated `brmangue_benchmark` executor whose sole job is to run both
substrates on the same input and report match %, MAE, and RMSE per band.
The raster substrate is additionally validated against TerraME's own
golden-output CSVs (`tests/fixtures/golden/step_NN.csv`) via a
`validation` executor — so the numerical-parity claim is not just
vector-vs-raster internal consistency, but raster-vs-TerraME-reference
correctness.

## 2.2 `SpatialModel` and the model lifecycle

Every model in DisSModel — cellular automaton, agent-free flood model, or
system-dynamics compartment — descends from `dissmodel.core.Model`, which
provides a four-hook lifecycle:

| Hook | Called | Purpose |
|---|---|---|
| `setup(**kwargs)` | once, right after `__init__` | one-time setup — build a neighborhood, initialize state |
| `pre_execute()` | once per tick, before `execute()` | snapshot state before the transition rule runs |
| `execute()` | once per tick | the model's actual transition rule (the only required override) |
| `post_execute()` | once per tick, after `execute()` | cleanup or snapshotting after the transition rule |

A `Model` auto-registers with the currently active `Environment` at
construction time — there is no explicit `env.add(model)` call, matching
TerraME's `Timer`/`Event` scheduling but without a separate registration
step. The `Environment.run()` loop then drives every registered model
tick-by-tick: on each simulation time `t`, every model whose
`_next_time <= t <= model.end_time` gets `pre_execute() → execute() →
post_execute()` called, and the clock jumps to the next pending event
(`dissmodel/core/environment.py`). `end_time` is inclusive on both the
environment and the model, explicitly matching TerraME's
`Timer:run(finalTime)` semantics (see the `Environment` docstring).

`SpatialModel` (`dissmodel/geo/vector/spatial_model.py`) extends `Model`
with spatial infrastructure — a shared `self.gdf` GeoDataFrame plus
neighborhood construction — without imposing a transition-rule contract:

```python
class SpatialModel(Model):
    def __init__(self, gdf, step=1, start_time=0, end_time=math.inf,
                 name="", **kwargs):
        self.gdf = gdf
        super().__init__(step=step, start_time=start_time,
                          end_time=end_time, name=name, **kwargs)

    def create_neighborhood(self, strategy=Queen, neighbors_dict=None, **kwargs):
        ...  # populates gdf["_neighs"] via libpysal (Queen, Rook, KNN, custom)
```

Two usage patterns sit on top of `SpatialModel`:

- **`CellularAutomaton`** — a *pull* model: each cell computes its own new
  state independently via a `rule(idx)` contract.
- **Free `execute()` models** (e.g. BR-MANGUE's Hydro model) — a *push*
  model: the transition modifies neighbors starting from a source cell,
  which doesn't fit the `rule(idx)` contract and instead overrides
  `execute()` directly.

This is the conceptual analogue of TerraME's `init(self)` / `execute(self)`
pair on a `Cell`/`CellularSpace`: `setup()` plays the role of one-time
`init`, and `execute()` plays the role of the per-tick transition rule —
except that in DisSModel both are plain Python methods, not Lua closures
registered on an `Event`.

## 2.3 `ModelExecutor`

`Model`/`SpatialModel`/`RasterModel` only know "math, geometry and time"
(paper.md) — they never touch a file path or an S3 URI. That separation
is enforced by `dissmodel.executor.ModelExecutor`, an abstract base class
every satellite package (`dissmodel-ca`, `disslucc-discrete`,
`disslucc-continuous`, `brmangue-dissmodel`) subclasses to expose its
models as CLI-runnable, reproducible experiments:

```python
class ModelExecutor(ABC):
    name: ClassVar[str]          # registry key — plain string

    def __init_subclass__(cls, **kwargs):
        super().__init_subclass__(**kwargs)
        ExecutorRegistry.register(cls)   # auto-registration, no boilerplate

    def validate(self, record: ExperimentRecord) -> None: ...  # optional, stateless
    @abstractmethod
    def load(self, record: ExperimentRecord) -> Any: ...       # I/O in
    @abstractmethod
    def run(self, data: Any, record: ExperimentRecord) -> Any: ...  # no I/O
    @abstractmethod
    def save(self, result: Any, record: ExperimentRecord) -> ExperimentRecord: ...  # I/O out
```

The platform always drives the same four-phase order:
`validate(record) → load(record) → run(data, record) → save(result, record)`.
Every subclass just needs a `name` class attribute (the registry key) and
the three abstract methods — registration happens automatically via
`__init_subclass__`, so nothing needs to `import` and manually add each
executor to a list.

`brmangue-dissmodel` illustrates the pattern with four executors
registered side by side under distinct names — `brmangue_raster`,
`brmangue_vector`, `brmangue_benchmark`, and `validation` — each wrapping
the same underlying models with a different I/O/reporting shape (see
2.1). `disslucc-continuous`/`disslucc-discrete` and `dissmodel-ca` follow
the identical contract for their own models.

**Connection to `dissmodel-configs`.** On the platform, an executor is
never referenced by Python import path directly — it's resolved from a
TOML config that names the package, the executor module, and the class:

```toml
# dissmodel-configs/models/brmangue_raster.toml
[model]
executor_module = "brmangue.executors"
name            = "brmangue_raster"
class           = "brmangue_raster"
package         = "git+https://github.com/DisSModel/brmangue-dissmodel@refactoring"

[model.parameters]
end_time      = 88
taxa_elevacao = 0.5
altura_mare   = 6.0
```

The `[model.parameters]` table becomes the `ExperimentRecord`'s
`resolved_spec`, merged with any CLI `--param` overrides — the same
mechanism used locally via `run_cli(MyExecutor)`. This is what lets the
platform (Chapter 9) run an executor from a config file with no code
change: the config is the only thing that differs between a local `python
my_executor.py run --toml model.toml` invocation and a platform job.

## 2.4 Core package structure

```
dissmodel/
  core/
  executor/
  geo/
    raster/
    vector/
  io/
  visualization/
```

## Summary

DisSModel's core rests on three layered ideas. First, every model —
regardless of substrate or domain — shares the same `setup → pre_execute →
execute → post_execute` lifecycle driven by `Environment.run()`, an
explicit, inclusive-`end_time` scheduler matching TerraME's
`Timer:run(finalTime)` semantics. Second, `SpatialModel` and `RasterModel`
add spatial infrastructure (`gdf`/`backend`, neighborhoods) on top of that
lifecycle without imposing a transition-rule contract, so both a *pull*
`CellularAutomaton.rule(idx)` model and a *push*, source-driven model like
BR-MANGUE's Hydro can share the same base class. Third, `ModelExecutor`
draws a hard line between science (models, which "only know math,
geometry and time") and infrastructure (I/O, CLI, provenance) — a
contract every satellite package implements identically, and that
`dissmodel-configs` lets the platform drive purely through TOML, with no
code change between a local run and a platform job.
