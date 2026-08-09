# Lesson 1 — Introduction to Computing

*Part VII — TerraME Compatibility Course*

*Kept as in the original course — does not depend on TerraME or DisSModel.*

## What is a computer?

A computer isn't just "another machine." A coffee maker or a calculator each do exactly one thing, in the
way they were built to do it. A computer is different: it is a **universal machine** — the same physical
machine can run a spreadsheet, a game, or a deforestation model, depending only on the program (software)
loaded onto it. This universality comes from the **von Neumann architecture**: memory, processing unit,
and data/instruction bus treated uniformly — programs and data live in the same memory space.

## Machines and abstractions

Before the general-purpose computer existed, computation theory already described increasingly powerful
machines, in layers of abstraction:

1. **Deterministic Finite Automaton (DFA)** — the simplest. It has an input alphabet (e.g. `0` and `1`), a
   finite set of states, a transition function between states, an initial state, and a set of accepting
   states. It reads an input tape from left to right, changing state at each symbol read. If it ends in an
   accepting state, the tape is accepted; otherwise, it is rejected. Classic example: a DFA that accepts
   only tapes ending in `01`, or tapes with an even number of zeros.
2. **Pushdown automaton** — adds memory (a stack) to the DFA, recognizing more complex languages (such as
   `aⁿbⁿ`, which a DFA cannot recognize).
3. **Turing machine** — the most general model, with an infinite read/write tape. It is the theoretical
   foundation of everything a computer can compute. The **Universal Turing Machine** is a Turing machine
   capable of simulating any other Turing machine — this is the idea that, decades later, materializes in
   the general-purpose computer.

## Programming languages

Programming means writing instructions for this universal machine to follow. Programming languages form a
hierarchy of abstraction:

- **Closer to the machine** (more computationally efficient, harder to write): machine code, Assembly.
- **Intermediate level**: Fortran, Cobol, C.
- **More abstract** (easier to write, at some cost to efficiency): C++/Java, and at the top,
  Python/R/Lua.

This hierarchy is the direct backdrop for the next lesson: why TerraME chose Lua, and why DisSModel chose
Python.

## Suggested exercises

Build DFAs that recognize tapes that:
- end with `01`
- contain two consecutive zeros
- have an even number of occurrences of `0`s and `1`s
- have every `0` always between two `1`s

---
*No corresponding chapter in the [dissmodel-book](https://github.com/DisSModel/dissmodel-book) — this is
computing-fundamentals content, independent of any framework.*
