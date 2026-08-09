# Lesson 13 — Agent-based Modeling

*Part VII — TerraME Compatibility Course*

## Many names, one idea

The same paradigm appears in the literature under several names: Agent-based Modelling (ABM), Agent-based
Modelling and Simulation (ABMS), Multi-agent Systems (MAS), Individual-based Modelling (IBM). It is used
in economics, sociology, archaeology, ecology, linguistics, political science, and other fields.

## What is ABM

A **bottom-up** approach to building complex systems, through the dynamic interaction of agents. An
**agent** is any actor within an environment — any entity capable of affecting itself, the environment,
and other agents. Advantages of this approach: flexibility, a natural way of representing the world
(instead of aggregate equations, individuals are represented directly), and **emergence** — macro
patterns that arise from local interaction, without being explicitly programmed.

## The four types of agents (Couclelis)

Helen Couclelis (UCSB) proposes classifying ABM applications by crossing two axes — natural/artificial
agent × natural/artificial environment:

| | Natural environment | Artificial environment |
|---|---|---|
| **Natural agent** | Behavioral experiments | Descriptive model |
| **Artificial agent** | Engineering applications | e-science |

This classification helps place where a specific model sits: `PredatorPreyModel` (Lesson 14), for
example, falls under "descriptive model" (artificial agents representing real animals, in a simplified
artificial environment).

## Representations in an agent model (Gilbert)

An agent has a representation of the environment, a goal, perception and action capabilities, and
communication (with the environment and with other agents) — the perceive → decide → act cycle is the
core of any agent.

## Why is ABM interesting (Gilbert, 2006)

- **Structure**: the system's structure emerges from agent interaction — this can be modeled directly,
  instead of assumed a priori.
- **Agency**: agents have goals, beliefs, and act — also directly modelable.
- **Dynamics**: things change, develop, evolve; agents move (in space and social position) and learn.

## Qualitative or quantitative?

Agent-based models can handle all kinds of data: quantitative attributes (age, size of an organization),
qualitative ordinal or categorical data (ethnicity), relational data (I am linked to him and to her), and
even vague data (A sends B a message about one time in three). This is an advantage over System Dynamics,
which works almost exclusively with aggregate continuous quantities.

## First practical models

1. **A single agent walking randomly**: a single agent moves randomly in a 30×30 cellular space.
2. **Society of moving agents**: 20 agents move randomly in the same 30×30 space, with the restriction
   that there cannot be more than one agent per cell.
3. **Agent-based SIR**: re-implement the SIR model from Lesson 5, but now with discrete individuals
   instead of continuous stocks — each agent has a state (susceptible/infected/recovered) and infection
   spreads by contact/neighborhood instead of an aggregate equation.

## Practice in DisSModel

`RandomWalkModel` directly covers exercises 1 and 2 (single agent / society of 20 agents). Exercise 3
(agent-based SIR) has no ready-made implementation yet — it would be built by combining `AgentModel` with
neighborhood-based contagion logic, in the same spirit as the `PredatorPreyModel` from the next lesson.

## TerraME vs DisSModel: side-by-side code

**TerraME** (`logo/lua/SingleAgent.lua`) — the agent lives in a discrete `CellularSpace`; `walkToEmpty()`
only moves to an empty neighboring cell (no two agents can occupy the same cell):

```lua
SingleAgent = Model{
    quantity = 1, dim = 15, finalTime = 100,
    init = function(model)
        model.cs = CellularSpace{xdim = model.dim}
        model.cs:createNeighborhood()
        model.agent = Agent{
            execute = function(self) self:walkToEmpty() end
        }
        model.soc = Society{instance = model.agent, quantity = model.quantity}
        model.env = Environment{model.cs, model.soc}
        model.env:createPlacement{}
        model.timer = Timer{Event{action = model.soc}, Event{action = model.map}}
    end
}
```

**DisSModel** (`dissmodel-abm/src/.../random_walk.py`) — agents are points (`Point`) in continuous space,
bounded by `bounds`, and the displacement is a continuous random offset (not a discrete cell swap):

```python
class RandomWalkModel(AgentModel):
    def setup(self, step_size: float = 1.0,
              bounds: tuple[float, float, float, float] = (0, 0, 100, 100)) -> None:
        self.step_size = step_size
        self.bounds = bounds

    def execute(self) -> None:
        self.society.walk_all(step_size=self.step_size, bounds=self.bounds)
```

Worth noting a real difference in the space model: `SingleAgent.lua` uses a discrete grid of cells
(`walkToEmpty()`, no two occupants per cell), while `RandomWalkModel` uses continuous geometric space
(`Point` + `bounds`), without that per-cell exclusivity restriction. To exactly reproduce the original
course's "society of 20 agents, no more than one per cell" exercise, you would need either to use an
equivalent discrete grid from `dissmodel-ca`, or to add a collision check to `RandomWalkModel`.

---
→ Corresponding chapter in the [dissmodel-book](https://github.com/DisSModel/dissmodel-book):
[Ch. 6 — Agent-Based Modeling](../part2-paradigms/ch06_abm.md)
(`RandomWalkModel` model)

> **Note:** Couclelis' "four types of agents" framework is modelling-science theory — deliberately outside
> the book's technical scope, but it was part of the original course's conceptual motivation and is worth
> keeping here.
