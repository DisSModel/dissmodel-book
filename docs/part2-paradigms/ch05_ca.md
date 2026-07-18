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

## Exercises

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
