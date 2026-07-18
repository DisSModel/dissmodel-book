# Chapter 6: Agent-Based Modeling

*Part II — Simulation Paradigms*

Implemented by the [`dissmodel-abm`](https://github.com/DisSModel/dissmodel-abm) package.

## Learning objectives

- Understand the `Society`/`Agent` protective layer over the vector substrate
- Translate `Agent`/`Society` concepts from TerraME to `dissmodel-abm`
- Write an agent-based model without touching `self.gdf` directly
- Know which models ship today, and what's explicitly still missing

## 6.1 `Society`/`Agent`: a protective layer over the substrate

The whole point of this package is that model code never touches
`self.gdf` directly. Today the substrate is a vector `GeoDataFrame`
(raster support is on the roadmap, 6.6); a model written against
`self.society` doesn't know or care about that — it only sees agents as
objects:

```python
for agent in self.society:
    if agent.energy <= 0:
        agent.die()
    elif agent.energy >= 15:
        agent.reproduce(energy=5.0)
```

instead of:

```python
self.gdf = self.gdf[self.gdf["energy"] > 0].reset_index(drop=True)
mask = self.gdf["energy"] >= 15
children = self.gdf.loc[mask].copy()
children["energy"] = 5.0
self.gdf = gpd.GeoDataFrame(pd.concat([self.gdf, children]), crs=self.gdf.crs)
```

`Society` owns no data of its own — it reads and writes through the host
model's `gdf` attribute, so `model.gdf` and `model.society` are always
the same data. `Agent` is a thin proxy over one row: `agent.energy = 5`
writes the underlying cell directly, with no copy to keep in sync.
`Map`, `Chart`, `ModelExecutor`, and any code that still expects
`model.gdf` keep working unmodified, whether or not a given model uses
`self.society` internally — `AgentModel` is a `SpatialModel` subclass
(Chapter 2) with a lazily-created `society` property, not a parallel
class hierarchy.

**Agents without a location.** Following TerraME — where an `Agent` may
exist with no `placement` until `Agent:enter()` is called — an agent
here can exist with `geometry = None`:

```python
orphan = self.society.add(energy=4.0)   # no position yet
orphan.has_location                     # False
orphan.enter(10, 10)                    # give it a position
orphan.leave()                          # remove the position, keep the agent
```

Calling a spatial method (`walk`, `neighbors`, `distance_to`) on a
location-less agent raises a clear `RuntimeError` rather than failing on
`None` deep inside geopandas.

**A structural difference worth naming explicitly** (from the package's
own `docs/agent.md`): TerraME agents are autonomous objects that carry
their own behavior (`Agent{execute = function(self) ... end}`).
`dissmodel-abm` agents are data with a uniform interface — behavior
lives once, in the owning model's `execute()`, applied identically to
every agent in the loop via `for agent in self.society`. This trades
TerraME's per-agent heterogeneous behavior for staying close to a
vectorizable substrate (a `GeoDataFrame`), the same trade-off Chapter 2
already described for `CellularAutomaton.rule(idx)`.

## 6.2 Concept mapping: TerraME → dissmodel-abm

| TerraME (`Agent`/`Society`) | `dissmodel-abm` |
|---|---|
| `execute(self)` | `execute()` (`Model` lifecycle, Chapter 2) |
| `init(self)` | `setup()` (`Model` lifecycle, Chapter 2) |
| `Society` (collection of Agents) | `self.society` — object-oriented view over `self.gdf` |
| `Agent` | `self.society[idx]` — a proxy over one row |
| `placement` / `getCell()` | `agent.geometry` |
| `enter(cell)` | `agent.enter(x, y)` — give a location-less agent a position |
| `leave()` | `agent.leave()` — remove an agent's position without removing it from its society |
| `move(cell)` / `walk()` | `agent.move_to(x, y)` / `agent.walk(step_size, bounds)` |
| `die()` | `agent.die()` |
| `reproduce()` | `agent.reproduce(**overrides)` |
| `emptyNeighbor` / neighborhood | `agent.neighbors(radius)` (points) or `agent.grid_neighbors()` (cells) |
| `Society:add` / `Society:remove` | `society.add(**attrs)` / `society.remove(agent_or_idx)` |
| `Society:sample` | `society.sample(n)` |
| `forEachAgent` | `for agent in society: ...` |
| `addSocialNetwork` / `message` | not provided yet (6.6) |
| `State` / `Jump` / `Flow` | not provided yet (6.6) |

Point-agent models (`RandomWalkModel`, `PredatorPreyModel`) back
`self.society` with Point geometry + state columns. One-agent-per-cell
models (`SchellingModel`) back it with a polygon grid
(`dissmodel.geo.vector.vector_grid`), matching the design of TerraME's
`logo` package — "implements spatial agent-based models with at most one
agent per cell."

A small set of legacy batch/vectorized methods (`walk`, `die_if`,
`reproduce_if`, `neighbors_within`) is still available directly on
`AgentModel` for backward compatibility and for cases where vectorized
performance matters more than per-agent readability. New models should
prefer `self.society`.

## 6.3 Included models

| Model | Substrate | Live-plotted attributes | Description |
|---|---|---|---|
| `RandomWalkModel` | Vector (points) | — | Independent random walk within a bounding box |
| `PredatorPreyModel` | Vector (points) | `sheep`, `wolves` | Wolf-sheep dynamics: movement, predation, death, reproduction, optional grazing |
| `SchellingModel` | Vector (grid cells) | `satisfaction` | Segregation model, ported from [TerraME's `logo` package](https://www.terrame.org/package/logo/models/#schelling): agents move to empty cells when unhappy with their same-type neighbor count |

`PredatorPreyModel` (`src/dissmodel_abm/models/predator_prey.py`) is
written entirely against `self.society` and demonstrates the full set of
TerraME-inspired operations in one model: `agent.walk()`,
`agent.neighbors()`, `agent.die()`, `agent.reproduce()`. Its rule set:

1. every agent takes one random walk step;
2. every agent loses `energy_loss` energy (sheep graze first if
   `graze_gain` is set);
3. wolves within `eat_radius` of a sheep eat it — the sheep dies, the
   wolf gains `energy_gain`;
4. agents with `energy <= 0` die;
5. agents with `energy >= reproduce_threshold` reproduce, the child
   starting at `reproduce_threshold / 2` energy.

`SchellingModel`'s parameters match TerraME's own defaults (`dim=25`,
`freeSpace=25%`, `preference=3`):

```python
from dissmodel.core import Environment
from dissmodel.geo.vector import vector_grid
from dissmodel_abm.models import SchellingModel

gdf = vector_grid(dimension=(25, 25), resolution=1)
env = Environment(start_time=0, end_time=30)

model = SchellingModel(gdf=gdf, free_space=0.25, preference=3, seed=0)
env.run()

print(model.fraction_satisfied())  # -> 1.0 once converged
```

Internally, every agent is reached through `self.society` —
`agent.agent_type`, `agent.grid_neighbors()` — never raw `gdf` rows.
`agent_type`: `-1` = empty, `0`/`1` = the two agent groups.
`model.fraction_satisfied()` returns the fraction of agents with at
least `preference` same-type Queen-neighbors — the model still builds an
internal `{id: type}` snapshot for the per-step happiness evaluation
(a local performance optimization, the TerraME-equivalent of evaluating
all agents against the same "tick" state before committing moves), which
is not something calling code needs to know about.

`PredatorPreyModel` and `SchellingModel` both use `@track_plot` (Chapter
2/4) to expose live-plotted attributes — `sheep`/`wolves` and
`satisfaction` respectively — with no separate tracker model needed, the
same convention `dissmodel-sysdyn`'s `SIR` uses (Chapter 4).

## 6.4 Installation and quick start

Like every other extension package (Chapter 3), `dissmodel-abm` has no
PyPI release — install from source:

```bash
git clone https://github.com/DisSModel/dissmodel-abm.git
cd dissmodel-abm
pip install -e .
```

This requires `dissmodel>=0.6.0` (which pulls in `geopandas`, `shapely`,
`numpy`, `pandas`, `libpysal` transitively).

```python
import geopandas as gpd
import numpy as np
from dissmodel.core import Environment
from dissmodel_abm.models import RandomWalkModel

n = 20
bounds = (0, 0, 100, 100)
rng = np.random.default_rng(42)
gdf = gpd.GeoDataFrame({
    "geometry": gpd.points_from_xy(
        rng.uniform(bounds[0], bounds[2], n),
        rng.uniform(bounds[1], bounds[3], n),
    )
})

env = Environment(start_time=0, end_time=20)
model = RandomWalkModel(gdf=gdf, step_size=2.0, bounds=bounds)
env.run()
```

Writing a new model looks exactly like writing a `dissmodel-ca` model
(Chapter 5), swapping `CellularAutomaton.rule(idx)` for a `society` loop:

```python
from dissmodel_abm.core import AgentModel

class MyModel(AgentModel):
    def setup(self, **params):
        ...  # one-time initialization

    def execute(self):
        for agent in self.society:
            agent.walk(step_size=1.0, bounds=(0, 0, 100, 100))
            if agent.energy <= 0:
                agent.die()
            elif agent.energy >= 15:
                agent.reproduce(energy=5.0)
```

Run the shipped examples directly, or explore interactively — the same
CLI/Streamlit split from Chapter 3, 3.3:

```bash
python examples/cli/abm_random_walk.py
python examples/cli/abm_predator_prey.py
python examples/cli/abm_schelling.py   # also writes PNG frames to ./map_frames/

pip install -e ".[viz]"
streamlit run examples/streamlit/abm_predator_prey.py   # Map + population Chart
streamlit run examples/streamlit/abm_schelling.py        # Map + satisfaction Chart
```

## 6.5 Repository structure

```
dissmodel-abm/
├── src/dissmodel_abm/
│   ├── core/
│   │   ├── agent_model.py   # AgentModel(SpatialModel) — no core changes
│   │   └── society.py       # Society / Agent — the protective layer
│   └── models/
│       ├── random_walk.py     # minimal example (point agents)
│       ├── predator_prey.py   # wolf-sheep, written against self.society
│       └── schelling.py       # segregation model, one agent per cell
├── examples/{cli,streamlit}/  # same convention as Chapters 4-5
├── tests/test_agent_model.py
└── docs/agent.md              # full Agent reference, mirroring TerraME's own docs
```

`AgentModel` is a `SpatialModel` subclass — no changes to `dissmodel`'s
core are required, the same "minimal core, additive extensions"
principle every satellite package follows (Chapter 12, 12.1). All other
`SpatialModel`/`Model` functionality — `self.env`, `create_neighborhood`,
`pre_execute`/`post_execute`, plot tracking, the `ModelExecutor`/
`ExperimentRecord` pipeline — is inherited unchanged.

## 6.6 What's not there yet

Stated directly in the package's own roadmap, not implied:

- **Raster substrate** — a `Society` backed by a NumPy/raster array
  instead of a `GeoDataFrame`, exposing the same `Agent`/`Society`
  interface, so model code written against `self.society` keeps working
  unchanged regardless of substrate. Vector support is being hardened
  first, deliberately.
- **Social networks** — TerraME's `addSocialNetwork`/`message`/
  `on_message`, likely as a thin layer over a graph library (e.g.
  `networkx`) keyed by `agent.id`.
- **State machines** — TerraME's `State`/`Jump`/`Flow`, for agents whose
  behavior depends on a discrete internal state.

`docs/agent.md` keeps an explicit list of TerraME `Agent` functions with
no `dissmodel-abm` equivalent yet, function by function — worth checking
before assuming a TerraME model can be migrated (Chapter 11) without any
gaps.

## Exercises

1. Open `src/dissmodel_abm/core/society.py` and `agent_model.py`. Explain
   why `Agent` needs no `__init__`-time snapshot of its data — what
   makes `agent.energy = 5` immediately visible in `model.gdf`?
2. Compare `PredatorPreyModel`'s use of `self.society` to
   `AgentModel`'s legacy `die_if`/`reproduce_if` methods. Both can
   remove and add agents — what does `die_if` do to the GeoDataFrame
   index that `agent.die()` via `Society.remove_if` avoids, and why
   would that matter for code holding onto agent references across a
   step?
3. `SchellingModel` is a one-agent-per-cell model built on
   `vector_grid` + `create_neighborhood`, while `RandomWalkModel`/
   `PredatorPreyModel` are point-agent models with no grid at all. Using
   6.2's mapping table, explain why `agent.grid_neighbors()` only makes
   sense for the former and `agent.neighbors(radius)` only for the
   latter.
4. Run `abm_predator_prey.py` and then `abm_schelling.py` via Streamlit.
   Both expose a live `Chart` via `@track_plot` with zero extra wiring
   (6.3) — identify the tracked attribute in each model's source and
   confirm it's a plain instance attribute, not a special decorator
   argument.
5. Pick one item from 6.6 (raster substrate, social networks, or state
   machines) and describe, from what you know of `Society`'s
   substrate-agnostic API design, why the *interface* (`add`, `remove`,
   `walk`, `neighbors`, ...) wouldn't need to change even once that
   feature is implemented.

## Summary

`dissmodel-abm` closes the paradigm gap the rest of this book's Chapters
4 (System Dynamics) and 5 (Cellular Automata) leave open: agents that
move, compete for cells, and are born/die at runtime, none of which fits
`CellularAutomaton.rule(idx)`'s pull contract or a plain `execute()`
push contract cleanly. `AgentModel` is a thin `SpatialModel` subclass
(Chapter 2) — no core changes — and `Society`/`Agent` is a protective,
substrate-agnostic layer over `self.gdf`: agents are read and written as
objects (`agent.energy`, `agent.die()`, `agent.reproduce()`), never as
raw DataFrame masks and `pd.concat` calls, while `Map`, `Chart`, and
`ModelExecutor` keep working unmodified underneath. The three shipped
models — `RandomWalkModel`, `PredatorPreyModel`, `SchellingModel` — cover
both point-agent and one-agent-per-cell styles, with `SchellingModel`
directly ported from TerraME's own `logo` package as a validated
reference point. The package is explicit about what it doesn't do yet —
no raster substrate, no social networks, no state machines — which
matters for Chapter 11: not every TerraME `Agent` model can be migrated
today without a gap, and `docs/agent.md`'s function-by-function
"not implemented" list is the place to check before assuming otherwise.
