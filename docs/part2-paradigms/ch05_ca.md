# Chapter 5: Cellular Automata

*Part II — Simulation Paradigms*

Implemented by the [`dissmodel-ca`](https://github.com/DisSModel/dissmodel-ca) package.

> **Live demo:** try the models in this chapter directly in the browser —
> [dissmodel-ca-demo on Hugging Face Spaces](https://huggingface.co/spaces/profsergiocosta/dissmodel-ca-demo).

## Learning objectives

- Install and use the `dissmodel-ca` extension
- Know the classic and research models included
- Choose between vector and raster substrate for a CA model

## 5.1 Overview

`dissmodel-ca` provides a collection of cellular automata models
implemented on top of the `dissmodel` engine, in both vector
(GeoDataFrame) and raster (NumPy) versions.

## 5.2 Included models

| Model | Substrate | Description |
|---|---|---|
| `GameOfLife` | Vector / Raster | Classic Conway's simulation |
| `FireModel` | Vector / Raster | Forest fire spread with probabilistic regrowth |
| `Snow` | Vector | Snowfall accumulation and gravity dynamics |
| `Growth` | Vector | Stochastic radial growth |
| `Anneal` | Vector | Binary system relaxation via majority-vote rule |
| `Excitable` | Vector | Excitable medium waves (spiral/ring patterns) |
| `Parasit` | Vector | Host-parasite spatial dynamics |
| `Interspecific` | Vector | Grass species competition model |

## 5.3 Installation and quick start

```bash
pip install .
python examples/cli/ca_game_of_life.py
streamlit run examples/streamlit/ca_all.py
jupyter lab examples/notebooks/ca_game_of_life.ipynb
```

## 5.4 Repository structure

- `src/dissmodel_ca/models/` — core implementations (the "Science" layer)
- `examples/notebooks/` — 15+ didactic notebooks (in Portuguese)
- `examples/cli/` — self-contained scripts for quick testing
- `examples/streamlit/` — reactive UI components

## 5.5 Theory: what is a cellular automaton

System Dynamics (Chapter 4) treats a whole stock as one homogeneous block — no spatial heterogeneity, no
explicit space at all. Cellular automata exist to fix exactly that: they add a grid of autonomous cells,
each changing state based only on its own state and its neighbors', with no central controller.

Cellular automata were proposed by John von Neumann — the same von Neumann of computer architecture — who
was looking for a model of logical systems capable of **self-replication**. Every cellular automaton is
defined by six elements: a grid of cells, a neighborhood, a finite set of discrete states, a finite set of
transition rules, an initial state, and discrete time.

> "Cellular automata contain enough complexity to simulate surprising and novel change as reflected in
> emergent phenomena" — Mike Batty

Two neighborhood shapes recur throughout this chapter: **Von Neumann** (the 4 orthogonal neighbors) and
**Moore** (all 8 surrounding neighbors, including diagonals) — in `dissmodel-ca` these map directly to
`Rook` and `Queen` from `libpysal` (see [§13.6](../part6-api/ch13_api_reference.md#136-dissmodelgeo-the-dual-substrate)).

### Synchronization: TerraME's `past` copy, made implicit

TerraME kept **two copies** of a `CellularSpace` — one holding past values, one holding present values —
and every rule had to read the past copy while writing the present one, or cells updated in different
order within the same tick would see an inconsistent mix of old and new state. `dissmodel-ca`'s
`rule(idx)` contract (§5.4 exercises) achieves the same consistency implicitly: `execute()` reads the
state consolidated at the *end* of the previous step, for every cell, before writing any new one — no
explicit `past` attribute is needed. `SyncSpatialModel`/`SyncRasterModel`
([§13.6](../part6-api/ch13_api_reference.md#136-dissmodelgeo-the-dual-substrate)) formalize this same
mechanism for models that need to name the past state explicitly, as TerraME did.

### `GameOfLife` — TerraME vs DisSModel

`ca/lua/Life.lua` reads `cell.past.state` and creates the neighborhood with `wrap = true` (toroidal
space — edges connect):

```lua
execute = function(cell)
    local alive = countNeighbors(cell, "alive")
    if alive < 2 then cell.state = "dead"
    elseif alive > 3 then cell.state = "dead"
    elseif alive == 3 and cell.past.state == "dead" then cell.state = "alive"
    end
end
model.cs:createNeighborhood{wrap = true}
```

`dissmodel_ca.models.game_of_life.GameOfLife`:

```python
class GameOfLife(CellularAutomaton):
    def setup(self) -> None:
        self.create_neighborhood(strategy=Queen, use_index=True)

    def rule(self, idx: Any) -> int:
        state = self.gdf.loc[idx, self.state_attr]
        live_neighbors = self.neighbor_values(idx, self.state_attr).sum()
        if state == 1:
            return 1 if 2 <= live_neighbors <= 3 else 0
        return 1 if live_neighbors == 3 else 0
```

Conway's rule is identical in both. **Known gap**: `Life.lua` wraps edges (`wrap = true`); the current
`GameOfLife` does not enable wrap by default, so behaviour differs at the grid boundary (not the
interior) — worth checking before porting a TerraME model that depends on toroidal edges.

## 5.6 Case study: Fire in the Forest

The classic first full application of the six CA elements above: a `forest`/`burning`/`burned` (or
`empty`/`forest`/`burning`/`burned`) state space, a Von Neumann neighborhood, and one rule — a `forest`
cell catches fire if any neighbor is `burning`; a `burning` cell becomes `burned` after one step. With a
random `forest`/`empty` initial mix, the model demonstrates **percolation**: below a density threshold
the fire self-extinguishes; above it, it consumes nearly the whole connected component.

### `FireModel` — TerraME vs DisSModel

`ca/lua/Fire.lua` uses the `"vonneumann"` neighborhood string and reads/writes past/present explicitly:

```lua
execute = function(cell)
    if cell.past.state == "burning" then
        cell.state = "burned"
    elseif cell.past.state == "forest" then
        if countNeighbors(cell, "burning") > 0 then cell.state = "burning" end
    end
end
model.cs:createNeighborhood{strategy = "vonneumann"}
```

`dissmodel_ca.models.fire_model.FireModel` uses `Rook` (`libpysal`'s Von Neumann equivalent) and an
`IntEnum` instead of strings:

```python
class FireModel(CellularAutomaton):
    def setup(self, initial_fire_density: float = 0.05, seed: int = 42) -> None:
        self.create_neighborhood(strategy=Rook, use_index=True)

    def rule(self, idx: Any) -> int:
        state = self.gdf.loc[idx, self.state_attr]
        if state == FireState.BURNING:
            return FireState.BURNED
        if state == FireState.FOREST:
            if (self.neighbor_values(idx, self.state_attr) == FireState.BURNING).any():
                return FireState.BURNING
        return state
```

The propagation rule matches step by step; `Rook` *is* Von Neumann, just a different library. The
original course exercise also asked for a `fire-average.lua` variant where ignition depends on the
*average* neighbor state rather than a binary "any burning" check — that variant maps onto the `mean`
operator documented for `disscube` ([§10.5](../part4-infra/ch10_disscube.md)), which can aggregate
neighbor state without writing custom percolation logic.



1. `CellularAutomaton.rule(idx)` is an abstract method — every model in
   5.2 must implement it. Open `src/dissmodel_ca/models/fire_model.py`
   and identify how `FireModel.rule(idx)` decides a cell's next
   `FireState` (`FOREST`/`BURNING`/`BURNED`) based on its Rook neighbors.
   Why does the model use `Rook` instead of the default `Queen`
   neighborhood used elsewhere in the package?
2. `CellularAutomaton.execute()` applies `rule` to every cell via
   `self.gdf.index.map(self.rule)`, which cannot be vectorized because
   `rule` is arbitrary Python. Compare the vector `FireModel` to
   `fire_model_raster.py` and describe, in your own words, what
   vectorization technique the raster version uses instead of per-cell
   `rule()` calls.
3. Every `CellularAutomaton` subclass overrides `initialize()` to set the
   starting state instead of doing it in `__init__`. Explain why — what
   does `initialize()` have access to that `__init__` doesn't, given the
   `Model.__init__` → `setup(**kwargs)` lifecycle from Chapter 2?
4. Pick one model from 5.2 you haven't read yet (e.g. `Anneal`,
   `Excitable`, or `Interspecific`) and, from its docstring and `rule()`
   body alone, write a one-paragraph description of the phenomenon it
   models and which neighborhood strategy it relies on.

## Summary

Every model in `dissmodel-ca` is a `CellularAutomaton` — a `SpatialModel`
that trades a free-form `execute()` for a stricter, cell-by-cell
`rule(idx)` contract: `initialize()` sets the starting state, and
`execute()` applies `rule` to every cell each tick via
`gdf.index.map(self.rule)`. That contract is what makes the package's
Streamlit explorer (Chapter 3) possible — auto-discovering every concrete
`CellularAutomaton` subclass works precisely because they all share the
same shape. Classic models (`GameOfLife`, `FireModel`) and research models
(`Snow`, `Growth`, `Anneal`, `Excitable`, `Parasit`, `Interspecific`) plug
into this same base class, differing only in their state space and
neighborhood strategy (`Queen` by default, `Rook` where fire/diffusion
logic calls for 4-directional spread). Vector and raster substrates
implement the same rule with different mechanics — per-cell `rule()`
lookups on `GeoDataFrame` versus vectorized NumPy operations on
`RasterBackend` — which is why `GameOfLife` and `FireModel` each ship both
a vector and a raster class.
