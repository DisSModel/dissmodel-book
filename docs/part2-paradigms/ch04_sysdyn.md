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

## 4.4 Theory: stocks and flows

Modelling a system means making explicit which measurable parts of reality it has, and how they affect
each other. System Dynamics (Meadows, 2008) represents any such part as one of two elements:

- **Stock**: a measurable quantity accumulated at a given moment — water in a tub, a population, a
  temperature.
- **Flow**: the change in a stock over time — what comes in (inflow) and what goes out (outflow).

```
stock(t) = stock(t − dt) + inflow × dt − outflow × dt
```

Three consequences follow directly from this, and they explain the shape of every model in this chapter:

1. **There are two ways to increase a stock** — raise the inflow, or lower the outflow.
2. **Stocks act as delays or buffers** — they don't react instantaneously to flow changes, which gives
   the system inertia.
3. **Stocks decouple inflows from outflows** — a tub can be filling and draining at the same time; the
   stock records the net balance of two independent forces.

### `Tub` — TerraME vs DisSModel

`sysdyn/lua/Tub.lua` uses an explicit `Timer`/`Event` loop, with the periodic inflow as a second `Event`:

```lua
Tub = Model{
    water = 40, outFlow = 5, inFlow = 0, finalTime = 8,
    execute = function(model)
        model.water = model.water - model.outFlow
        if model.water < 0 then model.water = 0 end
    end,
    init = function (model)
        model.timer = Timer{
            Event{action = model},
            Event{start = 10, period = 10, action = function()
                model.water = model.water + model.inFlow
            end},
        }
    end
}
```

`dissmodel_sysdyn.models.tub.Tub` reaches the same behaviour with no `Timer` at all — `Environment`
already calls `execute()` every step, and the periodic inflow is a step counter checked with `%`:

```python
@track_plot("Water", "blue")
class Tub(Model):
    def setup(self, water=40.0, out_flow=5.0, in_flow=0.0, in_period=10):
        self.water, self.out_flow = water, out_flow
        self.in_flow, self.in_period = in_flow, in_period
        self._step = 0

    def execute(self):
        self._step += 1
        self.water -= self.out_flow
        if self.water < 0.0:
            self.water = 0.0
        if self._step % self.in_period == 0:
            self.water += self.in_flow
```

The business logic is identical line by line. What disappears is TerraME's explicit `Timer`/`Event`
infrastructure — the `Environment`'s implicit `setup()`/`execute()` cycle absorbs it.

## 4.5 Feedbacks: balancing and reinforcing

A feedback is how a system affects itself: part of an outflow loops back to influence an inflow, closing
a causal loop. Because the effect always takes one `dt` to propagate, a feedback can only shape *future*
behaviour, never the present step.

**Balancing** (negative, self-correcting) feedback is goal-seeking — the system moves toward a reference
value and stabilizes there. `Coffee Cooling` is the canonical example:

```
temperature(t) = temperature(t − dt) − flow × dt
flow = discrepancy × 10%
discrepancy = coffee temperature − room temperature
```

The `flow` is proportional to the distance from equilibrium, so the system converges whether it starts
above or below the reference value.

**Reinforcing** (positive, self-amplifying) feedback has no such ceiling built in — it amplifies
deviations rather than correcting them, which is exactly how unconstrained population growth behaves:

```
population(t) = population(t − dt) + growth × dt
growth = population × rate
```

Reinforcing feedbacks always have limits in the real world — resources, space, or another balancing
feedback eventually intervenes; nothing grows exponentially forever.

### `Coffee` — TerraME vs DisSModel

```lua
Coffee = Model{
    temperature = 80, roomTemperature = 20, finalTime = 20,
    execute = function(model)
        local difference = model.temperature - model.roomTemperature
        model.temperature = model.temperature - difference * 0.1
    end,
}
```

```python
@track_plot("Temperature", "blue")
class Coffee(Model):
    def setup(self, temperature=80.0, room_temperature=20.0, cooling_rate=0.1):
        self.temperature, self.room_temperature = temperature, room_temperature
        self.cooling_rate = cooling_rate

    def execute(self):
        self.temperature -= self.cooling_rate * (self.temperature - self.room_temperature)
```

Newton's cooling equation is reproduced term for term — the Lua constant `0.1` becomes the named
`cooling_rate` parameter, easier to expose in a Streamlit sidebar without touching model code.

## 4.6 Epidemiology: the SIR model

The `SIR` model divides a population into three stocks that transform into one another: **S**usceptible →
**I**nfected → **R**ecovered. The transition rate depends on the infection's duration, the number of
contacts an infected individual has per unit time, and the fraction of those contacts sufficient to cause
infection — but only when the contact is with someone susceptible.

### `SIR` — TerraME vs DisSModel

`sysdyn/lua/SIR.lua` computes the force of infection inside a `Timer` `Event`:

```lua
SIR = Model{
    susceptible = 9998, infected = 2, recovered = 0,
    duration = 2, contacts = 6, probability = 0.25,
    init = function(model)
        model.timer = Timer{ Event{action = function()
            local proportion = model.susceptible /
                (model.susceptible + model.infected + model.recovered)
            local newInfected = model.infected * model.contacts * model.probability * proportion
            local newRecovered = model.infected / model.duration
            model.susceptible = model.susceptible - newInfected
            model.recovered   = model.recovered + newRecovered
            model.infected    = model.infected + newInfected - newRecovered
        end}}
    end
}
```

`dissmodel_sysdyn.models.sir.SIR` reproduces the same equation inside `execute()`:

```python
@track_plot("Susceptible", "green")
@track_plot("Infected",    "red")
@track_plot("Recovered",   "blue")
class SIR(Model):
    def setup(self, susceptible=9998, infected=2, recovered=0,
              duration=2, contacts=6, probability=0.25):
        self.susceptible, self.infected, self.recovered = susceptible, infected, recovered
        self.duration, self.contacts, self.probability = duration, contacts, probability

    def execute(self):
        total   = self.susceptible + self.infected + self.recovered
        alpha   = self.contacts * self.probability
        new_inf = self.infected * alpha * (self.susceptible / total)
        new_rec = self.infected / self.duration
        self.susceptible -= new_inf
        self.infected    += new_inf - new_rec
        self.recovered   += new_rec
```

**Known gap**: the original `SIR.lua` has a second `Event` implementing an educational campaign — contacts
are cut in half once infections cross a threshold. That branch is **not** ported to `sir.py` — anyone
relying on it needs to add the threshold check to `execute()` themselves.

## 4.7 Chaos and sensitivity to initial conditions

Edward Lorenz discovered deterministic chaos by accident in 1961, re-running a weather simulation with a
rounded input (`0.506` instead of `0.506127`) and watching the result diverge completely within a few
steps:

> "When the present determines the future, but the approximate present does not approximately determine
> the future."

Chaos requires three ingredients: **high sensitivity to initial conditions** (small differences grow
exponentially), **non-linearity** (no non-linear term, no chaos), and **determinism** (the same exact
state always leads to the same future — yet is unpredictable in practice, because initial conditions are
never known with infinite precision).

`ChaoticGrowth` is the logistic map; `Lorenz` is the three-equation convection model behind the
"butterfly" attractor:

```
growth(t) = rate × growth(t − dt) × (1 − growth(t − dt))

dx/dt = σ(y − x)
dy/dt = x(ρ − z) − y
dz/dt = xy − βz
```

### `ChaoticGrowth`/`Lorenz` — TerraME vs DisSModel

```lua
-- ChaoticGrowth.lua
execute = function(model)
    model.pop = model.rate * model.pop * (1 - model.pop)
end
```

```python
# chaotic_growth.py
def execute(self) -> None:
    self.pop = self.rate * self.pop * (1.0 - self.pop)
```

```lua
-- Lorenz.lua — inside a Timer Event
local nx = model.sigma * (model.y - model.x)
local ny = model.x * (model.rho - model.z) - model.y
local nz = model.x * model.y - model.beta * model.z
model.x = model.x + model.delta * nx
model.y = model.y + model.delta * ny
model.z = model.z + model.delta * nz
```

```python
# lorenz.py
def execute(self) -> None:
    dx = self.sigma * (self.y - self.x)
    dy = self.x * (self.rho - self.z) - self.y
    dz = self.x * self.y - self.beta * self.z
    self.x += self.delta * dx
    self.y += self.delta * dy
    self.z += self.delta * dz
```

Both default parameter sets (`rate=4.0`; `sigma=10, rho=28, beta=8/3, delta=0.01`) were preserved exactly
from the original TerraME models.

## 4.8 Predator-Prey and phase-plane analysis

The Lotka-Volterra model formalizes two rules: prey mostly die by predation, and predator birth/survival
depends on prey availability. A third principle gives the interaction its mathematical shape — two
species meet at a rate proportional to *both* populations:

```
dx/dt = r·x − a·x·y
dy/dt = e·a·x·y − m·y
```

Besides the usual population-vs-time chart, this system has a particularly revealing second
visualization: the **phase plane** — one population plotted against the other instead of against time. In
Lotka-Volterra the resulting **phase trajectory** is a closed loop, and the point sitting inside every
loop is the **equilibrium point** (`R=1000, W=80` for the classic rabbits/wolves parameterization) — the
stationary solution where neither population changes. `@track_plot`, documented in
[Chapter 13](../part6-api/ch13_api_reference.md#138-dissmodelvisualization-chart-map-rastermap), only
produces
time-series charts; a phase-plane plot has to be built from the model's tracked output manually.

### `PredatorPrey` — TerraME vs DisSModel

`sysdyn/lua/PredatorPrey.lua` uses `Choice{min=..., max=..., default=...}` to expose parameter bounds to
TerraME's graphical observer:

```lua
PredatorPrey = Model{
    predator = Choice{min = 1, default = 40},
    prey     = Choice{min = 1, default = 1000},
    preyGrowth      = Choice{default = 0.08},
    preyDeathPred   = Choice{default = 0.001},
    predDeath       = Choice{default = 0.02},
    predGrowthKills = Choice{default = 0.00002},
    execute = function(model)
        model.prey = model.prey + model.preyGrowth * model.prey
                        - model.preyDeathPred * model.prey * model.predator
        model.predator = model.predator - model.predDeath * model.predator
                      + model.predGrowthKills * model.prey * model.predator
    end
}
```

```python
@track_plot("Prey", "green")
@track_plot("Predator", "red")
class PredatorPrey(Model):
    def setup(self, predator=40.0, prey=1000.0, prey_growth=0.08,
              prey_death_pred=0.001, pred_death=0.02, pred_growth_kills=0.00002):
        self.predator, self.prey = predator, prey
        self.prey_growth, self.prey_death_pred = prey_growth, prey_death_pred
        self.pred_death, self.pred_growth_kills = pred_death, pred_growth_kills

    def execute(self) -> None:
        self.prey += (self.prey_growth * self.prey
                       - self.prey_death_pred * self.prey * self.predator)
        self.predator += (self.pred_growth_kills * self.prey * self.predator
                           - self.pred_death * self.predator)
```

All six default parameter values were preserved exactly — same numerical scenario (40 wolves, 1000
rabbits), only `camelCase` renamed to `snake_case`.

## 4.9 Daisyworld and the Gaia hypothesis

James Lovelock, analysing the Martian atmosphere at NASA, asked why Earth's surface temperature has
stayed within a narrow range for 3.6 billion years while solar output rose ~25% over that time. His
answer — the Gaia hypothesis — is that the biosphere maintains a homeostatic feedback loop with the
atmosphere: biotic factors regulate abiotic ones, which in turn constrain biological possibilities. There
is no intent involved, only feedback.

`Daisyworld` is the minimal model of this idea: black daisies (low albedo, warm the planet) and white
daisies (high albedo, cool it) compete for bare soil. When the planet is cold, black daisies dominate and
warm it; when it's hot, white daisies dominate and cool it — the result is a planet whose temperature
stays close to the daisies' optimum across a wide range of solar luminosity, with no central controller.

```
dαb/dt = αb (αg · β(Tb) − γ)          S·L·(1 − A) = σ·T⁴
```

### `Daisyworld` — TerraME vs DisSModel

*Note: in TerraME, `Daisyworld.lua` lives in the `ca` repository, not `sysdyn` — package boundaries were
looser than DisSModel's.*

```lua
local function daisyGrowthRate(tempK)
    local temp = tempK - 273
    if temp > 5.0 and temp < 40.0 then
        return 1 - 0.003265 * (22.5 - temp)^2
    end
    return 0.0
end
```

```python
def _daisy_growth_rate(temp_k: float) -> float:
    temp_c = temp_k - 273.0
    if 5.0 < temp_c < 40.0:
        return max(0.0, 1.0 - 0.003265 * (22.5 - temp_c) ** 2)
    return 0.0
```

The physical constants (Stefan-Boltzmann `σ = 5.67e-8`, solar flux, heat-transfer coefficient `Q =
2.06e9`) and the growth-curve formula — including the same `0.003265` "magic number" — were copied
exactly from the Lua original.



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
