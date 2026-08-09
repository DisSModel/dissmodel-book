# Lesson 6 — Chaos

*Part VII — TerraME Compatibility Course*

## The butterfly effect

Edward Lorenz, a meteorologist, discovered deterministic chaos by accident in 1961, when he re-ran a
weather simulation with a rounded input value (0.506 instead of 0.506127) and watched the result diverge
completely from the original within a few steps. This gave rise to the informal definition of chaos:

> "When the present determines the future, but the approximate present does not approximately determine
> the future."

## Three ingredients of chaos

1. **High sensitivity to initial conditions** — small differences in the initial state grow exponentially
   over time.
2. **Non-linearity** — without non-linear terms in the equation, there is no chaos (linear systems are
   always predictable).
3. **Determinism** — and here lies the central paradox: the system is entirely deterministic (the exact
   same initial state always leads to the same future), yet it is unpredictable in practice, because we
   never know the initial conditions with infinite precision.

## Chaotic growth (logistic map)

```
growth(t) = rate × growth(t − dt) × (1 − growth(t − dt))
```

- Initial stock: `growth = 0.1`
- `dt` = 1 year, 300-year simulation
- Test `rate` = 1, 2, 3, 4

This is the classic logistic map: for low `rate`, the system converges to a fixed point; for intermediate
values, it oscillates between a few values (bifurcation); for `rate` near 4, the behavior becomes chaotic
— deterministic, but visually indistinguishable from random noise.

## The Lorenz system

Lorenz modeled a two-dimensional fluid layer uniformly heated from below and cooled from above (a
simplification of atmospheric convection):

```
dx/dt = σ(y − x)
dy/dt = x(ρ − z) − y
dz/dt = xy − βz
```

- `x`: proportional to the intensity of convective motion
- `y`: horizontal temperature variation
- `z`: vertical temperature variation
- `σ` (Prandtl number) = 10, `ρ` (Rayleigh number) = 28, `β` = 8/3
- Initial state: x = y = z = 1, `dt` = 0.01

Running this system produces the famous **Lorenz attractor**, shaped like a butterfly — the trajectory
never exactly repeats, but stays confined to a region of state space.

## Why this matters: weather forecasting

From the conclusion of Lorenz's own 1963 paper, applied to the atmosphere:

> "[...] prediction of the sufficiently distant future is impossible by any method, unless the present
> conditions are known exactly. In view of the inevitable inaccuracy and incompleteness of weather
> observations, precise very-long-range forecasting would seem to be non-existent."

This is the practical origin of the ~10–14 day limit on weather forecasting, even with supercomputers.

## Practice in DisSModel

`Chaotic Growth` implements the logistic map directly; `Lorenz Attractor` implements the three coupled
differential equations.

## TerraME vs DisSModel: side-by-side code

**Chaotic growth** — `sysdyn/lua/ChaoticGrowth.lua`:

```lua
ChaoticGrowth = Model{
    pop = 0.10, rate = 4.0, finalTime = 300,
    execute = function(model)
        model.pop = model.rate * model.pop * (1 - model.pop)
    end,
    init = function(model)
        model.chart = Chart{target = model, select = "pop"}
        model.timer = Timer{Event{action = model}, Event{action = model.chart}}
    end
}
```

`dissmodel-sysdyn/src/.../chaotic_growth.py`:

```python
@track_plot("Pop", "red")
class ChaoticGrowth(Model):
    def setup(self, pop: float = 0.1, rate: float = 4.0) -> None:
        self.pop, self.rate = pop, rate

    def execute(self) -> None:
        self.pop = self.rate * self.pop * (1.0 - self.pop)
```

The logistic-map line (`model.pop = model.rate * model.pop * (1 - model.pop)`) is reproduced character by
character, only swapping Lua's assignment syntax for Python's.

**Lorenz system** — `sysdyn/lua/Lorenz.lua`:

```lua
Lorenz = Model{
    x = 1.0, y = 1.0, z = 1.0,
    delta = 0.01, rho = 28.0, sigma = 10.0, beta = 8.0 / 3.0,
    finalTime = 10000,
    init = function(model)
        model.timer = Timer{
            Event{action = function()
                local nx = model.sigma * (model.y - model.x)
                local ny = model.x * (model.rho - model.z) - model.y
                local nz = model.x * model.y - model.beta * model.z
                model.x = model.x + model.delta * nx
                model.y = model.y + model.delta * ny
                model.z = model.z + model.delta * nz
            end},
            Event{action = model.chart1}, Event{action = model.chart2}
        }
    end
}
```

`dissmodel-sysdyn/src/.../lorenz.py`:

```python
@track_plot("X", "blue")
@track_plot("Y", "orange")
@track_plot("Z", "red")
class Lorenz(Model):
    def setup(self, x=1.0, y=1.0, z=1.0, delta=0.01, sigma=10.0, rho=28.0, beta=8.0/3.0):
        self.x, self.y, self.z = x, y, z
        self.delta, self.sigma, self.rho, self.beta = delta, sigma, rho, beta

    def execute(self) -> None:
        dx = self.sigma * (self.y - self.x)
        dy = self.x * (self.rho - self.z) - self.y
        dz = self.x * self.y - self.beta * self.z
        self.x += self.delta * dx
        self.y += self.delta * dy
        self.z += self.delta * dz
```

The three differential equations and the Euler integration method (`x += delta * dx`) are identical —
even the default parameter values (`sigma=10`, `rho=28`, `beta=8/3`, `delta=0.01`) were preserved exactly
from the original TerraME version.

---
→ Corresponding chapter in the [dissmodel-book](https://github.com/DisSModel/dissmodel-book):
[Ch. 4 — System Dynamics](../part2-paradigms/ch04_sysdyn.md)
(`Chaotic Growth` and `Lorenz Attractor` models)

> **Note:** the book documents the implemented equations, but not the historical motivation (Lorenz's
> accidental discovery) nor the formal definition of sensitivity to initial conditions — worth keeping
> that part narrated in the lesson.
