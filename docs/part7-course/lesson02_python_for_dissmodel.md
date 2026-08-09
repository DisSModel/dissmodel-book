# Lesson 2 — Python for DisSModel

*Part VII — TerraME Compatibility Course*

*Replaces "Lua for TerraME" — the original lesson taught Lua syntax; here we swap it for DisSModel's
lifecycle pattern in Python.*

## Dynamic models

A dynamic model represents how a system (`S`) evolves over time: given the state of the world at time
`t`, a function `f(s)` produces the state at time `t+1`. The quote from Lewis Fry Richardson, a pioneer of
numerical weather prediction, sums up well why it's worth formalizing this mathematically:

> "The advantage of a mathematical statement is that it is so definite that it might be definitely wrong
> [...] Some verbal statements have not this merit; they are so vague that they could hardly be wrong, and
> are correspondingly useless."

This is the very reason both TerraME and DisSModel exist: to provide a computational environment for
expressing these models formally, covering the entire modelling cycle — design, implementation,
calibration, simulation, verification, and publication.

## Why swap Lua for Python?

TerraME chose Lua in the 2000s because it is a simple, small, efficient, and portable language — good for
embedding inside a C++ environment. But this brought a real cost: few people in the scientific community
know Lua, documentation was for a long time only available in Portuguese, and the ecosystem of scientific
libraries (statistics, geoprocessing, machine learning) is much shallower than Python's. This language
barrier is precisely the central motivation behind creating DisSModel.

## The `setup()` / `execute()` pattern

Where the original lesson taught Lua syntax (variables, tables, higher-order functions, `for`, `if`), the
DisSModel foundation is leaner: every model follows a two-phase lifecycle.

| Phase | When it runs | What it's for |
|---|---|---|
| `setup(**kwargs)` | once, right after `__init__` | initial configuration — build the neighborhood, initialize state |
| `execute()` | every time step | the model's transition rule |

This directly replaces Lua's "tables with functions" concept (`function MyLocation(locdata) ...`) with
ordinary Python classes, and TerraME's `CellularSpace`/neighborhood with `SpatialModel`/`RasterModel`,
with neighborhoods built via `libpysal` (`Queen`, `Rook`, `KNN`, custom).

## What to practice in this lesson

- Install DisSModel and one of the satellite packages (`dissmodel-sysdyn` is the simplest to start with).
- Run an example from `examples/cli/` and observe the `setup()`→`execute()` cycle in the code.
- Compare a Lua snippet from the original course side by side with its Python equivalent (Chapter 11 has
  a complete correspondence table, useful to consult here as well).

---
→ Corresponding chapters in the [dissmodel-book](https://github.com/DisSModel/dissmodel-book):
[Ch. 1 — Why DisSModel](../part1-core/ch01_why_dissmodel.md) ·
[Ch. 2 — Core concepts](../part1-core/ch02_core_concepts.md) ·
[Ch. 3 — Getting started](../part1-core/ch03_getting_started.md)
