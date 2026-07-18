# Chapter 4: System Dynamics

*Part II — Simulation Paradigms*

Implemented by the [`dissmodel-sysdyn`](https://github.com/DisSModel/dissmodel-sysdyn) package.

> **Live demo:** try the models in this chapter directly in the browser —
> [dissmodel-sysdyn-demo on Hugging Face Spaces](https://huggingface.co/spaces/profsergiocosta/dissmodel-sysdyn-demo).

## Learning objectives

- Install and explore the system dynamics model library
- Run models via CLI, Streamlit, and notebooks
- Recognize the categories of available models

## 4.1 Installation

`dissmodel-sysdyn`'s own README shows `pip install dissmodel-sysdyn`, but
the package has no PyPI release (no PyPI badge, no publish workflow) —
the main `dissmodel` repository's own ecosystem table documents the
actual install path as a direct GitHub install, same as every other
extension package (Chapter 3):

```bash
pip install "git+https://github.com/DisSModel/dissmodel-sysdyn.git"
```

## 4.2 Included models by category

| Category | Models |
|---|---|
| Epidemiology | SIR |
| Ecology & Biology | Predator-Prey (Lotka-Volterra), Yeast Growth, Daisyworld, Population Growth, Limited Growth, Chaotic Growth |
| Physics & Thermodynamics | Coffee Cooling, Room Temperature, Tub (Stock/Flow) |
| Complex Systems | Lorenz Attractor, Homeostasis |
| Environment | Mono Lake Water Balance |
| Stochasticity | Random Walk |

## 4.3 Usage

```bash
python examples/cli/sysdyn_sir.py
streamlit run examples/streamlit/sysdyn_all.py
jupyter notebook examples/notebooks/
```

There are 14 educational notebooks, each functioning as a self-contained
tutorial with scientific context, mathematical formulation, and guided
experiments (e.g. `sysdyn_daisyworld.ipynb` on the Gaia hypothesis,
`sysdyn_lorenz.ipynb` on deterministic chaos).

## Exercises

1. Open `src/dissmodel_sysdyn/models/sir.py`. Unlike every model in
   Chapter 5, `SIR` extends `dissmodel.core.Model` directly, not
   `SpatialModel`. Explain why a system-dynamics model has no need for a
   `gdf`/`backend` or a neighborhood.
2. `SIR` is decorated with three stacked `@track_plot("Susceptible",
   "green")` / `@track_plot("Infected", "red")` /
   `@track_plot("Recovered", "blue")` calls. Run
   `python examples/cli/sysdyn_sir.py` and identify, from the live chart,
   which compartment peaks first and why (hint: compare the `duration`
   and `contacts` default parameters documented in the class docstring).
3. Compare `sir.py` (a compartmental model with discrete stocks) to
   `lorenz.py` (a continuous three-variable chaotic system). Both
   override `execute()` on a plain `Model` — what does each model's
   `execute()` compute per tick, and why does neither need `pre_execute`
   or `post_execute`?
4. Read the `sysdyn_daisyworld.ipynb` notebook's introduction and
   describe, in your own words, what the Gaia hypothesis claims and how
   the model's stocks (black daisies, white daisies, bare ground)
   operationalize it.

## Summary

`dissmodel-sysdyn` shows that DisSModel's core lifecycle (`setup` →
`execute`, driven by `Environment.run()`) is not tied to spatial data at
all: every model here subclasses `dissmodel.core.Model` directly, with no
`gdf`, no `backend`, and no neighborhood — the state is just a handful of
numeric stocks (susceptible/infected/recovered, predator/prey
populations, x/y/z in the Lorenz system) advanced by ODE-like update
equations each tick. The `@track_plot` decorator is the piece that turns
any tracked attribute into an automatically-charted time series with no
extra plumbing, which is why every model in the package — from `SIR` to
`Daisyworld` to the `Lorenz` attractor — can be dropped into the same
Streamlit explorer (`sysdyn_all.py`) used for CA models in Chapter 5. The
14 notebooks are this package's main teaching surface: each pairs a
scientific framing (epidemiology, ecology, thermodynamics, deterministic
chaos) with the exact stock/flow equations implemented in
`src/dissmodel_sysdyn/models/`.
