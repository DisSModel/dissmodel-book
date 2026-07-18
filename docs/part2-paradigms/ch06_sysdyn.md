# Chapter 6: System Dynamics

*Part II — Simulation Paradigms*

Implemented by the [`dissmodel-sysdyn`](https://github.com/DisSModel/dissmodel-sysdyn) package.

## Learning objectives

- Install and explore the system dynamics model library
- Run models via CLI, Streamlit, and notebooks
- Recognize the categories of available models

## 6.1 Installation

```bash
pip install dissmodel-sysdyn
```

## 6.2 Included models by category

| Category | Models |
|---|---|
| Epidemiology | SIR |
| Ecology & Biology | Predator-Prey (Lotka-Volterra), Yeast Growth, Daisyworld, Population Growth, Limited Growth, Chaotic Growth |
| Physics & Thermodynamics | Coffee Cooling, Room Temperature, Tub (Stock/Flow) |
| Complex Systems | Lorenz Attractor, Homeostasis |
| Environment | Mono Lake Water Balance |
| Stochasticity | Random Walk |

## 6.3 Usage

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

*TODO: write exercises.*

## Summary

*TODO: summarize the chapter's key takeaways.*
