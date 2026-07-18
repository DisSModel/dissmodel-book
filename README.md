# DisSModel Book — Technical Reference & Ecosystem Guide

> Technical reference book for the DisSModel ecosystem — installation,
> API, architecture, and a migration guide from TerraME/LUCCME
> (INPE/CCST) to Python.

**LambdaGeo, UFMA**

Technical reference documentation for the **DisSModel** ecosystem: installation,
architecture, API for each satellite package, TerraME/LUCCME → DisSModel
migration guide, and the distributed execution platform.

> This book is **technical and reference-oriented**. For the didactic
> material on geographic data science in Python (from scratch up to
> DisSModel), see the companion textbook:
> [geospatial-modeling-python](https://github.com/LambdaGeo/geospatial-modeling-python).
> The two books cite each other — this one covers the "how it works / how
> to use it", that one covers the "how to think about it / why".

## Build

```bash
pip install -r requirements.txt
mkdocs serve        # preview at http://127.0.0.1:8000
mkdocs build        # build static site to site/
mkdocs gh-deploy    # deploy to GitHub Pages
```

## Structure

```
docs/
  index.md
  part1-core/        # Ch 1-3  — What DisSModel is, core concepts, installation
  part2-paradigms/    # Ch 4-6  — Cellular Automata, Agent-Based Modeling, System Dynamics
  part3-domain/        # Ch 7-8  — DisSLUCC (continuous + discrete), Coastal Dynamics case study
  part4-infra/          # Ch 9-10 — The DisSModel Platform (execution + registry), Spatial Data Cubes
  part5-reference/       # Ch 11-12 — TerraME→DisSModel migration, architecture & contributing
```

Chapter titles are concept-first (e.g. "Cellular Automata", not
"dissmodel-ca"); the underlying Python package is linked at the top of
each chapter instead of in the title.

## Contribution convention

Each chapter can evolve on its own branch: `chapter/<slug>`.
Open a PR to `main` when a chapter is ready for review.

When you finish a chapter here, add a "📖 Docs" link pointing to the
corresponding section in the relevant package's README (`dissmodel-ca`,
`dissmodel-abm`, etc.) to close the navigation loop.

---
*LambdaGEO Research Group · Federal University of Maranhão (UFMA)*
