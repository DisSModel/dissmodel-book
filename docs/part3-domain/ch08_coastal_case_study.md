# Chapter 8: Case Study — Coastal Dynamics

*Part III — Domain Modeling: Land Use & Coastal Systems*

Implemented by the [`brmangue-dissmodel`](https://github.com/DisSModel/brmangue-dissmodel) package.

## Learning objectives

- Understand the coupled coastal flood and mangrove migration model
- Compare the raster and vector implementations
- Run the raster vs. vector equivalence benchmark

## 8.1 Overview

`brmangue-dissmodel` implements spatially explicit models of coastal
ecosystem processes based on Bezerra et al. (2013), built on the
DisSModel framework. Two coupled processes are modelled:

1. **Flood dynamics** — sea-level rise propagation and terrain elevation adjustments.
2. **Mangrove migration** — ecosystem response to rising sea levels, soil transitions, and sediment accretion.

## 8.2 Two substrates

- **Raster** (`brmangue.models.raster`) — NumPy/RasterBackend, vectorized and fast. The canonical implementation, validated against TerraME golden outputs.
- **Vector** (`brmangue.models.vector`) — GeoDataFrame/libpysal, cell-by-cell over real polygon geometry. Numerically equivalent to the raster implementation (verified by the benchmark executor).

## 8.3 Running it

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

# Validation against TerraME golden CSVs (checkpointed comparison)
python src/brmangue/executors/validation_executor.py run \
  --input  examples/data/input/elevacao_pol.zip \
  --param  golden_dir=tests/fixtures/golden \
  --param  end_time=20 \
  --param  taxa_elevacao=0.05 \
  --param  altura_mare=6.0 \
  --param  checkpoints=[1,5,10,15,20]
```

The `main_benchmark.py` run reports match%, MAE, and RMSE per band
between the raster and vector substrates run on the same input — the
same benchmark pattern used by `disslucc-continuous` (Chapter 7). The
`validation_executor.py` run is the stricter check: it compares the
raster substrate's output directly against TerraME's own golden-output
CSVs (`tests/fixtures/golden/step_NN.csv`) at the listed checkpoints,
which is what backs the "validated against TerraME golden outputs" claim
in 8.2.

## Exercises

1. Run the raster and vector CLI examples from 8.3 over the same
   `synthetic_grid_60x60` input and compare the two output files
   (`saida.tiff` vs `saida.gpkg`). What has to match between them for the
   models to be considered equivalent, given both implement "identical
   equations, thresholds, parameter names, and update ordering" per the
   package README?
2. Run `main_benchmark.py` with `tolerance=0.05` as shown, then try
   `tolerance=0.01`. Does the benchmark still pass? What does changing
   the tolerance tell you about how close raster and vector actually are
   numerically, versus just "close enough for this threshold"?
3. The flood model uses a "push-based neighbourhood algorithm faithful to
   the original TerraME implementation" (Bezerra et al., 2013). Contrast
   this with `CellularAutomaton.rule(idx)` from Chapter 5, which is a
   *pull* model. Why does sea-level propagation need a push model instead
   of a pull rule?
4. Run the golden-CSV validation (`validation_executor.py`) with the
   default `checkpoints=[1,5,10,15,20]`. If step 20 diverges from the
   golden CSV but steps 1–15 match exactly, what does that suggest about
   where a numerical bug would most likely be — in initialization, in the
   per-step transition rule, or in an accumulating rounding error?

## Summary

`brmangue-dissmodel` is the ecosystem's deepest validation case study: the
same coupled flood/mangrove-migration model (Bezerra et al., 2013) is
implemented on both substrates with intentionally identical equations,
thresholds, and update ordering, then checked two different ways — a
raster-vs-vector benchmark (match%/MAE/RMSE, tolerance-based) and a
raster-vs-TerraME comparison against golden-output CSVs at fixed
checkpoints. Where Chapter 7's `disslucc-discrete` reaches exact 100%
cell-level parity because land use is categorical, BR-MANGUE's flood and
mangrove state is continuous (elevation, tidal height, sediment), so its
validation is necessarily tolerance-based rather than exact-match. The
push-based flood propagation is also a reminder that not every spatial
model fits `CellularAutomaton`'s pull-based `rule(idx)` contract (Chapter
2) — `SpatialModel` alone, with a free `execute()`, is the right base
class whenever a process (like a rising tide) originates at a source and
spreads outward rather than being computed independently per cell.
