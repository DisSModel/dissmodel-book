# Lesson 15 — Summary / Migrating from TerraME

*Part VII — TerraME Compatibility Course*

## Closing the course

The 14 previous lessons covered three modelling paradigms — System Dynamics (stocks and flows, Lessons
3-8), Cellular Automata (homogeneous spatial grid, Lessons 9-12), and Agent-based Modeling (discrete,
heterogeneous individuals, Lessons 13-14) — all re-implemented in the DisSModel ecosystem in Python,
instead of TerraME in Lua. This final lesson serves as a quick reference bridge for anyone who already
knows TerraME and just wants the conceptual translation, or for reviewing the whole course at once.

## General equivalence table

| TerraME concept (Lua) | DisSModel equivalent (Python) |
|---|---|
| `init(self)` | `setup()` |
| `execute(self)` | `execute()` |
| `CellularSpace` | `SpatialModel` / `RasterModel` |
| Von Neumann / Moore neighborhood | `Rook` / `Queen` (via `libpysal`) |
| `Agent`, `Society` | `dissmodel-abm` |
| `gis` package, `Layer:fill` | `disscube` |
| Stocks/flows (`sysdyn`) | `dissmodel-sysdyn` |
| Land change models (CLUE-like) | `disslucc-continuous` |

## Incremental migration roadmap

For anyone with a real model in TerraME/Lua who wants to migrate gradually, instead of rewriting
everything at once, the suggested path follows the same logical order as this course:

1. Swap the language syntax first (Lesson 2) — rewrite `init()`/`execute()` in Python, keeping the same
   business logic.
2. Migrate the spatial data structure (`CellularSpace` → `SpatialModel`/`RasterModel`, neighborhoods via
   `libpysal`) — Lessons 9-11.
3. Migrate paradigm-specific submodels (`sysdyn`, `ca`, `abm`, `disslucc`) according to the original
   model's type.
4. Validate against the original TerraME model's output — Ch. 7/11 documents the comparison methodology
   used internally within the ecosystem itself (MAE/RMSE/kappa) to ensure the migration did not change
   the model's behavior.

## What this course does not replace

Worth remembering, as we close: the [dissmodel-book](https://github.com/DisSModel/dissmodel-book) is the
ecosystem's **technical** reference — it documents what each package does and how to use it. The
conceptual theory behind each paradigm (balancing/reinforcing feedback, sensitivity to initial conditions,
phase plane, goodness-of-fit, Couclelis' four types of agents) is deliberately not there — and that is
what the 14 lessons of this course fill in, each with its own file, so nothing gets lost in the migration.

---
→ Corresponding chapter in the [dissmodel-book](https://github.com/DisSModel/dissmodel-book):
[Ch. 11 — Migrating from TerraME/LUCCME](../part5-reference/ch11_migration.md)
