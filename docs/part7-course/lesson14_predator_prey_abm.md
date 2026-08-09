# Lesson 14 — Predator-Prey (agents)

*Part VII — TerraME Compatibility Course*

## From differential equation to individual

Lesson 7 modeled rabbits and wolves as two continuous stocks governed by Lotka-Volterra. This lesson
rebuilds the same phenomenon — bottom-up — as discrete individuals that move, eat, reproduce, and die in
a cellular space. Translating the parameters `r`, `m`, `a`, `b` (prey growth, predator mortality,
predation, biomass conversion) into individual rules is the heart of the lesson:

| ODE parameter | Equivalent agent rule |
|---|---|
| `r` — prey growth (Malthus) | eating pasture increases energy by 20; with energy ≥ 30, reproduces if a random neighbor is empty, and energy is cut in half |
| `m` — predator natural mortality | dies when energy reaches ≤ 0 (rule that applies to both species) |
| `a` — prey mortality by predation | the predator searches a random neighboring cell; if there's a prey there, it is killed, and the predator gains 20% of the prey's energy |
| `b` — growth coefficient from predation | with energy ≥ 50, the predator reproduces if a random neighbor is empty, cutting its energy in half |

## Scenario parameters

- 30×30 space, **Moore** neighborhood (8 neighbors)
- Soil needs 4 time steps to become pasture again after being "consumed"
- Initial state: space entirely covered with pasture
- 20 wolves and 20 rabbits initially, each agent starts with 40 units of energy
- Rabbits consume 1 unit of energy per time step; wolves consume 4
- Simulation runs until time 500

## What changes relative to the aggregate model

The lesson's discussion raises important points about the trade-off between the two paradigms:

- **Discrete quantities of individuals**, instead of continuous real numbers — small populations can go
  extinct abruptly (an effect a continuous ODE never captures exactly).
- **Considerably more parameters** — energy, reproduction thresholds, per-species metabolic cost, pasture
  regeneration time, neighborhood size — against just 4 parameters (`r`, `m`, `a`, `b`) in Lotka-Volterra.
- **More complex behavior** — the aggregate outcome emerges from individual decisions and the geometry of
  space (who is near whom), not from a closed-form formula.
- **Harder to calibrate** — with more parameters and more randomness (multiple runs can give quite
  different results), calibrating against real data is considerably more work than fitting 4 ODE
  coefficients.
- On the upside: **many more possibilities** — spatial heterogeneity, individual variation, and questions
  that simply don't make sense in the aggregate model (e.g. "how long does a newborn rabbit survive near
  a high density of wolves?") become answerable.

## Practice in DisSModel

`PredatorPreyModel` (`dissmodel-abm`) implements this logic almost line by line: movement (`walk()`),
energy loss per step, predation by neighborhood/radius, death from zero energy, and reproduction by energy
threshold with a halving cut.

## TerraME vs DisSModel: side-by-side code

**TerraME** (`logo/lua/PredatorPrey.lua`) — rabbits eat `pasture`, turning the cell into `soil` (which
regenerates after 4 steps); predation happens through cell neighborhood (`getNeighborhood():sample()`):

```lua
model.rabbit = Agent{
    energy = 40, name = "rabbit",
    eat = function(self)
        if self:getCell().state == "pasture" then
            self:getCell().state = "soil"
            self.energy = self.energy + 20
        end
    end,
    execute = function(self)
        local cell = self:getCell():getNeighborhood():sample()
        if cell:isEmpty() then self:move(cell) end
        self.energy = self.energy - 1
        self:eat()
        cell = self:getCell():getNeighborhood():sample()
        if self.energy >= 30 and cell:isEmpty() then
            local child = self:reproduce()
            child:move(cell)
            self.energy = self.energy / 2
            child.energy = self.energy
        end
        if self.energy < 0 then self:die() end
    end
}

model.wolf = Agent{
    energy = 40, name = "wolf",
    execute = function(self)
        local cell = self:getCell():getNeighborhood():sample()
        if cell:isEmpty() then
            if self.energy >= 50 then
                local child = self:reproduce()
                child:move(cell)
                self.energy = self.energy / 2
            else
                self:move(cell)
            end
        elseif cell:getAgent().name == "rabbit" then
            local prey = cell:getAgent()
            self.energy = self.energy + prey.energy * 0.2
            prey:die()
        end
        self.energy = self.energy - 4
        if self.energy < 0 then self:die() end
    end
}
```

**DisSModel** (`dissmodel-abm/src/.../predator_prey.py`) — agents live in continuous space
(`geometry: Point`), predation uses `agent.neighbors(eat_radius)` (a radius-based search, not a
neighboring cell), and the five phases (movement, metabolism, predation, death, reproduction) are
explicitly separated instead of interleaved inside each agent type's `execute()`:

```python
class PredatorPreyModel(AgentModel):
    def execute(self) -> None:
        society = self.society

        # 1. Movement
        for agent in society:
            agent.walk(step_size=self.step_size, bounds=self.bounds)

        # 2. Metabolism (sheep graze, everyone loses energy)
        for agent in society:
            if self.graze_gain and agent.kind == "sheep":
                agent.energy += self.graze_gain
            agent.energy -= self.energy_loss

        # 3. Predation: wolves eat nearby sheep
        self._predation_step(society)

        # 4. Death
        society.remove_if(lambda agent: agent.energy <= 0)

        # 5. Reproduction
        for agent in society.select(lambda a: a.energy >= self.reproduce_threshold):
            agent.reproduce(energy=self.reproduce_threshold / 2.0)
```

**Real differences worth highlighting** (this is not a perfect 1:1 translation):

- **Space model**: TerraME uses a discrete `CellularSpace` (pasture that turns into soil for 4 steps,
  each cell occupied by at most one agent); the Python version uses continuous space with `eat_radius`,
  without the pasture-regeneration mechanism — there is a simpler `graze_gain` (fixed energy gain per
  step), but not the original slide's pasture↔soil spatial dynamics.
- **Reproduction thresholds**: in TerraME, rabbits reproduce at energy ≥ 30 and wolves at energy ≥ 50
  (different thresholds per species, as in the Lesson 14 slide); in the current `PredatorPreyModel`,
  there is a single shared `reproduce_threshold` (default 15) — to exactly reproduce the slide's numbers,
  this parameter would need to be differentiated by `kind`.
- **Energy gain from predation**: in TerraME the wolf gains 20% of the prey's energy (`prey.energy * 0.2`,
  variable); in Python the gain is a fixed value `energy_gain` (default 5.0), not proportional to the
  prey's energy at the moment of capture.

In other words: the **architecture** of the five rules (move, lose energy, eat, die, reproduce) is the
same, but the **exact numeric parameters** from the original course exercise (30/50/0.2×) would need to be
explicitly adjusted when instantiating `PredatorPreyModel` to faithfully reproduce the slide's scenario.

---
→ Corresponding chapter in the [dissmodel-book](https://github.com/DisSModel/dissmodel-book):
[Ch. 6 — Agent-Based Modeling](../part2-paradigms/ch06_abm.md)
(`PredatorPreyModel` model)
