# Chapter 5: Agent-Based Modeling

*Part II — Simulation Paradigms*

Implemented by the [`dissmodel-abm`](https://github.com/DisSModel/dissmodel-abm) package.

## Learning objectives

- Understand the `Society`/`Agent` layer over the vector substrate
- Translate `Agent`/`Society` concepts from TerraME to `dissmodel-abm`
- Write an agent-based model without touching `self.gdf` directly

## 5.1 `Society`/`Agent`: a protective layer over the substrate

The whole point of this package is that model code never touches
`self.gdf` directly. `Society` owns no data of its own — it reads and
writes through the host model's `gdf` attribute, so `model.gdf` and
`model.society` are always the same data.

```python
for agent in self.society:
    if agent.energy <= 0:
        agent.die()
    elif agent.energy >= 15:
        agent.reproduce(energy=5.0)
```

instead of manipulating the GeoDataFrame directly with masks and
`pd.concat`.

## 5.2 Concept mapping: TerraME → dissmodel-abm

| TerraME (`Agent`/`Society`) | dissmodel-abm |
|---|---|
| `execute(self)` | `execute()` (Model lifecycle) |
| `init(self)` | `setup()` (Model lifecycle) |
| `Society` (collection of Agents) | `self.society` — object-oriented view over `self.gdf` |
| `Agent` | `self.society[idx]` — a proxy over one row |
| `placement` / `getCell()` | `agent.geometry` |
| `enter(cell)` | `agent.enter(x, y)` |
| `leave()` | `agent.leave()` |

*TODO: complete the table with `move(cell)`/`walk()` and other methods.*

## 5.3 A standalone, additive package

`dissmodel-abm` is a **standalone, additive package** — it does not modify
dissmodel's core. It depends on `dissmodel` as a regular package
dependency, exactly like `dissmodel-ca`. `Map`, `Chart`, `ModelExecutor`
keep working unmodified, whether or not a given model uses `self.society`.

## Exercises

*TODO: write exercises.*

## Summary

*TODO: summarize the chapter's key takeaways.*
