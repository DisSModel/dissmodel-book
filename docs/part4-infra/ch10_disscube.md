# Chapter 10: Spatial Data Cubes

*Part IV — Data & Infrastructure*

Implemented by the [`disscube`](https://github.com/DisSModel/disscube) package.

## Learning objectives

- Understand the `SpatialSource → Derivation → Variable → DerivedVariable` pipeline
- Register a grid and a spatial source
- Derive a variable and integrate it into a LUCC model

## 10.1 Core concept

DisSCube is the spatial data cube engine of the DisSModel ecosystem. It
converts raw geospatial sources (raster, vector) into derived variables
aligned to LUCC modeling grids, ready for Cellular Automata models and
spatio-temporal analysis.

```
SpatialSource  →  Derivation  →  Variable  →  DerivedVariable (Zarr)
```

A **source** goes through a **derivation** that applies an **operator** to
a **grid**, producing a **derived variable** registered in the SQLite
catalog and stored in Zarr.

## 10.2 Basic flow

```python
from disscube.client import CubeClient
from disscube.utils.grids import register_local_grid

cube = CubeClient(catalog="catalog.db", store="./data/")
grid = register_local_grid(cube, name="AC", bbox_geo=(-73.99, -11.15, -66.62, -7.11), resolution=5_000.0)
```

```python
from disscube.models import SpatialSource

cube.register_spatial_source(SpatialSource(
    id="mapbiomas_2020", name="MapBiomas Acre 2020", format="raster",
    asset_url="data/raw/mapbiomas_2020.tif", crs="EPSG:4326", time=2020,
))
```

```python
from disscube.derivation import Derivation

d = Derivation(target="forest_pct", source_id="mapbiomas_2020",
                operator="percentage", class_code=3, role="driver",
                valid_from="2020", valid_until="2020")
cube.derive_declarative(d, grid_id="AC/5km")
```

## 10.3 Integration with DisSModel

```python
backend = cube.to_lucc_data(["forest_pct", "dist_roads"], grid_id="AC/5km")
```

*TODO: expand with the rest of the integration flow and correspondence
examples with TerraME's FillCell (`terrame_fill_correspondence.md`).*

## 10.4 Relationship to Chapter 7 (DisSLUCC)

*TODO: make explicit how derived variables produced here feed as drivers
into `disslucc-continuous` and `disslucc-discrete` models.*

## Exercises

*TODO: write exercises.*

## Summary

*TODO: summarize the chapter's key takeaways. Note: disscube is in Alpha
status — the declarative API is still evolving.*
