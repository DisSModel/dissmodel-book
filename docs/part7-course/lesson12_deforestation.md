# Lesson 12 — Deforestation: Top-down Modelling

*Part VII — TerraME Compatibility Course*

## Modelling and public policy

Land change models don't exist in a vacuum — they serve to connect a system (with external influences
from ecology, economics, and politics) to decisions: a policymaker compares possible scenarios to choose
policy options that move the system toward a desired state. The course's case study is deforestation in
the Brazilian Amazon, using PRODES/INPE data.

## The three-submodel (top-down) architecture

Classic "top-down" land change models (like CLUE) have three components that run in a loop at every time
step, transforming the land use map at `t` into the map at `t+1`:

1. **Demand submodel** ("how much?"): how much area changes from one class to another in that time step.
2. **Transition potential submodel** ("where?"): a potential map, indicating where the change is most
   likely to occur.
3. **Change allocation submodel**: allocates the demand in space, following the potential map.

### Demand

Can come from historical trend analysis, scenario building, a global economic model, or a transition
matrix (Markov chain). The simplest version used in the course ("Simple Demand") is just the area
difference between two consecutive years.

### Potential (CLUE-like)

The potential map combines layers of explanatory variables — in the Amazon example: distance to roads,
distance to ports, and protected areas (subtracted from the potential, since they reduce the chance of
deforestation). These variables, all in different formats, must first go through the cellular
homogenization from Lesson 11.

The most common statistical form is a multiple regression:

```
y = a0 + a1·x1 + a2·x2 + ... + ai·xi + ε
```

with the basic assumption that the process is stationary (the relationships between the variables and
deforestation don't change over time). In the real example with Amazon data (1996/2006, 26 variables),
variables such as soil fertility, distance to paved roads and to ports, and protected-area extent explain
between 45% and 51% of the observed variance (R² = 0.45–0.51).

Three strategies equivalent to the original course's `deforestation.lua` script:
- **Neighborhood**: potential based on the average deforestation of neighbors.
- **Regression**: potential based on regression against distance to roads, ports, and protected areas.
- **Mixed**: combines both — neighborhood + regression.

### Allocation

Given the potential map at `t` and the landscape map at `t`, allocation distributes the demand in space to
generate the landscape map at `t+1`. Common strategies: rank ordering (allocates first to the
highest-potential cells), stochastic, or iterative.

## Calibration and validation

Models need to be calibrated and "validated" against real data — and this requires splitting the
timeline into distinct periods: a **calibration** period (e.g. `tp-20` through `tp`, used to fit the
model's parameters) and a **validation** period (`tp` through `tp+10`, data never seen by the model during
fitting, used only to check that it generalizes).

### Goodness-of-fit (Costanza, 1989)

Comparing a simulated map against a real map cell by cell, exactly, is excessively rigid — two
simulations can be "equally good" without matching pixel by pixel. Costanza (1989) proposed a
**multiscale** fit: comparing the maps in windows of increasing size (3×3, 5×5, 9×9...) instead of cell by
cell, producing a fit curve as a function of window size — two maps can have low fit at fine resolution
but high fit at larger scales, which is still informative. A more recent variant normalizes the error by
demand instead of measuring fit directly, computing error instead of match.

## Are land change models cellular automata?

The lesson ends with a provocative question: a land change model has a grid of cells, a neighborhood, a
finite set of states, transition rules, an initial state, discrete time, parallel behavior in space, and
reads from neighbors to write to the cell itself — that is, it technically satisfies all the criteria of a
cellular automaton (Lesson 9). The difference is more one of emphasis and disciplinary origin (economic
geography vs. mathematics/physics) than of formal mechanism.

## Practice in DisSModel

The `disslucc-continuous` package implements exactly this Demand/Potential/Allocation tripod, with
`PotentialLinearRegression` using the same explanatory variables as the Amazon example.

---
→ Corresponding chapter in the [dissmodel-book](https://github.com/DisSModel/dissmodel-book):
[Ch. 7 — DisSLUCC](../part3-domain/ch07_disslucc.md)

> **Note:** Ch. 7/11 documents *engineering* validation (MAE/RMSE/kappa comparing DisSModel's output with
> TerraME's), but not the general scientific methodology of temporal calibration/validation nor Costanza's
> multiscale goodness-of-fit — worth keeping that part as separate theoretical content.
