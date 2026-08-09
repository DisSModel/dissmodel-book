# Lesson 11 — Cellular Data

*Part VII — TerraME Compatibility Course*

## The problem: data comes from very different sources

Real-world spatial data arrives in representations, formats, resolutions, extents, and units of
measurement that are completely different from one another (satellite images, road shapefiles, protected
area polygons, port points). Before running any model, this data needs to be **homogenized** into a
common representation.

## Homogenizing with a cellular representation

A cellular representation unifies different formats, resolutions, and extents: once converted to cells of
the same grid, the data can be treated as samples/"objects" without needing to consider its original
spatial representation. This helps both statistics and modelling — it's the same reasoning behind the
Amazon deforestation example: roads, ports, and protected areas (all in different vector formats) become
attributes of the same cells representing deforestation.

## Resolution: a trade-off choice

The resolution of the cellular grid is limited by three independent factors: the resolution of the
available input data, the computational capacity to process the grid, and the process under study (a
local fire model needs much finer resolution than a global climate-change model).

## TerraME's `gis` package

- **Project**: represents a GIS project, containing a set of layers.
- **Layer**: names a data source (which can be in different formats); can be used to create new data;
  offers functions to manipulate geospatial data.
- **`Layer:fill`**: creates a new attribute on the cells from an input data source, an aggregation
  strategy, and additional arguments.

## Pixel within polygon: overlap vs. centroid

Even before choosing an aggregation strategy, you need to decide **when a pixel belongs to a cell**:

- **Overlap**: the pixel counts toward every cell it intersects, even partially — the same pixel can
  contribute to several neighboring cells.
- **Centroid**: the pixel counts only toward the cell that contains the pixel's center — each pixel
  belongs to exactly one cell.

This choice directly affects the pixel count per cell at the edges, and therefore the result of any
aggregation strategy applied afterward.

## `Layer:fill` strategies

| Strategy | Input data type | What it does |
|---|---|---|
| `mode` (majority) | categorical raster | assigns the cell the most frequent class among the pixels |
| `average` / `mean` | continuous raster | assigns the cell the average of the pixel values (with `area=true`, weighted by intersection area — does not conserve mass) |
| `sum` | continuous raster | sums the pixel values, weighted by intersection area when `area=true` — conserves mass |
| `coverage` | categorical raster | creates one attribute per class, with each one's coverage fraction (summing to 1) |
| `presence` | vector | indicates whether any vector object is present in the cell |
| `count` | vector | counts how many vector objects are in the cell |
| `distance` | vector | computes the cell's distance to the nearest vector object |
| `length` / `area` | vector (line/polygon) | sums the length or area of the object within the cell |

For `sum` and `average`, the exact formulas use the intersection area `a(I)` between polygon `P` and cell
`C`, and the total area of the polygon `a(P)`:

```
sum:     f(C) = Σ f(P) × a(I) / a(P)      → conserves mass
average: f(C) = Σ f(P) × a(I) / a(P)      → does not conserve mass (varies by definition used)
```

## Associated exercise

Rebuild a runoff database from a DEM (`cabecadeboi.tif`) and an area clip (`cabecadeboi-box.shp`), first
at resolution 200, then redo everything at resolution 100 — a direct exercise on the resolution trade-off
mentioned above.

## Practice in DisSModel

`disscube` re-implements each of these strategies with an equivalent operator — the full correspondence
is documented, strategy by strategy, in the `terrame_fill_correspondence.md` file in Ch. 10.

---
→ Corresponding chapter in the [dissmodel-book](https://github.com/DisSModel/dissmodel-book):
[Ch. 10 — Spatial Data Cubes](../part4-infra/ch10_disscube.md)
(`terrame_fill_correspondence.md` table)

> **Note:** the `overlap` vs. `centroid` distinction for pixel geometric sampling has no dedicated section
> in the book — `disscube`'s operators document the aggregation itself, not this prior decision.
