# Lesson 5 — Epidemics

*Part VII — TerraME Compatibility Course*

## What is an epidemic

In epidemiology, an epidemic occurs when new cases of a certain disease, in a given population and time
period, substantially exceed what would be expected based on recent experience.

## The SIR model

The classic model for epidemic dynamics divides the population into three stocks that transform into one
another over time:

- **S**usceptible — people who can be infected
- **I**nfected — infected people, who can transmit the disease
- **R**ecovered — people who have already had the disease and can no longer be infected (or infect
  others)

The flow is always one-directional: S → I → R. The parameters that control the speed of these
transitions:

- Duration of the infection (how long someone remains infectious before "recovering")
- Number of contacts each infected person has per unit of time
- Fraction of those contacts sufficient to cause infection (but only if the contact is with someone
  susceptible — an infected person who only meets recovered people generates no new infections)

## Example parameters

- Initial stocks: susceptible = 9998, infected = 2, recovered = 0
- `dt` = 1 week, 30-week simulation
- Duration of an infection: 2 weeks
- Each infected person contacts 6 people per week
- 25% of those contacts are enough to cause infection

## Scenarios to explore

- Vary the duration of the infection: 2, 4, 8 weeks
- Vary the fraction of contacts that causes infection: 50%, 25%, 12.5%
- Simulate an educational campaign: cut the number of contacts in half when infections reach 2000 or
  1000 cases

These scenarios show in practice how small changes to transmission parameters produce dramatically
different epidemic curves — the same logic behind concepts like "flattening the curve."

## Practice in DisSModel

`SIR` is literally the central teaching example of the `dissmodel-sysdyn` package, with the same three
stocks and the same contact × infection-probability logic.

## TerraME vs DisSModel: side-by-side code

**TerraME** (`sysdyn/lua/SIR.lua`) — the contagion logic lives inside a `Timer` `Event`, and there is a
second `Event` implementing the educational campaign (halves contacts once a threshold is reached):

```lua
SIR = Model{
    susceptible = 9998, infected = 2, recovered = 0,
    duration = 2, finalTime = 30, contacts = 6, probability = 0.25,
    init = function(model)
        model.timer = Timer{
            Event{action = function()
                local proportion = model.susceptible /
                    (model.susceptible + model.infected + model.recovered)
                local newInfected = model.infected * model.contacts * model.probability * proportion
                local newRecovered = model.infected / model.duration
                model.susceptible = model.susceptible - newInfected
                model.recovered = model.recovered + newRecovered
                model.infected = model.infected + newInfected - newRecovered
            end},
            Event{action = model.chart}
        }
    end
}
```

**DisSModel** (`dissmodel-sysdyn/src/.../sir.py`) — the same force-of-infection equation, inside
`execute()`:

```python
@track_plot("Susceptible", "green")
@track_plot("Infected", "red")
@track_plot("Recovered", "blue")
class SIR(Model):
    def setup(self, susceptible=9998, infected=2, recovered=0,
              duration=2, contacts=6, probability=0.25):
        self.susceptible, self.infected, self.recovered = susceptible, infected, recovered
        self.duration, self.contacts, self.probability = duration, contacts, probability

    def execute(self):
        total = self.susceptible + self.infected + self.recovered
        alpha = self.contacts * self.probability
        new_infected  = self.infected * alpha * (self.susceptible / total)
        new_recovered = self.infected / self.duration
        self.susceptible -= new_infected
        self.infected    += new_infected - new_recovered
        self.recovered   += new_recovered
```

The translation is almost mechanical: `proportion` becomes `self.susceptible / total`, `contacts ×
probability` becomes the named variable `alpha`. The only piece left out is the second `Event` for the
educational campaign (halving contacts once an infected threshold is reached) — it's not implemented in
the current Python version, but it would be a one-line condition inside `execute()` itself.

---
→ Corresponding chapter in the [dissmodel-book](https://github.com/DisSModel/dissmodel-book):
[Ch. 4 — System Dynamics](../part2-paradigms/ch04_sysdyn.md)
(`SIR` model)
