# Chapter 3: Installation and first model

*Part I — DisSModel Core*

## Learning objectives

- Install `dissmodel` and an extension package
- Run a first model via CLI, Streamlit, and Jupyter
- Understand the minimal structure of a model project

## 3.1 Installation

`dissmodel` itself is published on PyPI:

```bash
pip install dissmodel
```

Extension packages such as `dissmodel-ca` are **not** published on PyPI —
install them straight from GitHub, either as a released version tag or
the development branch:

```bash
# latest development version of the core framework
pip install "git+https://github.com/DisSModel/dissmodel.git@main"

# an extension package (dissmodel-ca has no PyPI release)
pip install "git+https://github.com/DisSModel/dissmodel-ca.git"
```

If you're working from a local clone of an extension repo instead (e.g.
to run its bundled `examples/`), install it in editable mode from inside
the repo:

```bash
git clone https://github.com/DisSModel/dissmodel-ca.git
cd dissmodel-ca
pip install .
```

## 3.2 Running the first model (Game of Life)

`dissmodel-ca` ships the same Game of Life model in three forms, all
built on the identical `GameOfLife` class from `dissmodel_ca.models` — a
plain CLI script, a Streamlit dashboard, and a didactic Jupyter notebook.

**CLI** (`examples/cli/ca_game_of_life.py`):

```python
from matplotlib.colors import ListedColormap

from dissmodel.core import Environment
from dissmodel.geo import vector_grid
from dissmodel_ca.models import GameOfLife
from dissmodel.visualization import Map

gdf = vector_grid(dimension=(20, 20), resolution=1, attrs={"state": 0})
env = Environment(start_time=0, end_time=10)

gol = GameOfLife(gdf=gdf)
gol.initialize()

cmap = ListedColormap(["white", "black"])
Map(gdf=gdf, plot_params={"column": "state", "cmap": cmap, "ec": "gray"})

env.run()
```

```bash
python examples/cli/ca_game_of_life.py
```

This is the minimal pattern every model in this book follows: build a
`vector_grid`, open an `Environment` with a time window, instantiate the
model against the grid, optionally attach a `Map`/`Chart` for live
visualization, then call `env.run()`.

**Streamlit** (`examples/streamlit/ca_all.py`) goes one step further and
turns *every* `CellularAutomaton` subclass in `dissmodel_ca.models` into
an interactive app — it discovers models via
`inspect.getmembers(ca_models, inspect.isclass)`, filtering to concrete
`CellularAutomaton` subclasses, then renders sidebar controls
(model choice, simulation steps, grid size, colormap) automatically:

```bash
streamlit run examples/streamlit/ca_all.py
```

This works because every model follows the same conventions: annotated
attributes (`param: float`) are picked up by `display_inputs` and
rendered as widgets, `initialize()` consumes those parameters, and
`execute()` applies `rule()` to every cell each step.

**Jupyter** — the package's 15+ notebooks under `examples/notebooks/` are
the didactic entry point (the README calls them "the best way to learn
about each model"); note they are written in Portuguese:

```bash
jupyter lab examples/notebooks/ca_game_of_life.ipynb
```

`ca_game_of_life.ipynb` walks through the rule table (dies on
under/overpopulation, survives at 2–3 neighbors, born at exactly 3) and
includes a library of built-in patterns (`blinker`, `toad`, `beacon`,
`pulsar`, `glider`, `lwss`, `block`, ...) that can be seeded onto the grid
independently of the `GameOfLife` class itself.

## 3.3 Minimal structure of a model project

The "science" (models) vs. "runnable entry point" split shown here for
`dissmodel-ca` also recurs identically in `dissmodel-sysdyn`
(`src/dissmodel_sysdyn/models/` + `examples/{cli,streamlit,notebooks}/`).
Domain-executor packages built around `ModelExecutor` — `disslucc-continuous`,
`disslucc-discrete`, `brmangue-dissmodel` (Chapters 7–8) — use a related
but distinct convention: `src/<package>/{models,executors}/` for the
science/infrastructure split, and `examples/data/` for sample inputs run
through the executor's own CLI (`run_cli`) rather than through separate
`cli`/`streamlit`/`notebooks` folders.

```
dissmodel-ca/
  src/dissmodel_ca/models/     # the "Science" layer — CA model classes only
  examples/
    cli/                       # simplified, self-contained scripts for quick testing
    streamlit/                 # reactive UI components/apps
    notebooks/                 # didactic, step-by-step notebooks
```

- **`src/<package>/models/`** holds only model classes — subclasses of
  `SpatialModel`/`RasterModel`/`CellularAutomaton` with no CLI, no
  argparse, no I/O.
- **`examples/cli/`** is the fastest way to confirm a model runs at all —
  a linear script: build grid → `Environment` → model → `env.run()`.
- **`examples/streamlit/`** is for interactive parameter exploration; the
  `ca_all.py` pattern of auto-discovering model classes via `inspect` is
  reusable in any package that wants a one-file "explorer" app.
- **`examples/notebooks/`** is the teaching layer, with prose alongside
  runnable cells — the place to point students, not the CLI scripts.

When you install a package in editable mode (`pip install .` from inside
its repo, as in 3.1) you get direct access to run any of these three
example folders locally.

## Exercises

1. Install `dissmodel` from PyPI and `dissmodel-ca` from GitHub in a fresh
   virtual environment. Run `examples/cli/ca_game_of_life.py` and confirm
   the Game of Life pattern evolves over 10 steps.
2. Open `examples/streamlit/ca_all.py` and, without modifying the code,
   switch the model dropdown to `FireModel`. Identify which parameters
   change in the sidebar and explain why (hint: look at the annotated
   attributes on the model class in `src/dissmodel_ca/models/`).
3. Seed the grid in `ca_game_of_life.py` with the `"glider"` pattern from
   `PATTERNS` (see `examples/notebooks/ca_game_of_life.ipynb`) instead of
   an all-dead grid, and run for 20 steps. Does it behave as a spaceship
   on this substrate?
4. Compare the vector CLI example (`ca_game_of_life.py`, `vector_grid` +
   `GameOfLife`) with `ca_game_of_life_raster.py`. What changes in the
   setup code, and what stays the same in the model's `rule()`?

## Summary

Getting started with DisSModel means installing two different things in
two different ways: `dissmodel` itself from PyPI, and any extension
package (`dissmodel-ca`, and by extension `dissmodel-sysdyn`,
`disslucc-*`, `brmangue-dissmodel`) straight from its GitHub repo, since
none of the extensions are on PyPI yet. Once installed, every package
exposes the same three-tier example structure — `examples/cli` for the
fastest possible confirmation that a model runs, `examples/streamlit` for
interactive parameter exploration, and `examples/notebooks` for the
didactic explanation — all three driving the identical model classes
under `src/<package>/models/`. Domain-executor packages (Chapters 7–8)
replace this with an `executors/` + `examples/data/` + CLI convention,
which Chapter 2's `ModelExecutor` section already introduced.
