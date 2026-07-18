# Chapter 4: Cellular Automata

*Part II — Simulation Paradigms*

Implemented by the [`dissmodel-ca`](https://github.com/DisSModel/dissmodel-ca) package.

## Learning objectives

- Install and use the `dissmodel-ca` extension
- Know the classic and research models included
- Choose between vector and raster substrate for a CA model

## 4.1 Overview

`dissmodel-ca` provides a collection of cellular automata models
implemented on top of the `dissmodel` engine, in both vector
(GeoDataFrame) and raster (NumPy) versions.

## 4.2 Included models

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

## 4.3 Installation and quick start

```bash
pip install .
python examples/cli/ca_game_of_life.py
streamlit run examples/streamlit/ca_all.py
jupyter lab examples/notebooks/ca_game_of_life.ipynb
```

## 4.4 Repository structure

- `src/dissmodel_ca/models/` — core implementations (the "Science" layer)
- `examples/notebooks/` — 15+ didactic notebooks (in Portuguese)
- `examples/cli/` — self-contained scripts for quick testing
- `examples/streamlit/` — reactive UI components

## Exercises

*TODO: write exercises.*

## Summary

*TODO: summarize the chapter's key takeaways.*
