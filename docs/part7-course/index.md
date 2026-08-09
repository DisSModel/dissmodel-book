# Part VII: TerraME Compatibility Course

*A guided, code-level demonstration that the TerraME modelling ecosystem — System Dynamics, Cellular
Automata, Agent-based Modeling — is reproducible in DisSModel, lesson by lesson, model by model.*

## Why this part exists

Parts I–VI of this book are the ecosystem's **technical reference**: what each module does, and how to
use it. This part is different in kind — it adapts the original **TerraME/Lua** course
([wiki](https://github.com/TerraME/terrame/wiki/course), taught in São José dos Campos by Pedro R.
Andrade, Gilberto Câmara, and Francisco Gilney Bezerra) into 15 lessons that teach the *why* behind each
paradigm and prove compatibility with real code, not just narrative.

Each lesson takes a model from the original TerraME course and shows, side by side, the original Lua
implementation and its real Python equivalent in DisSModel — extracted directly from the source
repositories, not reconstructed — to demonstrate that the TerraME ecosystem is reproducible in DisSModel
without conceptual loss.

## How each lesson is organized

1. The detailed theory behind the original slide — the conceptual content the rest of this book
   deliberately leaves out (feedback theory, chaos, phase-plane analysis, goodness-of-fit, ABM theory).
2. A link to the corresponding technical chapter elsewhere in this book (Parts I–V), for the
   how-to-use-it reference.
3. In the System Dynamics, Cellular Automata, and Agent-based Modeling lessons: a **side-by-side code**
   section, with snippets pulled directly from the TerraME and DisSModel source repositories.

## Repositories used for the code comparisons

| Paradigm | TerraME (Lua) | DisSModel (Python) |
|---|---|---|
| System Dynamics | [TerraME/sysdyn](https://github.com/TerraME/sysdyn) | [dissmodel-sysdyn](https://github.com/DisSModel/dissmodel-sysdyn) |
| Cellular Automata | [TerraME/ca](https://github.com/TerraME/ca) | [dissmodel-ca](https://github.com/DisSModel/dissmodel-ca) |
| Agent-based Modeling | [TerraME/logo](https://github.com/TerraME/logo) | [dissmodel-abm](https://github.com/DisSModel/dissmodel-abm) |

## Lessons

| # | Lesson | Ties into |
|---|---|---|
| 1 | [Introduction to Computing](lesson01_intro_computing.md) | — (language-independent) |
| 2 | [Python for DisSModel](lesson02_python_for_dissmodel.md) | Ch. 1–3 |
| 3 | [System Dynamics](lesson03_system_dynamics.md) | Ch. 4 |
| 4 | [Feedbacks](lesson04_feedbacks.md) | Ch. 4 |
| 5 | [Epidemics](lesson05_epidemics.md) | Ch. 4 |
| 6 | [Chaos](lesson06_chaos.md) | Ch. 4 |
| 7 | [Predator-Prey (differential equations)](lesson07_predator_prey_ode.md) | Ch. 4 |
| 8 | [Daisyworld](lesson08_daisyworld.md) | Ch. 4 |
| 9 | [Cellular Automaton](lesson09_cellular_automaton.md) | Ch. 5 |
| 10 | [Fire in the Forest](lesson10_fire_in_the_forest.md) | Ch. 5 |
| 11 | [Cellular Data](lesson11_cellular_data.md) | Ch. 10 |
| 12 | [Deforestation](lesson12_deforestation.md) | Ch. 7 |
| 13 | [Agent-based Modeling](lesson13_agent_based_modeling.md) | Ch. 6 |
| 14 | [Predator-Prey (agents)](lesson14_predator_prey_abm.md) | Ch. 6 |
| 15 | [Summary / Migrating from TerraME](lesson15_summary_migration.md) | Ch. 11 |

## Out of scope for now

Two lessons from the original TerraME course have no ready-made model in the DisSModel ecosystem yet, and
therefore have no lesson here:

- **Runoff** — would require a `RasterModel` with a custom rule comparing cell elevation to its
  neighbors, using `disscube`/`RasterBackend` (Ch. 10) to ingest the DEM.
- **Agent-based SIR** (exercise from Lesson 13) — would combine `AgentModel` with neighborhood-based
  contagion logic, along the lines of `PredatorPreyModel` (Lesson 14).

When either lands in a satellite package, its lesson returns to this list.
