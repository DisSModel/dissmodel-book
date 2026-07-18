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

*TODO: expand with a code example showing the same model running on both
substrates, citing the numerical parity validation done in
`brmangue-dissmodel` (raster vs. vector against TerraME reference outputs).*

## 2.2 `SpatialModel` and the model lifecycle

*TODO: document `setup()` / `execute()` and the conceptual mapping to
TerraME (`init(self)` / `execute(self)`).*

## 2.3 `ModelExecutor`

*TODO: document the executor pattern used by all satellite packages
(dissmodel-ca, disslucc-discrete, brmangue-dissmodel) and how it connects
to `dissmodel-configs` on the platform.*

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

*TODO: summarize the chapter's key takeaways.*
