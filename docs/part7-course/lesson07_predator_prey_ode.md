# Lesson 7 — Predator-Prey (differential equations)

*Part VII — TerraME Compatibility Course*

## "Nature, red in tooth and claw"

One of the most studied ecological relationships: one species uses another as a food resource. The most
famous historical example comes from the fur-purchase records of the Hudson's Bay Company in Canada —
decades of data on hare (prey) and lynx (predator) populations showing **regular periodicity**, with the
lynx population peak consistently lagging just after the hare peak.

## The two rules that define the system

1. The principal cause of death among prey is being eaten by a predator.
2. The birth and survival rates of predators depend on the availability of food — that is, on the prey
   population.

A third principle gives mathematical form to the encounter between the two species: **two species
encounter each other at a rate proportional to both populations** (the more prey and the more predators,
the more encounters — and the more predation events).

## Generic model

```
dx/dt = f(x) − h(x, y)
dy/dt = e·h(x, y) − g(y)
```

- `f(x)`: prey growth term
- `g(y)`: predator mortality term
- `h(x, y)`: predation term (depends on both populations)
- `e`: prey-into-predator biomass conversion coefficient

## The Lotka-Volterra model

The classic, simplest form of this generic model:

```
dx/dt = r·x − a·x·y
dy/dt = e·a·x·y − m·y      (with b = e·a)
```

- `r`: prey growth rate (Malthus law — exponential growth in the absence of predators)
- `m`: predator natural mortality rate
- `a`, `b`: predation coefficients (`b = e·a`)

**Numerical example**: rabbits and wolves, with `r = 0.08`, `m = 0.02`, `b = 0.00002`, `a = 0.001`, `t` in
years, 40 wolves and 1000 rabbits initially. The result is periodic oscillations in both populations,
replicating the pattern observed in the historical hare/lynx data.

## Phase plane, trajectories, and equilibrium point

Besides the traditional chart (each population vs. time), this system has a particularly revealing
visualization: the **phase plane** — one population plotted against the other, instead of against time.

- **Phase trajectory**: the path traced out by the solution `(R, W)` (Rabbits, Wolves) as time goes by —
  in Lotka-Volterra, this trajectory forms a closed loop (a periodic orbit).
- **Equilibrium point**: the point that sits inside all the solution curves — in the numerical example,
  `(1000, 80)` — corresponding to the stationary solution `R = 1000, W = 80`, where the two populations do
  not change.

## Practice in DisSModel

The `Predator-Prey (Lotka-Volterra)` model implements exactly these two coupled equations.

## TerraME vs DisSModel: side-by-side code

**TerraME** (`sysdyn/lua/PredatorPrey.lua`) — parameters use `Choice{min=..., max=..., default=...}` to
expose bounds in TerraME's graphical observer:

```lua
PredatorPrey = Model{
    predator = Choice{min = 1, default = 40},
    prey = Choice{min = 1, default = 1000},
    preyGrowth = Choice{min = 0.000001, max = 1, default = 0.08},
    preyDeathPred = Choice{min = 0.000001, max = 0.5, default = 0.001},
    predDeath = Choice{min = 0.000001, max = 0.5, default = 0.02},
    predGrowthKills = Choice{min = 0, max = 0.5, default = 0.00002},
    execute = function(model)
        model.prey = model.prey + model.preyGrowth * model.prey
                        - model.preyDeathPred * model.prey * model.predator
        model.predator = model.predator - model.predDeath * model.predator
                      + model.predGrowthKills * model.prey * model.predator
    end
}
```

**DisSModel** (`dissmodel-sysdyn/src/.../predatorprey.py`):

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
        self.prey += (
            self.prey_growth * self.prey
            - self.prey_death_pred * self.prey * self.predator
        )
        self.predator += (
            self.pred_growth_kills * self.prey * self.predator
            - self.pred_death * self.predator
        )
```

Note that the default values of all 6 parameters (`predator=40`, `prey=1000`, `prey_growth=0.08`,
`prey_death_pred=0.001`, `pred_death=0.02`, `pred_growth_kills=0.00002`) were preserved exactly — the same
numerical scenario as the original slide (rabbits and wolves), only renaming the variables to
`snake_case` (`preyDeathPred` → `prey_death_pred`) instead of `camelCase`.

---
→ Corresponding chapter in the [dissmodel-book](https://github.com/DisSModel/dissmodel-book):
[Ch. 4 — System Dynamics](../part2-paradigms/ch04_sysdyn.md)
(`Predator-Prey (Lotka-Volterra)` model)

> **Note:** the book documents `@track_plot` for time-series charts, but not the phase-plane visualization
> (one population vs. the other) — to reproduce the rabbits×wolves chart from the slide, you would need to
> build the scatter/line plot manually from the model's output data.
