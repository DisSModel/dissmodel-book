# Chapter 8: Case Study — Coastal Dynamics

*Part III — Domain Modeling: Land Use & Coastal Systems*

Implemented by the [`brmangue-dissmodel`](https://github.com/DisSModel/brmangue-dissmodel) package.

## Learning objectives

- Understand the coupled coastal flood and mangrove migration model
- Compare the raster and vector implementations
- Run the raster vs. vector equivalence benchmark

## 10.1 Overview

`brmangue-dissmodel` implements spatially explicit models of coastal
ecosystem processes based on Bezerra et al. (2013), built on the
DisSModel framework. Two coupled processes are modelled:

1. **Flood dynamics** — sea-level rise propagation and terrain elevation adjustments.
2. **Mangrove migration** — ecosystem response to rising sea levels, soil transitions, and sediment accretion.

## 10.2 Two substrates

- **Raster** (`brmangue.models.raster`) — NumPy/RasterBackend, vectorized and fast. The canonical implementation, validated against TerraME golden outputs.
- **Vector** (`brmangue.models.vector`) — GeoDataFrame/libpysal, cell-by-cell over real polygon geometry. Numerically equivalent to the raster implementation (verified by the benchmark executor).

## 10.3 Running it

```bash
# Raster simulation (NumPy-based, fast)
python examples/main_raster.py run \
  --input  examples/data/input/synthetic_grid_60x60_tiff.zip \
  --output examples/data/output/saida.tiff \
  --param  interactive=true \
  --param  end_time=20

# Vector simulation (GeoDataFrame-based)
python examples/main_vector.py run \
  --input  examples/data/input/synthetic_grid_60x60_shp.zip \
  --output examples/data/output/saida.gpkg \
  --param  end_time=20

# Vector vs raster equivalence benchmark
python examples/main_benchmark.py run \
  --input  examples/data/input/synthetic_grid_60x60_shp.zip \
  --param  end_time=10 \
  --param  taxa_elevacao=0.011 \
  --param  tolerance=0.05
```

## Exercises

*TODO: write exercises.*

## Summary

*TODO: summarize the chapter's key takeaways.*
