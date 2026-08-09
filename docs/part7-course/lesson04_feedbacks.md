# Lesson 4 — Feedbacks

*Part VII — TerraME Compatibility Course*

## What is a feedback

Feedback is how a system affects itself: part of what leaves a system (outflow) loops back to influence
what enters it (inflow), closing a causal loop. Examples cited in the course: population growth (births
increase the population, which in turn determines how many future births occur, via fertility), albedo
(ice reflects energy, which cools things down and generates more ice), and the water dam (rain fills the
dam, which generates energy, which supplies the city, whose growth increases consumption — a causal loop
with a delay).

An important point: **the information delivered by a feedback can only affect future behavior** — never
the present, because it takes one time step (`dt`) for the effect to propagate back.

## Type 1: Balancing feedback

Also called negative, self-correcting, discrepancy-reducing, regenerative. It is an equilibrating or
"goal-seeking" structure — the system moves toward a reference value and stabilizes there.

**Example: coffee cooling/warming**

```
temperature(t) = temperature(t − dt) − flow × dt
flow = discrepancy × 10%
discrepancy = coffee temperature − room temperature
```

- Initial stock: 80 °C, 20 °C, or 5 °C (hot coffee cooling, coffee already at equilibrium, or cold coffee
  warming up)
- Room temperature: 20 °C
- `dt` = 1 minute, 20-minute simulation

Notice that `flow` is proportional to the distance from equilibrium (the discrepancy) — which is why the
system always converges, regardless of whether it starts above or below the reference value.

## Type 2: Reinforcing feedback

Also called positive, self-reinforcing, discrepancy-enhancing, degenerative. Instead of seeking an
equilibrium, it amplifies deviations — self-reinforcing behavior that leads to growth (or collapse).

**Example: population growth**

```
population(t) = population(t − dt) + growth × dt
growth = population × rate
```

- Initial stocks: 60 or 20
- Inflow rates tested: 50% and 90%
- `dt` = 1 year, 7-year simulation

A key point: **reinforcing feedbacks have limits!** No population growth is exponential forever — at some
point another factor (resources, space, another balancing feedback) comes into play. A good exercise is
to simulate what happens if the growth rate decreases by 20% every year, for 30 years — the system stops
being purely reinforcing and starts behaving like an S-curve.

## Practice in DisSModel

`Coffee Cooling` implements the balancing-feedback example with the same discrepancy × time-constant
reasoning; reinforcing population growth can be built on top of the base `Model` following the same
equation.

## TerraME vs DisSModel: side-by-side code

**TerraME** (`sysdyn/lua/Coffee.lua`):

```lua
Coffee = Model{
    temperature = 80, roomTemperature = 20, finalTime = 20,
    execute = function(model)
        local difference = model.temperature - model.roomTemperature
        model.temperature = model.temperature - difference * 0.1
    end,
    init = function(model)
        model.chart = Chart{target = model, select = {"temperature"}}
        model.timer = Timer{Event{action = model}, Event{action = model.chart}}
    end
}
```

**DisSModel** (`dissmodel-sysdyn/src/.../cofee.py`):

```python
@track_plot("Temperature", "blue")
class Coffee(Model):
    def setup(self, temperature=80.0, room_temperature=20.0, cooling_rate=0.1):
        self.temperature = temperature
        self.room_temperature = room_temperature
        self.cooling_rate = cooling_rate

    def execute(self):
        self.temperature -= self.cooling_rate * (self.temperature - self.room_temperature)
```

Newton's cooling equation (`temperature -= rate × discrepancy`) is reproduced term by term — the Lua
constant `0.1` becomes the named parameter `cooling_rate` in Python, easier to expose in a UI or notebook
without touching the model's code.

---
→ Corresponding chapter in the [dissmodel-book](https://github.com/DisSModel/dissmodel-book):
[Ch. 4 — System Dynamics](../part2-paradigms/ch04_sysdyn.md)
(`Coffee Cooling` model)

> **Note:** Meadows' *balancing/reinforcing* vocabulary used in this lesson does not appear in the book —
> it documents the ready-made model, not the theoretical classification of feedback types.
