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

*TODO: expand with concrete data — TerraME's latest release is 2.0.1
(August 2020), C++/Lua, OS-specific installers. Discuss the cost of
maintaining a custom DSL (Lua) vs. leveraging the already-mature Python
scientific ecosystem (geopandas, rasterio, xarray).*

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

*TODO: summarize the chapter's key takeaways.*
