# Chapter 11: Migrating from TerraME/LUCCME to DisSModel

*Part V — Migration & Reference*

## Learning objectives

- Translate TerraME/LUCCME concepts to their DisSModel equivalents
- Know what numerical equivalence guarantees already exist
- Plan an incremental migration of an existing model

## 14.1 Why migrate

TerraME is C++/Lua, with its latest release (2.0.1) from August 2020 and
installation via OS-specific installers. DisSModel is Python-native
(`pip install`), runs from notebook to distributed cluster with the same
code, and is numerically validated against TerraME reference outputs
where applicable (e.g. `brmangue-dissmodel`).

## 14.2 General equivalence table

| TerraME/LUCCME | DisSModel |
|---|---|
| TerraME (framework) | `dissmodel` |
| LUCCME | `disslucc-continuous` |
| CLUE-S-like (discrete) | `disslucc-discrete` |
| TerraLib | `geopandas` / `rasterio` |
| FillCell | `disscube` / `disslucc_continuous.io` |
| `Agent`/`Society` | `dissmodel-abm` — see Ch. 5 (Agent-Based Modeling) |
| `init(self)` | `setup()` |
| `execute(self)` | `execute()` |

## 14.3 Step-by-step incremental migration

*TODO: write a practical roadmap — (1) identify the original model's
paradigm (CA, ABM, continuous/discrete LUCC, system dynamics), (2) choose
the corresponding satellite package, (3) map input data (TerraLib →
geopandas/rasterio), (4) validate outputs against the original model,
citing existing benchmarks (`benchmark/` in disslucc-continuous and
disslucc-discrete, `main_benchmark.py` in brmangue-dissmodel).*

## 14.4 Existing validation cases in the ecosystem

- `brmangue-dissmodel`: numerical parity raster vs. vector, both against TerraME golden outputs.
- `disslucc-discrete`: Lab6 validation (TerraME reference) — *TODO: detail*.

## Exercises

*TODO: write exercises.*

## Summary

*TODO: summarize the chapter's key takeaways.*
