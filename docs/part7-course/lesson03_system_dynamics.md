# Lesson 3 — System Dynamics

*Part VII — TerraME Compatibility Course*

## How to model nature-society systems?

Modelling a system such as deforestation, for example, requires connecting knowledge from different
fields (ecology, economics, geography) and making the different conceptions of how these components
relate to each other explicit. A **system** is a group of different components that interact with each
other — the question that guides all modelling is: what are the parts, and how do they affect each other?

## Stocks and flows

System Dynamics (Meadows, 2008) represents any measurable part of reality as a combination of two
elements:

- **Stock**: a measurable element — energy, matter, or information accumulated at a given moment (e.g.
  water in a tub, population, temperature).
- **Flow**: the change in a stock over time — what comes in (inflow) and what goes out (outflow).

The generic stock equation is:

```
stock(t) = stock(t − dt) + inflow × dt − outflow × dt
```

## Example: water in the tub

- Initial stock: 40 gallons
- `outflow` = 5 gal/min
- `dt` = 1 minute, 8-minute simulation
- In a second version, an `inflow` of 40 gallons is added every 10 minutes

Running these two versions (outflow only vs. inflow+outflow) makes the role of stocks in the system
visible.

## Three important conclusions

1. **There are two ways to increase a stock**: increase the inflow or decrease the outflow.
2. **Stocks act as delays or buffers** — they don't react instantaneously to flow changes, which gives the
   system inertia.
3. **Stocks allow inflows and outflows to be decoupled** — the tub can be filling and draining at the same
   time, and the stock is what records the net balance of these two independent forces.

These three conclusions are the conceptual foundation for understanding feedbacks (next lesson): a
feedback is nothing more than a flow whose rate depends on the stock's own value.

## Practice in DisSModel

The `Tub` example in `dissmodel-sysdyn` implements exactly this stock/flow model, with `setup()` defining
the initial stock and `execute()` applying inflow/outflow at every step.

## TerraME vs DisSModel: side-by-side code

**TerraME** (`sysdyn/lua/Tub.lua`) — the event loop is explicit, via `Timer`/`Event`, and the chart is
wired manually with a `Chart` object:

```lua
Tub = Model{
    water = 40, outFlow = 5, inFlow = 0, finalTime = 8,
    execute = function(model)
        model.water = model.water - model.outFlow
        if model.water < 0 then model.water = 0 end
    end,
    init = function (model)
        model.chart = Chart{target = model, select = "water"}
        model.timer = Timer{
            Event{action = model},
            Event{start = 10, period = 10, action = function()
                model.water = model.water + model.inFlow
            end},
            Event{action = model.chart}
        }
    end
}
```

**DisSModel** (`dissmodel-sysdyn/src/.../tub.py`) — the same behavior in `setup()`/`execute()`, with no
explicit `Timer` (DisSModel's `Environment` already calls `execute()` every step), and the chart declared
as a decorator on the class:

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

The business logic (subtract `out_flow`, clamp to zero, add `in_flow` every `in_period` steps) is
identical line by line — the real difference is in the surrounding infrastructure: TerraME's explicit
`Timer`/`Event` becomes the `Environment`'s implicit `setup()`/`execute()` cycle, and the manual `Chart`
becomes a declarative decorator.

---
→ Corresponding chapter in the [dissmodel-book](https://github.com/DisSModel/dissmodel-book):
[Ch. 4 — System Dynamics](../part2-paradigms/ch04_sysdyn.md)
(`Tub` model)
