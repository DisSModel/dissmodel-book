# Lesson 10 — Fire in the Forest

*Part VII — TerraME Compatibility Course*

## The classic application example of cellular automata

After the theory in Lesson 9, this is the first complete model: simulating fire spread in a forest,
applying the six elements of a cellular automaton (grid, neighborhood, states, rules, initial state,
discrete time) to a concrete scenario.

## Version 1: three states

- 50×50 cell space
- Three states: **forest**, **burning**, **burned**
- Initial state: all cells start as forest, except a random initial cell in `burning`
- **Von Neumann** neighborhood (4 neighbors)
- Transition rules:
  - a `forest` cell becomes `burning` if a neighbor is `burning`
  - a `burning` cell becomes `burned` after one time step
- Simulate for 90 time steps

This model already shows the classic emergent behavior of cellular automata: a fire front spreads
radially from the initial point, following the topology of the Von Neumann neighborhood (which is why the
front tends to be diamond-shaped rather than circular).

## Version 2: four states

The second version adds a fourth, more realistic state:

- Same 50×50 space, same Von Neumann neighborhood, same 90 iterations
- Four states: **empty** (no fuel), **forest**, **burning**, **burned**
- Initial state: each cell is randomly drawn between `empty` and `forest`
- Same spread rules: forest catches fire if a neighbor is burning; a burning cell becomes burned after one
  step

Since not every cell has fuel (`forest`), the fire can stop spreading before consuming the whole space —
depending on the sampled forest density, the fire can go out early (low density, below the percolation
threshold) or consume nearly the whole connected area (high density). This is a natural example of
**percolation** — an important concept in statistical physics that the cellular automaton illustrates
directly.

## Associated exercise (from the original course)

The "Fire in the forest" exercise asked students to implement the two versions above, and then a variant
(`fire-average.lua`) in which the decision to catch fire depends on the **average** state of the
neighbors, instead of a binary "if any neighbor is burning" rule — smoothing the spread.

## Practice in DisSModel

`FireModel` (Ch. 5) implements this model in two versions (vector and raster), with exactly the same
rules and neighborhood. The neighbor-average variant of the exercise corresponds to the `mean` operator
from Ch. 10 (`disscube`), which can be reused to aggregate neighbor state instead of writing the
percolation logic from scratch.

## TerraME vs DisSModel: side-by-side code

**TerraME** (`ca/lua/Fire.lua`) — here the use of `cell.past.state` is quite explicit (the cell reads the
*past* state of itself and its neighbors, writes to the *present* state), and the neighborhood is created
with the `"vonneumann"` string:

```lua
model.cell = Cell{
    init = function(cell)
        cell.state = (Random():number() > model.empty) and "forest" or "empty"
    end,
    execute = function(cell)
        if cell.past.state == "burning" then
            cell.state = "burned"
        elseif cell.past.state == "forest" then
            local burning = countNeighbors(cell, "burning")
            if burning > 0 then cell.state = "burning" end
        end
    end
}
model.cs = CellularSpace{xdim = model.dim, instance = model.cell}
model.cs:sample().state = "burning"
model.cs:createNeighborhood{strategy = "vonneumann"}
```

**DisSModel** (`dissmodel-ca/src/.../fire_model.py`) — the neighborhood becomes
`create_neighborhood(strategy=Rook)` (Rook = Von Neumann, 4 cardinal neighbors), and the initial state is
filled with `fill(strategy=FillStrategy.RANDOM_SAMPLE, ...)` instead of a Lua cell `init`:

```python
class FireModel(CellularAutomaton):
    def setup(self, initial_fire_density: float = 0.05, seed: int = 42) -> None:
        self.initial_fire_density, self.seed = initial_fire_density, seed
        self.create_neighborhood(strategy=Rook, use_index=True)

    def initialize(self) -> None:
        fill(
            strategy=FillStrategy.RANDOM_SAMPLE, gdf=self.gdf, attr="state",
            data={FireState.FOREST: 1 - self.initial_fire_density,
                  FireState.BURNING: self.initial_fire_density},
            seed=self.seed,
        )

    def rule(self, idx: Any) -> int:
        state = self.gdf.loc[idx, self.state_attr]
        if state == FireState.BURNING:
            return FireState.BURNED
        if state == FireState.FOREST:
            if (self.neighbor_values(idx, self.state_attr) == FireState.BURNING).any():
                return FireState.BURNING
        return state
```

The spread rule is identical step by step: burning → burned; forest with a burning neighbor → burning;
any other state, unchanged. `Rook` (from `libpysal`) is literally the implementation of the Von Neumann
neighborhood used in TerraME — same 4-neighbor topology, different library. The most visible difference
is stylistic: strings (`"forest"`, `"burning"`, `"burned"`) become an `IntEnum` (`FireState.FOREST`,
etc.), which gives type checking and avoids typos in state names.

---
→ Corresponding chapter in the [dissmodel-book](https://github.com/DisSModel/dissmodel-book):
[Ch. 5 — Cellular Automata](../part2-paradigms/ch05_ca.md)
(`FireModel`, vector and raster)
