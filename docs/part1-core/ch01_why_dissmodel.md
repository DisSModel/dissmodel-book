# Chapter 1: Why DisSModel — trajectory and motivation

*Part I — DisSModel Core*

## Learning objectives

- Understand the research trajectory that led to DisSModel
- Situate DisSModel relative to TerraME/LUCCME (INPE/CCST)
- Recognize the three principles that guide the ecosystem

## 1.1 A research agenda, not an isolated project

DisSModel did not emerge from a blank slate. It is the current expression
of an agenda that began in 2001 with an undergraduate thesis on
geographic data interoperability using XML and open standards:

| Period | Project | Contribution to the agenda |
|---|---|---|
| 2001–2002 | Terra Translator (XML, ontologies) | Geographic data needs semantics and open standards |
| 2005 | TerraHS (Haskell + GIS) | Scientific models as verifiable, executable artifacts |
| 2007–2010 | TerraME / LuccME (INPE) | Spatially explicit dynamic models as scientific objects |
| 2015–2024 | DbCells, Linked Data, QGIS plugins | Reproducibility demands rich metadata and federated access |
| 2024–2026 | **DisSModel** (Python, FAIR, cloud-native) | Synthesis: the same code runs from Jupyter to a distributed cluster |

## 1.2 Why leave TerraME

TerraME is conceptually robust — it introduced `CellularSpace`, spatially
explicit dynamic modeling, and a discrete-event simulation engine that
DisSLUCC and DisSModel still trace their lineage to. But two structural
costs motivated the move away from it, as stated in DisSModel's own JOSS
paper (`dissmodel/paper.md`):

- **Language barrier.** TerraME models are written in Lua, a language with
  substantially smaller adoption in the data science ecosystem than
  Python. Every new contributor has to learn a DSL that exists nowhere
  else in their toolchain, instead of reusing skills they already have
  from `pandas`/`numpy`.
- **Maintenance status.** The framework "has seen no new release since
  August 2020" (per the paper's Statement of Need) — general-purpose
  simulation libraries built afterward gained no native synchronization
  between a time-stepped clock and the geographic state of a
  `GeoDataFrame`, a gap TerraME's ecosystem never closed.

DisSModel's own **State of the Field** comparison (`paper.md`) makes the
trade-off concrete:

| Aspect | TerraME | Dinamica EGO | DisSModel |
|---|---|---|---|
| Language | Lua | Visual/Internal | Python |
| Simulation engine | Discrete event | Cellular automata | Time-stepped scheduler |
| Spatial structure | `CellularSpace` (fixed) | Cellular grid | `GeoDataFrame` + NumPy (dual) |
| GIS integration | TerraLib | Native raster | GeoPandas / Rasterio |
| Extensibility | Script-based | Block-based | Class inheritance |
| Reproducibility | Manual | Manual | Automated (`ExperimentRecord`) |
| Neighborhoods | GPM support | Limited | libpysal weights (Queen, Rook, KNN, custom) |

In short: TerraME's DSL and TerraLib binding traded portability and
community size for a bespoke, no-longer-maintained stack, while the
already-mature Python scientific ecosystem (`geopandas`, `rasterio`,
`xarray`) offered the same spatial capabilities with an actively
maintained dependency chain and, critically, a reproducibility layer
(`ExperimentRecord`) that TerraME never provided out of the box.

## 1.3 The three principles

1. **Openness as method** — open source and open data as conditions for scientific validation.
2. **Interoperability as architecture** — systems designed to communicate, avoiding silos.
3. **Reproducibility as requirement** — publishing conditions for re-execution, not just results.

## 1.4 Ecosystem map

| INPE/TerraME | DisSModel | Role |
|---|---|---|
| TerraME | `dissmodel` | Generic framework for dynamic spatial modeling |
| LUCCME | `disslucc-continuous` / `disslucc-discrete` | LUCC domain models |
| TerraLib | `geopandas` / `rasterio` | Geographic data handling |
| — | `disscube` | Spatial data cube (no direct equivalent in the original stack) |

## See also

- [dissmodel on GitHub](https://github.com/DisSModel/dissmodel)

## Summary

DisSModel is not a rewrite for its own sake — it is the 2024–2026 synthesis
of a research agenda running since 2001, from XML-based geographic
interoperability (Terra Translator) through verifiable executable models
(TerraHS) to spatially explicit dynamic modeling as a scientific object
(TerraME/LuccME). The move away from TerraME is driven by two concrete
costs — a Lua DSL with limited data-science adoption, and no release since
August 2020 — not by a rejection of TerraME's modeling paradigm, which
DisSModel and its DisSLUCC satellite packages deliberately preserve
(`CellularSpace` → `SpatialModel`/`RasterModel`, LuccME's Demand/Potential/
Allocation → `disslucc-continuous`/`disslucc-discrete`). The three
principles — openness, interoperability, reproducibility — are what the
rest of this book keeps coming back to: every chapter after this one is,
in some way, an elaboration of how DisSModel operationalizes them in
Python.
