# Lesson 9 — Cellular Automaton

*Part VII — TerraME Compatibility Course*

## System Dynamics has limits

This lesson closes the previous topic with an honest comparison. System Dynamics (Lessons 3-8) has clear
advantages — simple and visual representation of the world, modular and hierarchical — but also important
disadvantages: **it does not represent heterogeneity** (the entire "stock" is treated as a single,
homogeneous block), spatial representation is implicit (or non-existent), and connections between stocks
are fixed. Cellular automata solve exactly these three limitations, introducing explicit, heterogeneous
space.

## Historical origin

Cellular automata were proposed by the Hungarian mathematician John von Neumann (the same von Neumann of
computer architecture), who was looking for a model of logical systems capable of **self-replication** —
the idea of a machine that builds copies of itself, motivated by questions about the nature of life and
computation.

## Definition: basic cellular automaton

Every cellular automaton is defined by six elements:

1. A grid of cells
2. A neighborhood
3. A finite set of discrete states
4. A finite set of transition rules
5. An initial state
6. Discrete time

Formally, in a 2D cellular automaton, the state of a cell at time `t` is a function of the states of its
neighbors at time `t-1`. Each cell is **autonomous**: it changes state according to its own current state
and the state of its neighborhood — there is no central controller.

> "Cellular automata contain enough complexity to simulate surprising and novel change as reflected in
> emergent phenomena" — Mike Batty

## Game of Life

The most famous cellular automaton example: each cell is alive or dead, and at every time step simple
rules based on the number of live neighbors are applied (born with 3 live neighbors, survives with 2 or
3, dies of loneliness or overpopulation otherwise). Despite the simplicity of the rules, the system
produces complex, recognizable patterns: oscillators (`blinker`, `toad`), spaceships that move across the
grid (`glider`, `LWSS`), and stable structures.

## New vocabulary concepts

- **Cell**: a spatial location with homogeneous internal content.
- **CellularSpace**: a set of cells — represents the area of interest.
- **Neighborhood**: the proximity relations of a cell.
- **Map**: a visual component for visualizing `CellularSpace`s.

## Neighborhood types

- **Von Neumann**: the 4 orthogonal neighbors (up, down, left, right).
- **Moore**: the 8 surrounding neighbors, including diagonals.

## Synchronizing a CellularSpace

An important technical detail from the original TerraME: it keeps **two copies** of a `CellularSpace` in
memory — one holds the past values of the cells, another holds the current (present) values. The model's
equations must **read the past copy and write to the present copy** — otherwise, different cells would be
updated using an inconsistent mix of old and new states within the same time step. At the right moment,
TerraME synchronizes the past copy with the current values, closing the cycle.

## Practice in DisSModel

`GameOfLife` (Ch. 5) implements exactly this logic; the Von Neumann and Moore neighborhoods correspond to
the `Rook` and `Queen` types built via `libpysal`.

## TerraME vs DisSModel: side-by-side code

**TerraME** (`ca/lua/Life.lua`) — the transition rule reads `cell.past.state` (the explicit "past" copy
mentioned above) and uses `countNeighbors`; the neighborhood is created with `wrap = true` (toroidal
space — edges wrap around):

```lua
model.cell = Cell{
    init = function(cell) cell.state = "dead" end,
    execute = function(cell)
        local alive = countNeighbors(cell, "alive")
        if alive < 2 then
            cell.state = "dead"
        elseif alive > 3 then
            cell.state = "dead"
        elseif alive == 3 and cell.past.state == "dead" then
            cell.state = "alive"
        end
    end
}
model.cs = CellularSpace{xdim = model.dim, instance = model.cell}
model.cs:createNeighborhood{wrap = true}
```

**DisSModel** (`dissmodel-ca/src/.../game_of_life.py`) — the neighborhood becomes
`create_neighborhood(strategy=Queen)` (Moore = 8 neighbors, via `libpysal`), and the rule is a `rule(idx)`
method that reads and returns the new state, with no need for an explicit "past" copy:

```python
class GameOfLife(CellularAutomaton):
    def setup(self) -> None:
        self.create_neighborhood(strategy=Queen, use_index=True)

    def rule(self, idx: Any) -> int:
        state = self.gdf.loc[idx, self.state_attr]
        live_neighbors = (self.neighbor_values(idx, self.state_attr)).sum()
        if state == 1:
            return 1 if 2 <= live_neighbors <= 3 else 0
        return 1 if live_neighbors == 3 else 0
```

Conway's classic rule (born with 3 live neighbors, survives with 2-3, dies otherwise) is identical in both
versions. A real difference worth noting: `Life.lua` uses `wrap = true` (toroidal space, edges connected),
and the current Python `GameOfLife` does **not** enable wrap by default — `libpysal`'s `Queen` treats
edges as real boundaries. This is a small behavioral divergence at the edges of the grid (not in the
center), documented in other models of the same package (`wolfram.py`, `parity.py`) as something to
consider when porting a TerraME model that relies on toroidal edges.

---
→ Corresponding chapter in the [dissmodel-book](https://github.com/DisSModel/dissmodel-book):
[Ch. 5 — Cellular Automata](../part2-paradigms/ch05_ca.md)
(`GameOfLife` model)

> **Note:** the explicit two-copy mechanism (past/present) has no equivalent section in the book — the
> DisSModel `setup()→execute()` cycle resolves this consistency implicitly (each `execute()` reads the
> consolidated state from the previous step before any cell is updated), but it's worth explaining that
> equivalence in the lesson for anyone already familiar with TerraME's mental model.
